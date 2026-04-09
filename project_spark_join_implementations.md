---
name: Spark Join Implementations Deep Dive
description: All 5 physical join operators (BroadcastHash, SortMerge, ShuffledHash, BroadcastNL, Cartesian), join types, selection priority, build side rules, HashedRelation internals, AQE dynamic re-selection, config parameters
type: project
tags: [spark, joins, physical-planning, broadcast, sort-merge, hash-join, aqe]
---

## Join Types (Logical)

Defined in `sql/catalyst/.../plans/joinTypes.scala`:
- **Inner/Cross** (InnerLike) — matching rows / cartesian product
- **LeftOuter/RightOuter/FullOuter** — all rows from one/both sides, NULLs where no match
- **LeftSemi** — left rows where match exists (no right columns)
- **LeftAnti** — left rows where NO match exists
- **ExistenceJoin(exists: Attribute)** — internal, adds boolean column
- **LeftSingle** — at most one right row per left (scalar subquery)
- **NaturalJoin/UsingJoin** — wrappers for NATURAL/USING syntax

## 5 Physical Join Operators

### 1. BroadcastHashJoinExec
- **When:** Equi-join, one side < `autoBroadcastJoinThreshold` (10MB), binary-comparable keys
- **Algorithm:** Broadcast small side → build HashedRelation → stream large side → hash probe
- **No shuffle needed**
- **Supports:** Inner, Cross, LeftOuter (build right), RightOuter (build left), LeftSemi, LeftAnti (build right)
- **Performance:** O(N) stream × O(1) probe. Risk: OOM if broadcast larger than stats suggest

### 2. SortMergeJoinExec
- **When:** Equi-join with sortable keys, default strategy
- **Algorithm:** Shuffle both sides by key → sort → two-pointer merge on sorted streams
- **Buffer:** ExternalAppendOnlyUnsafeRowArray for matching rows, spills to disk
- **Supports:** All join types except LeftSingle
- **Performance:** Shuffle + sort cost, but bounded memory with spilling. Best for large-large joins

### 3. ShuffledHashJoinExec
- **When:** Equi-join, one side much smaller (build × 20 ≤ stream), partition fits in memory, preferSortMergeJoin=false
- **Algorithm:** Shuffle both sides → build hash table per partition → stream + probe
- **Supports:** All join types including FullOuter
- **Performance:** Faster than SMJ when build partition small (no sort). Risk: OOM if skewed partition

### 4. BroadcastNestedLoopJoinExec
- **When:** Non-equi join (no join keys), or keys not binary-comparable
- **Algorithm:** Broadcast small side → nested loop: for each stream row × each broadcast row → check condition
- **Supports:** All join types
- **Performance:** O(N × M). Only viable when broadcast side very small

### 5. CartesianProductExec
- **When:** Inner/Cross with no condition, or SHUFFLE_REPLICATE_NL hint
- **Algorithm:** Every left partition × every right partition, emit all combinations
- **Supports:** Inner/Cross only
- **Performance:** Output = |left| × |right|. Extremely expensive

## HashedRelation (Hash Table)

Three implementations in `HashedRelation.scala`:
- **LongHashedRelation** — single Long key, sparse (hash probe) or dense (direct array) mode
- **UnsafeHashedRelation** — generic keys, uses BytesToBytesMap (off-heap)
- **EmptyHashedRelation** — empty build side, zero memory

Probe logic (HashJoin.scala): unique keys → `getValue()` O(1), duplicate keys → `get()` iterator.

## Join Selection Priority

### With Hints:
1. BROADCAST → BroadcastHashJoin
2. SHUFFLE_MERGE → SortMergeJoin
3. SHUFFLE_HASH → ShuffledHashJoin
4. SHUFFLE_REPLICATE_NL → CartesianProduct

### Without Hints (equi-join):
1. One side < 10MB → BroadcastHashJoin
2. One side much smaller, partition fits, preferSMJ=false → ShuffledHashJoin
3. Keys sortable → **SortMergeJoin** (default landing)
4. Inner-like → CartesianProduct
5. Else → BroadcastNestedLoopJoin

### Without Hints (non-equi):
1. One side < 10MB → BroadcastNestedLoopJoin
2. Inner-like → CartesianProduct
3. Else → BroadcastNestedLoopJoin (last resort)

### AQE Dynamic Re-Selection:
- Demote broadcast if many empty partitions
- Prefer shuffle hash if every partition fits in memory
- Force shuffle hash if both conditions apply

## Build Side Rules

| Join Type | Can Build Left? | Can Build Right? |
|---|---|---|
| Inner/Cross | Yes | Yes |
| LeftOuter | No | Yes |
| RightOuter | Yes | No |
| LeftSemi/LeftAnti | No | Yes |
| FullOuter | Yes (ShuffledHash only) | Yes (ShuffledHash only) |

If both sides eligible → pick smaller by stats.sizeInBytes.

## Key Configs

- `spark.sql.autoBroadcastJoinThreshold` = 10MB
- `spark.sql.join.preferSortMergeJoin` = true
- `spark.sql.shuffle.partitions` = 200
- `spark.sql.adaptive.enabled` = true
- `SHUFFLE_HASH_JOIN_FACTOR` = 0.05 (build × 20 ≤ stream)

## Key Source Files

- `sql/catalyst/.../plans/joinTypes.scala` — JoinType hierarchy
- `sql/core/.../execution/SparkStrategies.scala:181-424` — JoinSelection strategy
- `sql/catalyst/.../optimizer/joins.scala:290-562` — JoinSelectionHelper (size checks, build side)
- `sql/core/.../execution/joins/BroadcastHashJoinExec.scala` — broadcast hash join
- `sql/core/.../execution/joins/SortMergeJoinExec.scala` — sort merge join
- `sql/core/.../execution/joins/ShuffledHashJoinExec.scala` — shuffled hash join
- `sql/core/.../execution/joins/BroadcastNestedLoopJoinExec.scala` — broadcast nested loop
- `sql/core/.../execution/joins/CartesianProductExec.scala` — cartesian product
- `sql/core/.../execution/joins/HashJoin.scala` — shared hash probe logic
- `sql/core/.../execution/joins/HashedRelation.scala` — hash table implementations
- `sql/core/.../execution/adaptive/DynamicJoinSelection.scala` — AQE re-selection
