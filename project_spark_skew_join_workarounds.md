---
name: Spark Skew Join Workarounds
description: How to circumvent AQE skew limitations for FullOuter joins — split into LeftOuter+LeftAnti, salt the key, separate skewed keys + broadcast, increase parallelism, when to use which approach
type: project
tags: [spark-aqe, spark-joins]
---

## Workarounds for Skew in Unsupported Join Types (FullOuter, etc.)

### 1. Split Into LeftOuter + LeftAnti (most common)

Convert FullOuter into two joins that both support AQE skew splitting:

```sql
-- LeftOuter: all matched + unmatched left with NULLs
SELECT o.*, r.reason
FROM orders o LEFT OUTER JOIN returns r ON o.id = r.order_id
UNION ALL
-- LeftAnti: unmatched right rows (missing piece from FullOuter)
SELECT NULL, NULL, ..., r.reason
FROM returns r LEFT ANTI JOIN orders o ON r.order_id = o.id
```

Both support AQE left-side splitting.

### 2. Salt the Skewed Key

Add random suffix to spread skew across partitions:

```scala
val saltBuckets = 10
val saltedOrders = orders
  .withColumn("salt", (rand() * saltBuckets).cast("int"))
  .withColumn("salted_key", concat(col("country_code"), lit("_"), col("salt")))

val explodedCountries = countries
  .withColumn("salt", explode(array((0 until saltBuckets).map(lit(_)): _*)))
  .withColumn("salted_key", concat(col("code"), lit("_"), col("salt")))

saltedOrders.join(explodedCountries, "salted_key", "full_outer")
```

Works for ANY join type. P_US 4.5GB → 10 partitions of 450MB.

### 3. Separate Skewed Keys + Broadcast

```scala
val skewedKeys = Set("US")
// Non-skewed: normal FullOuter (no skew)
val nonSkewed = orders.filter(!col("country_code").isin(skewedKeys.toSeq: _*))
  .join(countries.filter(!col("code").isin(skewedKeys.toSeq: _*)), ..., "full_outer")
// Skewed: broadcast the small side
val skewed = orders.filter(col("country_code").isin(skewedKeys.toSeq: _*))
  .join(broadcast(countries.filter(...)), ..., "full_outer")
val result = nonSkewed.unionByName(skewed)
```

### 4. Broadcast If One Side Is Small

```scala
orders.join(broadcast(countries), ..., "full_outer")
// No shuffle → no skew problem
```

### 5. Increase Parallelism

```scala
spark.conf.set("spark.sql.shuffle.partitions", 2000)
```

Reduces impact but doesn't solve root cause.

### When to Use Which

| Situation | Best Approach |
|-----------|-------------|
| One side small (< 1GB) | Broadcast — simplest |
| Known skewed keys | Separate + broadcast — targeted |
| Unknown/dynamic skew | Salt the key — generic |
| Can restructure query | LeftOuter + LeftAnti — lets AQE handle it |
| Quick fix | Increase partitions — reduces impact |

### All workarounds avoid the core problem by:
- **Eliminating the shuffle** (broadcast)
- **Distributing the skew** (salting)
- **Decomposing into supported join types** (LeftOuter + LeftAnti)

## Related Notes

- [[project_spark_aqe_skew_join_type_constraints]] — Why these workarounds are needed
- [[project_spark_aqe_skew_join_handling]] — AQE skew handling for supported join types
- [[project_spark_join_strategy_selection]] — Broadcast hint as workaround
