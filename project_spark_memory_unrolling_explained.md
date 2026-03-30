---
name: Spark Memory Unrolling Process Explained
description: How Spark gradually materializes an iterator into memory (unrolling), periodic memory checks, what happens when memory is full — MEMORY_ONLY vs MEMORY_AND_DISK fallback, with concrete step-by-step example
type: project
tags: [spark-storage, spark-core]
---

## What is "Unrolling"?

When Spark caches an RDD partition, it doesn't know the final size upfront. Data comes as a lazy `Iterator[T]`. Spark gradually materializes the iterator into an array, checking memory at each step. This is called **unrolling**.

### Step-by-Step Example

Executor with **1 GB storage memory**, partition that turns out to be **1.5 GB**:

```
Step 1: Reserve initial memory (16 KB default)
  unrollMemory = 16KB
  vector = SizeTrackingVector[] (empty)

Step 2: Start unrolling — add elements one by one
  values.next() → Row(0, "name_0", 0.0) → vector += row
  values.next() → Row(1, "name_1", 1.5) → vector += row
  ...

Step 3: Every 16 elements, CHECK memory
  elementsUnrolled = 16 → check!
  currentSize = 2KB → still under threshold → keep going

Step 4: Vector grows, needs more memory
  elementsUnrolled = 10000 → check!
  currentSize = 1MB → exceeds threshold!
  Ask MemoryManager: "Can I have 500KB more?"
  MemoryManager: "Yes" → keepUnrolling = true
  Continue unrolling...

Step 5: Reaches ~700MB
  Ask MemoryManager: "Can I have 350MB more?"
  MemoryManager: "Yes, but I had to evict some LRU blocks"
  Continue unrolling...

Step 6: Reaches ~1GB — MEMORY IS FULL
  Ask MemoryManager: "Can I have 500MB more?"
  MemoryManager: "NO — nothing left to evict"
  keepUnrolling = false  ← STOP!
```

### What Happens Next? Depends on StorageLevel

**Case A: `MEMORY_ONLY`** — data is lost, not cached

```
vector has 10M rows (partially unrolled)
remaining iterator has 90M rows (not yet read)

Returns: PartiallyUnrolledIterator
  = vector.iterator ++ remaining original iterator

Task continues processing ALL 100M rows from this combined iterator.
But the block is NOT cached — next time → recomputed from scratch.
```

**Case B: `MEMORY_AND_DISK`** — spill to disk

```
Memory failed → BlockManager catches this:

  diskStore.put("rdd_5_3") { channel →
    serializerManager.dataSerializeStream(blockId, outputStream,
      partiallyUnrolledIterator ++ remainingIterator)
  }

Result: entire 1.5GB partition written to disk file.
Next time → read from disk (not recomputed).
```

### Visual Timeline

```
Memory: [==========] 1GB total

Start unrolling rdd_5_3:
  [▓.........] 100MB   — check: OK, keep going
  [▓▓▓.......] 300MB   — check: OK, keep going
  [▓▓▓▓▓▓....] 600MB   — check: OK, evict some old blocks
  [▓▓▓▓▓▓▓▓▓.] 900MB   — check: OK, evicted more
  [▓▓▓▓▓▓▓▓▓▓] 1000MB  — check: FAIL! no more memory

MEMORY_ONLY:     → discard vector, block NOT cached
MEMORY_AND_DISK: → write everything to disk, block cached on disk
```

### Why Check Periodically?

Checking every element is too slow. Spark checks every `memoryCheckPeriod` elements (starts at 16, doubles):
```
Check at:  16, 32, 64, 128, 256, 512, 1024, ...
```

Balances between catching memory exhaustion early and not adding overhead.

### Key Takeaway

Spark never blindly loads an entire partition into memory. It's a **gradual, memory-aware process** that stops as soon as memory is exhausted and gracefully falls back to disk or recomputation.

### Key Source Location

`MemoryStore.scala:230-296` — the unrolling while loop with periodic memory checks

## Related Notes

- [[project_spark_blockmanager_internals]] — BlockManager that triggers unrolling
- [[project_spark_rdd_block_creation_format]] — Block formats after successful unrolling
- [[project_spark_blockmanager_file_processing]] — When unrolling happens vs streaming
