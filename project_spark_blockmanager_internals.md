---
name: Spark BlockManager Internals
description: BlockManager architecture — MemoryStore, DiskStore, BlockInfoManager, BlockManagerMaster, read/write/spill flows, block types, locking, replication, driver vs executor roles
type: project
---

## BlockManager — Spark's Distributed Storage Engine

BlockManager exists on **both driver and every executor**. It stores and retrieves all data blocks — cached RDDs, shuffle data, broadcast variables, task results.

### What It Manages

| Data Type | BlockId Format | Example |
|-----------|---------------|---------|
| Cached RDD partitions | `rdd_<rddId>_<partition>` | `rdd_5_3` |
| Shuffle data | `shuffle_<shuffleId>_<mapId>_<reduceId>` | `shuffle_1_0_2` |
| Broadcast variables | `broadcast_<id>` | `broadcast_7` |
| Task results | `taskresult_<taskId>` | `taskresult_42` |
| Streaming blocks | `input-<streamId>-<uniqueId>` | streaming micro-batches |

### Architecture

```
Each Executor (and Driver):
┌─────────────────────────────────────────────────┐
│ BlockManager                                     │
│                                                  │
│  ┌────────────┐  ┌───────────┐  ┌─────────────┐ │
│  │ MemoryStore │  │ DiskStore  │  │ BlockInfo   │ │
│  │ (LinkedHash │  │ (files on  │  │ Manager     │ │
│  │  Map, LRU)  │  │  local     │  │ (locks +    │ │
│  │             │  │  disk)     │  │  metadata)  │ │
│  └──────┬──────┘  └─────┬─────┘  └─────────────┘ │
│         │               │                         │
│         └───────┬───────┘                         │
│                 ↓                                  │
│  ┌──────────────────────────┐                     │
│  │ BlockTransferService     │  ← fetch from       │
│  │ (Netty-based)            │    remote executors  │
│  └──────────────────────────┘                     │
│                 │                                  │
└─────────────────┼──────────────────────────────────┘
                  │ RPC
                  ↓
┌─────────────────────────────────────┐
│ BlockManagerMaster (on Driver)       │
│                                      │
│  blockLocations:                     │
│    Map[BlockId → Set[ExecutorId]]    │
│                                      │
│  blockManagerInfo:                   │
│    Map[ExecutorId → BlockManagerInfo] │
│    (memory used, blocks held)        │
└─────────────────────────────────────┘
```

### How Reads Work — Memory → Disk → Remote

`get(blockId)` (line 1347):

```
1. getLocalValues(blockId)
   ├── Check MemoryStore → found? return from memory
   ├── Check DiskStore   → found? read from disk
   │                        (maybe re-cache to memory if MEMORY_AND_DISK)
   └── Not found locally

2. getRemoteValues(blockId)
   ├── Ask BlockManagerMaster: "who has this block?"
   ├── Try same-host executor first (local directory read)
   └── Fetch over network via BlockTransferService
```

### How Writes Work

`getOrElseUpdate(blockId, storageLevel, makeIterator)` (line 1432):

```
1. Block already cached and visible? → return it
2. Not cached → compute via makeIterator()
3. Store based on StorageLevel:
   ├── MEMORY_ONLY      → MemoryStore.putIterator()
   ├── DISK_ONLY        → DiskStore.put()
   ├── MEMORY_AND_DISK  → try memory first, spill to disk if full
   └── With _2 suffix   → replicate to another executor
4. Report to BlockManagerMaster: "I have block X, size Y"
```

### Memory Store vs Disk Store

**MemoryStore** (`memory/MemoryStore.scala`):
- LinkedHashMap (access-ordered → LRU eviction)
- Two formats: DeserializedMemoryEntry (Java objects) vs SerializedMemoryEntry (ByteBuffer)
- When full → evict oldest blocks (LRU) → spill to DiskStore if level allows

**DiskStore** (`DiskStore.scala`):
- Files on local disk managed by DiskBlockManager
- Supports optional encryption
- Can memory-map large files for efficient reads

**Spill flow:**
```
MemoryStore full → evictBlocksToFreeSpace()
  → pick LRU blocks
  → if StorageLevel.useDisk → write to DiskStore
  → else → data lost (MEMORY_ONLY level)
  → free memory
```

### Block Locking (BlockInfoManager)

```
lockForReading(blockId)   → shared lock (multiple readers OK)
lockForWriting(blockId)   → exclusive lock (blocks all others)
downgradeLock()           → convert write → read lock
releaseAllLocksForTask()  → auto-cleanup when task ends
```

### BlockManagerMaster (Driver Side)

Tracks **which executor has which blocks** cluster-wide:

```
blockLocations: Map[BlockId → Set[BlockManagerId]]
blockManagerInfo: Map[BlockManagerId → BlockManagerInfo]
```

Key operations: `registerBlockManager()`, `updateBlockInfo()`, `getLocations()`, `removeRdd()`, `removeExecutor()`

### Replication

When StorageLevel has `_2` suffix (e.g., MEMORY_ONLY_2):
```
Executor-1 stores block locally
  → BlockReplicationPolicy picks a peer
  → BlockTransferService uploads block to peer
  → Both report to BlockManagerMaster
  → If Executor-1 dies → peer still has the block
```

### Driver vs Executor BlockManager

| | Driver | Executor |
|--|--------|----------|
| Has BlockManager? | Yes | Yes |
| Has BlockManagerMaster? | Yes (runs master endpoint) | No (talks via RPC) |
| Stores what? | Broadcast vars, small data | RDD partitions, shuffle, broadcasts, task results |
| Main role | Coordinate block locations | Store/retrieve data for tasks |

### Key Source Files

| Component | File |
|-----------|------|
| BlockManager | `core/.../storage/BlockManager.scala` |
| MemoryStore | `core/.../storage/memory/MemoryStore.scala` |
| DiskStore | `core/.../storage/DiskStore.scala` |
| BlockInfoManager | `core/.../storage/BlockInfoManager.scala` |
| BlockManagerMaster | `core/.../storage/BlockManagerMaster.scala` |
| BlockManagerMasterEndpoint | `core/.../storage/BlockManagerMasterEndpoint.scala` |
| BlockId types | `core/.../storage/BlockId.scala` |
