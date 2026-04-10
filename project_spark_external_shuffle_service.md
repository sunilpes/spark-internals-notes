---
name: External Shuffle Service Architecture and Network Impact
description: How ESS works: separate JVM process, executor registration, shuffle file layout, block fetch flow, Netty zero-copy, index caching, network latency sources, single-process bottleneck, tradeoffs vs direct executor serving
type: project
tags: [spark, shuffle, external-shuffle-service, network, latency, fault-tolerance]
---

## What It Is

Separate long-lived JVM process (one per worker node) that serves shuffle data to reducers. Executors can die and shuffle data is still available.

## Shuffle File Layout

IndexShuffleBlockResolver writes two files per map task:
- `shuffle_<shuffleId>_<mapId>_0.data` — all partitions concatenated
- `shuffle_<shuffleId>_<mapId>_0.index` — byte offsets (8 bytes each) per partition

## End-to-End Flow

### 1. Registration
Executor's BlockManager.initialize() sends `RegisterExecutor(execId, localDirs, subDirsPerLocalDir)` RPC to ESS:7337. ESS stores in ConcurrentMap + optional RocksDB for restart recovery.

### 2. Map Task Writes
SortShuffleWriter → IndexShuffleBlockResolver.writeMetadataFileAndCommit() → .index + .data files on local disk. No ESS interaction.

### 3. Reducer Fetches
- MapOutputTracker returns BlockManagerId pointing to ESS host:7337 (not executor)
- OneForOneBlockFetcher sends `FetchShuffleBlocks(appId, execId, shuffleId, mapIds, reduceIds)` to ESS
- ESS looks up executor's localDirs, reads index (cached in 100MB Guava cache), finds byte range
- Returns FileSegmentManagedBuffer → Netty zero-copy streams to reducer

## Network Latency Sources

### 1. No extra network hop
ESS is co-located with data on same node. Reducer→ESS is same network distance as reducer→executor.

### 2. Single-process bottleneck (KEY concern)
One ESS per node serves ALL apps and ALL reducers. Under heavy load: thread contention, disk I/O contention, network bandwidth saturation through one process.

### 3. Always reads from disk
Without ESS: data might be in executor's page cache or memory buffer. With ESS: always disk I/O (relies on OS page cache for warm reads).

### 4. Fewer connections
N executors on 5 nodes → 5 ESS connections vs N executor connections.

## Optimizations

- **Batch fetch**: contiguous reduce partitions in one RPC
- **Index caching**: 100MB Guava LoadingCache avoids re-reading .index files
- **Netty zero-copy**: sendfile() syscall, data never enters ESS JVM heap
- **Throttled fetching**: reducer limits maxBytesInFlight

## Tradeoff Summary

Without ESS: faster hot path (memory serving), load spread across executors, BUT executor death = data lost, no dynamic scaling.

With ESS: shuffle survives executor death, enables dynamic allocation, fewer connections, BUT single-process bottleneck, always disk reads, shared across all apps.

## Config

- `spark.shuffle.service.enabled` = false (default)
- `spark.shuffle.service.port` = 7337
- `spark.shuffle.service.index.cache.size` = 100m
- `spark.shuffle.service.db.enabled` = true (RocksDB persistence)

## Key Source Files

- `deploy/ExternalShuffleService.scala` — entry point, TransportServer
- `network/shuffle/ExternalBlockHandler.java` — RPC handler
- `network/shuffle/ExternalShuffleBlockResolver.java` — file lookup, index cache
- `storage/BlockManager.scala:639` — registration with ESS
- `shuffle/IndexShuffleBlockResolver.scala` — .data/.index file layout
- `storage/ShuffleBlockFetcherIterator.scala` — reducer fetch logic
