---
name: Spark Cost-Based Optimization (CBO)
description: How CBO works in Spark — disabled by default, stats estimation (Filter/Join/Aggregate), DP join reordering, selectivity computation, cardinality estimation, ANALYZE TABLE, CBO vs default comparison
type: project
tags: [spark-optimization, spark-sql, spark-joins]
---

## Where Spark Does Cost-Based Optimization

### Disabled by Default

```scala
spark.sql.cbo.enabled = false   // default
```

Without CBO: only `sizeInBytes` (file size). With CBO: rowCount, column stats (min/max/nullCount/distinctCount).

### 3 Places CBO Is Used

### 1. Stats Estimation

`LogicalPlanStats.stats` (`statsEstimation/LogicalPlanStats.scala:33`):

```scala
if (conf.cboEnabled)
  BasicStatsPlanVisitor.visit(self)      // CBO: rich stats
else
  SizeInBytesOnlyStatsPlanVisitor.visit(self)  // Default: just file sizes
```

**BasicStatsPlanVisitor** estimates per operator:

| Operator | Class | What It Computes |
|----------|-------|------------------|
| Filter | `FilterEstimation` | Selectivity per predicate, updated row count |
| Join | `JoinEstimation` | Join cardinality from column stats |
| Aggregate | `AggregateEstimation` | Output rows from distinct counts |
| Project | `ProjectEstimation` | Updated column stats after projection |

### FilterEstimation — Selectivity

`WHERE age > 25` with column stats: min=0, max=100:

```
Selectivity = (max - 25) / (max - min) = 75/100 = 0.75
Row count: 10M * 0.75 = 7.5M rows

AND: p1 * p2 (independence assumed)
OR:  p1 + p2 - p1*p2 (inclusion-exclusion)
NOT: 1 - p
```

### JoinEstimation — Cardinality

`users JOIN orders ON users.id = orders.user_id`:

```
users: 10M rows, id distinctCount = 10M
orders: 50M rows, user_id distinctCount = 5M

Cardinality = (10M * 50M) / max(10M, 5M) = 50M rows
```

### 2. Join Reordering (DP Algorithm)

**Requires**: `cbo.enabled=true` AND `cbo.joinReorder.enabled=true`

`CostBasedJoinReorder.scala` — dynamic programming (Selinger paper):

```
4-table join: A(100K), B(10M), C(500), D(1M)

Level 0: individual tables
Level 1: best 2-way plans (A⋈B, B⋈C, C⋈D)
Level 2: best 3-way plans
Level 3: final best plan

Winner: (C ⋈ D) ⋈ (A ⋈ B)  — start with smallest!
```

Cost model: `Cost(card: BigInt, size: BigInt)` — card=CPU cost, size=I/O cost.

Conditions for reorder (line 63):
- More than 2 tables
- ≤ 12 tables (dp.threshold)
- Has join conditions
- ALL tables have rowCount stats

### 3. Join Strategy Selection (indirect)

With CBO, `JoinSelection` gets accurate size estimates after filters:

```
Without CBO: users = 500MB → too big to broadcast
With CBO: WHERE age > 90 → selectivity 10% → 50MB → broadcast!
```

### How Stats Get Into the System

```sql
ANALYZE TABLE users COMPUTE STATISTICS
ANALYZE TABLE users COMPUTE STATISTICS FOR COLUMNS id, name, age

-- Stored in metastore:
-- Table: sizeInBytes=500MB, rowCount=10M
-- Column age: min=0, max=100, nullCount=0, distinctCount=101
```

### CBO vs Default

| | Default | With CBO |
|--|---------|----------|
| Stats | sizeInBytes only | rowCount, columnStats |
| Filter estimation | No selectivity | Calculates from column range |
| Join estimation | Product of file sizes | Cardinality from distinct counts |
| Join reorder | Write order preserved | DP finds optimal order |
| Broadcast decision | Raw file size | Estimated output size after filters |
| Requires | Nothing | ANALYZE TABLE ... FOR COLUMNS |

### Key Configs

| Config | Default | Purpose |
|--------|---------|---------|
| `spark.sql.cbo.enabled` | false | Enable CBO stats |
| `spark.sql.cbo.joinReorder.enabled` | false | Enable DP join reordering |
| `spark.sql.cbo.joinReorder.dp.threshold` | 12 | Max tables for DP |
| `spark.sql.cbo.starSchemaDetection` | false | Star schema optimization |

### Key Source Files

| File | What |
|------|------|
| `statsEstimation/LogicalPlanStats.scala` | CBO toggle |
| `statsEstimation/FilterEstimation.scala` | Selectivity computation |
| `statsEstimation/JoinEstimation.scala` | Join cardinality |
| `optimizer/CostBasedJoinReorder.scala` | DP join reorder |

## Related Notes

- [[project_spark_cbo_external_sources]] — CBO with JDBC, Postgres, raw files
- [[project_spark_join_strategy_selection]] — How CBO stats improve join strategy picks
- [[project_spark_optimizer_rule_execution]] — CostBasedJoinReorder in the optimizer pipeline
- [[project_spark_all_optimizer_rules]] — Where CBO rules sit in batch execution order
- [[project_spark_aqe_stats_flow]] — AQE as runtime alternative to CBO
- [[project_spark_query_planning_caveats]] — CBO off by default, stale stats
