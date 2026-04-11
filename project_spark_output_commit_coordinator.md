---

name: OutputCommitCoordinator Atomic Commit Protocol
description: "How OutputCommitCoordinator ensures atomic output writes: first-committer-wins policy, two-phase commit (task+job), speculative/retry/zombie handling, StageState tracking, RPC coordination, HadoopMapReduceCommitProtocol integration"
type: project
tags: [spark, output-commit, atomicity, speculation, fault-tolerance, hdfs]
---

## OutputCommitCoordinator — Atomic Output Commit

### Problem
With speculative execution or task retries, multiple attempts of the same task run simultaneously. Without coordination, both write output → duplicate/corrupt data. Coordinator enforces "first committer wins".

### Core Data Structure

```scala
private case class StageState(numPartitions: Int) {
  val authorizedCommitters = Array.fill[TaskIdentifier](numPartitions)(null)  // one slot per partition
  val failures = mutable.Map[Int, mutable.Set[TaskIdentifier]]()              // blacklisted attempts
}
private case class TaskIdentifier(stageAttempt: Int, taskAttempt: Int)
```

### Two-Phase Commit Protocol

**Phase 1 — Task Commit (executor):**
```
FileFormatDataWriter.commit()
  → HadoopMapReduceCommitProtocol.commitTask()
    → SparkHadoopMapRedUtil.commitTask()
      → outputCommitCoordinator.canCommit() → RPC to driver
        → handleAskPermissionToCommit()
```
Each task writes to STAGING dir, then asks coordinator. If YES → move to task-committed area. If NO → abort, delete.

**Phase 2 — Job Commit (driver):**
After ALL tasks succeed:
```
committer.commitJob() → rename files from staging to final destination → delete staging
```
If any failure: `committer.abortJob()` rolls back everything.

### Authorization Logic (handleAskPermissionToCommit)

Three rules evaluated in order:
1. This attempt in `failures` set? → **DENY** (zombie prevention)
2. Partition slot is `null` (open)? → **AUTHORIZE** (first wins), set slot
3. Slot already taken? → **DENY**

All operations `synchronized` on coordinator instance.

### Failure Handling

**Authorized committer dies:**
- `taskCompleted()` called by DAGScheduler
- Clears lock: `authorizedCommitters[partition] = null`
- Blacklists: `failures[partition].add(taskId)`
- Next retry attempt can now be authorized

**Speculative execution:**
- First attempt to ask wins, second gets `CommitDeniedException`
- Denied task aborts its output files

**Zombie attempt (failed then comes back):**
- `attemptFailed()` check catches it — attempt is in failures set
- Denied even though slot might be open

### Atomicity Guarantees

| Level | Mechanism |
|---|---|
| Per-partition | Only one task attempt can commit — coordinator lock |
| Per-job | commitJob() moves all files atomically; abortJob() on failure |
| Against zombies | Failed attempts blacklisted, can never re-authorize |
| Thread safety | All methods synchronized on coordinator |
| No distributed consensus | Single driver decides via serialized RPC |

### DAGScheduler Integration

- `stageStart()` — called when stage starts (DAGScheduler:1641,1657), creates StageState
- `taskCompleted()` — called on task end (DAGScheduler:2176), tracks failures, clears locks
- `stageEnd()` — called on final stage completion (DAGScheduler:3261), removes state

### Key Source Files

- `core/.../scheduler/OutputCommitCoordinator.scala` — core logic
- `core/.../mapred/SparkHadoopMapRedUtil.scala:78` — canCommit() caller
- `core/.../io/HadoopMapReduceCommitProtocol.scala` — commitTask()/commitJob() file moves
- `sql/.../datasources/FileFormatWriter.scala:262` — setupJob → tasks → commitJob
- `sql/.../datasources/FileFormatDataWriter.scala:123` — per-task write + commit
- `sql/.../datasources/v2/WriteToDataSourceV2Exec.scala:537` — V2 canCommit call

### Config

`spark.hadoop.outputCommitCoordination.enabled = true` (default). When disabled, all attempts commit directly — unsafe with speculation.
