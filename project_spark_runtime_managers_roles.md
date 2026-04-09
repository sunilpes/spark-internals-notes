---
name: Spark Runtime Managers and Their Roles
description: Roles of all key runtime managers: CoarseGrainedSchedulerBackend, CoarseGrainedExecutorBackend, TaskSchedulerImpl, TaskSetManager, BlockManager, BlockManagerMaster, DiskBlockManager, MemoryManager, MapOutputTracker, ShuffleManager, OutputCommitCoordinator, SparkEnv as container
type: project
tags: [spark, architecture, managers, runtime, scheduling, storage, memory, shuffle]
---

## SparkEnv — The Container

SparkEnv (one per JVM) holds all runtime managers:
- rpcEnv, serializer, closureSerializer, serializerManager
- mapOutputTracker, broadcastManager, blockManager
- securityManager, metricsSystem, memoryManager
- shuffleManager, outputCommitCoordinator

## Scheduling Layer (Driver-side)

### CoarseGrainedSchedulerBackend
Bridge between Spark and cluster manager. Holds registry of live executors (`executorDataMap`).
Contains `DriverEndpoint` RPC endpoint that:
- Accepts executor registrations
- `makeOffers()` → collects free resources → asks TaskSchedulerImpl to match tasks
- `launchTasks()` → sends LaunchTask RPC to executors
- Detects executor death via `onDisconnected()`
- Implements `ExecutorAllocationClient` for dynamic allocation

### CoarseGrainedExecutorBackend
Executor-side RPC endpoint — receives and dispatches work.
- Receives `LaunchTask` → decodes TaskDescription → `executor.launchTask()`
- Receives `KillTask` → `executor.killTask()`
- Sends `StatusUpdate(taskId, state, result)` back to driver
- Detects driver disconnection → shuts down

### TaskSchedulerImpl
Task-level scheduler — decides WHICH task runs WHERE.
- `submitTasks(TaskSet)` → creates TaskSetManager, adds to scheduling pool (FIFO/Fair)
- `resourceOffers(WorkerOffer[])` → core matching: iterates TaskSetManagers, asks each for best task per executor (locality-aware)
- `statusUpdate()` → routes completion/failure to correct TaskSetManager
- `executorLost()` → propagates to all TaskSetManagers
- Called from multiple threads (DAGScheduler event loop, RPC handlers, periodic revival)

### TaskSetManager
Manages tasks within a SINGLE stage — locality and retry brain. One per active stage attempt.
- **Locality scheduling**: pending queues per level (PROCESS_LOCAL → NODE_LOCAL → NO_PREF → RACK_LOCAL → ANY), delay scheduling
- **`resourceOffer()`**: picks best pending task for an executor
- **Failure tracking**: `numFailures[taskIndex]`, aborts after `maxTaskFailures` (default 4)
- **Speculative execution**: detects slow tasks, launches duplicates
- **Executor loss**: re-queues running tasks, invalidates shuffle outputs

## Storage Layer

### BlockManager
Distributed key-value store for blocks (RDD partitions, shuffle data, broadcasts). One per JVM.
- **MemoryStore** — on-heap/off-heap storage
- **DiskStore** — spill-to-disk
- Read: `getLocalBytes()` / `getRemoteBytes()` — local first, then remote fetch
- Write: `putBytes()` / `putIterator()` with StorageLevel
- Spill/eviction, replication, per-task lock management

### BlockManagerMaster
Driver-side coordinator knowing where every block lives.
- Global registry: blockId → executor(s)
- `getLocations(blockId)` → used for locality scheduling
- Communicates via RPC to BlockManagerMasterEndpoint

### DiskBlockManager
Maps logical block IDs to physical files on disk.
- Creates dirs from `spark.local.dir`, hashes blocks across subdirectories
- `getFile(blockId)` → File path

## Memory Layer

### MemoryManager → UnifiedMemoryManager
Enforces memory boundaries between execution and storage. One per JVM. Four pools:
- On-Heap/Off-Heap × Execution/Storage
- UnifiedMemoryManager allows borrowing:
  - Storage borrows execution → evicted when execution reclaims
  - Execution borrows storage → never evicted by storage
- `spark.memory.fraction` (0.6) × `spark.memory.storageFraction` (0.5)

## Shuffle Layer

### ShuffleManager
Pluggable trait for shuffle implementations (default: SortShuffleManager).
- `registerShuffle()` — driver creates shuffle stage
- `getWriter()` — ShuffleMapTask writes partitioned output
- `getReader()` — reducer reads shuffle data

### MapOutputTracker (Master / Worker)
Tracks where shuffle map outputs are stored.
- **Master (driver)**: `shuffleId → ShuffleStatus → MapStatus[]`, `registerMapOutput()`, `removeOutputsOnExecutor()`, `incrementEpoch()`
- **Worker (executor)**: local cache of map output locations, `getMapSizesByExecutorId()`

## Other Managers

### OutputCommitCoordinator
"First committer wins" — prevents duplicate output writes to HDFS.
- `canCommit(stage, partition, attempt)` → asks driver via RPC
- Maintains `authorizedCommitters` per stage/partition

### SecurityManager
Authentication, authorization (ACLs), I/O encryption key management.

### BroadcastManager
Factory for Broadcast[T] objects (TorrentBroadcast).

### MetricsSystem
Collects/reports metrics. Sources → Sinks (console, CSV, JMX, Graphite).

## Architecture Diagram

```
                         DRIVER
┌─────────────────────────────────────────────────┐
│  DAGScheduler                                    │
│      │ submitTasks(TaskSet)                      │
│      ▼                                           │
│  TaskSchedulerImpl ←──── SchedulerBackend trait  │
│      │                        │                  │
│      │ resourceOffers()       │                  │
│      ▼                        ▼                  │
│  TaskSetManager[]    CoarseGrainedSchedulerBackend│
│  (locality, retry,       │                       │
│   speculation)           │ makeOffers()           │
│                          │ launchTasks()          │
│                          ▼                       │
│  MapOutputTrackerMaster  DriverEndpoint (RPC)    │
│  BlockManagerMaster      OutputCommitCoordinator │
│  MemoryManager           BroadcastManager        │
└──────────────────────────┬───────────────────────┘
                           │ LaunchTask / StatusUpdate (RPC)
                           ▼
                        EXECUTOR
┌─────────────────────────────────────────────────┐
│  CoarseGrainedExecutorBackend (RPC endpoint)     │
│      │                                           │
│      ▼                                           │
│  Executor → ThreadPool → TaskRunner.run()        │
│      │                                           │
│  BlockManager ←→ MemoryStore + DiskStore         │
│      │               │                           │
│      │           MemoryManager                   │
│      │         (UnifiedMemoryManager)            │
│      │                                           │
│  MapOutputTrackerWorker (cache of shuffle locs)  │
│  ShuffleManager (SortShuffleManager)             │
│  DiskBlockManager (file paths)                   │
└─────────────────────────────────────────────────┘
```
