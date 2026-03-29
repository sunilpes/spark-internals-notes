---
name: Spark Physical Planning Walkthrough
description: How physical plans are prepared from optimized logical plans — recursive top-down algorithm, PlanLater placeholders, Strategy priority order, complex 3-table join example, logical-to-physical node mapping
type: project
---

## How Physical Plans Are Prepared

### The Planning Algorithm

`QueryPlanner.plan()` (`QueryPlanner.scala:59`) is **recursive, top-down**:

```
1. Take the root logical node
2. Try each Strategy in order until one matches
3. The matching strategy returns a physical node with PlanLater() placeholders for children
4. Recursively plan each PlanLater() child (go back to step 1)
5. Replace placeholders with the physical children
```

### Strategies — Tried in This Order (first match wins)

```scala
// SparkPlanner.scala:38-57
strategies = Seq(
  extraStrategies,              // user-injected (Delta, Iceberg, etc.)
  LogicalQueryStageStrategy,    // AQE already-materialized stages
  PythonEvals,                  // Python UDFs
  DataSourceV2Strategy,         // V2 data sources
  V2CommandStrategy,            // V2 DDL commands
  FileSourceStrategy,           // Parquet/ORC/CSV file scans
  DataSourceStrategy,           // V1 data sources
  SpecialLimits,                // LIMIT optimizations
  Aggregation,                  // GROUP BY → HashAggregate/SortAggregate
  Window,                       // Window functions
  WindowGroupLimit,             // Window + LIMIT optimization
  JoinSelection,                // Joins → BroadcastHash/SortMerge/ShuffledHash
  InMemoryScans,                // Cached tables
  SparkScripts,                 // Script transforms
  Pipelines,                    // Pipe operators
  BasicOperators,               // Filter, Project, Sort, Union, etc.
)
```

### Complex Join Example

```sql
SELECT u.name, o.amount, p.product_name
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN products p ON o.product_id = p.id
WHERE u.age > 25 AND o.amount > 100
ORDER BY o.amount DESC
LIMIT 10
```

Assume: users = 50MB, orders = 5GB, products = 2MB

### Optimized Logical Plan

```
GlobalLimit 10
└── LocalLimit 10
    └── Sort [amount DESC]
        └── Project [name, amount, product_name]
            └── Join (Inner, o.product_id = p.id)
                ├── Join (Inner, u.id = o.user_id)
                │   ├── Filter (age > 25)
                │   │   └── Relation users [id, name, age]      (50MB)
                │   └── Filter (amount > 100)
                │       └── Relation orders [user_id, product_id, amount]  (5GB)
                └── Relation products [id, product_name]         (2MB)
```

### Recursive Planning — Round by Round

**Round 1: `GlobalLimit 10` + `Sort` + `Project`**
```
SpecialLimits → MATCH!
  → TakeOrderedAndProjectExec(10, [amount DESC], child = PlanLater(Join...))
```

**Round 2: `Join(orders_result, products)` on product_id = id**
```
JoinSelection → MATCH!
  products = 2MB ≤ 10MB → BroadcastHashJoin
  → BroadcastHashJoinExec(build=PlanLater(products), stream=PlanLater(Join...))
```

**Round 3: `Join(users, orders)` on id = user_id**
```
JoinSelection → MATCH!
  Both > 10MB → SortMergeJoin
  → SortMergeJoinExec(left=PlanLater(Filter+users), right=PlanLater(Filter+orders))
```

**Round 4-6: Leaf nodes**
```
FileSourceStrategy → MATCH for each Relation
  → FileSourceScanExec with pushed filters and column pruning
```

### Assembled Physical Plan

```
TakeOrderedAndProjectExec(10, [amount DESC], [name, amount, product_name])
└── BroadcastHashJoinExec (o.product_id = p.id, buildRight)
    ├── SortMergeJoinExec (u.id = o.user_id)
    │   ├── Sort [id ASC]
    │   │   └── Exchange (HashPartitioning(id))        ← shuffle
    │   │       └── FilterExec (age > 25)
    │   │           └── FileSourceScanExec (users)
    │   └── Sort [user_id ASC]
    │       └── Exchange (HashPartitioning(user_id))   ← shuffle
    │           └── FilterExec (amount > 100)
    │               └── FileSourceScanExec (orders)
    └── BroadcastExchange                              ← broadcast
        └── FileSourceScanExec (products)
```

### Post-Processing Rules (after planning)

```
EnsureRequirements      → inserts Exchange (shuffle) and Sort nodes
CollapseCodegenStages   → wraps chains in WholeStageCodegenExec
ReuseExchange           → if same shuffle appears twice, reuse it
ReuseSubquery           → reuse identical subquery results
```

### The Key Mechanism: PlanLater()

Strategies don't plan children — they wrap them in `PlanLater()`:

```scala
case Join(left, right, Inner, condition, _) =>
  Seq(SortMergeJoinExec(
    leftKeys, rightKeys,
    planLater(left),     // "I'll let someone else plan this"
    planLater(right),    // "and this too"
    condition))
```

`QueryPlanner.plan()` collects all PlanLater placeholders and recursively plans each by trying all strategies again.

### Logical → Physical Node Mapping

| Logical Node | Strategy | Physical Node |
|-------------|----------|--------------|
| LogicalRelation(HadoopFsRelation) | FileSourceStrategy | FileSourceScanExec |
| DataSourceV2Relation | DataSourceV2Strategy | BatchScanExec |
| Filter | BasicOperators | FilterExec |
| Project | BasicOperators | ProjectExec |
| Sort | BasicOperators | SortExec |
| Join (small ≤ 10MB) | JoinSelection | BroadcastHashJoinExec |
| Join (both large, equi) | JoinSelection | SortMergeJoinExec |
| Join (non-equi) | JoinSelection | BroadcastNestedLoopJoinExec |
| Aggregate | Aggregation | HashAggregateExec (partial + final) |
| Window | Window | WindowExec |
| Limit + Sort | SpecialLimits | TakeOrderedAndProjectExec |
| Union | BasicOperators | UnionExec |
| InMemoryRelation | InMemoryScans | InMemoryTableScanExec |

### Key Source Files

| File | What |
|------|------|
| `QueryPlanner.scala:59` | Recursive plan() algorithm |
| `SparkPlanner.scala:38` | Strategy list and order |
| `SparkStrategies.scala` | All strategy implementations (JoinSelection, Aggregation, etc.) |
| `FileSourceStrategy.scala` | File scan physical planning |
