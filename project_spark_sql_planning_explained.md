---
name: Spark SQL Planning Explained From Scratch
description: Complete walkthrough of how Spark SQL planning works — TreeNode hierarchy, Logical vs Physical plans, LeafNode/UnaryNode/BinaryNode, QueryExecution pipeline, Strategies, and recursive planning algorithm with source code references
type: project
tags: [spark-sql, spark-execution]
---

## How Spark SQL Planning Works — From Scratch

### Part 1: The Tree — Everything is a Tree

Every query in Spark is represented as a **tree of nodes**. Before understanding planning, you need to understand what a "node" is.

#### What is a TreeNode?

At `TreeNode.scala:70`:
```scala
abstract class TreeNode[BaseType <: TreeNode[BaseType]] extends Product {
  def children: Seq[BaseType]    // every node knows its children
}
```

Think of it like a family tree. Every node can have zero or more children. Spark defines exactly **three shapes** of nodes (at `TreeNode.scala:1256-1316`):

#### LeafLike — A node with NO children (a data source)

```scala
trait LeafLike[T <: TreeNode[T]] {
  override final def children: Seq[T] = Nil    // always empty!
}
```

Example: A table scan. It reads from disk — there's nothing below it.
```
  [TableScan("users")]    <- leaf, no children
```

#### UnaryLike — A node with exactly ONE child

```scala
trait UnaryLike[T <: TreeNode[T]] {
  def child: T                                          // singular!
  override final lazy val children: Seq[T] = IndexedSeq(child)  // wraps in list
}
```

Example: A `Filter` takes ONE input and filters rows from it.
```
  [Filter: age > 25]      <- has exactly one child
       |
  [TableScan("users")]    <- the child
```

#### BinaryLike — A node with exactly TWO children (left + right)

```scala
trait BinaryLike[T <: TreeNode[T]] {
  def left: T
  def right: T
  override final lazy val children: Seq[T] = IndexedSeq(left, right)
}
```

Example: A `Join` combines two inputs.
```
     [Join: u.id = o.user_id]     <- has two children
         /            \
[Scan("users")]   [Scan("orders")]
    (left)            (right)
```

### Part 2: Two Worlds of Trees — Logical vs Physical

Spark has two parallel hierarchies built on `TreeNode`:

```
                  TreeNode[T]
                      |
                  QueryPlan[T]          <- adds: def output: Seq[Attribute]
                  /           \
         LogicalPlan        SparkPlan
         (the "what")       (the "how")
```

**`QueryPlan`** at `QueryPlan.scala:53` adds the concept of `output` — what columns this node produces:
```scala
abstract class QueryPlan[PlanType <: QueryPlan[PlanType]] extends TreeNode[PlanType] {
  def output: Seq[Attribute]    // the schema of rows coming out of this node
}
```

**`LogicalPlan`** at `LogicalPlan.scala:37` — describes **what** you want:
```scala
abstract class LogicalPlan extends QueryPlan[LogicalPlan]
```

**`SparkPlan`** (physical) — describes **how** to execute it. Same tree structure but with concrete implementations that produce RDDs.

Each world has its own Leaf/Unary/Binary variants:

| Children | Logical | Physical |
|----------|---------|----------|
| 0 (leaf) | `LeafNode` | `LeafExecNode` |
| 1 (unary) | `UnaryNode` | `UnaryExecNode` |
| 2 (binary) | `BinaryNode` | `BinaryExecNode` |

### Part 3: Concrete Examples in Source Code

**`Project`** (SELECT columns) — `basicLogicalOperators.scala:73`:
```scala
case class Project(projectList: Seq[NamedExpression], child: LogicalPlan)
    extends OrderPreservingUnaryNode {
  override def output: Seq[Attribute] = projectList.map(_.toAttribute)
  //                                     ^ output is ONLY the selected columns
}
```
One child (unary) — the data it reads from. Output is the projected columns.

**`Filter`** (WHERE clause) — `basicLogicalOperators.scala:335`:
```scala
case class Filter(condition: Expression, child: LogicalPlan)
  extends OrderPreservingUnaryNode {
  override def output: Seq[Attribute] = child.output
  //                                     ^ output is SAME as child (just fewer rows)
}
```
One child (unary). Output schema is identical to the child — filtering doesn't change columns, just removes rows.

