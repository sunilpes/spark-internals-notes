---
name: Spark Query Planning Caveats
description: Key limitations and pitfalls of Spark's query planning — CBO off by default, single plan selection, join order, UDF pushdown blocking, Python codegen breaks, no indexes, broadcast threshold blindspot, stale stats, AQE limits
type: project
tags: [spark-sql, spark-optimization, spark-aqe]
---

## Spark Query Planning Caveats

### 1. CBO Is Off by Default — Stats Are Often Wrong
Default uses only file size (sizeInBytes). Filter on 1% of rows? Spark still thinks output is full file size → wrong join strategy.

### 2. Single Physical Plan — No Cost-Based Plan Selection
```scala
// QueryPlanner.scala: "TODO: ONLY ONE PLAN IS RETURNED EVER..."
```
Picks first matching strategy, doesn't generate multiple plans and compare costs. `prunePlans()` is a no-op.

### 3. Join Order Depends on How You Write the Query
Without CBO join reorder (off by default), Spark joins in the order you wrote them. Manual ordering or CBO needed.

### 4. Predicate Pushdown Doesn't Always Happen
- UDFs block pushdown: `WHERE my_udf(age) > 25` → reads ALL rows
- Complex expressions: `WHERE age + 10 > 35` → not rearranged to `age > 25`

### 5. Column Pruning Breaks With SELECT *
`SELECT *` propagates all columns through the plan. Columnar formats lose their biggest advantage.

### 6. Python UDFs Break Codegen Pipeline
Every Python UDF inserts a codegen boundary + socket serialization overhead. Chaining multiplies the cost.

### 7. Optimizer Rules Can Conflict
Rules in fixed-point loops can undo each other's work → wasted iterations until maxIterations (100) limit.

### 8. AQE Can't Fix Everything
Only re-optimizes at shuffle boundaries. No shuffle = no AQE. First stage always uses static estimates.

### 9. Broadcast Threshold Is Blind
`autoBroadcastJoinThreshold = 10MB` is file size on disk. Compressed Parquet at 10MB might decompress to 500MB → OOM during broadcast.

### 10. No Index Support
Every query is a full scan with filter pushdown at best. Data skipping (Delta Z-ordering, Iceberg hidden partitioning) are workarounds, not true indexes.

### 11. Subquery Performance Trap
Correlated subqueries can execute as separate jobs per row. IN/EXISTS rewritten to semi join, but scalar subqueries can be expensive.

### 12. Skew Isn't Detected Until Runtime
No compile-time detection. Only after shuffle completes (expensive write already done).

### 13. Statistics Become Stale
No automatic stats refresh. Old ANALYZE TABLE stats → terrible CBO decisions on grown tables.

### 14. Hints Are All-or-Nothing
`/*+ BROADCAST(users) */` overrides planner entirely. If table grows, hint still forces broadcast → OOM.

### Summary

| Caveat | Impact | Mitigation |
|--------|--------|-----------|
| CBO off by default | Wrong join strategies | Enable CBO or rely on AQE |
| Single plan (no exploration) | Possibly suboptimal | Manual hints |
| Write-order joins | Bad join ordering | CBO join reorder or manual |
| UDFs block pushdown | Full table scans | Use native Spark expressions |
| Python UDFs break codegen | Performance cliff | Use Pandas UDFs or Scala |
| No indexes | Full scans always | Partition pruning, Z-ordering |
| Broadcast threshold = disk size | OOM on decompression | Conservative threshold |
| Stale statistics | Wrong decisions | Re-run ANALYZE or use AQE |
| AQE only at shuffle boundaries | First stage unoptimized | Accurate initial stats |

## Related Notes

- [[project_spark_cbo_explained]] — CBO off by default (Caveat 1)
- [[project_spark_optimizer_rule_execution]] — Rule conflicts (Caveat 7)
- [[project_spark_udf_physical_planning]] — UDF pushdown blocking, codegen breaks (Caveats 4, 6)
- [[project_whole_stage_codegen_explained]] — Codegen fallbacks (Caveat 6)
- [[project_spark_aqe_stats_flow]] — AQE limitations at shuffle boundaries (Caveat 8)
- [[project_spark_join_strategy_selection]] — Broadcast threshold blindspot (Caveat 9)
- [[project_spark_format_aware_optimization]] — Format-aware mitigations for full scans
