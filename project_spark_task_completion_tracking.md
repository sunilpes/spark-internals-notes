---
name: Spark Task and Job Completion Tracking
description: How Spark tracks task progress and marks stages/jobs complete — StatusUpdate flow, TaskSetManager counters, DAGScheduler stage tracking, JobWaiter notification, child stage triggering
type: project
---

## How Spark Tracks Task Progress and Marks Completion

### The Full Flow: Executor → Task → Stage → Job

```
Executor finishes task
  ↓
StatusUpdate RPC to driver
  ↓
TaskSchedulerImpl.statusUpdate()
  ↓
TaskResultGetter (deserialize result)
  ↓
TaskSetManager (track per-task state)
  ↓
DAGScheduler.handleTaskCompletion()
  ├── ShuffleMapTask done → update stage.pendingPartitions
  └── ResultTask done → update job.finished[partition]
  ↓
Stage complete? → submit child stages
  ↓
Job complete? → notify caller via JobWaiter.jobPromise
```

### Step 1: Executor → Driver (RPC)

```
Executor sends:
  StatusUpdate(executorId="1", taskId=42, state=FINISHED, data=serializedResult)

CoarseGrainedSchedulerBackend.receive()  [line 172]
  → TaskSchedulerImpl.statusUpdate()     [line 774]
    → TaskResultGetter.enqueueSuccessfulTask()  [line 58]
      → (thread pool deserializes result)
      → TaskSetManager.handleSuccessfulTask()   [line 805]
```

### Step 2: TaskSetManager — Track Individual Tasks

```scala
class TaskSetManager {
  val successful = Array.fill[Boolean](numTasks)(false)  // which partitions done
  var tasksSuccessful = 0                                 // count of successes
  val numFailures = Array.fill[Int](numTasks)(0)         // failures per partition
}
```

On success (`handleSuccessfulTask`, line 805):
```
1. Mark task finished: info.markFinished(FINISHED)
2. Kill duplicate attempts for same partition
3. tasksSuccessful += 1
4. If tasksSuccessful == numTasks → isZombie = true (all done)
5. Notify DAGScheduler → CompletionEvent
```

On failure (`handleFailedTask`, line 944):
```
1. numFailures[partition] += 1
2. If numFailures > maxTaskFailures (default 4) → abort entire stage
3. Otherwise → add back to pending, retry on another executor
```

### Step 3: DAGScheduler — Track Stages

`handleTaskCompletion()` (line 2172):

**ShuffleMapTask completion:**
```
shuffleStage.pendingPartitions -= task.partitionId
mapOutputTracker.registerMapOutput(shuffleId, partition, mapStatus)

If pendingPartitions.isEmpty:
  → markStageAsFinished(shuffleStage)
  → submitWaitingChildStages(shuffleStage)   ← triggers next stage!
```

**ResultTask completion:**
```
job.finished[partitionId] = true
job.numFinished += 1

If job.numFinished == job.numPartitions:
  → markStageAsFinished(resultStage)
  → post SparkListenerJobEnd(JobSucceeded)
  → job.listener.taskSucceeded()             ← notifies caller!
```

### Step 4: Job Completion — Notify Caller

```scala
class JobWaiter {
  val finishedTasks = new AtomicInteger(0)
  val jobPromise: Promise[Unit]

  def taskSucceeded(index, result) = {
    resultHandler(index, result)
    if (finishedTasks.incrementAndGet() == totalTasks)
      jobPromise.success(())                  // UNBLOCKS the caller!
  }
}
```

```
df.collect()
  → DAGScheduler.submitJob() → returns Future from JobWaiter
  → Future blocks waiting...
  → All ResultTasks complete → jobPromise.success()
  → Future completes → collect() returns results
```

### Tracking Structures

```
ActiveJob
  finished[]: Array[Boolean]  ← which result partitions done
  numFinished: Int            ← count of completed
  listener: JobWaiter         ← notifies caller when done

ResultStage
  TaskSetManager
    tasksSuccessful: Int
    successful[]: Boolean[]

ShuffleMapStage
  pendingPartitions: HashSet[Int]  ← shrinks as tasks complete
  TaskSetManager
    tasksSuccessful: Int
    successful[]: Boolean[]
```

### How Child Stages Get Triggered

```
ShuffleMapStage 0 completes (pendingPartitions empty)
  → processShuffleMapStageCompletion()
    → submitWaitingChildStages(stage0)
      → finds Stage 1 was waiting → submitStage(stage1)

All ResultTasks complete
  → job.numFinished == job.numPartitions
  → JobWaiter.jobPromise.success()
  → df.collect() returns!
```

### Task States

```
LAUNCHING → RUNNING → FINISHED (success)
                   → FAILED   (retry up to 4 times)
                   → KILLED   (cancelled)
                   → LOST     (executor died)
```

### Key Source Files

| Component | File | Key Lines |
|-----------|------|-----------|
| StatusUpdate receive | CoarseGrainedSchedulerBackend.scala | 172 |
| statusUpdate | TaskSchedulerImpl.scala | 774 |
| TaskResultGetter | TaskResultGetter.scala | 58, 136 |
| handleSuccessfulTask | TaskSetManager.scala | 805 |
| handleFailedTask | TaskSetManager.scala | 944 |
| handleTaskCompletion | DAGScheduler.scala | 2172 |
| processShuffleMapStageCompletion | DAGScheduler.scala | 2935 |
| markStageAsFinished | DAGScheduler.scala | 3236 |
| JobWaiter | JobWaiter.scala | 30-79 |
| ActiveJob | ActiveJob.scala | 45-66 |
