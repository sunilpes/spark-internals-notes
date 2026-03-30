---
name: Spark Optimizer Rule Execution
description: How Catalyst optimizer picks and applies rules — batch ordering, fixed-point loop, pattern matching, RuleExecutor.execute() walkthrough, and concrete example with SELECT query
type: project
tags: [spark-sql, spark-optimization]
---

## How Optimizer Rules Are Picked and Applied

### Step 1: Rules are organized into ordered Batches

The optimizer defines a **fixed sequence** of batches (Optimizer.scala line 176-273). Rules don't "decide" to run — they're all registered upfront:

```
Batch "Finish Analysis"           (FixedPoint(1))  → [FinishAnalysis]
Batch "Eliminate Distinct"        (Once)           → [EliminateDistinct]
Batch "Inline CTE"               (Once)           → [InlineCTE]
Batch "Union"                     (fixedPoint)     → [RemoveNoopOperators, CombineUnions, ...]
Batch "LocalRelation early"       (fixedPoint)     → [ConvertToLocalRelation, ...]
Batch "Operator Optimization ..." (fixedPoint)     → [PushDownPredicates, ColumnPruning,
                                                       CollapseProject, ConstantFolding, ...
                                                       ~50 rules total]
Batch "Join Reorder"              (FixedPoint(1))  → [CostBasedJoinReorder]
Batch "RewriteSubquery"           (Once)           → [ColumnPruning, CollapseProject, ...]
... ~20 batches total
```

### Step 2: RuleExecutor.execute() drives everything

`RuleExecutor.scala:215-325` — the engine:

```scala
def execute(plan: TreeType): TreeType = {
  var curPlan = plan

  batches.foreach { batch →              // 1. For each batch IN ORDER
    var continue = true
    while (continue) {                    // 2. Loop (fixed-point)
      curPlan = batch.rules.foldLeft(curPlan) {
        case (plan, rule) →
          val result = rule(plan)         // 3. Apply each rule sequentially
          val effective = !result.fastEquals(plan)  // 4. Did it change?
          result
      }
      if (curPlan.fastEquals(lastPlan))   // 5. Plan unchanged? → stop
        continue = false
      if (iteration > maxIterations)      // 6. Hit limit (100)? → stop
        continue = false
    }
  }
  curPlan
}
```

### Step 3: Each rule walks the tree with pattern matching

When `rule(plan)` is called, the rule uses `transformWithPruning` to walk the **entire plan tree bottom-up** and pattern-match nodes:

```scala
// ColumnPruning example (line 1066)
def apply(plan: LogicalPlan): LogicalPlan =
  plan.transformWithPruning(AlwaysProcess.fn, ruleId) {
    // Match: Project on top of another Project with unused columns
    case p @ Project(_, p2: Project) if !p2.outputSet.subsetOf(p.references) =>
      p.copy(child = p2.copy(projectList = p2.projectList.filter(p.references.contains)))

    // Match: Project on top of Aggregate with unused columns
    case p @ Project(_, a: Aggregate) if !a.outputSet.subsetOf(p.references) =>
      p.copy(child = a.copy(aggregateExpressions = ...))

    // ... more patterns
  }
```

**If a node matches** → returns a transformed copy
**If no pattern matches** → node passes through unchanged

### Walkthrough With Example Query

```sql
SELECT name FROM users WHERE age > 25
```

**Batch: "Operator Optimization before Inferring Filters"** (up to 100 iterations)

**Iteration 1** — rules run sequentially on the plan:

```
Rule 1: PushProjectionThroughUnion  → no Union node → NO MATCH → plan unchanged
Rule 2: ReorderJoin                 → no Join node  → NO MATCH → plan unchanged
Rule 3: PushDownPredicates          → MATCHES Filter → pushes predicate down
Rule 4: ColumnPruning              → MATCHES Project → prunes "id" column
Rule 5: CollapseProject            → MATCHES adjacent Projects → merges them
Rule 6: ConstantFolding            → no foldable expr → NO MATCH
Rule 7: BooleanSimplification      → no simplifiable bool → NO MATCH
... (all ~50 rules run, most NO MATCH)
```

Plan changed from iteration start → **loop again**

**Iteration 2** — all ~50 rules run again:

```
Rule 1-50: all NO MATCH (already optimized)
```

Plan unchanged → **fixed point reached** → exit batch loop

### Visual Summary

```
Optimizer.execute(analyzedPlan)
│
├─ Batch 1: "Finish Analysis" (max 1 iter)
│   └─ iter 1: [FinishAnalysis] → apply → done
│
├─ Batch 2: "Eliminate Distinct" (Once)
│   └─ iter 1: [EliminateDistinct] → no match → done
│
├─ ...
│
├─ Batch N: "Operator Optimization" (max 100 iters)
│   ├─ iter 1: [Rule1 → Rule2 → ... → Rule50] → plan changed!
│   ├─ iter 2: [Rule1 → Rule2 → ... → Rule50] → plan changed!
│   ├─ iter 3: [Rule1 → Rule2 → ... → Rule50] → plan unchanged → STOP
│   └─ (fixed point reached after 3 iterations)
│
├─ Batch M: "RewriteSubquery" (Once)
│   └─ iter 1: [RewritePredicateSubquery → ColumnPruning → ...] → done
│
└─ return optimizedPlan
```

### Why ColumnPruning Runs Multiple Times

1. **Appears in 2 batches** — "Operator Optimization" (line 114) and "RewriteSubquery" (line 270)
2. **Fixed-point loop** within each batch — up to 100 iterations (default `spark.sql.optimizer.maxIterations`)
3. **Other rules create new opportunities** — e.g., PushDownPredicates rearranges plan, creating new pruning opportunities for ColumnPruning

### Key Insight

**Every rule runs on every iteration** regardless of whether it matched before. Rules don't "know" if they're relevant — they just pattern-match the tree. Most calls result in no match and return the plan unchanged. The cost is low because `transformWithPruning` can skip subtrees that don't contain relevant node types.

### Batch Strategies

| Strategy | Behavior |
|----------|----------|
| `Once` | Run all rules exactly once |
| `FixedPoint(1)` | Run once, same as Once but with idempotence check |
| `fixedPoint` | Loop until plan stops changing or 100 iterations (`spark.sql.optimizer.maxIterations`) |

## Related Notes

- [[project_spark_top_optimizer_rules_explained]] — Detailed look at top 6 rules
- [[project_spark_all_optimizer_rules]] — Complete catalog of 80+ rules
- [[project_spark_format_aware_optimization]] — Format-specific optimization rules
- [[project_spark_cbo_explained]] — Cost-based optimization rules
- [[project_spark_query_planning_caveats]] — Optimizer limitations and pitfalls
- [[project_spark_sql_planning_explained]] — Planning pipeline where optimizer sits
