## 1. Project Title

Custom LSM-Tree Storage Engine

## 2. Difficulty

Mid-Level

### Rationale
This project dives into database internals and storage architecture. The developer must implement a Log-Structured Merge-tree (LSM-tree) from scratch, managing in-memory sorted structures (MemTables), flushing to immutable disk files (SSTables) with binary indexing, and executing background compaction algorithms. It teaches disk I/O optimization, binary serialization, and concurrency control.

## 3. Project Overview

The Custom LSM-Tree Storage Engine is an embedded key-value database library. Designed for high-throughput write workloads, it buffers incoming writes in a memory-based sorted structure (MemTable) backed by an append-only write-ahead log (WAL) for durability. When the MemTable reaches capacity, it flushes to disk as an immutable Sorted String Table (SSTable) accompanied by sparse index files for fast binary searches. It includes background compaction routines to merge older SSTables and reclaim disk space from deleted or overwritten keys.

## 4. Problem Statement

Traditional B-Tree databases perform random disk I/O for every write operation, which becomes a major performance bottleneck for write-heavy applications. 
- Random disk writes cause disk head thrashing on HDDs and wear out SSD cells prematurely.
- Without a write-ahead log (WAL), server crashes result in data loss for in-memory updates.
- As databases grow, fragmented files degrade read and write performance without structured compaction.

Building an LSM-tree engine reveals how modern high-performance databases (like RocksDB, Cassandra, and LevelDB) achieve massive write throughput by turning random writes into sequential disk operations.

## 5. Proposed Solution

The storage engine implements the classic LSM-tree architecture:
1. **Write Path**: Incoming `PUT` or `DELETE` requests are appended to a disk WAL for crash recovery, then inserted into an in-memory sorted structure (MemTable, e.g., a SkipList or Red-Black Tree).
2. **Flush Mechanism**: When the MemTable reaches a threshold size, it is frozen and flushed to disk as an immutable SSTable file. A new active MemTable takes its place.
3. **Read Path**: The engine checks the active MemTable first. If not found, it checks immutable MemTables, then searches SSTables on disk from newest to oldest using sparse index files and Bloom filters.
4. **Compaction**: A background worker periodically merges multiple SSTables, discarding obsolete tombstones and overwritten keys to reclaim disk space.

## 6. Project Goal

To build a fully functional, embedded key-value store that supports fast sequential writes, persistent crash recovery via WAL, efficient point lookups, and background SSTable compaction.

## 7. Core Workflow

```text
Client Write (PUT key=val) ──> Append to Write-Ahead Log (WAL)
                                      │
                                      ▼
                               Insert into MemTable (In-Memory Sorted Map)
                                      │
                                      ├─────── Memtable Full? ───────┐
                                      │                              │ [Yes]
                                      │                              ▼
                                      │                    Flush to SSTable (Disk)
                                      │                    + Build Sparse Index
                                      │                              │
                                      │<─────────────────────────────┘
                                      │
                                   [Success]
```

## 8. Functional Requirements

### Core Operations
- **PUT(key, value)**: Insert or update a key-value pair.
- **GET(key)**: Retrieve the value for a key across MemTables and SSTables.
- **DELETE(key)**: Write a tombstone marker to signify deletion.

### Persistence & Durability
- **Write-Ahead Log (WAL)**: Append all writes to a disk log file to ensure durability across crashes.
- **SSTable Serialization**: Serialize sorted key-value pairs into immutable disk files with binary headers and sparse index offsets.

### Compaction
- **Background Compaction**: Implement a merge-sort compaction algorithm that combines multiple SSTables into a single new SSTable while purging tombstones.

## 9. Non-Functional Requirements

### Performance
- **Write Throughput**: MemTable buffering must ensure high write speeds (limited only by sequential disk I/O for the WAL).
- **Read Efficiency**: Sparse indexes and optional Bloom filters must ensure point lookups require scanning minimal disk blocks.

