---
name: Spark SQL Core Architecture
description: Key classes, inheritance hierarchies, and query execution pipeline in Spark SQL core and catalyst modules
type: project
---

## Spark SQL Core — Architecture Overview

### Entry Points
- **SparkSession** (`sql/core/.../classic/SparkSession.scala`) — Main entry point. Extends `sql.SparkSession` with `Logging`. Holds `sparkContext`, `sharedState`, `sessionState`, `sqlConf`. Creates `Dataset`/`DataFrame` via `sql()`, `read`, `createDataFrame()`.
- **Dataset[T]** (`sql/core/.../classic/Dataset.scala`) — Core data abstraction. `DataFrame` is just `Dataset[Row]` (type alias). Holds `queryExecution: QueryExecution` and `encoder: Encoder[T]`.
- **SQLContext** (`sql/core/.../classic/SQLContext.scala`) — Deprecated wrapper around SparkSession for Spark 1.x compat.

### Session Management
- **SessionState** (`sql/core/.../internal/SessionState.scala`) — Per-session state. Holds `catalog`, `analyzer`, `optimizer`, `planner`, `sqlParser`, `functionRegistry`, `listenerManager`. Key method: `executePlan(): QueryExecution`.
- **SharedState** (`sql/core/.../internal/SharedState.scala`) — Shared across sessions. Holds `sparkContext`, `cacheManager`, `externalCatalog`, `statusStore`, `relationCache`.
- **SessionCatalog** (`sql/catalyst/.../catalog/SessionCatalog.scala`) — Manages tables, views, functions. Proxies to `ExternalCatalog` (e.g., Hive metastore).

### Query Execution Pipeline (in order)
1. **SQL/DataFrame API** → User input
2. **Parser** (`ParserInterface`) → Unresolved `LogicalPlan`
3. **Analyzer** → Resolved `LogicalPlan` (attributes, tables resolved)
4. **Optimizer** → Optimized `LogicalPlan` (pushdowns, pruning, folding)
5. **SparkPlanner** → `SparkPlan` (physical plan)
6. **Preparation Rules** → Executed `SparkPlan`
7. **execute()** → `RDD[InternalRow]`

**QueryExecution** (`sql/core/.../execution/QueryExecution.scala`) orchestrates this entire pipeline. Lazy vals: `logical` → `analyzed` → `optimizedPlan` → `sparkPlan` → `executedPlan` → `toRdd`.

### Catalyst Plan Hierarchy
```
TreeNode[T]  (catalyst/trees)
  └── QueryPlan[T]  (catalyst/plans)
        ├── LogicalPlan  (catalyst/plans/logical)
        │     - extends QueryPlan[LogicalPlan] with AnalysisHelper, LogicalPlanStats
        └── SparkPlan  (core/execution)
              - extends QueryPlan[SparkPlan] with Serializable
              - execute(): RDD[InternalRow]
```

### Rule Execution Framework
- **RuleExecutor[TreeType]** (`catalyst/rules`) — Base for both Analyzer and Optimizer. Applies batches of transformation rules to tree nodes.
- **Analyzer** extends `RuleExecutor[LogicalPlan]` with `CheckAnalysis` — resolves attributes, tables, functions, does type coercion.
- **Optimizer** extends `RuleExecutor[LogicalPlan]` — predicate pushdown, column pruning, constant folding, join reordering.
- **SparkPlanner** extends `SparkStrategies` — converts LogicalPlan → SparkPlan using strategies.

### Key Module Boundaries
- **sql/catalyst** — Plans (LogicalPlan, QueryPlan, TreeNode), Analyzer, Optimizer, RuleExecutor, SessionCatalog, expressions
- **sql/core** — SparkSession, Dataset, QueryExecution, SparkPlan, SessionState, SharedState, physical operators (*Exec classes)

**Why:** Understanding these relationships is essential for navigating the codebase when debugging query planning, adding new optimizations, or tracing how a user query becomes an RDD execution.

**How to apply:** Use this as a map when the user asks about query execution flow, plan transformations, or where to find specific Spark SQL components.
