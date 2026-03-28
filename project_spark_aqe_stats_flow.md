---
name: Spark AQE Stats Collection Flow
description: How AQE gets shuffle statistics from executors to driver — MapStatus, MapOutputTracker, MapOutputStatistics, re-optimization flow, and what AQE decides based on runtime stats
type: project
---

## How AQE Gets Shuffle Stats from Executors to Driver

### The Complete Flow

```
EXECUTOR                           NETWORK (RPC)                    DRIVER
────────                           ─────────────                    ──────

ShuffleMapTask runs
  ↓
ShuffleWriter writes
shuffle output to disk
  ↓
Creates MapStatus
(compressed sizes per             ───────────────────→    MapOutputTrackerMaster
 reduce partition)                  StatusUpdate RPC       .addMapOutput(mapIndex, status)
                                                             ↓
                                                          ShuffleStatus.mapStatuses[i] = status
                                                             ↓
                                                          All tasks done?
                                                             ↓ YES
                                                          DAGScheduler
                                                          .markMapStageJobsAsFinished()
                                                             ↓
                                                          mapOutputTracker.getStatistics()
                                                          → aggregate all MapStatus
                                                          → MapOutputStatistics(shuffleId,
                                                               bytesByPartitionId: Array[Long])
                                                             ↓
                                                          ShuffleQueryStageExec
                                                          .resultOption = MapOutputStatistics
                                                             ↓
                                                          AdaptiveSparkPlanExec
                                                          .reOptimize(logicalPlan)
                                                             ↓
                                                          New physical plan!
```

### Step by Step

**Step 1: Executor creates MapStatus** (`MapStatus.scala:31`)

After a `ShuffleMapTask` finishes writing shuffle data, it creates a `MapStatus` — a compressed representation of output sizes:

```scala
// Two implementations:
CompressedMapStatus(location: BlockManagerId, compressedSizes: Array[Byte])
  → stores each partition size as 1 byte (log-compressed)

HighlyCompressedMapStatus(location, avgSize, emptyBlocks, hugeBlockSizes)
  → for large partition counts: avg + bitmap of empties + outliers
```

**Step 2: MapStatus sent to driver via RPC**

The executor sends `StatusUpdate` → DAGScheduler receives it → calls `MapOutputTrackerMaster.addMapOutput()`:

```
MapOutputTrackerMaster
└── shuffleStatuses: Map[Int, ShuffleStatus]
    └── ShuffleStatus
        └── mapStatuses: Array[MapStatus]   ← one slot per map task
            [0] = MapStatus from task 0
            [1] = MapStatus from task 1
            [2] = MapStatus from task 2
            ...
```

**Step 3: All tasks complete → aggregate into MapOutputStatistics**

`DAGScheduler.markMapStageJobsAsFinished()` (line 3041):

```scala
val stats = mapOutputTracker.getStatistics(shuffleStage.shuffleDep)
```

`getStatistics()` (`MapOutputTracker.scala:1013`) does:

```
For each reduce partition i:
  totalSizes[i] = sum of mapStatus.getSizeForBlock(i) across ALL map tasks

Returns: MapOutputStatistics(shuffleId, totalSizes)
```

Example with 3 mappers, 4 reduce partitions:
```
MapStatus[0]: [10MB, 5MB, 0MB, 2MB]    (mapper 0's output per reduce partition)
MapStatus[1]: [8MB,  3MB, 1MB, 4MB]    (mapper 1's output)
MapStatus[2]: [12MB, 2MB, 0MB, 1MB]    (mapper 2's output)
                ↓      ↓    ↓    ↓
totalSizes:   [30MB, 10MB, 1MB, 7MB]   ← this is what AQE sees
```

**Step 4: AQE receives stats and re-optimizes**

`AdaptiveSparkPlanExec` (`AdaptiveSparkPlanExec.scala:325`):

```scala
case StageSuccess(stage, res) =>
  stage.resultOption.set(Some(res))   // res = MapOutputStatistics
  // ...
  val afterReOptimize = reOptimize(logicalPlan)  // RE-PLAN!
```

**Step 5: Optimization rules read the stats**

```scala
// CoalesceShufflePartitions reads actual partition sizes:
shuffleStage.mapStats.map(_.bytesByPartitionId)
// → [30MB, 10MB, 1MB, 7MB]
// "Partitions 2 and 3 are tiny, merge them into one!"

// JoinSelection with runtime stats:
plan.stats.isRuntime = true   // signals these are actual, not estimated
// "This side is only 5MB at runtime, switch to BroadcastHashJoin!"
```

### What AQE Decides Based on These Stats

| Stats show | AQE action |
|-----------|------------|
| One join side is actually small | **SortMergeJoin → BroadcastHashJoin** |
| Many tiny partitions | **Coalesce** them into fewer, larger partitions |
| One partition is 10x bigger than others | **Split** the skewed partition |
| A side produces 0 rows | **Short-circuit** with empty relation |

### Key Insight: Everything Happens on the Driver

- **Executors** only report compressed `MapStatus` (a few bytes per task)
- **Driver** aggregates all statuses into `MapOutputStatistics`
- **Driver** re-optimizes the remaining plan
- **Executors** never know about AQE — they just receive new tasks from the updated plan

The data itself stays on executors. Only **metadata about sizes** flows to the driver.

### Key Source Files

| Component | File | Key Lines |
|-----------|------|-----------|
| MapOutputStatistics | `core/.../MapOutputStatistics.scala` | 20-27 |
| MapStatus (compressed) | `core/.../scheduler/MapStatus.scala` | 31-263 |
| MapOutputTrackerMaster | `core/.../MapOutputTracker.scala` | 58-193 |
| getStatistics() | `core/.../MapOutputTracker.scala` | 1013-1044 |
| submitMapStage | `core/.../scheduler/DAGScheduler.scala` | 1084-1113 |
| markMapStageJobsAsFinished | `core/.../scheduler/DAGScheduler.scala` | 3041-3049 |
| ShuffleQueryStageExec.mapStats | `sql/core/.../adaptive/QueryStageExec.scala` | 231-235 |
| AdaptiveSparkPlanExec | `sql/core/.../adaptive/AdaptiveSparkPlanExec.scala` | 268-384 |
| reOptimize | `sql/core/.../adaptive/AdaptiveSparkPlanExec.scala` | 793-823 |
