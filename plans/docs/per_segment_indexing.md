# Per-Segment Indexing: No Global Index, No Merge

This document explains how indexing works when new segments are flushed alongside
already-indexed segments. The key insight: **there is no index merge**. Each segment
has its own completely independent index.

---

## Scenario

You have many old flushed segments, each with a finished HNSW/IVF/DISKANN index.
A new segment flushes. How does it get indexed?

---

## Step by Step

### 1. Segment Flushes (DataNode → Object Storage)

- DataNode writes the segment's insert binlogs to `insert_log/.../newSegmentID/...`
- DataCoord updates segment state: `Growing → Sealed → Flushing → Flushed`
- DataCoord sends the segment ID to `flushCh`

### 2. IndexInspector Detects the Unindexed Segment (DataCoord)

- The `flushCh` event (or periodic timer) triggers `IndexInspector`
- It calls `IsUnIndexedSegment(collectionID, newSegmentID)` — returns true
- It finds the existing index definition (the one your old segments were built with)
- Creates a `SegmentIndex` record: `{segmentID: new, indexID: existing, state: Unissued}`
- Enqueues an `IndexBuildTask` to the global scheduler

### 3. Index Is Built from Scratch for This Segment Alone (DataNode)

- Scheduler assigns the task to a DataNode
- DataNode downloads **only this segment's binlogs** from object storage
- Calls C++ Knowhere to build a full index (HNSW/IVF/DISKANN) on just this segment's data
- Uploads the resulting index files to `index_files/...`
- Updates `SegmentIndex` state → `Finished`

### 4. QueryNode Loads the New Segment (QueryCoord → QueryNode)

- QueryCoord's `SegmentChecker` sees the new flushed segment in `NextTarget`
- Sends `LoadSegmentsRequest` to a QueryNode with both binlog paths and index file paths
- QueryNode loads the data + index into memory

### 5. Search Fans Out to All Segments Independently

```
Search query arrives
    │
    ▼
Proxy fans out to QueryNodes
    │
    ├── QueryNode 1
    │     ├── Old Segment A → search using A's own HNSW index → top-K results
    │     └── Old Segment B → search using B's own HNSW index → top-K results
    │
    └── QueryNode 2
          ├── Old Segment C → search using C's own HNSW index → top-K results
          └── New Segment D → search using D's own HNSW index → top-K results
          │                    (built independently, never merged with A/B/C)
    │
    ▼
Proxy merges all top-K results into final top-K
```

---

## There Is No Index Merge

```
Old segments:                         New segment:

Segment A: HNSW index (100K vectors)
Segment B: HNSW index (100K vectors)      These indexes know
Segment C: HNSW index (100K vectors)      nothing about each other.
                                           No merge ever happens.
                         Segment D: HNSW index (100K vectors)
                         (built from D's binlogs only)
```

Each segment's index covers **only that segment's data**. At search time, every
segment is searched independently and results are merged by the proxy (top-K
reduction). This is how Milvus scales — adding new data means adding new segments
with new indexes, never rebuilding or merging into a single global index.

---

## The Only "Consolidation": Compaction

The only time indexes get consolidated is through **compaction**: if Mix compaction
merges segments A+B+C into a new segment E, then:

1. DataNode merges A+B+C binlogs into E's binlogs (new data files)
2. A, B, C are marked `Dropped`
3. IndexInspector detects E as unindexed
4. A fresh index is built from scratch for E's data
5. Old indexes for A, B, C are garbage-collected

This is **segment replacement**, not index merging. E's index is built from E's
binlogs alone, with no reference to A/B/C's old indexes.

```
BEFORE compaction:                    AFTER compaction:

Segment A: index_A (100K)
Segment B: index_B (100K)    ──▶     Segment E: index_E (300K)
Segment C: index_C (100K)            (built from scratch, A/B/C dropped)
```

---

## Key Source Files

- `internal/datacoord/index_inspector.go:87-229` — detects unindexed flushed segments
- `internal/datacoord/index_meta.go` — `IsUnIndexedSegment()`, per-segment index tracking
- `internal/datanode/index/task_index.go` — builds index from one segment's binlogs
- `internal/querycoordv2/checkers/segment_checker.go` — loads new segments onto QueryNodes
- `internal/proxy/task_search.go` — fans out search to all shards, merges top-K
