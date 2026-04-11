---

name: Interesting Facts About SQL Query Planning and Execution
description: "15 non-obvious facts about Spark SQL planning: no plan comparison, top-down planning, 5 tree transformations, 80+ rules in 27 batches, broadcast threshold blindspot, Python UDF codegen break, spill behavior, AQE mid-query changes, HashedRelation optimization, runtime codegen, predicate pushdown at 3 levels, ANALYZE TABLE storage, Exchange stage boundaries, broadcast serialization"
type: project
tags: [spark, query-planning, codegen, optimizer, aqe, joins, broadcast, interesting-facts]
---

Here are some non-obvious facts about Spark SQL planning and execution, drawn from what we've explored in the source code:

---

### 1. Spark Never Compares Alternative Plans

Unlike traditional databases (Postgres, Oracle) that generate multiple candidate plans and pick the cheapest, Spark's `QueryPlanner.plan()` takes the **first plan** each Strategy returns. If `JoinSelection` says "BroadcastHashJoin" — that's it. No cost comparison, no plan enumeration.

The only exception: CBO's `CostBasedJoinReorder` (disabled by default) uses dynamic programming to reorder multi-way joins — but even that only reorders, it doesn't compare different physical operators.

### 2. The Physical Plan is Built Top-Down, Not Bottom-Up

`QueryPlanner` starts at the **root** of the logical plan and recursively plans downward. When it encounters a child it can't plan yet, it inserts a `PlanLater` placeholder. After all strategies run, it replaces PlanLaters with actual physical nodes. This is the opposite of how most textbooks describe query compilation.

### 3. Your SQL String Becomes 5 Different Trees

A single query like `SELECT * FROM t WHERE x > 5` passes through:

```
1. ANTLR Parse Tree (CST)     — grammar-level, every token
2. Unresolved LogicalPlan      — AstBuilder output, names not resolved
3. Analyzed LogicalPlan        — Analyzer resolves names, types
4. Optimized LogicalPlan       — Optimizer applies 80+ rules
5. Physical SparkPlan          — concrete operators with doExecute()
```

Each is a completely separate immutable tree. Transformations create new trees — the old ones are never mutated.

### 4. The Optimizer Runs 80+ Rules in 27 Batches, Some Repeatedly

`RuleExecutor.execute()` processes rules in **batches**. Some batches use `FixedPoint` strategy — they loop until the plan stops changing (or hit `maxIterations=100`). A single query can have hundreds of rule applications before the plan stabilizes.

The `transformWithPruning` optimization skips subtrees that can't possibly match a rule's pattern, using TreeNode bit tags — without this, optimizer would be much slower.

### 5. Broadcast Threshold Uses Disk Size, Not Memory Size

`spark.sql.autoBroadcastJoinThreshold` (10MB) compares against `stats.sizeInBytes` — which is the **serialized, compressed** size on disk. After deserialization, the data in memory can be **10-100x larger** (especially with string-heavy data). This is why broadcast joins sometimes OOM despite the table being "only 10MB".

### 6. Python UDFs Break the Codegen Pipeline

WholeStageCodegenExec fuses operators into a single Java method using the produce/consume protocol. But Python UDFs get extracted into a separate `BatchEvalPythonExec` node with an `InputAdapter` boundary — **breaking the codegen chain** into two separate generated methods. Each break adds overhead (materialization of intermediate rows).

Scala UDFs don't have this problem — they're inlined directly into the generated code.

### 7. Sort Merge Join Can Spill to Disk; Broadcast Hash Join Cannot

`SortMergeJoinExec` uses `ExternalAppendOnlyUnsafeRowArray` to buffer matching rows — if too many matches for a key, it spills to disk. This makes it **memory-safe** for any data size.

`BroadcastHashJoinExec` builds the entire `HashedRelation` in memory. If the build side is larger than expected (stale stats, post-filter expansion), it's OOM with no fallback.

### 8. AQE Can Change the Join Strategy Mid-Query

`AdaptiveSparkPlanExec` runs stages bottom-up. After a shuffle stage completes, it collects real `MapOutputStatistics` (actual partition sizes) and re-optimizes the remaining plan. A query planned as SortMergeJoin can **switch to BroadcastHashJoin** at runtime if the shuffle reveals one side is actually small.

This happens in `DynamicJoinSelection` — it can also **demote** broadcast joins if it detects many empty partitions (broadcast would waste time).

### 9. HashedRelation Has Two Implementations for Different Key Types

- **Single `Long` key** → `LongHashedRelation` using `LongToUnsafeRowMap` with **dense mode** (direct array indexing when key range is small) or **sparse mode** (quadratic probing hash table)
- **Everything else** → `UnsafeHashedRelation` using `BytesToBytesMap` (off-heap, Unsafe API)

The Long optimization makes joins on integer/long keys significantly faster — no hashing overhead, just array[key - minKey].

### 10. Codegen Generates a Literal Java Class at Runtime

`WholeStageCodegenExec` produces actual Java source code (as a string), compiles it with Janino, and runs it. You can see the generated code with:
```scala
df.queryExecution.debug.codegen
```

A typical generated class has a `processNext()` method that fuses Filter + Project + Scan into a tight loop with no virtual dispatch — similar to hand-written C performance.

### 11. The "Broadcast" in BroadcastHashJoin is a Spark Broadcast Variable

The small table is collected to the driver, wrapped in a `Broadcast[HashedRelation]`, and sent to all executors via the TorrentBroadcast protocol (chunked, peer-to-peer distribution). The hash table is built **once on the driver** and shared read-only across all executor tasks.

### 12. Predicate Pushdown Happens at Three Different Levels

1. **Logical**: `PushDownPredicates` rule pushes Filter below Join/Project
2. **Physical**: `FileSourceStrategy` pushes filters into `FileSourceScanExec` as data filters
3. **Format-level**: Parquet/ORC readers use pushed predicates to skip entire row groups via min/max statistics in file footer

A filter like `WHERE year = 2024` can skip 99% of data before any rows are deserialized — but only if partition pruning or row group stats allow it.

### 13. ANALYZE TABLE Statistics Are Stored in the Hive Metastore

`ANALYZE TABLE ... COMPUTE STATISTICS` computes row count, total size, and per-column stats (min, max, NDV, null count, avg length). These are persisted in the **Hive metastore** catalog. Without them, the optimizer falls back to `defaultSizeInBytes` — a hardcoded guess that leads to poor join strategy selection.

For JDBC/external sources, **no statistics are available at all** — CBO is effectively blind. AQE is the practical workaround.

### 14. Exchange Nodes Are the Stage Boundaries

Every `ShuffleExchangeExec` in the physical plan becomes a **stage boundary** when DAGScheduler converts the plan to stages. The `EnsureRequirements` post-processing rule inserts these Exchanges automatically when a child's output partitioning doesn't match the parent's requirements (e.g., SortMergeJoin needs both sides hash-partitioned by join key).

### 15. The Entire Plan Is Serialized as a Broadcast Variable

When `DAGScheduler.submitMissingTasks()` creates tasks, the RDD + function are serialized **once** as a broadcast variable (`taskBinary`). Each `ShuffleMapTask` or `ResultTask` just carries a reference to this broadcast — not a copy. This avoids sending the plan N times for N tasks.