**`Join`** — `basicLogicalOperators.scala:714`:
```scala
case class Join(
    left: LogicalPlan,      // <- first child
    right: LogicalPlan,     // <- second child
    joinType: JoinType,
    condition: Option[Expression],
    hint: JoinHint)
  extends BinaryNode {
  override def output: Seq[Attribute] = Join.computeOutput(joinType, left.output, right.output)
  //                                     ^ combines columns from both children
}
```
Two children (binary). Output merges columns from left and right.

### Part 4: A Query as a Tree

For `SELECT name FROM users WHERE age > 25`:

```
Logical Plan Tree:

    Project(name)              <- UnaryNode, output = [name]
         |
    Filter(age > 25)           <- UnaryNode, output = [id, name, age]
         |
    UnresolvedRelation("users")  <- LeafNode, output = [id, name, age]
```

Data flows **bottom-up**: the leaf produces all rows, Filter removes some, Project keeps only the `name` column.

### Part 5: The Planning Pipeline

This is the journey from SQL string to actual execution. It's orchestrated by **`QueryExecution`** at `QueryExecution.scala:65`:

```scala
class QueryExecution(val sparkSession: SparkSession, val logical: LogicalPlan) {
```

It defines a chain of lazy vals — each one transforms the plan further:

```
SQL String
    |
    v
+----------+   Parser (not in QueryExecution)
| logical   |   Unresolved tree: table names are strings, columns unvalidated
+----------+
    |  line 144: analyzer.executeAndCheck(logical)
    v
+----------+   ANALYSIS
| analyzed  |   Resolve table/column names, check types, validate semantics
+----------+
    |  line 249: optimizer.executeAndTrack(withCachedData)
    v
+--------------+   OPTIMIZATION
| optimizedPlan |   Push down filters, prune columns, fold constants, etc.
+--------------+
    |  line 270: createSparkPlan(planner, optimizedPlan)
    v
+-----------+   PLANNING (logical -> physical)
| sparkPlan  |   Convert LogicalPlan tree -> SparkPlan tree
+-----------+
    |  line 285: prepareForExecution(preparations, sparkPlan)
    v
+--------------+   PREPARATION
| executedPlan  |   Insert shuffles, codegen wrappers, etc.
+--------------+
    |  line 300: executedPlan.execute()
    v
+-------+
| toRdd  |   Actual RDD[InternalRow] — Spark jobs run!
+-------+
```

### Part 6: How Planning Works (Logical -> Physical)

This is the most interesting step. The **`SparkPlanner`** at `SparkPlanner.scala:31` has a list of **strategies**:

```scala
class SparkPlanner(...) extends SparkStrategies {
  override def strategies: Seq[Strategy] =
    LogicalQueryStageStrategy ::
    PythonEvals ::
    DataSourceV2Strategy ::
    FileSourceStrategy ::
    DataSourceStrategy ::
    SpecialLimits ::
    Aggregation ::
    Window ::
    JoinSelection ::
    InMemoryScans ::
    BasicOperators ::       // <- the catch-all
    EventTimeWatermarkStrategy :: Nil
}
```

Each `Strategy` is simple — it's a pattern match. At `SparkStrategies.scala:61`:
```scala
abstract class SparkStrategy extends GenericStrategy[SparkPlan] {
  def apply(plan: LogicalPlan): Seq[SparkPlan]   // match logical -> return physical
}
```

The **`BasicOperators`** strategy at line 903 is the catch-all that handles `Project`, `Filter`, `Sort`, etc.:

```scala
object BasicOperators extends Strategy {
  def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
    case logical.Project(projectList, child) =>
      execution.ProjectExec(projectList, planLater(child)) :: Nil      // line 1032

    case logical.Filter(condition, child) =>
      execution.FilterExec(condition, planLater(child)) :: Nil         // line 1034

    case logical.Sort(sortExprs, global, child, _) =>
      execution.SortExec(sortExprs, global, planLater(child)) :: Nil   // line 1030

    // ... many more
  }
}
```

Notice `planLater(child)` — this is a **placeholder**. The strategy says "I know how to convert this Filter node, but I'll let someone else figure out the child."

### Part 7: The Recursive Planning Algorithm

At `QueryPlanner.scala:59`, the `plan()` method ties it all together:

