---
name: Spark AQE Stage Splitting and Re-optimization
description: How AQE splits physical plan into stages at Exchange boundaries, the submit-wait-reoptimize loop, LogicalQueryStage with runtime stats, and concrete walkthrough of join strategy change at runtime
type: project
---

## How the Plan is Split into Stages and Which Parts AQE Re-optimizes

### Key Insight: Exchange Nodes = Stage Boundaries

Spark doesn't split at the **logical plan** level. It splits at the **physical plan** level — every `ShuffleExchange` or `BroadcastExchange` node becomes a stage boundary.

```
Physical Plan:
Project [name]
└── SortMergeJoin [id = user_id]
    ├── ShuffleExchange (HashPartitioning(id))     ← STAGE BOUNDARY
    │   └── Filter (age > 25)
    │       └── FileScan users
    └── ShuffleExchange (HashPartitioning(user_id)) ← STAGE BOUNDARY
        └── FileScan orders
```

AQE splits this into:

```
Stage 0: Filter + FileScan users → ShuffleWrite
Stage 1: FileScan orders → ShuffleWrite
Stage 2: ShuffleRead + ShuffleRead → SortMergeJoin → Project  (waits for 0 & 1)
```

### How AQE Creates Stages — Bottom-Up

`createNonResultQueryStages()` (`AdaptiveSparkPlanExec.scala:586`) walks the plan tree **bottom-up**:

```scala
plan match {
  case exchange: Exchange =>
    val childResult = createNonResultQueryStages(exchange.child)
    if (childResult.allChildStagesMaterialized) {
      val stage = newQueryStage(exchange)  // CREATE this stage
    } else {
      exchange  // Children not ready → can't create, wait
    }
  case other =>
    other.mapChildren(createNonResultQueryStages)  // recurse
}
```

**The rule**: A stage is created **only when all its child stages are already materialized**.

### The Main Loop — Submit → Wait → Re-optimize → Repeat

`withFinalPlanUpdate()` (`AdaptiveSparkPlanExec.scala:268`):

```
while (!allStagesMaterialized) {

  1. CREATE stages whose children are ready
     createQueryStages(plan)

  2. SUBMIT new stages for async execution
     stage.materialize() → returns Future
     (broadcasts submitted first for overlap)

  3. WAIT for at least one stage to complete
     events.take() ← blocks here
     → StageSuccess(stage, MapOutputStatistics)
     → stage.resultOption.set(Some(stats))

  4. RE-OPTIMIZE the plan with new stats
     a) replaceWithQueryStagesInLogicalPlan()
        → completed stages become LogicalQueryStage nodes
        → these carry REAL runtime stats
     b) reOptimize(logicalPlan)
        → optimizer runs again with runtime stats
     c) re-plan to physical
     d) adopt new plan only if cost improves

  5. LOOP BACK to step 1
}
```

### Concrete Walkthrough

```sql
SELECT u.name, count(*)
FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.age > 25
GROUP BY u.name
```

```
INITIAL PHYSICAL PLAN:
ResultStage
└── HashAggregate (final)
    └── ShuffleExchange③ (HashPartitioning(name))
        └── HashAggregate (partial)
            └── SortMergeJoin [id = user_id]
                ├── ShuffleExchange① (HashPartitioning(id))
                │   └── Filter (age > 25)
                │       └── FileScan users
                └── ShuffleExchange② (HashPartitioning(user_id))
                    └── FileScan orders
```

**Iteration 1:**

```
createQueryStages() walks bottom-up:
  ShuffleExchange① → children are leaf nodes (materialized) → CREATE Stage 0
  ShuffleExchange② → children are leaf nodes (materialized) → CREATE Stage 1
  ShuffleExchange③ → children include Stage 0 & 1 (NOT materialized) → SKIP

Submit: Stage 0, Stage 1 (run in parallel)
Wait...
  Stage 0 completes: MapOutputStatistics → users after filter = 3 MB
  Stage 1 completes: MapOutputStatistics → orders = 800 MB
```

**RE-OPTIMIZE with runtime stats:**

```
Logical plan now has:
  LogicalQueryStage(users_filtered, stats=3MB)   ← REAL size
  LogicalQueryStage(orders, stats=800MB)          ← REAL size

Optimizer sees: 3MB ≤ 10MB threshold!
  → SortMergeJoin converted to BroadcastHashJoin!

NEW PHYSICAL PLAN:
ResultStage
└── HashAggregate (final)
    └── ShuffleExchange③ (HashPartitioning(name))
        └── HashAggregate (partial)
            └── BroadcastHashJoin [id = user_id]    ← CHANGED!
                ├── BroadcastExchange④               ← NEW!
                │   └── Stage 0 (already done)
                └── Stage 1 (already done)
```

**Iteration 2:**

```
BroadcastExchange④ → child Stage 0 materialized → CREATE Stage 3 (broadcast)
Submit: Stage 3
Wait... Stage 3 completes
```

**Iteration 3:**

```
ShuffleExchange③ → all children materialized → CREATE Stage 4
Submit: Stage 4
Wait... Stage 4 completes → check if partitions need coalescing
```

**Iteration 4:**

```
All stages materialized → create ResultQueryStageExec → done!
```

### How AQE Knows Which Parts to Re-optimize

It doesn't "mark" specific parts. The mechanism is:

1. **Completed stages** get replaced with `LogicalQueryStage` nodes in the logical plan
2. `LogicalQueryStage` provides **real runtime stats** (not estimates)
3. The **entire remaining plan** is re-optimized with these real stats
4. The optimizer naturally finds better strategies where the real stats differ from estimates

```
BEFORE re-optimization:                AFTER re-optimization:
LogicalPlan                            LogicalPlan
├── Join                               ├── Join
│   ├── LogicalQueryStage              │   ├── LogicalQueryStage
│   │   (stats: 3MB ← REAL)           │   │   (stats: 3MB)
│   └── LogicalQueryStage              │   └── LogicalQueryStage
│       (stats: 800MB ← REAL)         │       (stats: 800MB)
└── Aggregate                          └── Aggregate

JoinSelection now sees 3MB → picks BroadcastHashJoin!
```

### Summary

| Question | Answer |
|----------|--------|
| Where is the plan split? | At **Exchange nodes** (shuffle/broadcast) in the physical plan |
| What triggers a stage? | Its **child stages are all materialized** |
| What gets re-optimized? | The **entire remaining logical plan**, with completed stages carrying runtime stats |
| How does AQE know what to change? | It re-runs the optimizer, which **naturally** finds better plans with accurate stats |
| Who drives the loop? | `AdaptiveSparkPlanExec.withFinalPlanUpdate()` — submit → wait → re-optimize → repeat |

### Key Source Files

| Component | File | Key Lines |
|-----------|------|-----------|
| Main loop | `AdaptiveSparkPlanExec.scala` | 268-384 |
| Stage creation | `AdaptiveSparkPlanExec.scala` | 531-574, 586-649 |
| newQueryStage | `AdaptiveSparkPlanExec.scala` | 665-701 |
| reOptimize | `AdaptiveSparkPlanExec.scala` | 793-823 |
| replaceWithQueryStagesInLogicalPlan | `AdaptiveSparkPlanExec.scala` | 759-788 |
| QueryStageExec (base) | `QueryStageExec.scala` | 47-160 |
| ShuffleQueryStageExec | `QueryStageExec.scala` | 198-238 |
| LogicalQueryStage | `LogicalQueryStage.scala` | 38-113 |
| AQEOptimizer | `AQEOptimizer.scala` | 39-47 |
