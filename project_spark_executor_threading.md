---
name: Spark Executor Multithreading
description: How multithreading works inside executor — CachedThreadPool for tasks, TaskRunner lifecycle, all thread types (heartbeat, metrics, reaper, RPC, shuffle), shared resources, thread safety patterns, memory sharing
type: project
tags: [spark, executor, threading, concurrency, task-execution]
---

#spark-core #spark-execution #spark-distributed

## Multithreading Inside a Spark Executor

### All Threads in an Executor JVM

```
Executor JVM
├── Task Thread Pool (CachedThreadPool — unbounded)
│   ├── Thread "task-0" → TaskRunner → task.run()
│   ├── Thread "task-1" → TaskRunner → task.run()
│   └── ...
├── Heartbeat Thread (1 thread, every 10s)
├── Metrics Poller Thread (1 thread)
├── Kill Mark Cleanup Thread (1 thread)
├── Task Reaper Pool (dynamic, for killing stuck tasks)
├── RPC/Netty Threads (handle driver messages)
└── Shuffle Fetch Threads (Netty transport)
```

### How a Task Gets Launched

```
LaunchTask RPC → CoarseGrainedExecutorBackend.receive() [line 181]
  → Executor.launchTask() [line 550]
    → createTaskRunner() → runningTasks.put(taskId, tr) → threadPool.execute(tr)
      → TaskRunner.run() [line 805]:
        Set classloader → Deserialize task → task.run() → Serialize result → StatusUpdate
```

### Task Thread Pool

- `Executors.newCachedThreadPool()` — unbounded, creates threads on demand
- Uses `UninterruptibleThread` — prevents library hangs on interrupt
- Driver limits tasks sent based on `spark.executor.cores`

### Shared Resources & Thread Safety

```
Shared (thread-safe):
├── runningTasks: ConcurrentHashMap       ← lock-free
├── killMarks: ConcurrentHashMap          ← lock-free
├── BlockManager                          ← internal locks
├── MemoryManager                         ← synchronized
└── SerializerManager                     ← newInstance() per task

Per-thread (no sharing):
├── TaskContext, Serializer instance, ClassLoader, Task metrics

Protected by locks:
├── updateDependencies: ReentrantLock     ← JAR downloads
├── taskReaperForTask: synchronized       ← reaper creation
└── TaskRunner.finished: synchronized     ← completion + notifyAll
```

### Concurrency Patterns

| Pattern | Where | Why |
|---------|-------|-----|
| ConcurrentHashMap | runningTasks, killMarks | High-concurrency from task + heartbeat threads |
| volatile | reasonIfKilled, task, threadId | Cross-thread visibility |
| synchronized | TaskRunner.finished | Coordinate with TaskReaper via wait/notifyAll |
| ReentrantLock | updateDependenciesLock | Interruptible JAR downloads |
| Thread-local | ClassLoader, taskDeserializationProps | Per-task isolation |
| UninterruptibleThread | All task threads | Defer interrupts |

### Memory Sharing Between Concurrent Tasks

```
8GB executor, 4 tasks:
├── Execution Memory (synchronized)
│   ├── task-0: 500MB, task-1: 500MB, task-2: 500MB, task-3: 500MB
├── Storage Memory: 2GB shared cached blocks
└── Tasks compete — if one needs more, can spill another's data
```

### Key Source Files

| Component | File | Lines |
|-----------|------|-------|
| Thread Pool | Executor.scala | 306-313 |
| launchTask() | Executor.scala | 550-600 |
| TaskRunner.run() | Executor.scala | 805-1070 |
| TaskReaper | Executor.scala | 1225-1318 |
| Heartbeat | Executor.scala | 482-485 |
| LaunchTask handler | CoarseGrainedExecutorBackend.scala | 181-188 |

## Related Notes
- [[project_spark_rpc_inbox_dispatcher]] — RPC system that delivers LaunchTask to executor
- [[project_spark_rpc_message_format]] — LaunchTask, StatusUpdate message format
- [[project_spark_heartbeat_contents]] — What heartbeat thread sends to driver
- [[project_spark_task_completion_tracking]] — How task results flow back to driver
- [[project_spark_blockmanager_internals]] — BlockManager shared across task threads
- [[project_spark_memory_unrolling_explained]] — Memory allocation for caching within tasks
- [[project_spark_distributed_systems_topics]] — Broader distributed systems context
