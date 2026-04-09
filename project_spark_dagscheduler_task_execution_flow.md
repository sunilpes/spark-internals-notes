---
name: DAGScheduler and CoarseGrainedSchedulerBackend Task Execution Flow
description: End-to-end flow from physical plan execution through DAGScheduler stage/task creation, TaskSchedulerImpl scheduling, CoarseGrainedSchedulerBackend resource offers, to executor-side TaskRunner execution and result return
type: project
tags: [spark, scheduler, dagscheduler, executor, task-execution]
---

## End-to-End: From Query Planning to Task Execution

### Phase 1: Physical Plan → RDD

When you call an action like `df.collect()`:

```
QueryExecution.executedPlan    →  SparkPlan (physical plan tree)
SparkPlan.execute()            →  RDD[InternalRow]  (each operator's doExecute())
SQLExecution.withNewExecutionId() wraps the action, assigns executionId
```

Each physical operator overrides `doExecute()` to build an RDD by transforming its child's RDD. The RDD action (e.g., `collect()`) calls **`SparkContext.runJob()`** — the bridge from SQL to core scheduler.

### Phase 2: SparkContext → DAGScheduler

`SparkContext.runJob()` (SparkContext.scala:2481):
1. Cleans the closure
2. Calls `dagScheduler.runJob()`, which **blocks** until job finishes

```scala
val waiter = submitJob(rdd, func, partitions, ...)
ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)  // blocks
```

`submitJob()` posts a **JobSubmitted** event to DAGScheduler's single-threaded event loop, returns a `JobWaiter`.

### Phase 3: DAGScheduler — Stage Creation

`handleJobSubmitted()` (DAGScheduler.scala:1362):

**Step 1 — Create stages by walking RDD lineage backwards:**
- `createResultStage(finalRDD)` → finds shuffle boundaries → creates ShuffleMapStages for parents
- Every `ShuffleDependency` = stage boundary
- Final action = `ResultStage`; before shuffle = `ShuffleMapStage`

**Step 2 — Submit stages recursively via `submitStage()`:**
- `getMissingParentStages(stage)` — find unfinished parents
- If no missing parents → `submitMissingTasks(stage)` (leaf stage, submit now)
- If missing parents → submit parents first, add current to `waitingStages`
- Bottom-up: leaf stages run first; child stages wait

### Phase 4: DAGScheduler — Task Creation

`submitMissingTasks()` (DAGScheduler.scala:1597):

1. Find partitions to compute: `stage.findMissingPartitions()`
2. Serialize RDD + function as **broadcast variable**: `sc.broadcast(serialize(stage.rdd, stage.shuffleDep))`
3. Create Task objects per partition:
   - `ShuffleMapStage` → `ShuffleMapTask` per partition
   - `ResultStage` → `ResultTask` per partition
4. Submit: `taskScheduler.submitTasks(new TaskSet(tasks))`

**Key**: RDD serialized once as broadcast; each Task references the broadcast, not a copy.

### Phase 5: TaskSchedulerImpl — Scheduling

`submitTasks()` (TaskSchedulerImpl.scala:243):

```scala
val manager = createTaskSetManager(taskSet, maxTaskFailures)
schedulableBuilder.addTaskSetManager(manager, ...)  // FIFO or Fair pool
backend.reviveOffers()                               // trigger resource allocation
```

**TaskSetManager** (TaskSetManager.scala) handles:
- Pending task queues by **locality level** (PROCESS_LOCAL → NODE_LOCAL → RACK_LOCAL → ANY)
- **Delay scheduling** — waits before relaxing locality
- **Speculative execution** — re-launching slow tasks
- Failure tracking per task (up to `maxTaskFailures`)

### Phase 6: CoarseGrainedSchedulerBackend — Resource Offers

`reviveOffers()` → sends `ReviveOffers` to **DriverEndpoint** RPC endpoint.

`makeOffers()` (CoarseGrainedSchedulerBackend.scala:376):
```scala
val workOffers = activeExecutors.map(buildWorkerOffer)  // free cores per executor
val taskDescs = scheduler.resourceOffers(workOffers)     // match tasks to executors
launchTasks(taskDescs)                                    // send to executors
```

`resourceOffers()` in TaskSchedulerImpl:
1. Shuffles offers randomly (avoid hot-spotting)
2. Iterates TaskSetManagers in scheduling order
3. TaskSetManager picks best task per executor based on data locality

`launchTasks()` (line 430):
```scala
executorData.freeCores -= task.cpus
executorData.executorEndpoint.send(LaunchTask(serializedTask))  // RPC to executor
```

### Phase 7: Executor — Task Execution

**CoarseGrainedExecutorBackend** receives `LaunchTask`:
```scala
val taskDesc = TaskDescription.decode(data.value)
executor.launchTask(this, taskDesc)
```

