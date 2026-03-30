---
name: Spark 5-Week Learning Plan
description: Structured 5-week plan to learn Spark internals — core/scheduler/shuffle, catalyst deep dive, physical execution, data sources/streaming, networking/storage/tuning
type: project
tags: [spark-core, spark-sql]
---

## 5-Week Spark Internals Learning Plan

**Started:** 2026-03-24

**Why:** Systematic deep-dive into Spark internals, building on existing knowledge of parsing, planning, and codegen.

**How to apply:** Use this to track progress week by week. Each topic has key source locations for IntelliJ debugging.

### Completed Topics (before this plan)

- [x] TreeNode hierarchy (LeafNode, UnaryNode, BinaryNode)
- [x] LogicalPlan vs SparkPlan
- [x] SQL Parsing pipeline (ANTLR -> AstBuilder -> UnresolvedRelation)
- [x] Planning (Strategies, SparkPlanner, planLater)
- [x] WholeStageCodegenExec (produce/consume)
- [x] Streaming dedup internals

---

### Week 1: Core Foundation (Mar 24 - Mar 30)

- [ ] RDD internals — `core/src/main/scala/org/apache/spark/rdd/` — RDD lineage, partitions, dependencies (narrow vs wide), compute()
- [ ] DAGScheduler — `core/src/main/scala/org/apache/spark/scheduler/DAGScheduler.scala` — How RDD graph becomes stages and tasks
- [ ] TaskScheduler — `core/src/main/scala/org/apache/spark/scheduler/TaskSchedulerImpl.scala` — Task distribution to executors
- [ ] Shuffle system — `core/src/main/scala/org/apache/spark/shuffle/` — ShuffleManager, sort-based shuffle, shuffle read/write
- [ ] Memory management — `core/src/main/scala/org/apache/spark/memory/` — Unified memory manager, on-heap vs off-heap

### Week 2: Catalyst Deep Dive (Mar 31 - Apr 6)

- [ ] Expressions — `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/expressions/` — Expression evaluation, codegen for expressions
- [ ] Analyzer rules — `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/Analyzer.scala` — ResolveRelations, ResolveReferences, type coercion
- [ ] Optimizer rules — `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala` — Predicate pushdown, constant folding, column pruning
- [ ] Rule executor — `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/rules/RuleExecutor.scala` — Batches, fixed-point iteration, how rules compose
- [ ] Type system — `sql/catalyst/src/main/scala/org/apache/spark/sql/types/` — DataType hierarchy, schema resolution

### Week 3: Physical Execution (Apr 7 - Apr 13)

- [ ] Join algorithms — `sql/core/src/main/scala/org/apache/spark/sql/execution/joins/` — BroadcastHashJoin, SortMergeJoin, ShuffledHashJoin — when each is picked
- [ ] Aggregation — `sql/core/src/main/scala/org/apache/spark/sql/execution/aggregate/` — HashAggregate vs SortAggregate, partial/final aggregation
- [ ] Exchange (shuffle) — `sql/core/src/main/scala/org/apache/spark/sql/execution/exchange/` — ShuffleExchangeExec, BroadcastExchangeExec
- [ ] Whole-stage codegen (deeper) — `sql/core/src/main/scala/org/apache/spark/sql/execution/WholeStageCodegenExec.scala` — Go deeper into produce/consume with IntelliJ debugging
- [ ] Adaptive Query Execution — `sql/core/src/main/scala/org/apache/spark/sql/execution/adaptive/` — Runtime re-optimization, coalescing partitions, skew handling

### Week 4: Data Sources & Streaming (Apr 14 - Apr 20)

- [ ] DataSource V2 API — `sql/catalyst/src/main/scala/org/apache/spark/sql/connector/` — Table, ScanBuilder, Batch, PartitionReader interfaces
- [ ] Parquet reader — `sql/core/src/main/scala/org/apache/spark/sql/execution/datasources/parquet/` — Column pruning, predicate pushdown at file level
- [ ] Structured Streaming — `sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/` — MicroBatchExecution, IncrementalExecution, watermarks
- [ ] State store — `sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/state/` — How stateful operators persist state across batches
- [ ] Checkpointing — `sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/CheckpointFileManager.scala` — Offset log, commit log, fault tolerance

### Week 5: Networking, Storage & Tuning (Apr 21 - Apr 27)

- [ ] Block manager — `core/src/main/scala/org/apache/spark/storage/BlockManager.scala` — How data blocks are stored, replicated, evicted
- [ ] Tungsten — `sql/core/src/main/scala/org/apache/spark/sql/execution/UnsafeRow*` — Binary row format, off-heap memory, cache-friendly layout
- [ ] Broadcast — `core/src/main/scala/org/apache/spark/broadcast/` — TorrentBroadcast, how driver distributes data to executors
- [ ] Metrics and SparkUI — `core/src/main/scala/org/apache/spark/ui/` — How execution metrics are collected and displayed
- [ ] SparkSession internals — `sql/core/src/main/scala/org/apache/spark/sql/SparkSession.scala` — SessionState, catalog, how everything ties together

### Study Method

1. Read memory notes first for existing context
2. Set breakpoints in IntelliJ and trace a simple query
3. Use `df.queryExecution.analyzed/optimizedPlan/executedPlan.treeString` to inspect each stage
4. Ask Claude to explain each topic with a simple example walkthrough

## Related Notes

- [[project_spark_sql_architecture]] — Architecture overview for Week 2
- [[project_spark_distributed_systems_topics]] — Distributed systems context for Week 1
- [[project_spark_blockmanager_internals]] — BlockManager for Week 5
- [[project_sparksession_internals]] — SparkSession for Week 5
