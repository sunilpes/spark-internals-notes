---
name: Spark Join Strategy Selection
description: How Spark selects join strategies (BroadcastHash, SortMerge, ShuffledHash) based on data statistics — stats collection, size thresholds, JoinSelection logic, priority order, hints, AQE re-planning
type: project
---

## How Spark Selects Join Strategies Based on Data Statistics

### Where Does Spark Get the Data Size?

Stats are **NOT** collected during analysis. They're computed **lazily during planning** when the planner accesses `plan.stats`.

**Two sources of stats:**

| Source | When | How |
|--------|------|-----|
| **File listing** (default) | Always available | `PartitioningAwareFileIndex.sizeInBytes` = sum of `FileStatus.getLen()` from HDFS/local FS |
| **Catalog stats** (richer) | After `ANALYZE TABLE` | Stored in metastore: sizeInBytes, rowCount, column stats, histograms |

```
LogicalRelation.computeStats()
  → catalogTable.stats.map(_.toPlanStats())    // if ANALYZE was run
  → OR: Statistics(sizeInBytes = relation.sizeInBytes)  // fallback: file sizes
```

### Join Strategy Selection — Priority Order

`JoinSelection.apply()` in `SparkStrategies.scala:217` checks strategies **in this order**:

```
1. BroadcastHashJoin     ← if either side ≤ 10 MB (default)
2. ShuffledHashJoin      ← if one side is 3x smaller AND fits in memory
3. SortMergeJoin         ← default for large equi-joins
4. CartesianProduct      ← inner join with no join keys
5. BroadcastNestedLoopJoin ← final fallback (non-equi joins)
```

### The Key Decision: Can We Broadcast?

`JoinSelectionHelper.canBroadcastBySize()` (`joins.scala:367`):

```scala
def canBroadcastBySize(plan: LogicalPlan, conf: SQLConf): Boolean = {
  plan.stats.sizeInBytes >= 0 &&
  plan.stats.sizeInBytes <= conf.autoBroadcastJoinThreshold  // default 10 MB
}
```

### Concrete Example — Small + Large Table

```sql
-- users: 500 bytes (tiny), orders: 2 GB (huge)
SELECT * FROM users JOIN orders ON users.id = orders.user_id
```

```
JoinSelection.apply() checks:

1. BroadcastHashJoin?
   canBroadcastBySize(users)?  → 500 bytes ≤ 10MB → YES
   canBroadcastBySize(orders)? → 2GB ≤ 10MB → NO
   → Pick BroadcastHashJoin, broadcast "users" side

Result:
BroadcastHashJoinExec [id = user_id]
├── BroadcastExchange (users)     ← small table sent to all executors
└── FileScan orders               ← large table stays partitioned
```

### Concrete Example — Both Tables Large

```sql
-- users: 5 GB, orders: 20 GB
SELECT * FROM users JOIN orders ON users.id = orders.user_id
```

```
1. BroadcastHashJoin?
   canBroadcastBySize(users)?  → 5GB > 10MB → NO
   canBroadcastBySize(orders)? → 20GB > 10MB → NO
   → Skip

2. ShuffledHashJoin?
   canBuildLocalHashMap(users)? → 5GB < 10MB * 200 partitions = 2GB → NO
   → Skip

3. SortMergeJoin? → keys are sortable → YES

Result:
SortMergeJoinExec [id = user_id]
├── Sort [id] → Exchange(HashPartitioning) → FileScan users
└── Sort [user_id] → Exchange(HashPartitioning) → FileScan orders
```

### Stats Flow — End to End

```
                        ANALYZE TABLE users    (optional, richer stats)
                              ↓
                        CatalogStatistics stored in metastore
                              ↓
Parquet files on disk → FileStatus.getLen() → sizeInBytes (always available)
                              ↓
              LogicalRelation.computeStats()
                              ↓
                    Statistics(sizeInBytes = ...)
                              ↓ cached in plan node
              JoinSelection accesses plan.stats
                              ↓
              canBroadcastBySize(left/right)?
                    ↓                ↓
                   YES              NO
                    ↓                ↓
          BroadcastHashJoin    SortMergeJoin (or ShuffledHash)
```

### Key Configs

| Config | Default | Effect |
|--------|---------|--------|
| `spark.sql.autoBroadcastJoinThreshold` | **10 MB** | Max size to broadcast; set `-1` to disable |
| `spark.sql.shuffledHashJoinFactor` | **3** | Build side must be 3x smaller than probe side |
| `spark.sql.join.preferSortMergeJoin` | **true** | Prefer SortMerge over ShuffledHash |
| `spark.sql.cbo.enabled` | **false** | Enable richer stats (row count, column stats) |

### Hints Override Everything

```sql
SELECT /*+ BROADCAST(orders) */ * FROM users JOIN orders ON ...
SELECT /*+ MERGE(users, orders) */ * FROM users JOIN orders ON ...
SELECT /*+ SHUFFLE_HASH(users) */ * FROM users JOIN orders ON ...
```

Hints are checked **before** size-based logic in `JoinSelection.apply()` (lines 219-236).

### AQE (Adaptive Query Execution) Can Re-decide

At runtime, after shuffle stages complete, AQE knows the **actual data sizes** (not estimates). If a shuffle output turns out to be ≤ threshold, AQE can **convert SortMergeJoin → BroadcastHashJoin** on the fly. This uses `stats.isRuntime = true` with a potentially different adaptive threshold.

### Key Source Files

| Component | File | Key Lines |
|-----------|------|-----------|
| Statistics case class | `catalyst/.../plans/logical/Statistics.scala` | 55-74 |
| CatalogStatistics | `catalyst/.../catalog/interface.scala` | 842-878 |
| LogicalPlanStats | `catalyst/.../statsEstimation/LogicalPlanStats.scala` | 25-50 |
| JoinSelection strategy | `sql/core/.../execution/SparkStrategies.scala` | 181-424 |
| JoinSelectionHelper | `catalyst/.../optimizer/joins.scala` | 290-562 |
| canBroadcastBySize | `catalyst/.../optimizer/joins.scala` | 367-375 |
| File size collection | `sql/core/.../datasources/PartitioningAwareFileIndex.scala` | 117 |
