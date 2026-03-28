---
name: Distributed Systems Topics for Spark
description: Key distributed systems topics to learn for understanding Spark codebase — RPC, scheduling, shuffle, fault tolerance, memory management, with learning order and resources
type: project
---

## Core Distributed Systems Concepts for Spark

### 1. RPC & Message Passing
- Request/response patterns, async messaging, serialization formats
- **Spark**: Netty RPC (Inbox/Outbox/Dispatcher), driver↔executor communication, MapStatus reporting

### 2. Scheduling & Task Distribution
- DAG-based scheduling, data locality, speculative execution, task retry
- **Spark**: `DAGScheduler` (stage creation), `TaskScheduler` (task assignment), `TaskSetManager` (retry/speculative), locality levels (PROCESS_LOCAL → NODE_LOCAL → RACK_LOCAL → ANY)

### 3. Shuffle (Distributed Data Exchange)
- Hash partitioning, sort-based merge, external sorting, disk spillover
- **Spark**: `ShuffleManager`, `SortShuffleWriter`, `BlockStoreShuffleReader`, shuffle files on disk, MapOutputTracker

### 4. Fault Tolerance & Lineage
- Lineage-based recovery vs checkpointing, idempotent operations, at-least-once vs exactly-once
- **Spark**: RDD lineage (recompute lost partitions), `OutputCommitCoordinator`, streaming checkpoints

### 5. Memory Management
- Off-heap vs on-heap, memory pools, eviction policies, spilling to disk
- **Spark**: `UnifiedMemoryManager` (execution vs storage split), `Tungsten` (off-heap binary format), `ExternalSorter` (spill when memory full)

### 6. Distributed Storage & Caching
- Block storage, replication, cache eviction (LRU)
- **Spark**: `BlockManager`, `MemoryStore`/`DiskStore`, broadcast variables, cached RDDs/DataFrames

### 7. Consensus & Coordination
- Leader election, heartbeats, failure detection, liveness vs safety
- **Spark**: `HeartbeatReceiver`, `DriverEndpoint` (executor registration), executor timeout/removal

### 8. Partitioning & Data Distribution
- Hash partitioning, range partitioning, skew handling, partition coalescing
- **Spark**: `HashPartitioner`, `RangePartitioner`, AQE skew handling, `CoalesceShufflePartitions`

### 9. Backpressure & Flow Control
- Rate limiting, bounded queues, push vs pull models
- **Spark**: Structured Streaming rate sources, `LinkedBlockingQueue` in RPC, AQE staged execution

### 10. Distributed Joins
- Broadcast join, shuffle join, sort-merge join, hash join, skew join
- **Spark**: `BroadcastHashJoinExec`, `SortMergeJoinExec`, `ShuffledHashJoinExec`, AQE join strategy switching

## Suggested Learning Order

```
Phase 1: Foundations (understand the plumbing)
  ├── RPC & Message Passing         ← already explored
  ├── Scheduling & Task Distribution
  └── Partitioning & Data Distribution

Phase 2: Data Movement (the expensive part)
  ├── Shuffle                       ← most performance issues live here
  ├── Distributed Joins             ← already explored
  └── Memory Management

Phase 3: Resilience (what happens when things fail)
  ├── Fault Tolerance & Lineage
  ├── Consensus & Coordination
  └── Distributed Storage & Caching

Phase 4: Advanced (optimization)
  ├── Backpressure & Flow Control
  └── AQE / adaptive systems       ← already explored
```

## Recommended Resources

| Resource | Covers |
|----------|--------|
| **"Designing Data-Intensive Applications" (Kleppmann)** | Partitioning, replication, serialization, fault tolerance |
| **MIT 6.824 Distributed Systems** (free lectures) | MapReduce, RPC, fault tolerance, consistency |
| **Spark source code** | Theory becomes concrete |

## Current Gaps (as of 2026-03-28)

Already covered: RPC, Catalyst optimizer, AQE, join strategies, file splitting, Hadoop integration
Remaining gaps: **scheduling**, **shuffle internals**, **memory management** — these three would give the deepest understanding of Spark's runtime behavior.
