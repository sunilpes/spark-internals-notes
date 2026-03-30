---
name: Spark AQE Skew Join Handling
description: How AQE detects and handles skewed partitions — detection threshold, mapper-level splitting, replication of non-skewed side, PartialReducerPartitionSpec, Cartesian product for both-sides skew, concrete example with country skew
type: project
---

## How AQE Handles Skewed Partitions

### The Problem

After shuffle, some partitions get way more data than others:

```
P0: 50MB, P1: 60MB, P2: 5GB(!), P3: 40MB, ..., P199: 55MB
Median: ~50MB, P2 = 100x larger → SKEW
199 tasks finish in 1 minute, P2 takes 2 hours
```

### How AQE Detects Skew

`OptimizeSkewedJoin.scala:65` — skewed if size exceeds BOTH:

```
skewThreshold = max(
  256MB,                        // absolute threshold
  medianPartitionSize × 5.0     // relative threshold
)
```

### The Fix: Split + Replicate

### Concrete Example

```sql
-- orders: 1B rows, 90% have country='US' (skew!)
SELECT o.*, c.country_name
FROM orders o JOIN countries c ON o.country_code = c.code
```

After shuffle:
```
LEFT (orders):              RIGHT (countries):
  P_US:  4.5GB ← SKEWED      P_US:  200 bytes
  P_UK:  100MB                P_UK:  200 bytes
  P_DE:  80MB                 P_DE:  200 bytes
  ...                         ...
```

**Step 1: Detect** — P_US = 4.5GB > max(256MB, 50MB×5) → SKEWED

**Step 2: Target size** — max(64MB advisory, 60MB avg non-skewed) = 64MB

**Step 3: Split by mapper ranges**

```
P_US produced by 200 mappers (~25MB each)
Split into ~64MB chunks:
  Split 0: Mappers 0-2   → 71MB  (PartialReducerPartitionSpec)
  Split 1: Mappers 3-5   → 65MB
  ...
  Split 69: Mappers 197-199 → 66MB

4.5GB → 70 chunks of ~64MB each
```

**Step 4: Replicate other side**

```
BEFORE: 1 task: (4.5GB) ⋈ (200B) → 2 hours
AFTER:  70 tasks, each: (64MB) ⋈ (200B) → 30 seconds, all parallel
```

### When BOTH Sides Are Skewed

Cartesian product of splits:
```
Left splits: [L2-1, L2-2, L2-3]  (3 splits)
Right splits: [R2-1, R2-2]        (2 splits)
= 6 tasks: (L2-1,R2-1), (L2-1,R2-2), (L2-2,R2-1), ...
```

### How Splitting Works Internally

Split at **mapper boundaries** using `PartialReducerPartitionSpec`:

```scala
case class PartialReducerPartitionSpec(
    reducerIndex: Int,      // which partition (P_US)
    startMapIndex: Int,     // read from mapper N
    endMapIndex: Int,       // to mapper M (exclusive)
    dataSize: Long)
```

Non-skewed side replicated by creating **multiple specs pointing to same partition**.

### Join Type Constraints

| Join Type | Split Left? | Split Right? |
|-----------|------------|-------------|
| Inner/Cross | Yes | Yes |
| LeftOuter | Yes | No |
| RightOuter | No | Yes |
| LeftSemi/Anti | Yes | No |
| FullOuter | No | No |

### Key Configs

| Config | Default | Purpose |
|--------|---------|---------|
| `spark.sql.adaptive.skewJoin.enabled` | true | Enable skew optimization |
| `spark.sql.adaptive.skewJoin.skewedPartitionFactor` | 5.0 | Skewed if > median × factor |
| `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` | 256MB | Absolute minimum for skew |
| `spark.sql.adaptive.advisoryPartitionSizeInBytes` | 64MB | Target size for splits |

### Key Source Files

| File | What |
|------|------|
| `adaptive/OptimizeSkewedJoin.scala` | Skew detection + split + replicate |
| `adaptive/ShufflePartitionsUtil.scala:398` | `createSkewPartitionSpecs()` mapper-level splitting |
| `adaptive/AQEShuffleReadExec.scala` | Executes rearranged partition specs |
| `ShuffledRowRDD.scala:48` | `PartialReducerPartitionSpec` |