**Executor.launchTask()** (Executor.scala:550):
```scala
val tr = createTaskRunner(context, taskDescription)
runningTasks.put(taskId, tr)
threadPool.execute(tr)  // cached thread pool
```

**TaskRunner.run()** execution sequence:
1. `statusUpdate(RUNNING)` → driver
2. Deserialize Task from broadcast `taskBinary`
3. Create `TaskContext` (metrics, memory manager)
4. Call `task.run()`:
   - **ShuffleMapTask.runTask()**: `rdd.iterator(partition)` → `shuffleWriter.write(iterator)` → returns `MapStatus`
   - **ResultTask.runTask()**: `rdd.iterator(partition)` → `func(context, iterator)` → returns result
5. Serialize result:
   - Small → `DirectTaskResult` (sent inline in RPC)
   - Medium → `IndirectTaskResult` (stored in BlockManager, reference sent)
   - Huge → dropped
6. `statusUpdate(FINISHED, result)` → driver

### Phase 8: Completion — Back to DAGScheduler

Driver receives `StatusUpdate` → `TaskSchedulerImpl.statusUpdate()` → `DAGScheduler.taskEnded()`.

**ShuffleMapTask completion:**
- `mapOutputTracker.registerMapOutput(shuffleId, partitionId, mapStatus)`
- If all partitions done → `submitWaitingChildStages(stage)`

**ResultTask completion:**
- `jobWaiter.taskSucceeded(outputId, result)`
- If all partitions done → `JobWaiter.completionFuture` completes → `sc.runJob()` unblocks → returns to user

### Visual Summary

```
df.collect()
    │
    ▼
SparkPlan.execute() ──→ RDD lineage (lazy)
    │
    ▼
SparkContext.runJob() ──→ DAGScheduler.submitJob()
    │                         │
    │ (blocks on              ▼
    │  JobWaiter)     handleJobSubmitted()
    │                    ├─ createResultStage() ← walks RDD lineage, finds shuffles
    │                    └─ submitStage()       ← recursive, parents first
    │                         │
    │                         ▼
    │                 submitMissingTasks()
    │                    ├─ broadcast(RDD + func)
    │                    ├─ create ShuffleMapTask / ResultTask per partition
    │                    └─ taskScheduler.submitTasks(TaskSet)
    │                         │
    │                         ▼
    │                 TaskSchedulerImpl
    │                    ├─ create TaskSetManager (locality, speculation, retries)
    │                    └─ backend.reviveOffers()
    │                         │
    │                         ▼
    │                 CoarseGrainedSchedulerBackend.DriverEndpoint
    │                    ├─ makeOffers() → collect free executor cores
    │                    ├─ resourceOffers() → match tasks to executors (locality-aware)
    │                    └─ launchTasks() → send LaunchTask RPC per task
    │                         │
    │                         ▼  (network)
    │                 CoarseGrainedExecutorBackend.receive(LaunchTask)
    │                    └─ executor.launchTask() → threadPool.execute(TaskRunner)
    │                         │
    │                         ▼
    │                 TaskRunner.run()
    │                    ├─ deserialize task from broadcast
    │                    ├─ task.run() → rdd.iterator(partition) → compute
    │                    ├─ serialize result
    │                    └─ statusUpdate(FINISHED) → driver
    │                         │
    │                         ▼
    │                 DAGScheduler.handleTaskCompletion()
    │                    ├─ ShuffleMap: register output, submit child stages
    │                    └─ Result: jobWaiter.taskSucceeded()
    │                         │
    └─────────────────────────┘  (JobWaiter completes → runJob returns)
```

### Key Source Files

- `sql/core/.../execution/QueryExecution.scala` — executedPlan, RDD creation
- `sql/core/.../execution/SparkPlan.scala` — execute()/doExecute()
- `sql/core/.../execution/SQLExecution.scala` — withNewExecutionId wrapper
- `core/.../SparkContext.scala` — runJob() entry point
- `core/.../scheduler/DAGScheduler.scala` — stage creation, task submission, completion handling
- `core/.../scheduler/TaskSchedulerImpl.scala` — submitTasks, resourceOffers
- `core/.../scheduler/TaskSetManager.scala` — locality scheduling, speculation, failure tracking
- `core/.../scheduler/cluster/CoarseGrainedSchedulerBackend.scala` — makeOffers, launchTasks
- `core/.../executor/CoarseGrainedExecutorBackend.scala` — LaunchTask reception
- `core/.../executor/Executor.scala` — TaskRunner, thread pool execution
- `core/.../scheduler/Task.scala` — run() template method
- `core/.../scheduler/ShuffleMapTask.scala` — runTask() with shuffle write
- `core/.../scheduler/ResultTask.scala` — runTask() with user function
