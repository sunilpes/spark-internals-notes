---
name: Spark Parquet Count Optimization
description: Why count() reads full file by default, aggregate pushdown to Parquet footer metadata (disabled by default), how to enable it, catalog stats via ANALYZE TABLE
type: project
---

## Why count() Reads the Whole Parquet File by Default

### Spark CAN Read Count from Parquet Footer

Parquet stores row counts in footer metadata. Spark has code for this (`ParquetUtils.scala:366`):

```scala
case _: CountStar =>
  rowCount += block.getRowCount   // reads from metadata, not data
```

### But It's Disabled by Default!

```scala
spark.sql.parquet.aggregatePushdown = false   // default since Spark 3.3
```

**Why disabled?** If statistics are missing from any Parquet file footer, an exception is thrown. Not all files have valid stats (non-Spark writers, old versions, corrupted footers).

### To Enable Metadata-Only Count

```scala
spark.conf.set("spark.sql.parquet.aggregatePushdown", "true")
```

Plan changes from:
```
WITHOUT pushdown (default):
  HashAggregate (count)
  └── FileScan parquet [reads ALL data]

WITH pushdown:
  HashAggregate (final count)
  └── FileScan parquet [reads ONLY footer metadata]
      → one row per file from block.getRowCount()
```

### Supports: MIN, MAX, COUNT

- COUNT: all data types
- MIN/MAX: boolean, integer, float, date
- All read from Parquet footer statistics, not data

### Alternative: Catalog Stats

```sql
ANALYZE TABLE my_table COMPUTE STATISTICS
```

Then catalog has the row count — Spark can use it without reading files at all.

### Summary

| Scenario | Reads Data? | Requirement |
|----------|------------|-------------|
| `count()` default | Yes, full scan | Default behavior |
| `count()` with pushdown | No, footer only | `spark.sql.parquet.aggregatePushdown=true` |
| `count()` on analyzed table | No, catalog stats | `ANALYZE TABLE` was run |
| `count()` with filter | Yes, must read data | Filter needs row-level evaluation |
