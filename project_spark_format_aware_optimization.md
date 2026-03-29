---
name: Spark Format-Aware Optimization Rules
description: How Spark applies optimizations based on file format — partition pruning, predicate pushdown to row groups/stripes, column pruning, aggregate pushdown, bucket pruning, V2 push-down interfaces, per-format capability matrix
type: project
---

## Format-Aware Optimization Rules in Query Planning

### Layer 1: Partition Pruning (skip directories)

**Rule**: `PruneFileSourcePartitions` — works on any partitioned format

```sql
SELECT * FROM events WHERE date = '2024-01-15'
→ Only reads events/date=2024-01-15/, skips all other date directories
```

### Layer 2: Predicate Pushdown to File Format (skip row groups/stripes)

**Rule**: `FileSourceStrategy` pushes filters into format reader (line 207)

**Parquet**: Checks row group min/max statistics → skip entire row groups
**ORC**: Checks stripe statistics + bloom filters → skip stripes
**CSV/JSON**: No statistics → no pushdown, must read everything

### Layer 3: Column Pruning / Schema Pruning (skip columns)

**Rule**: `SchemaPruning`

**Parquet/ORC** (columnar): Only reads needed column chunks — huge I/O savings
**CSV/JSON** (row-based): Must read entire rows, discard columns in Spark

### Layer 4: Aggregate Pushdown (metadata only)

**Rule**: `V2ScanRelationPushDown` — only for `SupportsPushDownAggregates`

```sql
SELECT count(*), max(age) FROM users
```

**Parquet/ORC**: Read from footer stats, no data read
**CSV/JSON**: Not supported → full scan

### Layer 5: Bucket Pruning (skip files)

**Rule**: `FileSourceStrategy` (line 179)

```sql
SELECT * FROM users WHERE user_id = 42
→ hash(42) % 100 = bucket 42 → only reads part-00042-*.parquet
```

### Layer 6: V2 Push-Down Interfaces

Format self-declares capabilities via interfaces:

| Interface | What Gets Pushed | Parquet | ORC | JDBC | CSV |
|-----------|-----------------|---------|-----|------|-----|
| `SupportsPushDownFilters` | WHERE predicates | Yes | Yes | Yes | No |
| `SupportsPushDownRequiredColumns` | Column pruning | Yes | Yes | Yes | No |
| `SupportsPushDownAggregates` | COUNT/MIN/MAX | Yes | Yes | Yes | No |
| `SupportsPushDownLimit` | LIMIT N | No | No | Yes | No |
| `SupportsPushDownTopN` | ORDER BY + LIMIT | No | No | Yes | No |

### How Rules Know the Format

Rules don't hardcode formats. They **ask** via interfaces:

```scala
scanBuilder match {
  case s: SupportsPushDownFilters     => s.pushFilters(filters)
  case s: SupportsPushDownAggregates  => s.pushAggregation(agg)
  case s: SupportsPushDownRequiredColumns => s.pruneColumns(schema)
}
```

### Per-Format Capability Matrix

```
                    Parquet         ORC            CSV/JSON
Partition Pruning   ✅ skip dirs    ✅ skip dirs    ✅ skip dirs
Predicate Pushdown  ✅ row groups   ✅ stripes      ❌ read all
Column Pruning      ✅ skip cols    ✅ skip cols    ❌ read all rows
Aggregate Pushdown  ✅ footer stats ✅ footer stats ❌ full scan
Bucket Pruning      ✅ skip files   ✅ skip files   ✅ skip files
Splittable          ✅              ✅              ❌ if compressed
Vectorized Read     ✅              ✅              ❌
```

This is why **Parquet is the default recommendation** — benefits from every optimization layer.

### Key Rules in SparkOptimizer (earlyScanPushDownRules)

```scala
Seq(
  SchemaPruning,                       // column/nested field pruning
  V2ScanRelationPushDown,              // filter/aggregate/limit pushdown for V2
  V2ScanPartitioningAndOrdering,       // partition-aware scan
  PruneFileSourcePartitions,           // partition directory pruning
  PushVariantIntoScan                  // variant type pushdown
)
```

### Key Source Files

| Component | File |
|-----------|------|
| FileSourceStrategy | `sql/core/.../datasources/FileSourceStrategy.scala` |
| V2ScanRelationPushDown | `sql/core/.../datasources/v2/V2ScanRelationPushDown.scala` |
| PruneFileSourcePartitions | `sql/core/.../datasources/PruneFileSourcePartitions.scala` |
| SchemaPruning | `sql/core/.../datasources/SchemaPruning.scala` |
| ParquetFilters | `sql/core/.../datasources/parquet/ParquetFilters.scala` |
| OrcFilters | `sql/core/.../datasources/orc/OrcFilters.scala` |
| ParquetScanBuilder | `sql/core/.../datasources/v2/parquet/ParquetScanBuilder.scala` |
| SparkOptimizer | `sql/core/.../execution/SparkOptimizer.scala` |
