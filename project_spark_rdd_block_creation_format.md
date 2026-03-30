---
name: Spark RDD Block Creation and Storage Format
description: How RDD blocks are created lazily, what they look like in memory (DeserializedMemoryEntry vs SerializedMemoryEntry) and on disk (file layout, serialization), StorageLevel options, spill/eviction
type: project
tags: [spark-storage, spark-core]
---

## How RDD Blocks Are Created and What They Look Like

### `.cache()` Does Nothing — Blocks Are Created Lazily

```scala
rdd.cache()    // just sets storageLevel = MEMORY_ONLY, no blocks yet
rdd.count()    // NOW blocks are created on executors
```

The block creation happens when a task calls `rdd.iterator(partition)` (RDD.scala:334):

```
rdd.iterator(partition)
  ↓
storageLevel != NONE? → getOrCompute()
  ↓
blockId = RDDBlockId(rddId=5, partition=3)  →  "rdd_5_3"
  ↓
BlockManager.getOrElseUpdateRDDBlock()
  ├── Cache hit? → return cached data
  └── Cache miss? → compute partition → store in MemoryStore/DiskStore
```

### Block Format IN MEMORY

MemoryStore stores blocks in a `LinkedHashMap[BlockId, MemoryEntry]` (access-ordered for LRU).

**Option A: Deserialized** (`MEMORY_ONLY`, `MEMORY_AND_DISK`)

```
entries["rdd_5_3"] = DeserializedMemoryEntry(
    value: Array[T],        ← actual Java objects in an array
    size: 48576,            ← estimated bytes (via SizeEstimator)
    memoryMode: ON_HEAP,
    classTag: ClassTag[Row]
)
```

```
In JVM heap:
  Array[Row]
    [0] → Row(1, "alice", 30)     ← live Java objects
    [1] → Row(2, "bob", 25)
    [2] → Row(3, "charlie", 35)
```

Fast to read (no deserialization), but uses more memory (object headers, pointers).

**Option B: Serialized** (`MEMORY_ONLY_SER`, `MEMORY_AND_DISK_SER`)

```
entries["rdd_5_3"] = SerializedMemoryEntry(
    buffer: ChunkedByteBuffer,  ← serialized bytes in chunks
    memoryMode: ON_HEAP,        ← (or OFF_HEAP)
    classTag: ClassTag[Row]
)
```

Compact (no object overhead), but needs deserialization on read. Can be **off-heap**.

### Block Format ON DISK

**Directory structure** (DiskBlockManager.scala:95):

```
spark.local.dir/
├── blockmgr-<UUID>/
│   ├── 00/
│   │   ├── rdd_5_3          ← block file
│   │   ├── shuffle_1_0_2
│   │   └── broadcast_7
│   ├── 01/
│   ├── 02/
│   │   └── rdd_5_7
│   └── 3f/                  ← hex subdirs (default 64)
```

Subdirectory chosen by: `hash(blockName) % numSubDirs`

**File content** — raw serialized bytes, no header:

```
rdd_5_3 file on disk:
┌──────────────────────────────────────────┐
│ [Serialized object 1][Serialized obj 2]  │
│ [Serialized obj 3]...                    │
│ Format: Serializer stream (Java/Kryo)    │
│ Optional: compressed (LZ4/Snappy/Zstd)   │
│ Optional: encrypted (AES)                │
│ No header, no index — just a stream      │
└──────────────────────────────────────────┘
```

Serialization path:
```
Iterator[T] → Serializer.serializeStream() → wrapForCompression() → wrapForEncryption() → File
```

### StorageLevel Options

| Level | Disk | Memory | Off-Heap | Deserialized | Replicas |
|-------|------|--------|----------|-------------|----------|
| `MEMORY_ONLY` | - | yes | - | yes | 1 |
| `MEMORY_ONLY_SER` | - | yes | - | no | 1 |
| `MEMORY_AND_DISK` | yes | yes | - | yes | 1 |
| `MEMORY_AND_DISK_SER` | yes | yes | - | no | 1 |
| `DISK_ONLY` | yes | - | - | - | 1 |
| `OFF_HEAP` | yes | yes | yes | no | 1 |
| `*_2` variants | | | | | 2 |

### What Happens When Memory Is Full

```
MEMORY_ONLY:
  Memory full → data LOST (not cached, recomputed next time)

MEMORY_AND_DISK:
  Memory full → evict LRU blocks to disk → store new block in memory
  Eviction: LinkedHashMap access-order → oldest accessed block evicted first
```

### Complete Lifecycle

```
1. rdd.cache()              → sets storageLevel, no blocks created
2. rdd.count() triggers task → rdd.iterator() → getOrCompute()
3. Creates RDDBlockId("rdd_5_3")
4. BlockManager: cache miss → compute → store
   MEMORY_ONLY:     → Array[T] in DeserializedMemoryEntry
   MEMORY_ONLY_SER: → ChunkedByteBuffer in SerializedMemoryEntry
   MEMORY_AND_DISK: → try memory, spill to disk if full
   DISK_ONLY:       → serialize to file blockmgr-xxx/0a/rdd_5_3
5. Report to BlockManagerMaster: "I have rdd_5_3, size=48KB"
6. Next action → cache HIT → no recomputation
```

### Key Source Files

| Component | File | Key Lines |
|-----------|------|-----------|
| cache/persist API | RDD.scala | 166-205 |
| iterator & getOrCompute | RDD.scala | 334-409 |
| MemoryEntry types | MemoryStore.scala | 43-59 |
| LinkedHashMap storage | MemoryStore.scala | 93 |
| Deserialized put | MemoryStore.scala | 309-333 |
| Serialized put | MemoryStore.scala | 345-386 |
| DiskStore put | DiskStore.scala | 64-115 |
| File paths | DiskBlockManager.scala | 95-128 |
| StorageLevel | StorageLevel.scala | 39-44, 148-162 |

## Related Notes

- [[project_spark_blockmanager_internals]] — BlockManager that stores these blocks
- [[project_spark_memory_unrolling_explained]] — How blocks are gradually materialized
- [[project_spark_blockmanager_file_processing]] — When BlockManager creates blocks vs streaming
