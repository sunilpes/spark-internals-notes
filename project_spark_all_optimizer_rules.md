---
name: Complete Spark Optimizer Rules Catalog
description: All 80+ Catalyst optimizer rules organized by category — push down, combine, eliminate, column pruning, constant folding, filter, join, subquery, set ops, aggregate, local relation, window, and Spark-specific rules with batch execution order
type: project
---

## All Optimizer Rules — Location and What They Do

**File**: `sql/catalyst/.../optimizer/Optimizer.scala` (lines 100-276)
**Spark-specific**: `sql/core/.../execution/SparkOptimizer.scala`

### Category 1: Operator Push Down

| Rule | What It Does |
|------|-------------|
| `PushProjectionThroughUnion` | Push SELECT cols down into each side of UNION |
| `PushProjectionThroughLimitAndOffset` | Push column pruning below LIMIT/OFFSET |
| `PushDownPredicates` | Push WHERE filters down through Project, Join, etc. |
| `PushDownLeftSemiAntiJoin` | Push semi/anti join below other operators |
| `PushLeftSemiLeftAntiThroughJoin` | Push semi/anti join through inner joins |
| `PushExtraPredicateThroughJoin` | Push extra predicates derived from join conditions |
| `LimitPushDown` | Push LIMIT into Union, Join children |
| `LimitPushDownThroughWindow` | Push LIMIT below Window operators |
| `PushdownPredicatesAndPruneColumnsForCTEDef` | Push filters/pruning into CTE definitions |

### Category 2: Operator Combine / Collapse

| Rule | What It Does |
|------|-------------|
| `CollapseProject` | Merge adjacent Project → Project into one |
| `CollapseRepartition` | Merge adjacent Repartition → Repartition into one |
| `CollapseWindow` | Merge adjacent Window operators with same partition/order |
| `CombineUnions` | Merge nested Union(Union(a,b), c) → Union(a,b,c) |
| `CombineTypedFilters` | Merge adjacent typed filters (Dataset API) |
| `CombineConcats` | Merge nested concat(concat(a,b), c) → concat(a,b,c) |

### Category 3: Eliminate / Remove

| Rule | What It Does |
|------|-------------|
| `EliminateDistinct` | Remove DISTINCT from MAX/MIN (already duplicate-agnostic) |
| `EliminateOuterJoin` | Convert outer join to inner if null side is filtered out |
| `EliminateOffsets` | Remove OFFSET 0 (no-op) |
| `EliminateLimits` | Remove redundant nested limits |
| `EliminateSorts` | Remove sorts when output order doesn't matter |
| `EliminateSerialization` | Remove serialize→deserialize roundtrips |
| `EliminateAggregateFilter` | Remove always-true aggregate filters |
| `EliminateMapObjects` | Remove unnecessary MapObjects in typed operations |
| `EliminateWindowPartitions` | Remove unused window partitioning |
| `RemoveRedundantAliases` | Remove aliases that don't rename anything |
| `RemoveRedundantAggregates` | Remove aggregates that duplicate a child aggregate |
| `RemoveRedundantSorts` | Remove sorts when data is already sorted |
| `RemoveNoopOperators` | Remove operators that don't change anything |
| `RemoveNoopUnion` | Remove UNION with single child |

### Category 4: Column / Schema Pruning

| Rule | What It Does |
|------|-------------|
| `ColumnPruning` | Drop unused columns from Project, Aggregate, Expand |
| `GenerateOptimization` | Prune unused outputs from Generate (explode, etc.) |
| `ObjectSerializerPruning` | Prune unused fields in Dataset serialization |

### Category 5: Constant Folding & Simplification

| Rule | What It Does |
|------|-------------|
| `ConstantFolding` | `1 + 2` → `3`, evaluate at plan time |
| `ConstantPropagation` | `a = 5 AND b = a` → `a = 5 AND b = 5` |
| `FoldablePropagation` | Propagate foldable expressions through plan |
| `NullPropagation` | `null + x` → `null` |
| `NullDownPropagation` | Propagate null-ness downward |
| `BooleanSimplification` | `a AND true` → `a`, deduplicate |
| `SimplifyConditionals` | `IF(true, a, b)` → `a` |
| `SimplifyBinaryComparison` | `x = x` → `true` (non-nullable) |
| `SimplifyCasts` | Remove redundant casts |
| `SimplifyCaseConversionExpressions` | `upper(upper(x))` → `upper(x)` |
| `SimplifyDateTimeConversions` | Simplify redundant date/time conversions |
| `SimplifyExtractValueOps` | Simplify nested struct/array access |
| `PushFoldableIntoBranches` | Push constants into IF/CASE branches |
| `ReplaceNullWithFalseInPredicate` | Replace null with false in filters |
| `UnwrapCastInBinaryComparison` | `CAST(int AS long) = 5L` → `int = 5` |
| `ReorderAssociativeOperator` | `(a + 1) + 2` → `a + 3` |
| `OptimizeIn` | IN list → HashSet lookup |
| `OptimizeRand` | Optimize RAND expressions |
| `LikeSimplification` | `LIKE 'abc'` → `= 'abc'` |
| `OptimizeCsvJsonExprs` | Optimize repeated CSV/JSON parsing |
| `OptimizeUpdateFields` | Simplify nested field updates |

