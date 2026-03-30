---
tags: [spark-sql, spark-core, spark-storage, spark-aqe, spark-joins, spark-optimization, spark-distributed, spark-execution]
---

# Spark Internals -- Map of Content

This is the entry point for navigating the Spark internals knowledge graph. Notes are organized by topic area. Open the **Graph View** (Ctrl/Cmd+G) to visualize connections.

---

## SQL Pipeline (parsing to execution)

The complete journey of a SQL query through Spark:

1. [[project_spark_sql_parsing_explained]] -- SQL string to unresolved LogicalPlan
2. [[project_spark_ast_tree_building]] -- How ANTLR parse tree becomes LogicalPlan via AstBuilder
3. [[project_spark_sql_planning_explained]] -- TreeNode hierarchy, planning pipeline
3. [[project_spark_optimizer_rule_execution]] -- How optimizer picks and applies rules
4. [[project_spark_top_optimizer_rules_explained]] -- Top 6 rules with before/after examples
5. [[project_spark_all_optimizer_rules]] -- Complete catalog of 80+ rules
6. [[project_spark_physical_planning_walkthrough]] -- Logical to physical plan conversion
7. [[project_whole_stage_codegen_explained]] -- Codegen fuses operators into Java
8. [[project_spark_udf_physical_planning]] -- How UDFs integrate into physical plans

## Architecture and Session

- [[project_spark_sql_architecture]] -- Key classes, inheritance hierarchies, query execution pipeline
- [[project_sparksession_internals]] -- SparkSession, SharedState, SessionState
- [[spark-sql-planning-classes]] -- Mermaid class diagram of plan hierarchy
- [[project_spark_query_planning_caveats]] -- Limitations and pitfalls of query planning

## Optimizer and CBO

- [[project_spark_optimizer_rule_execution]] -- Batch ordering, fixed-point loop
- [[project_spark_top_optimizer_rules_explained]] -- PushDown, ColumnPruning, ConstantFolding
- [[project_spark_all_optimizer_rules]] -- All 80+ rules by category
- [[project_spark_format_aware_optimization]] -- Format-specific rules (Parquet, ORC, CSV)
- [[project_spark_cbo_explained]] -- Cost-based optimization, ANALYZE TABLE
- [[project_spark_cbo_external_sources]] -- CBO with JDBC, Postgres, raw files
- [[project_spark_parquet_count_optimization]] -- count() and aggregate pushdown

## Adaptive Query Execution (AQE)

- [[project_spark_aqe_stats_flow]] -- MapStatus, MapOutputTracker, runtime stats
- [[project_spark_aqe_stage_splitting]] -- Exchange boundaries, submit-wait-reoptimize loop
- [[project_spark_aqe_skew_join_handling]] -- Skew detection, split + replicate
- [[project_spark_aqe_skew_join_type_constraints]] -- NULL correctness constraints
- [[project_spark_skew_join_workarounds]] -- LeftOuter+LeftAnti, salting, broadcast

## Joins

- [[project_spark_join_strategy_selection]] -- BroadcastHash vs SortMerge vs ShuffledHash
- [[project_spark_cbo_explained]] -- CBO improves join decisions
- [[project_spark_aqe_skew_join_handling]] -- Runtime skew handling
- [[project_spark_aqe_skew_join_type_constraints]] -- Which join types support skew splitting
- [[project_spark_skew_join_workarounds]] -- Manual workarounds for unsupported types

## Storage and BlockManager

- [[project_spark_blockmanager_internals]] -- MemoryStore, DiskStore, locking, replication
- [[project_spark_rdd_block_creation_format]] -- Block creation, memory/disk formats
- [[project_spark_memory_unrolling_explained]] -- Gradual materialization into memory
- [[project_spark_blockmanager_file_processing]] -- When BlockManager IS and ISN'T involved
- [[project_spark_file_split_partitioning]] -- File splitting and bin-packing
- [[project_spark_parquet_count_optimization]] -- Parquet footer stats

## RPC and Communication

- [[project_spark_rpc_inbox_dispatcher]] -- Dispatcher, MessageLoop, Inbox
- [[project_spark_rpc_message_format]] -- Wire format, LaunchTask, StatusUpdate
- [[project_spark_heartbeat_contents]] -- Executor heartbeat: metrics, accumulators
- [[project_spark_task_completion_tracking]] -- Task/stage/job completion flow
- [[project_spark_listener_bus]] -- Driver-only async event bus

## Distributed Systems

- [[project_spark_distributed_systems_topics]] -- Key topics for understanding Spark
- [[project_spark_hadoop_integration]] -- FileSystem API, YARN, Kerberos
- [[project_spark_rpc_inbox_dispatcher]] -- Netty RPC internals
- [[project_spark_no_consensus_algo]] -- No Raft/Paxos — single driver, consensus only in YARN/HDFS/ZK

## Streaming

- [[project_streaming_dedup_internals]] -- dropDuplicates: state store vs batch path

## Meta

- [[project_spark_5week_learning_plan]] -- Structured learning plan (started 2026-03-24)
- [[_Graph Guide]] -- How to use the Obsidian graph view with these notes
