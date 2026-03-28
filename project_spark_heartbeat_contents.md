---
name: Spark Heartbeat Contents
description: What a single executor heartbeat carries — executorId, per-task accumulators, blockManagerId, 20 executor metrics (JVM, GC, memory pools, process tree), and what driver does with it
type: project
---

## Heartbeat Contents

```scala
case class Heartbeat(
    executorId: String,                                          // "1"
    accumUpdates: Array[(Long, Seq[AccumulatorV2[_, _]])],       // per-task counters
    blockManagerId: BlockManagerId,                              // host:port identity
    executorUpdates: Map[(Int, Int), ExecutorMetrics])            // per-stage peak metrics
```

### Field 1: `executorId`
Just the executor's ID string (e.g., `"0"`, `"1"`, `"2"`).

### Field 2: `accumUpdates` — per-task accumulator snapshots
For each **running task** on this executor:

| Data | Example |
|------|---------|
| `taskId` | `42` |
| Accumulators | bytes read, records read, bytes written, shuffle bytes, custom accumulators |

### Field 3: `blockManagerId` — executor's identity
```scala
BlockManagerId(executorId, host, port)
// e.g., BlockManagerId("1", "worker-node-3", 43721)
```

### Field 4: `executorUpdates` — peak resource metrics per active stage

Keyed by `(stageId, stageAttemptId)`, values are `ExecutorMetrics` containing **20 metrics**:

| Category | Metrics |
|----------|---------|
| **JVM Memory** | `JVMHeapMemory`, `JVMOffHeapMemory` |
| **Spark Memory** | `OnHeapExecutionMemory`, `OffHeapExecutionMemory`, `OnHeapStorageMemory`, `OffHeapStorageMemory`, `OnHeapUnifiedMemory`, `OffHeapUnifiedMemory` |
| **NIO Buffers** | `DirectPoolMemory`, `MappedPoolMemory` |
| **GC** | `MinorGCCount`, `MinorGCTime`, `MajorGCCount`, `MajorGCTime` |
| **Process Tree** | `ProcessTreeJVMVMemory`, `ProcessTreeJVMRSSMemory`, `ProcessTreePythonVMemory`, `ProcessTreePythonRSSMemory`, `ProcessTreeOtherVMemory`, `ProcessTreeOtherRSSMemory` |

### What the Driver Does With It

```
HeartbeatReceiver.receiveAndReply(Heartbeat)
  ├── Mark executor as alive (reset timeout clock)
  ├── Forward accumUpdates to TaskScheduler
  ├── Forward executorUpdates to SparkListenerBus
  └── Reply with HeartbeatResponse(reregisterBlockManager: Boolean)
```

If an executor **misses heartbeats** for `spark.network.timeout` (default 120s), the driver considers it dead and removes it.

### Key Source Files

| File | What |
|------|------|
| `core/.../HeartbeatReceiver.scala:43` | Heartbeat case class |
| `core/.../metrics/ExecutorMetricType.scala` | 20 metric definitions |
| `core/.../executor/ExecutorMetrics.scala` | Metrics collection |