### Category 6: Filter Optimization

| Rule | What It Does |
|------|-------------|
| `PruneFilters` | Remove always-true filters, replace always-false with empty |
| `InferFiltersFromConstraints` | `a = b AND a > 5` → adds `b > 5` |
| `InferFiltersFromGenerate` | Infer not-null filters from Generate |

### Category 7: Join Optimization

| Rule | What It Does |
|------|-------------|
| `ReorderJoin` | Reorder inner joins to push filters closer to leaves |
| `OptimizeJoinCondition` | Simplify join conditions |
| `CostBasedJoinReorder` | Reorder joins based on table statistics (CBO) |
| `CheckCartesianProducts` | Warn/error on unintended cartesian products |

### Category 8: Subquery Optimization

| Rule | What It Does |
|------|-------------|
| `OptimizeSubqueries` | Recursively optimize subquery plans |
| `OptimizeOneRowRelationSubquery` | Simplify subqueries returning one row |
| `RewriteCorrelatedScalarSubquery` | Rewrite correlated scalar subquery as left join |
| `RewriteLateralSubquery` | Rewrite lateral subquery |
| `RewritePredicateSubquery` | Rewrite IN/EXISTS as semi/anti joins |
| `PullupCorrelatedPredicates` | Pull correlated predicates out of subqueries |
| `InlineCTE()` | Inline CTE if used once |
| `RewriteNonCorrelatedExists` | Rewrite non-correlated EXISTS |

### Category 9: Set Operation Rewrites

| Rule | What It Does |
|------|-------------|
| `RewriteExceptAll` | EXCEPT ALL → window functions |
| `RewriteIntersectAll` | INTERSECT ALL → window functions |
| `ReplaceIntersectWithSemiJoin` | INTERSECT → left semi join |
| `ReplaceExceptWithFilter` | EXCEPT → filter (same source) |
| `ReplaceExceptWithAntiJoin` | EXCEPT → left anti join |
| `ReplaceDistinctWithAggregate` | DISTINCT → GROUP BY |
| `ReplaceDeduplicateWithAggregate` | Deduplicate → aggregate |

### Category 10: Aggregate Optimization

| Rule | What It Does |
|------|-------------|
| `RemoveLiteralFromGroupExpressions` | `GROUP BY name, 1` → `GROUP BY name` |
| `RemoveRepetitionFromGroupExpressions` | `GROUP BY a, a, b` → `GROUP BY a, b` |
| `RewriteDistinctAggregates` | Rewrite multiple DISTINCT aggregates |
| `DecimalAggregates` | Optimize decimal SUM/AVG with unscaled longs |

### Category 11: Local / Empty Relation

| Rule | What It Does |
|------|-------------|
| `ConvertToLocalRelation` | Convert small scans to in-memory local relations |
| `PropagateEmptyRelation` | `Join(empty, x)` → empty, eliminate subtrees |
| `UpdateAttributeNullability` | Fix nullability after empty relation removal |
| `OptimizeOneRowPlan` | Optimize plans producing exactly one row |

### Category 12: Spark-Specific (SparkOptimizer)

| Rule | What It Does |
|------|-------------|
| `SchemaPruning` | Push column pruning into nested fields for Parquet/ORC |
| `V2ScanRelationPushDown` | Push filters/aggregates/limits into V2 data sources |
| `V2ScanPartitioningAndOrdering` | Push partitioning/ordering into V2 scans |
| `PruneFileSourcePartitions` | Skip partition directories based on filters |
| `PushVariantIntoScan` | Push variant type extraction into scan |
| `PartitionPruning` | Dynamic partition pruning (runtime filter from join) |
| `InjectRuntimeFilter` | Inject bloom filters at runtime for joins |
| `OptimizeMetadataOnlyQuery` | Skip data scan for metadata-only queries |

### Batch Execution Order

```
1.  Finish Analysis
2.  Rewrite With expression
3.  Eliminate Distinct
4.  Inline CTE
5.  Union
6.  LocalRelation early
7.  Pullup Correlated Expressions
8.  Subquery
9.  Replace Operators
10. Aggregate
11. Operator Optimization before Inferring Filters  (fixedPoint, ~50 rules)
12. Infer Filters
13. Operator Optimization after Inferring Filters   (fixedPoint, ~50 rules again)
14. Push extra predicate through join
15. Clean Up Temporary CTE Info
16. Pre CBO Rules
17. Early Filter and Projection Push-Down           (format-aware)
18. Update CTE Relation Stats
19. Join Reorder (CBO)
20. Eliminate Sorts
21. Decimal Optimizations
22. Distinct Aggregate Rewrite
23. Object Expressions Optimization
24. LocalRelation
25. Optimize One Row Plan
26. Check Cartesian Products
27. RewriteSubquery
```

~80+ distinct rules across ~27 batches.
