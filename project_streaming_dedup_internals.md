---
name: Streaming dropDuplicates Internals
description: How dropDuplicates works in Spark Structured Streaming — stateful (state store) path vs batch (foreachBatch) path, key source files, and execution flow
type: project
tags: [spark-sql, spark-execution]
---

## Streaming `dropDuplicates` — Two Distinct Code Paths

### Path 1: Native Streaming (State Store Based)

**API entry point**: `sql/core/src/main/scala/org/apache/spark/sql/classic/Dataset.scala:1355`
- `dropDuplicates(colNames)` creates a `Deduplicate(groupCols, logicalPlan)` logical node

**Logical plan**: `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/plans/logical/basicLogicalOperators.scala:2127`
- `case class Deduplicate(keys: Seq[Attribute], child: LogicalPlan)`
- `case class DeduplicateWithinWatermark(keys, child)` at line 2137

**Strategy (logical → physical)**: `sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala:511`
- `StreamingDeduplicationStrategy` matches `Deduplicate` when `child.isStreaming`
- Converts to `StreamingDeduplicateExec` or `StreamingDeduplicateWithinWatermarkExec`

**Physical execution**: `sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/operators/stateful/statefulOperators.scala:1310`
- `BaseStreamingDeduplicateExec.doExecute()` (line 1328): uses `mapPartitionsWithStateStore`
- Core logic (line 1356-1370): sequential `filter` over partition iterator
  - For each row: project dedup key → `store.keyExists(key)`
  - If key NOT exists: emit row + `store.put(key, value)` → deduplicates within AND across batches
  - If key exists: drop the row
- After processing: `evictDupInfoFromState(store)` then `store.commit()`

**`StreamingDeduplicateExec`** (line 1407): stores `EMPTY_ROW` (dummy null) as value — only key existence matters. Eviction via `removeKeysOlderThanWatermark`.

**`StreamingDeduplicateWithinWatermarkExec`** (line 1464): stores `expiresAtMicros = event_time + delayThreshold`. Eviction scans all entries, removes where `watermark >= expiresAt`. Allows same key to reappear after expiration.

**Key behavior**: Within-batch dedup works because `store.put()` is called immediately during iteration, so subsequent rows with the same key see `keyExists = true`.

### Path 2: Inside `foreachBatch` (Batch DataFrame)

When `dropDuplicates` is called on the batch DataFrame inside `foreachBatch`, `child.isStreaming = false`.

**Optimizer rule**: `ReplaceDeduplicateWithAggregate` at `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala:2641`
- Rewrites `Deduplicate(keys, child)` into `Aggregate(keys, [keys + First(non-key-cols)], child)`
- Non-key columns use `First()` aggregate — which row survives is non-deterministic
- Executed via `HashAggregate` or `SortAggregate` — no state store involved

**Critical implication**: foreachBatch dedup only works within a single micro-batch. No cross-batch dedup. Same record arriving in different micro-batches will NOT be deduplicated.

### Comparison Table

| Aspect | Streaming `dropDuplicates` | `foreachBatch` + `dropDuplicates` |
|--------|---------------------------|-----------------------------------|
| Physical plan | `StreamingDeduplicateExec` | `HashAggregate` / `SortAggregate` |
| Mechanism | State store key lookup | Hash-based GROUP BY |
| Cross-batch dedup | Yes | No |
| Within-batch dedup | Yes | Yes |
| State growth | Unbounded without watermark | None |
| Surviving row | First encountered in partition | Non-deterministic (First()) |

## Related Notes

- [[project_spark_sql_architecture]] — Query execution pipeline context
- [[project_sparksession_internals]] — StreamingQueryManager lives in SessionState
- [[project_spark_sql_planning_explained]] — How Deduplicate becomes StreamingDeduplicateExec
