---

name: Executor OOM Death Recovery Flow
description: "Complete workflow when executor dies from OOM: JVM exit, driver detection via RPC disconnect, TaskSetManager re-queuing (no failure count), shuffle output loss, FetchFailed, stage resubmission, external shuffle service difference"
type: project
tags: [spark, fault-tolerance, executor, oom, scheduler, shuffle, recovery]
---

## Executor OOM Death — Complete Recovery Workflow

### Step 1: Executor JVM Dies Immediately

When `java.lang.OutOfMemoryError` thrown during `TaskRunner.run()`:
- `Executor.isFatalError()` returns true for OOM
- `SparkUncaughtExceptionHandler` catches it → `System.exit(SparkExitCode.OOM)`
- JVM killed instantly, **no graceful notification to driver**

Key files: `Executor.scala` (TaskRunner.run lines 805-1114), `SparkUncaughtExceptionHandler.scala`

### Step 2: Driver Detects Disconnection (Netty RPC timeout)

```
CoarseGrainedSchedulerBackend.DriverEndpoint.onDisconnected(remoteAddress)
  → removeExecutor(executorId, ExecutorProcessLost(...))
    → executorDataMap -= executorId, totalCoreCount reduced
    → scheduler.executorLost()    // → TaskSchedulerImpl
    → dagScheduler.executorLost() // → DAGScheduler
    → backend.reviveOffers()      // reschedule work
```

Key file: `CoarseGrainedSchedulerBackend.scala` (onDisconnected line 402, removeExecutor line 463)

### Step 3: TaskSchedulerImpl — Clean Up

`executorLost()` (TaskSchedulerImpl.scala:982):
- Removes executor from tracking maps
- Calls `rootPool.executorLost()` → propagates to every TaskSetManager

### Step 4: TaskSetManager — Re-queue Tasks

`executorLost()` (TaskSetManager.scala:1143) does two things:

**A) Running tasks on dead executor → mark failed, re-enqueue:**
- `handleFailedTask(tid, FAILED, ExecutorLostFailure(exitCausedByApp=false))`
- **Critical: failure does NOT count toward stage abort** because `countTowardsTaskFailures=false`
- Task re-added to pending queue via `addPendingTask()`

**B) Completed shuffle map tasks whose output was on dead executor → re-run:**
- `successful(index) = false`, `tasksSuccessful -= 1`, `addPendingTask(index)`
- Only when external shuffle service is NOT enabled

### Step 5: DAGScheduler — Handle Shuffle Output Loss

`handleExecutorLost()` (DAGScheduler.scala:3059):
```scala
val fileLost = !supportsReliableStorage &&
  (workerHost.isDefined || !externalShuffleServiceEnabled)
if (fileLost) {
  mapOutputTracker.removeOutputsOnExecutor(execId)  // null out outputs
  mapOutputTracker.incrementEpoch()                   // bump epoch
}
```

### Step 6: Downstream Tasks Hit FetchFailed

Reducer tries to fetch from dead executor → output location is null → FetchFailed:
- `DAGScheduler.handleTaskCompletion(FetchFailed)`
- `failedStages += mapStage` (producer: must re-run)
- `failedStages += failedStage` (consumer: must re-run)
- Schedule `ResubmitFailedStages` after ~200ms delay

### Step 7: Stages Resubmitted

- `submitStage(mapStage)` — re-runs only lost partitions
- `submitStage(reduceStage)` — waits for map stage, then re-runs
- Tasks scheduled on remaining live executors (or new ones from cluster manager)

## Timeline

```
t=0   Task hits OOM → System.exit() → executor JVM dies
t=1   Netty detects TCP drop → DriverEndpoint.onDisconnected()
t=2   removeExecutor() → TaskSchedulerImpl.executorLost()
        ├─ TaskSetManager: re-queue running tasks (no failure count)
        ├─ TaskSetManager: invalidate completed shuffle tasks on dead executor
        └─ backend.reviveOffers() → re-schedule on live executors
t=3   DAGScheduler.handleExecutorLost()
        └─ mapOutputTracker.removeOutputsOnExecutor() → outputs nulled
t=4   Downstream reducer hits FetchFailed
        └─ DAGScheduler: mark both stages failed, schedule resubmit
t=5   ResubmitFailedStages → map stage re-runs lost partitions
t=6   Map stage completes → reduce stage re-runs → job continues
```

## Key Nuances

| Question | Answer |
|---|---|
| Does OOM count as task failure? | **No.** `ExecutorLostFailure(exitCausedByApp=false)` — doesn't count toward `maxTaskFailures` (default 4) |
| Does the same task get retried? | **Yes**, re-enqueued on a different executor |
| Completed tasks on dead executor? | Non-shuffle results already sent to driver are safe. Shuffle outputs are **lost** (unless ESS enabled) |
| External shuffle service difference? | With ESS: `fileLost=false`, shuffle data survives. Without ESS: all shuffle outputs gone, map stage must re-run |
| How many stage retries? | `spark.stage.maxConsecutiveAttempts` (default 4). After that, job aborts |
| Replacement executor? | Cluster manager (YARN/K8s) typically launches a replacement |

## Key Source Files

- `executor/Executor.scala` — TaskRunner.run(), isFatalError()
- `util/SparkUncaughtExceptionHandler.scala` — OOM → System.exit()
- `scheduler/cluster/CoarseGrainedSchedulerBackend.scala` — onDisconnected(), removeExecutor()
- `scheduler/TaskSchedulerImpl.scala` — executorLost(), removeExecutor()
- `scheduler/TaskSetManager.scala` — executorLost(), handleFailedTask(), addPendingTask()
- `scheduler/DAGScheduler.scala` — handleExecutorLost(), handleTaskCompletion(FetchFailed), ResubmitFailedStages
- `MapOutputTracker.scala` — removeOutputsOnExecutor(), incrementEpoch()
- `scheduler/TaskEndReason.scala` — ExecutorLostFailure, countTowardsTaskFailures
