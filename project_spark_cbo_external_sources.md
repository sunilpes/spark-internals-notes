---
name: Spark CBO with External Sources
description: How to use CBO with non-Hive sources (JDBC, Postgres, MySQL, raw files) — stats availability per source, workarounds (save to table, hints, V2 JDBC pushdown, AQE), best practices
type: project
---

## How to Make Spark Use CBO for Different Sources

### CBO Needs Stats in the Spark Catalog

CBO works through catalog stats, not just Hive metastore. External sources (JDBC, raw files) don't have catalog stats by default.

### Stats Availability Per Source

| Source | Has Stats? | CBO Works? |
|--------|-----------|------------|
| Hive table | Yes (metastore) | Yes, after ANALYZE |
| Spark-managed Parquet/ORC table | Yes (catalog) | Yes, after ANALYZE |
| `spark.read.parquet("path/")` | File size only | Partial (sizeInBytes only) |
| `spark.read.jdbc(...)` | No stats at all | No — uses `defaultSizeInBytes` |
| `spark.read.csv("path/")` | File size only | Partial |
| Temp views from DataFrames | Inherits source stats | Depends on source |

### The JDBC Problem

```scala
val users = spark.read.jdbc(url, "users", props)   // actually 50MB
val orders = spark.read.jdbc(url, "orders", props)  // actually 5GB

// Spark sees: both = Long.MaxValue (8.4 exabytes!)
// Result: SortMergeJoin (even though users should be broadcast)
```

### Solutions

**Option 1: Save to Spark table first, then analyze**
```scala
spark.read.jdbc(url, "users", props).write.saveAsTable("users_local")
spark.sql("ANALYZE TABLE users_local COMPUTE STATISTICS FOR COLUMNS id, name, age")
// CBO works on this table
```
Most reliable, but duplicates data.

**Option 2: Use V2 JDBC with pushdown**
```scala
spark.read.format("jdbc")
  .option("pushDownAggregate", "true")
  .option("pushDownLimit", "true")
  .load()
```
Pushes count/min/max/filters to Postgres/MySQL directly. Not CBO, but similar effect.

**Option 3: Broadcast hint (manual override)**
```scala
users.hint("broadcast").join(orders, "id")
// Forces BroadcastHashJoin regardless of stats
```

**Option 4: Rely on AQE (runtime stats)**
AQE discovers actual sizes after shuffle stages complete and re-optimizes. No pre-collected stats needed.

### Best Practice by Source Type

| Source | Recommendation |
|--------|---------------|
| Hive/managed tables | `ANALYZE TABLE ... FOR COLUMNS` + enable CBO |
| Parquet/ORC paths | AQE handles most cases; optionally register as table + analyze |
| JDBC (Postgres/MySQL) | V2 JDBC with pushdown; `broadcast()` hint for small tables; or save to Spark table |
| CSV/JSON | Save to Parquet first, then analyze |
| Mixed sources | AQE (runtime stats) + explicit hints for known-small tables |

### TL;DR

For most real-world pipelines with mixed sources, **AQE + broadcast hints** is more practical than enabling CBO. CBO shines when you have managed tables with fresh stats from ANALYZE TABLE.