```scala
def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {
  // 1. Try each strategy until one matches
  val candidates = strategies.iterator.flatMap(_(plan))

  // 2. For each candidate, find planLater() placeholders
  val plans = candidates.flatMap { candidate =>
    val placeholders = collectPlaceholders(candidate)

    if (placeholders.isEmpty) {
      Iterator(candidate)       // done! no placeholders left
    } else {
      // 3. RECURSIVELY plan each placeholder
      placeholders.iterator.foldLeft(Iterator(candidate)) {
        case (candidates, (placeholder, logicalPlan)) =>
          val childPlans = this.plan(logicalPlan)    // <- recursive call!
          candidates.flatMap { c =>
            childPlans.map { childPlan =>
              c.transformUp {
                case p if p.eq(placeholder) => childPlan  // replace placeholder
              }
            }
          }
      }
    }
  }
  prunePlans(plans)
}
```

### Part 8: Full Walkthrough — `SELECT name FROM users WHERE age > 25`

**Step 1: Logical Plan (after analysis + optimization)**
```
Project([name])
    |
Filter(age > 25)
    |
LogicalRelation(ParquetTable("users"), [id, name, age])
```

**Step 2: Planner starts with the root — `Project`**
- Strategies tried in order: `LogicalQueryStageStrategy` -> no match, `DataSourceV2Strategy` -> no match, ..., `BasicOperators` -> **match!**
```scala
case logical.Project(projectList, child) =>
  ProjectExec(projectList, planLater(child)) :: Nil
```
Result: `ProjectExec([name], PlanLater(Filter(...)))`

**Step 3: Planner finds the `PlanLater` placeholder, recursively plans `Filter`**
- `BasicOperators` matches again:
```scala
case logical.Filter(condition, child) =>
  FilterExec(condition, planLater(child)) :: Nil
```
Result: `FilterExec(age > 25, PlanLater(LogicalRelation(...)))`

**Step 4: Planner recursively plans `LogicalRelation`**
- `FileSourceStrategy` matches (Parquet is a file source). It creates a `FileSourceScanExec` — a `LeafExecNode` with no children. No more `planLater` placeholders.

**Step 5: Assemble the final physical tree**
```
ProjectExec([name])                    <- UnaryExecNode
    |
FilterExec(age > 25)                   <- UnaryExecNode
    |
FileSourceScanExec("users", [id,name,age])  <- LeafExecNode
```

**Step 6: Preparation rules add execution details**
```
WholeStageCodegenExec                  <- codegen wrapper
    |
  ProjectExec([name])
      |
  FilterExec(age > 25)
      |
  FileSourceScanExec("users")
```

**Step 7: `execute()` produces an RDD**
- `FileSourceScanExec.execute()` -> RDD that reads Parquet files
- `FilterExec.execute()` -> wraps the child RDD with a `.filter()` on each row
- `ProjectExec.execute()` -> wraps with a `.map()` that keeps only `name`
- `WholeStageCodegenExec` -> compiles all of the above into a single Java method for speed

### Visual Summary

```
                    LOGICAL WORLD                    PHYSICAL WORLD
                    (the "what")                     (the "how")

                    Project(name)       ------>    ProjectExec(name)
                        |              Strategy        |
                    Filter(age>25)      ------>    FilterExec(age>25)
                        |              Strategy        |
                    LogicalRelation     ------>    FileSourceScanExec
                    ("users")          Strategy    (reads Parquet files)

                        ^                              ^
                    UnaryNode                      UnaryExecNode
                    (1 child)                      (1 child)

                    LeafNode                       LeafExecNode
                    (0 children)                   (0 children)
```

The key insight: **the tree shape is preserved**. A unary logical node becomes a unary physical node. A binary Join becomes a binary BroadcastHashJoinExec or SortMergeJoinExec. The planning step translates each node while keeping the same parent-child relationships.

## Related Notes

- [[project_spark_sql_parsing_explained]] — Previous step: SQL string to unresolved LogicalPlan
- [[project_spark_optimizer_rule_execution]] — Optimizer transforms analyzed plan
- [[project_spark_physical_planning_walkthrough]] — Deeper dive into physical planning
- [[project_whole_stage_codegen_explained]] — Next step: codegen wraps the physical plan
- [[project_spark_sql_architecture]] — Overall architecture context
- [[spark-sql-planning-classes]] — Class diagram of TreeNode/QueryPlan hierarchy