## 10. Main Entities / Data Model

### MemTable
- In-memory sorted structure (e.g., SkipList or balanced tree mapping Keys to Values/Tombstones).

### SSTable (Sorted String Table)
- **Data File**: Sequential list of sorted key-value pairs on disk.
- **Index File**: Sparse index mapping specific keys to byte offsets in the data file.

### Tombstone
- A special marker value indicating that a key has been deleted.

## 11. System Components

- **WAL Manager**: Handles append-only logging for crash recovery.
- **MemTable Engine**: In-memory sorted container.
- **SSTable Writer / Reader**: Handles binary file serialization, sparse indexing, and disk lookups.
- **Compactor Worker**: Background thread running merge-compaction on SSTables.

## 12. Important Technical Challenges

### Disk Layout and Sparse Indexing
- **Challenge**: Searching an entire SSTable file on disk for every `GET` request is too slow. You must build a sparse index that stores key offsets every $N$ entries.
- **Concepts**: Binary search on disk, byte offset seeking, file framing.

### Handling Deletions (Tombstones)
- **Challenge**: Since SSTables are immutable, you cannot simply delete a key from disk. You must write a "tombstone" entry and clean it up during compaction.
- **Concepts**: Soft deletes, garbage collection through compaction.

## 13. Suggested Technology Areas

- **Language**: Go, Rust, or Java (languages with strong low-level binary manipulation and concurrency primitives).
- **Storage**: Direct file system disk operations (`os.File`, `std::fs`).

## 14. Skills and Knowledge Gained

### Database Internals
- LSM-tree architecture and write-optimized data structures.
- Crash recovery using Write-Ahead Logs (WAL).
- Disk file formats, binary serialization, and sparse indexing.

## 15. Recommended Development Phases

1. **Phase 1 - In-Memory MemTable & WAL**: Build an in-memory sorted map and append every write to a simple WAL file on disk. Implement crash recovery by replaying the WAL on startup.
2. **Phase 2 - SSTable Flushing**: When the MemTable hits capacity, serialize its sorted entries into an immutable SSTable file on disk along with a sparse index file.
3. **Phase 3 - Multi-Level Read Path**: Implement the `GET` lookup path: check MemTable first, then scan SSTables using sparse index binary searches.
4. **Phase 4 - Tombstones & Compaction**: Add support for `DELETE` tombstones and write a background compactor worker that merges old SSTables.

## 16. Testing Requirements

- **Unit Tests**: Test MemTable sorting, binary serialization/deserialization of key-value pairs, and sparse index lookups.
- **Crash Recovery Tests**: Perform 10,000 writes, abruptly kill the process, restart, and verify all data is successfully recovered from the WAL.
- **Compaction Tests**: Trigger multiple flushes, verify that compaction successfully merges files and removes tombstones.

## 17. Security Considerations

- **File Permissions**: Ensure database data files and WAL logs are created with restricted file permissions (`0600`) to prevent unauthorized local users from reading sensitive database contents.

## 18. Possible Extensions

- **Bloom Filters**: Add in-memory Bloom filters for each SSTable to instantly skip reading disk files when a key definitely does not exist.
- **Range Queries**: Implement iterator support to scan keys within a specific range (`SCAN start_key end_key`).

## 19. Learning Questions

- Why are LSM-trees faster for write-heavy workloads compared to B-Trees?
- What is the purpose of a Write-Ahead Log (WAL) if SSTables are already persisted to disk?
- How does sparse indexing speed up disk lookups without storing an index for every single key?

## 20. Completion Criteria

- [ ] `PUT`, `GET`, and `DELETE` operations work correctly.
- [ ] Writes are durably logged to a WAL and successfully recovered on restart.
- [ ] MemTable flushes to immutable SSTables with sparse index files when full.
- [ ] Read lookups correctly search MemTables and SSTables in the correct order.
- [ ] Background compaction successfully merges SSTables and cleans up tombstones.