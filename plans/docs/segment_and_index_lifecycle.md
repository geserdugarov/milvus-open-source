# Milvus Segment & Index Lifecycle

This document describes how data records and index records flow through the Milvus
segment lifecycle — from insert to query, including compaction, deletion, and
garbage collection. It also covers how raw data and index files relate within
a segment: how they are stored, tracked, and loaded independently.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Segment States](#2-segment-states)
3. [Segment Levels](#3-segment-levels)
4. [Data Record Flow](#4-data-record-flow)
5. [Sealing Policies](#5-sealing-policies)
6. [Data and Index: Separate Entities](#6-data-and-index-separate-entities)
7. [Index Record Flow](#7-index-record-flow)
8. [Growing Segment In-Memory Indexing](#8-growing-segment-in-memory-indexing)
9. [Query Path: Loading Segments and Indexes](#9-query-path-loading-segments-and-indexes)
10. [Delete Flow and L0 Segments](#10-delete-flow-and-l0-segments)
11. [Compaction](#11-compaction)
12. [Garbage Collection](#12-garbage-collection)
13. [End-to-End Lifecycle Diagram](#13-end-to-end-lifecycle-diagram)

---

## 1. Overview

Milvus is a distributed vector database built around a **segment-centric architecture**.
All data — inserts, deletes, indexes, and statistics — is organized into segments.
A segment is the fundamental unit of data storage, indexing, and query execution.

A segment is a **logical grouping** — the raw data (binlogs) and the index are stored,
tracked, and loaded **independently**. The segment ID is the key that ties them together.

Key architectural roles:

- **Proxy** — user-facing gateway; routes inserts and queries
- **DataCoord** — manages segment metadata, allocation, sealing, flushing, index
  scheduling, and compaction orchestration
- **DataNode** — executes writes (buffering, flushing binlogs to object storage),
  index builds, and compaction tasks
- **QueryCoord** — decides which segments to load on which QueryNodes
- **QueryNode** — loads segments + indexes into memory and executes searches

```
                          ┌──────────────────────────────────────────┐
                          │              Object Storage              │
                          │  (S3 / MinIO / Azure Blob / Local Disk)  │
                          │                                          │
                          │  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
                          │  │ Insert   │ │ Delta    │ │ Index    │ │
                          │  │ Binlogs  │ │ Binlogs  │ │ Files    │ │
                          │  └──────────┘ └──────────┘ └──────────┘ │
                          └──────────┬───────────┬───────────┬──────┘
                                     │           │           │
                     ┌───────────────┼───────────┼───────────┼──────────┐
                     │               │  DataCoord│(metadata) │          │
                     │  ┌────────────▼───────────▼───────────▼───────┐  │
                     │  │         Segment Metadata (etcd)            │  │
                     │  │  state, level, binlog paths, index info    │  │
                     │  └────────────────────────────────────────────┘  │
                     └──────────────────────────────────────────────────┘
                              │                              │
              ┌───────────────▼──────────┐     ┌─────────────▼──────────────┐
              │         DataNode         │     │         QueryNode          │
              │  • write buffer          │     │  • load segment data       │
              │  • flush to binlogs      │     │  • load index files        │
              │  • build indexes         │     │  • execute search/query    │
              │  • run compaction        │     │  • growing + sealed search │
              └──────────────────────────┘     └────────────────────────────┘
```

---

## 2. Segment States

A segment transitions through the following states, defined in `commonpb.SegmentState`:

```
                     ┌────────────┐
         Allocation  │  Growing   │  Actively receiving inserts
                     └─────┬──────┘
                           │ seal policy triggered
                     ┌─────▼──────┐
                     │  Sealed    │  Closed to inserts, awaiting flush
                     └─────┬──────┘
                           │ flush triggered
                     ┌─────▼──────┐
                     │  Flushing  │  Writing binlogs to object storage
                     └─────┬──────┘
                           │ flush complete
                     ┌─────▼──────┐
                     │  Flushed   │  Immutable, persisted, ready for indexing
                     └─────┬──────┘
                           │ explicit drop / compaction replacement
                     ┌─────▼──────┐
                     │  Dropped   │  Pending garbage collection
                     └────────────┘


                     ┌────────────┐
         Bulk import │ Importing  │  Created during bulk import
                     └─────┬──────┘
                           │ import complete
                     ┌─────▼──────┐
                     │  Flushed   │
                     └────────────┘
```

| State      | Mutable? | On Disk? | Indexed? | Queryable?                     |
|------------|----------|----------|----------|--------------------------------|
| Growing    | Yes      | No*      | In-memory only | Yes (brute-force / in-memory) |
| Sealed     | No       | No*      | No       | Yes (brute-force)              |
| Flushing   | No       | Partial  | No       | Yes (brute-force)              |
| Flushed    | No       | Yes      | Yes (if index exists) | Yes (with index)      |
| Dropped    | No       | Yes**    | N/A      | No                             |
| Importing  | Yes      | Partial  | No       | No                             |

\* Data is in DataNode memory / write buffer.
\** Binlogs remain until garbage-collected.

**Key source files:**
- `internal/datacoord/segment_manager.go` — allocation, sealing
- `internal/datacoord/meta.go` — `isFlushState()`, state transitions

---

## 3. Segment Levels

Segments have a **level** attribute (`datapb.SegmentLevel`) that determines their role
in the storage hierarchy:

```
  Level    Value   Description
  ─────    ─────   ───────────────────────────────────────────────────
  Legacy     0     Zero value for backward compatibility
  L0         1     Delta-only segments (stores deletes for a channel)
  L1         2     Normal data segments (default for new segments)
  L2         3     Segments with data distribution info (post-clustering)
```

**How levels interact with the lifecycle:**

```
  ┌─────────────────────────────────────────────────────────┐
  │                    L0 (Delete Layer)                     │
  │  Accumulates delete delta logs per channel.             │
  │  Periodically compacted into L1/L2 segments.            │
  │  Never contains insert binlogs.                         │
  └──────────────────────────┬──────────────────────────────┘
                             │ L0 compaction
  ┌──────────────────────────▼──────────────────────────────┐
  │                    L1 (Normal Layer)                     │
  │  Standard segments from inserts or mix compaction.      │
  │  Created as Growing, transitions through lifecycle.     │
  │  Can be compacted (mix) to reduce segment count.        │
  └──────────────────────────┬──────────────────────────────┘
                             │ clustering compaction
  ┌──────────────────────────▼──────────────────────────────┐
  │                    L2 (Distribution Layer)               │
  │  Segments reorganized by clustering key.                │
  │  Carry partition statistics for query pruning.          │
  │  Created by clustering compaction only.                 │
  └─────────────────────────────────────────────────────────┘
```

**Key source file:** `pkg/proto/datapb/data_coord.pb.go:84-87`

---

## 4. Data Record Flow

### 4.1 Insert Path

```
  Client
    │
    │  Insert(collection, data)
    ▼
  ┌──────────┐   hash by     ┌────────────────────┐
  │  Proxy   │──primary key──▶│  Virtual Channels  │ (one per shard)
  └──────────┘   (sharding)   └────────┬───────────┘
                                       │ MsgStream (Pulsar/Kafka/RocksMQ)
                                       ▼
                              ┌─────────────────────────────┐
                              │        DataNode             │
                              │                             │
                              │  ┌───────────────────────┐  │
                              │  │    Flow Graph          │  │
                              │  │                        │  │
                              │  │  DDNode (DDL filter)   │  │
                              │  │    │                   │  │
                              │  │  WriteNode             │  │
                              │  │    │                   │  │
                              │  │  DeleteNode            │  │
                              │  └────┬──────────────────┘  │
                              │       │                     │
                              │  ┌────▼──────────────────┐  │
                              │  │  Write Buffer Manager │  │
                              │  │                       │  │
                              │  │  Buffers inserts in   │  │
                              │  │  memory per segment.  │  │
                              │  │  Tracks allocations.  │  │
                              │  └────┬──────────────────┘  │
                              │       │                     │
                              │       │ seal / flush        │
                              │       ▼                     │
                              │  ┌───────────────────────┐  │
                              │  │  Sync Manager         │  │
                              │  │                       │  │
                              │  │  Serializes data to   │  │
                              │  │  binlog format and    │  │
                              │  │  uploads to object    │  │
                              │  │  storage.             │  │
                              │  └───────────────────────┘  │
                              └─────────────────────────────┘
```

**Key source files:**
- `internal/flushcommon/pipeline/flow_graph_write_node.go` — WriteNode.Operate()
- `internal/flushcommon/writebuffer/` — write buffer manager
- `internal/flushcommon/syncmgr/sync_manager.go` — sync (flush) manager

### 4.2 Segment Allocation

When DataNode needs space for new inserts, DataCoord allocates rows in a growing
segment via `SegmentManager.AllocSegment()`:

1. Lock the target channel
2. Check existing growing segments in the partition for available capacity
3. If capacity exists → allocate rows in existing segment
4. If not → call `openNewSegment()` to create a new Growing/L1 segment

A new segment is created with:
- Unique segment ID (from allocator)
- State = `Growing`, Level = `L1`
- Max row count estimated from collection schema
- Assigned to a specific virtual channel

**Key source file:** `internal/datacoord/segment_manager.go:306-460`

### 4.3 Binlog Structure

Each flushed segment produces three types of binlogs stored in object storage:

```
  Object Storage Path Layout:
  ─────────────────────────────────────────────────────────────
  {rootPath}/{binlogType}/{collectionID}/{partitionID}/{segmentID}/{fieldID}/{logID}

  Binlog Types:
  ┌────────────────┬──────────────────────────────────────────────────┐
  │ Insert Binlogs │ Field-level data files. One set of binlogs per  │
  │                │ field per segment. Contains actual row data.     │
  ├────────────────┼──────────────────────────────────────────────────┤
  │ Stats Binlogs  │ Per-field statistics: min, max, row count,      │
  │                │ null counts. Used for query pruning.             │
  ├────────────────┼──────────────────────────────────────────────────┤
  │ Delta Binlogs  │ Delete records: primary keys + timestamps.      │
  │                │ JSON-serialized entries.                         │
  ├────────────────┼──────────────────────────────────────────────────┤
  │ BM25 Binlogs   │ BM25 statistics for full-text search fields.    │
  └────────────────┴──────────────────────────────────────────────────┘
```

Binlog file format uses a custom binary layout:
- Magic number (`0xfffabc`)
- Descriptor event (collection/partition/segment metadata)
- Data events with compressed payloads

**Key source files:**
- `internal/storage/binlog_writer.go` — binlog serialization
- `internal/metastore/kv/binlog/binlog.go` — path building and compression

### 4.4 Segment Metadata

Each segment carries metadata tracked by DataCoord in etcd:

```
  SegmentInfo {
      ID, CollectionID, PartitionID     // identity
      InsertChannel                      // virtual channel assignment
      State                              // Growing/Sealed/Flushed/...
      Level                              // L0/L1/L2
      NumOfRows                          // current row count
      MaxRowNum                          // capacity estimate

      Binlogs[]                          // insert binlog references
      Statslogs[]                        // stats binlog references
      Deltalogs[]                        // delta binlog references
      BM25Statslogs[]                    // BM25 stats references

      StartPosition, DmlPosition         // recovery checkpoints
      LastExpireTime                      // allocation expiration
      CreatedByCompaction                 // origin flag
      CompactionFrom[]                    // source segment IDs
      Compacted                           // compaction completed flag
      IsImporting                         // bulk import flag
      StorageVersion                      // V1 (binlogs) or V2 (manifest)
      ManifestPath                        // V2 manifest location
  }
```

**Key source file:** `internal/datacoord/segment_info.go:52-62`

---

## 5. Sealing Policies

DataCoord's `SegmentManager` applies sealing policies to transition segments from
Growing to Sealed. Policies are evaluated in `tryToSealSegment()`.

### 5.1 Segment-Level Policies

```
  Policy                         Trigger Condition
  ──────────────────────────     ──────────────────────────────────────────
  sealL1SegmentByCapacity        Row count >= sealProportion * maxRows
  sealL1SegmentByLifetime        Time since creation >= SegmentMaxLifetime
  sealL1SegmentByIdleTime        No writes for >= SegmentMaxIdleTime
                                 AND row count above minimum threshold
  sealL1SegmentByBinlogFileNumber  Binlog file count >= SegmentMaxBinlogFileNumber
```

### 5.2 Channel-Level Policies

```
  Policy                         Trigger Condition
  ──────────────────────────     ──────────────────────────────────────────
  sealByTotalGrowingSegmentsSize  Total size of all growing segments in
                                  the channel exceeds GrowingSegmentsMemSizeInMB.
                                  Seals the LARGEST growing segment.

  sealByBlockingL0               L0 segments accumulate beyond threshold
                                 (BlockingL0SizeInMB or BlockingL0EntryNum).
                                 Seals growing segments whose time range
                                 overlaps with L0 segments, enabling
                                 L0 compaction to proceed.
```

### 5.3 Flush Trigger

After sealing, a separate flush policy determines when to actually persist:

```
  flushPolicyL1:
    segment.State == Sealed
    AND segment.Level != L0
    AND time since last flush >= SegmentFlushInterval
    AND segment.LastExpireTime <= current timestamp
    AND segment.NumOfRows != 0
    AND NOT segment.IsImporting
```

**Key source file:** `internal/datacoord/segment_allocation_policy.go`

---

## 6. Data and Index: Separate Entities

A segment's raw data and its index are **stored, tracked, and loaded independently**.
The segment ID ties them together, but they have separate lifecycles.

### 6.1 Separate Storage Paths

Raw binlogs and index files live at completely different paths in object storage:

```
  Object Storage
  ├── insert_log/.../segmentID/...    ← data binlogs (uploaded by flush pipeline)
  ├── delta_log/.../segmentID/...     ← deletes
  ├── stats_log/.../segmentID/...     ← statistics
  └── index_files/...                 ← index files (uploaded by index build task)
```

**Key source files:**
- `pkg/common/common.go:122` — `SegmentInsertLogPath`
- `pkg/common/common.go:131` — `SegmentIndexPath = "index_files"`

### 6.2 Separate Metadata Stores in DataCoord

```
  ┌──────────────────────────────────────────────────────┐
  │                    DataCoord                          │
  │                                                      │
  │  ┌─────────────────────────┐  ┌────────────────────┐ │
  │  │    Segment Metadata     │  │   Index Metadata   │ │
  │  │                         │  │                    │ │
  │  │  SegmentInfo:           │  │  indexMeta:        │ │
  │  │  • segment ID           │  │  • segmentID →    │ │
  │  │  • state                │  │    indexID →       │ │
  │  │  • binlog paths         │  │    SegmentIndex   │ │
  │  │  • row count            │  │  • index state    │ │
  │  │  • level (L0/L1/L2)    │  │    (Unissued /    │ │
  │  │  • delta logs           │  │     InProgress /  │ │
  │  │  • stats logs           │  │     Finished /    │ │
  │  │                         │  │     Failed)       │ │
  │  │                         │  │  • index file keys│ │
  │  │                         │  │  • index size     │ │
  │  └─────────────────────────┘  └────────────────────┘ │
  └──────────────────────────────────────────────────────┘
```

A segment can exist without any index. The index has its own lifecycle and state
machine, completely independent from segment state transitions.

**Key source files:**
- `internal/datacoord/segment_info.go:52-62` — SegmentInfo wrapper
- `internal/datacoord/index_meta.go:60-76` — indexMeta with `segmentIndexes` map

### 6.3 Independent Loading via LoadScope

The `LoadScope` enum proves data and index can be loaded independently:

```
  LoadScope_Full   = 0    // all data + index + deltas
  LoadScope_Delta  = 1    // only delta logs
  LoadScope_Index  = 2    // only index files
  LoadScope_Stats  = 3    // only stats logs
  LoadScope_Reopen = 4    // reopen with new info
```

### 6.4 In-Memory: Data and Index Coexist

On a QueryNode, a `LocalSegment` holds both raw data and indexes as separate maps:

```go
  type LocalSegment struct {
      fields       *ConcurrentMap[int64, *FieldInfo]        // raw field data
      fieldIndexes *ConcurrentMap[int64, *IndexedFieldInfo]  // index per field
  }

  type IndexedFieldInfo struct {
      FieldBinlog *datapb.FieldBinlog      // reference to raw data binlog
      IndexInfo   *querypb.FieldIndexInfo  // index metadata
      IsLoaded    bool                     // whether index is loaded
  }
```

When an index is loaded, raw vector data is **NOT deleted** — they coexist.
The segment tracks `HasRawData(fieldID)` per field.
Exception: BM25 indexes do remove raw data after index load.

**Key source file:** `internal/querynodev2/segments/segment.go:331-350`

### 6.5 One Segment on a QueryNode

```
  Object Storage:
  ┌──────────────────────────────────────────────────────────┐
  │  Segment 1234:                                           │
  │    insert_log/.../1234/field_100/log_A  (vector binlog)  │
  │    insert_log/.../1234/field_101/log_B  (scalar binlog)  │
  │    delta_log/.../1234/...               (deletes)        │
  │    stats_log/.../1234/...               (statistics)     │
  │                                                          │
  │  Index for Segment 1234, Field 100:                      │
  │    index_files/build_567/...            (HNSW files)     │
  └──────────────────────────────────────────────────────────┘
           │                           │
           │  loaded separately        │  loaded separately
           ▼                           ▼
  ┌─────────────────────────────────────────────────────┐
  │  QueryNode 1: LocalSegment 1234                     │
  │                                                     │
  │    fields:                                          │
  │      field_100 → raw vector data (in memory/mmap)   │
  │      field_101 → raw scalar data                    │
  │                                                     │
  │    fieldIndexes:                                    │
  │      field_100 → HNSW index (in memory)             │
  │                                                     │
  │    Search: uses HNSW index for field_100            │
  │            raw scan for field_101 (scalar filter)   │
  └─────────────────────────────────────────────────────┘
```

A segment's data and index are never on different nodes. They are loaded together
onto the assigned QueryNode, though in separate phases. The segment is the atomic
unit of distribution — it is never split across nodes.

```
  QueryCoord assigns whole segments to nodes:

  QueryNode 1                    QueryNode 2
  ┌────────────────────┐        ┌────────────────────┐
  │ Segment A          │        │ Segment C          │
  │   data + index     │        │   data + index     │
  │                    │        │                    │
  │ Segment B          │        │ Segment D          │
  │   data + index     │        │   data (no index   │
  │                    │        │         yet →       │
  │                    │        │         brute force)│
  └────────────────────┘        └────────────────────┘

  Proxy fans out search to both nodes,
  merges top-K results.
```

---

## 7. Index Record Flow

### 7.1 Index Creation

```
  Client
    │
    │  CreateIndex(collection, field, params)
    ▼
  ┌──────────┐    validate schema      ┌──────────────────┐
  │  Proxy   │─────────────────────────▶│    DataCoord     │
  └──────────┘                          │                  │
                                        │  1. Create Index │
                                        │     definition   │
                                        │     in IndexMeta │
                                        │                  │
                                        │  2. Return       │
                                        │     immediately  │
                                        │     (async build)│
                                        └────────┬─────────┘
                                                 │
                                                 │ IndexInspector loop
                                                 ▼
```

Index creation is **fully asynchronous**. The API returns immediately; actual builds
happen in the background.

**Key source file:** `internal/datacoord/index_service.go:129+`

### 7.2 Index Build Scheduling

The `IndexInspector` runs a continuous loop detecting segments that need indexing:

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                     IndexInspector Loop                         │
  │                                                                 │
  │   Triggers:                                                     │
  │   ├── Periodic timer (TaskCheckInterval)                        │
  │   ├── CreateIndex event (notifyIndexChan)                       │
  │   └── Segment flush event (flushCh)                             │
  │                                                                 │
  │   For each trigger:                                             │
  │   1. Find all Flushed segments                                  │
  │   2. Filter to unindexed: IsUnIndexedSegment(collID, segID)     │
  │   3. For each unindexed segment + index definition:             │
  │      a. Allocate unique BuildID                                 │
  │      b. Create SegmentIndex record (state = Unissued)           │
  │      c. Enqueue IndexBuildTask to global scheduler              │
  └─────────────────────────────────────────────────────────────────┘
```

**Key source file:** `internal/datacoord/index_inspector.go:87-229`

### 7.3 Index Build Execution

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    DataNode: Index Build Task                        │
  │                                                                      │
  │  Phase 1: PreExecute                                                 │
  │  ├── Resolve binlog paths from segment metadata                      │
  │  ├── Parse type params (dim, data type) and index params (nlist...)   │
  │  └── Determine index engine version compatibility                    │
  │                                                                      │
  │  Phase 2: Execute                                                    │
  │  ├── Load segment data from object storage (binlogs)                 │
  │  ├── Call C++ Knowhere index builder via CGO                         │
  │  └── Build index (HNSW, IVF_FLAT, DISKANN, etc.)                    │
  │                                                                      │
  │  Phase 3: PostExecute                                                │
  │  ├── Serialize index to files                                        │
  │  ├── Upload index files to object storage (index_files/ path)        │
  │  └── Update SegmentIndex: state=Finished, save file keys and sizes   │
  └──────────────────────────────────────────────────────────────────────┘
```

**Key source file:** `internal/datanode/index/task_index.go`

### 7.4 Index States

Each (segment, index) pair is tracked by a `SegmentIndex` record:

```
  SegmentIndex {
      SegmentID, IndexID, BuildID       // identity
      NodeID                             // DataNode executing the build
      IndexState                         // current state (below)
      FailReason                         // error message if failed
      IndexVersion                       // version for retry tracking
      IndexFileKeys[]                    // file paths in object storage
      IndexSerializedSize                // compressed size
      IndexMemSize                       // uncompressed in-memory size
      FinishedUTCTime                    // completion timestamp
  }
```

State machine:

```
                ┌──────────┐
    task created│ Unissued │
                └────┬─────┘
                     │ assigned to DataNode
                ┌────▼─────┐
                │InProgress│
                └────┬─────┘
                     │
            ┌────────┼────────┐
            │        │        │
       ┌────▼───┐ ┌──▼───┐ ┌─▼────┐
       │Finished│ │Failed│ │Retry │
       └────────┘ └──────┘ └──┬───┘
                               │ re-enqueued
                          ┌────▼─────┐
                          │ Unissued │
                          └──────────┘
```

**Key source files:**
- `internal/datacoord/index_meta.go` — state tracking
- `internal/metastore/model/segment_index.go` — SegmentIndex model

---

## 8. Growing Segment In-Memory Indexing

While a segment is still Growing (before flush), the C++ segcore maintains
**incremental in-memory indexes** so that recent data is searchable:

```
  ┌─────────────────────────────────────────────────────────────────┐
  │              C++ Segcore: FieldIndexing                         │
  │                                                                 │
  │  As rows arrive, the segcore appends to in-memory indexes:     │
  │                                                                 │
  │  AppendSegmentIndexDense()   — dense vector fields              │
  │  AppendSegmentIndexSparse()  — sparse vector fields             │
  │  AppendSegmentIndex()        — scalar / string / JSON fields    │
  │                                                                 │
  │  Build Threshold:                                               │
  │  ┌─────────────────────────────────────────────────────────┐    │
  │  │ Rows < threshold  →  raw data kept, partial index       │    │
  │  │ Rows >= threshold →  full in-memory index built         │    │
  │  └─────────────────────────────────────────────────────────┘    │
  │                                                                 │
  │  This enables search on growing segments without waiting        │
  │  for flush + persistent index build.                            │
  └─────────────────────────────────────────────────────────────────┘
```

**Key source file:** `internal/core/src/segcore/FieldIndexing.h:67-102`

---

## 9. Query Path: Loading Segments and Indexes

### 9.1 QueryCoord Segment Assignment

QueryCoord's `SegmentChecker` continuously reconciles the desired state
(target segments) with the actual state (loaded segments on QueryNodes):

```
  ┌────────────────────────────────────────────────────────────────┐
  │                 QueryCoord: SegmentChecker                     │
  │                                                                │
  │  1. Get NextTarget: all sealed segments that should be loaded  │
  │  2. Get CurrentTarget: segments already loaded on QueryNodes   │
  │  3. Diff:                                                      │
  │     • toLoad   = in NextTarget but not loaded                  │
  │     • toRelease = loaded but not in NextTarget                 │
  │     • toUpdate  = loaded but need index/config update          │
  │                                                                │
  │  Load priorities:                                              │
  │     HIGH   — recovery (segment was in CurrentTarget)           │
  │     MEDIUM — import/refresh                                    │
  │     LOW    — handoff (growing → sealed transition)             │
  └────────────────────────────────────────────────────────────────┘
```

### 9.2 Two-Phase Loading on QueryNode

```
  QueryCoord
    │
    │  SegmentLoadInfo (binlogs + index paths combined)
    ▼
  QueryNode Segment Loader
    │
    │  Phase 1: Load raw binlogs + deltas
    │  ├── Read insert binlogs from object storage
    │  ├── Deserialize field data into memory
    │  ├── Load delta logs (deleted PKs)
    │  └── separateIndexAndBinlog() splits indexed vs non-indexed fields
    │
    │  Phase 2: Load index (independent, can happen later)
    │  ├── Read index files from object storage
    │  ├── Load into Knowhere index structure
    │  └── Mark field as indexed (IsLoaded = true)
    │
    ▼
  LocalSegment ready for queries
```

The `SegmentLoadInfo` bundles both data and index paths:

```
  SegmentLoadInfo {
      BinlogPaths[]    ← raw field data (insert binlogs)
      IndexInfos[]     ← index file paths + params per field
      Deltalogs[]      ← delete records
      Statslogs[]      ← field statistics
  }
```

**Key source files:**
- `internal/querynodev2/segments/segment_loader.go:244-448` — Load() method
- `internal/querynodev2/segments/segment_loader.go:2244-2307` — LoadIndex() method
- `internal/querycoordv2/utils/types.go:63-104` — PackSegmentLoadInfo()

### 9.3 Search: Growing vs Sealed

```
  ┌────────────────────────────────────────────────────────────────────┐
  │                    QueryNode: Delegator.Search()                   │
  │                                                                    │
  │  Search request arrives at shard delegator                         │
  │                                                                    │
  │  ┌─────────────────────────┐    ┌──────────────────────────────┐  │
  │  │   Growing Segments      │    │    Sealed Segments           │  │
  │  │                         │    │                              │  │
  │  │  • In-memory data       │    │  • Persistent index loaded   │  │
  │  │  • Brute-force scan     │    │  • HNSW / IVF / DISKANN     │  │
  │  │    or simple in-memory  │    │    search                    │  │
  │  │    index                │    │  • Much faster for large     │  │
  │  │  • Includes latest      │    │    segments                  │  │
  │  │    unfinished inserts   │    │  • Immutable snapshot        │  │
  │  └────────────┬────────────┘    └──────────────┬───────────────┘  │
  │               │                                │                   │
  │               └───────────┬────────────────────┘                   │
  │                           ▼                                        │
  │                    Merge results                                   │
  │                    (top-K reduction)                                │
  └────────────────────────────────────────────────────────────────────┘

  Per-segment search decision:

  Search arrives at segment
    │
    ├─ Index loaded for this vector field?
    │    YES → use index (HNSW / IVF_FLAT / DISKANN search)
    │    NO  → brute-force scan on raw vector binlog data
    │
    ├─ Scalar field filter?
    │    → evaluated on raw scalar data (or scalar index if exists)
    │
    └─ Delta logs applied as bitset to exclude deleted rows
```

A sealed segment without an index **still works** — it falls back to brute force
on raw vectors. This means segments become queryable before their indexes are built.

### 9.4 Growing-to-Sealed Handoff

There is no explicit "handoff" RPC. Instead, the transition happens through
the target management system:

1. DataNode flushes a growing segment → state becomes Flushed
2. DataCoord updates the target: segment moves from growing list to sealed list
3. QueryCoord's SegmentChecker detects the new sealed segment in NextTarget
4. QueryNode loads the sealed segment + index
5. QueryCoord calls `SyncTargetVersion` on QueryNode's delegator
6. Delegator moves the segment from `growingSegments` to `sealedSegments`
7. Growing copy is released; subsequent searches use the indexed sealed copy

**Key source files:**
- `internal/querycoordv2/checkers/segment_checker.go:313-399`
- `internal/querycoordv2/task/executor.go:243-319`
- `internal/querynodev2/segments/segment_loader.go:78-107`
- `internal/querynodev2/delegator/distribution.go:430-449`

---

## 10. Delete Flow and L0 Segments

### 10.1 Delete Processing

```
  Client
    │
    │  Delete(collection, filter_expr)
    ▼
  ┌──────────┐
  │  Proxy   │
  │          │
  │  Simple delete (PK = X):                                 
  │    → extract PKs directly from expression               
  │                                                          
  │  Complex delete (expr filter):                           
  │    → query QueryNode to find matching PKs first          
  │    → then produce delete messages for found PKs          
  └────┬─────┘
       │  Delete messages (PKs + timestamps)
       │  sent to MsgStream per channel
       ▼
  ┌──────────────────┐
  │    DataNode       │
  │                   │
  │  DeleteNode in    │
  │  flow graph       │
  │  buffers deletes  │
  │       │           │
  │       ▼           │
  │  Written to       │
  │  Delta Binlogs    │──────▶  Object Storage
  │  (JSON format:    │         (delta binlog files)
  │   PK + timestamp) │
  └──────────────────┘
```

### 10.2 L0 Segments

L0 segments are special delta-only segments that accumulate deletes for a channel:

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        L0 Segment                               │
  │                                                                 │
  │  • Level = L0                                                   │
  │  • Contains ONLY delta binlogs (no insert binlogs)              │
  │  • One per channel (accumulates all deletes for that channel)   │
  │  • Created in Flushed state directly                            │
  │  • Cannot be sealed or flushed via normal policies              │
  │  • Excluded from normal flush policy (flushPolicyL1)            │
  │                                                                 │
  │  Delta binlogs contain:                                         │
  │    { primary_key: "...", timestamp: ... }                       │
  │  serialized as JSON strings                                     │
  └─────────────────────────────────────────────────────────────────┘
```

### 10.3 L0 Compaction

When L0 segments accumulate enough deletes, L0 compaction merges them into
the target L1/L2 segments:

```
  BEFORE L0 Compaction:
  ┌───────────┐  ┌───────────┐  ┌───────────┐
  │ L0 seg A  │  │ L0 seg B  │  │ L0 seg C  │   (delta-only)
  │ deltas:50 │  │ deltas:30 │  │ deltas:20 │
  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
        │               │               │
        └───────────────┼───────────────┘
                        │ merge deletes into
                        ▼
  ┌───────────────┐  ┌───────────────┐
  │  L1 segment X │  │  L1 segment Y │   (target segments)
  │  +delta logs  │  │  +delta logs  │
  └───────────────┘  └───────────────┘

  AFTER L0 Compaction:
  • L0 segments A, B, C → state=Dropped, Compacted=true
  • L1 segments X, Y → delta logs updated (V1) or manifest updated (V2)
  • NO new segments created; deletes are applied to existing segments
```

**L0 compaction trigger policies:**
- `LevelZeroCompactionTriggerMinSize` — minimum delta size to trigger
- `LevelZeroCompactionTriggerMaxSize` — maximum delta size per compaction
- `LevelZeroCompactionTriggerDeltalogMinNum` — minimum deltalog count
- `LevelZeroCompactionTriggerDeltalogMaxNum` — maximum deltalog count

**Key source files:**
- `internal/proxy/task_delete.go` — delete request processing
- `internal/storage/serde_delta.go` — deltalog serialization
- `internal/datacoord/compaction_task_l0.go` — L0 compaction task
- `internal/datacoord/compaction_policy_l0.go` — L0 trigger policies

---

## 11. Compaction

Compaction merges, reorganizes, or cleans up segments. All compaction types create
new segments and trigger fresh index builds — indexes are never reused from
source segments.

### 11.1 Compaction Types

```
  Type          Input                Output              Purpose
  ────────────  ───────────────────  ──────────────────  ─────────────────────────────
  L0            L0 delta segments    Updated L1/L2 segs  Apply accumulated deletes
  Mix           Multiple L1/L2 segs  Fewer larger segs   Reduce segment count
  Sort          Single L2 segment    Single sorted seg   Optimize sort/filter perf
  Clustering    L1/L2 segments       L2 segments          Reorganize by clustering key
```

### 11.2 Compaction Flow

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │               DataCoord: Compaction Trigger Manager                  │
  │                                                                      │
  │  Periodic tickers:                                                   │
  │  ├── L0CompactionTriggerInterval      → L0 policy                   │
  │  ├── MixCompactionTriggerInterval     → Mix + Single + Sort policy  │
  │  └── ClusteringCompactionTriggerInterval → Clustering policy        │
  │                                                                      │
  │  Each tick:                                                          │
  │  1. Policy evaluates segment views (sizes, counts, levels)           │
  │  2. If trigger condition met → create CompactionTask                 │
  │  3. Enqueue task to priority queue                                   │
  └──────────────────────────┬───────────────────────────────────────────┘
                             │
                             ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │               Priority Queue                                         │
  │                                                                      │
  │  Default priority (LevelPrioritizer):                                │
  │    L0DeleteCompaction:      1  (highest)                             │
  │    MixCompaction:          10                                        │
  │    ClusteringCompaction:  100                                        │
  │    Other:                1000  (lowest)                              │
  └──────────────────────────┬───────────────────────────────────────────┘
                             │
                             ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │               DataNode: Compaction Execution                         │
  │                                                                      │
  │  1. Read input segment binlogs from object storage                   │
  │  2. Merge data, apply deletes, sort if needed                        │
  │  3. Write output segment binlogs to object storage                   │
  │  4. Return CompactionPlanResult to DataCoord                         │
  └──────────────────────────┬───────────────────────────────────────────┘
                             │
                             ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │               DataCoord: Post-Compaction                             │
  │                                                                      │
  │  1. Validate segment states (ensure inputs unchanged during compact) │
  │  2. Register new output segments in metadata (data only, no index)   │
  │  3. Mark input segments: State=Dropped, Compacted=true               │
  │  4. IndexInspector detects new unindexed Flushed segments            │
  │  5. Fresh index builds triggered for output segments                 │
  │  6. QueryCoord loads new segments (data + index), releases old ones  │
  └──────────────────────────────────────────────────────────────────────┘
```

### 11.3 Compaction Task State Machine

```
                 ┌───────────┐
    task created │ pipelining│  Waiting for DataNode assignment
                 └─────┬─────┘
                       │ assigned
                 ┌─────▼─────┐
                 │ executing │  Running on DataNode
                 └─────┬─────┘
                       │ result received
                 ┌─────▼──────┐
                 │ meta_saved │  Metadata updated
                 └─────┬──────┘
                       │ cleanup done
                 ┌─────▼──────┐
                 │ completed  │  Done
                 └────────────┘

        On error:  ┌────────┐
                   │ failed │
                   └────────┘
        On timeout:┌─────────┐
                   │ timeout │
                   └─────────┘
```

### 11.4 Mix Compaction Details

Merges multiple small L1/L2 segments into fewer larger segments:

```
  Input:                           Output:
  ┌──────┐ ┌──────┐ ┌──────┐     ┌────────────────┐
  │seg A │ │seg B │ │seg C │ ──▶ │   seg D (new)  │
  │100 MB│ │50 MB │ │75 MB │     │   225 MB       │
  └──────┘ └──────┘ └──────┘     └────────────────┘
      ↓        ↓        ↓
   Dropped  Dropped  Dropped      → IndexInspector builds index for seg D
```

### 11.5 Clustering Compaction Details

Reorganizes data by a clustering key field to enable query pruning:

```
  Input (random data distribution):    Output (clustered by field "age"):
  ┌──────────────────┐                 ┌──────────────────┐
  │seg A: ages 5-95  │                 │seg X: ages 0-30  │  L2 + partition stats
  ├──────────────────┤    clustering   ├──────────────────┤
  │seg B: ages 10-88 │  ──────────▶   │seg Y: ages 31-60 │  L2 + partition stats
  ├──────────────────┤    compaction   ├──────────────────┤
  │seg C: ages 2-99  │                 │seg Z: ages 61-99 │  L2 + partition stats
  └──────────────────┘                 └──────────────────┘

  Query: "age BETWEEN 31 AND 60"
  → Only reads seg Y (prunes X and Z)
  → Reported 25x+ QPS improvement for scalar-filtered queries
```

For vector clustering keys, an analyze phase runs first to determine cluster
centroids before the actual compaction.

**Key source files:**
- `internal/datacoord/compaction_trigger_v2.go` — trigger manager
- `internal/datacoord/compaction_queue.go` — priority queue
- `internal/datacoord/compaction_task_l0.go` — L0 compaction
- `internal/datacoord/compaction_task_mix.go` — mix compaction
- `internal/datacoord/compaction_task_clustering.go` — clustering compaction
- `internal/datanode/compactor/sort_compaction.go` — sort compaction execution

---

## 12. Garbage Collection

After segments are dropped (via compaction or explicit collection drop), the
garbage collector cleans up their storage:

```
  ┌─────────────────────────────────────────────────────────┐
  │              DataCoord: Garbage Collector                │
  │                                                         │
  │  Periodically scans for Dropped segments:               │
  │                                                         │
  │  1. Check drop tolerance (time since segment dropped)   │
  │  2. If tolerance exceeded:                              │
  │     a. Remove insert binlog files from object storage   │
  │     b. Remove stats binlog files                        │
  │     c. Remove delta binlog files                        │
  │     d. Remove index files                               │
  │     e. Remove segment metadata from etcd                │
  │                                                         │
  │  Safety:                                                │
  │  • Respects dropTolerance TTL before cleanup            │
  │  • Handles missing binlogs with missingTolerance        │
  │  • Pauses when system CPU is high                       │
  │  • Concurrent removal with configurable pool size       │
  └─────────────────────────────────────────────────────────┘
```

**Key source file:** `internal/datacoord/garbage_collector.go`

---

## 13. End-to-End Lifecycle Diagram

```
  ┌─────────┐
  │ Client  │
  └────┬────┘
       │
       │ Insert / Delete / CreateIndex / Search
       ▼
  ┌─────────┐
  │  Proxy  │
  └────┬────┘
       │
  ═════╪═══════════════════════════ MsgStream (Pulsar/Kafka) ══════════
       │
       ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                          DataNode                                │
  │                                                                  │
  │  ┌─────────┐    ┌──────────┐    ┌────────────┐                  │
  │  │WriteNode│───▶│WriteBuf  │───▶│SyncManager │──▶ Object Storage│
  │  └─────────┘    │Manager   │    │(flush)     │   (binlogs)      │
  │                 └──────────┘    └────────────┘                   │
  │                                                                  │
  │  ┌──────────────┐    ┌────────────────┐                         │
  │  │IndexBuildTask│───▶│C++ Knowhere    │──▶ Object Storage       │
  │  │(from sched.) │    │(build index)   │   (index files)         │
  │  └──────────────┘    └────────────────┘                         │
  │                                                                  │
  │  ┌──────────────┐    ┌────────────────┐                         │
  │  │CompactTask   │───▶│Merge/Sort      │──▶ Object Storage       │
  │  │(from sched.) │    │segments        │   (new binlogs)         │
  │  └──────────────┘    └────────────────┘                         │
  └──────────────────────────────────────────────────────────────────┘
       │
       │ metadata updates
       ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                         DataCoord                                │
  │                                                                  │
  │  ┌────────────────┐  ┌───────────────┐  ┌──────────────────┐   │
  │  │SegmentManager  │  │IndexInspector │  │CompactionTrigger │   │
  │  │• allocate      │  │• detect       │  │• L0/Mix/Cluster  │   │
  │  │• seal          │  │  unindexed    │  │  policies        │   │
  │  │• flush         │  │• schedule     │  │• priority queue  │   │
  │  │                │  │  builds       │  │• schedule tasks  │   │
  │  └────────────────┘  └───────────────┘  └──────────────────┘   │
  │                                                                  │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  Segment Metadata (etcd)  │  Index Metadata (etcd)       │   │
  │  │  state, level, binlogs    │  segmentID→indexID→state     │   │
  │  │  (independent stores)     │  index file keys, sizes      │   │
  │  └──────────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘
       │
       │ target assignment
       ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                         QueryCoord                               │
  │                                                                  │
  │  ┌────────────────┐  ┌────────────────┐                         │
  │  │TargetManager   │  │SegmentChecker  │                         │
  │  │• NextTarget    │  │• diff target   │                         │
  │  │• CurrentTarget │  │  vs loaded     │                         │
  │  │                │  │• load/release  │                         │
  │  └────────────────┘  └────────┬───────┘                         │
  └───────────────────────────────┼─────────────────────────────────┘
                                  │ LoadSegmentsRequest
                                  │ (binlog paths + index file paths)
                                  ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                         QueryNode                                │
  │                                                                  │
  │  ┌───────────────┐  ┌───────────────────────────────────┐       │
  │  │SegmentLoader  │  │         Delegator                 │       │
  │  │• Phase 1:     │  │                                   │       │
  │  │  load binlogs │  │  Search fan-out:                  │       │
  │  │• Phase 2:     │  │  ┌─────────┐    ┌─────────────┐  │       │
  │  │  load indexes │  │  │Growing  │    │  Sealed     │  │       │
  │  │• load deltas  │  │  │(brute   │    │  (indexed   │  │       │
  │  │               │  │  │ force)  │    │   search)   │  │       │
  │  └───────────────┘  │  └────┬────┘    └──────┬──────┘  │       │
  │                      │       └──────┬─────────┘         │       │
  │                      │              ▼                    │       │
  │                      │        Merge top-K               │       │
  │                      └───────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────────────┘
```

### Complete Segment State + Level Lifecycle

```
  INSERT arrives
       │
       ▼
  ┌──────────────┐
  │  Growing/L1  │ ◄── Allocated by DataCoord SegmentManager
  └──────┬───────┘
         │ seal policy
         ▼
  ┌──────────────┐
  │  Sealed/L1   │
  └──────┬───────┘
         │ flush
         ▼
  ┌──────────────┐     index build      ┌──────────────────┐
  │  Flushed/L1  │────────────────────▶│  Index: Finished  │
  └──────┬───────┘  (data in binlogs,   └──────────────────┘
         │           index in index_files/
         │           — separate stores)
    ┌────┴────────────────────────┐
    │                             │
    ▼                             ▼
  Mix compaction              Clustering compaction
    │                             │
    ▼                             ▼
  ┌──────────────┐          ┌──────────────┐
  │  Flushed/L1  │ (new)    │  Flushed/L2  │ (new, with partition stats)
  └──────┬───────┘          └──────┬───────┘
         │                         │
         │   index build           │   index build
         ▼                         ▼
  ┌──────────────┐          ┌──────────────┐
  │Index:Finished│          │Index:Finished│
  └──────────────┘          └──────────────┘

  Original segments after compaction:
  ┌──────────────────┐
  │  Dropped         │ ──▶ Garbage Collector ──▶ Storage cleanup
  │  Compacted=true  │     (removes both binlogs AND index files)
  └──────────────────┘


  DELETE arrives
       │
       ▼
  ┌──────────────┐
  │  L0 segment  │ ◄── Delta-only, accumulates deletes per channel
  └──────┬───────┘
         │ L0 compaction trigger
         ▼
  Deletes merged into target L1/L2 segments
  L0 segments → Dropped
```
