---

name: Spark AST Tree Building
description: "How AST trees are built — two stages: ANTLR parse tree from grammar, then AstBuilder visitor converts to LogicalPlan bottom-up, with concrete SQL examples showing node creation"
type: project
tags: [spark, spark-sql, parsing, antlr, ast, catalyst]
---

#spark-sql #spark-optimization

## How AST Trees Are Built in Spark

### Two Stages

```
SQL string → ANTLR Lexer → Tokens → ANTLR Parser → ParseTree → AstBuilder → LogicalPlan (AST)
```

### Stage 1: ANTLR Generates Parse Tree

ANTLR uses grammar (`SqlBaseParser.g4`) to tokenize and parse into a concrete syntax tree that mirrors grammar rules:

```
singleStatement
└── regularQuerySpecification
    ├── selectClause → namedExpression("name")
    ├── fromClause → tableName("users")
    └── whereClause → comparison("age", ">", "25")
```

### Stage 2: AstBuilder Visits Parse Tree → LogicalPlan

AstBuilder extends ANTLR's `SqlBaseParserBaseVisitor`, overrides visit methods, walks bottom-up:

```
visitRegularQuerySpecification()          [AstBuilder.scala:1441]
  ├── visitFromClause()                   [line 1750]
  │   → UnresolvedRelation(["users"])
  ├── withWhereClause()                   [line 1498]
  │   → Filter(GreaterThan(UnresolvedAttribute("age"), Literal(25)), child)
  └── Project([UnresolvedAttribute("name")], child = Filter(...))
```

### Bottom-Up Construction Pattern

```
Step 1: UnresolvedRelation(["users"])              ← leaf
Step 2: Filter(age > 25, child = step1)            ← wraps step 1
Step 3: Project([name], child = step2)             ← wraps step 2
```

Each `visit*` method returns a LogicalPlan/Expression node. Caller wraps it as child.

### How Each Node Gets Created

**Table**: `visitTableName()` → `UnresolvedRelation(tableId)`
**WHERE**: `withWhereClause()` → `Filter(expression, childPlan)`
**Comparison**: `visitComparison()` → pattern match on operator → `GreaterThan/LessThan/EqualTo`
**SELECT**: wraps in `Project(namedExpressions, childPlan)`

### JOIN Example

```sql
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id WHERE u.age > 25
```

Built bottom-up:
```
Project [u.name, o.amount]
└── Filter (u.age > 25)
    └── Join (Inner, u.id = o.user_id)
        ├── SubqueryAlias "u" → UnresolvedRelation ["users"]
        └── SubqueryAlias "o" → UnresolvedRelation ["orders"]
```

### Why "Unresolved"?

At parse time, Spark doesn't know if tables/columns exist or their types. Everything is `Unresolved*`. The Analyzer resolves them:

```
UnresolvedRelation(["users"]) → LogicalRelation(HadoopFsRelation)
UnresolvedAttribute("name")   → AttributeReference("name", StringType)
```

### Key Distinction

The AST is Spark's `LogicalPlan` tree, NOT ANTLR's parse tree. AstBuilder is the translator between the two.

## Related Notes
- [[project_spark_sql_parsing_explained]] — Full parsing walkthrough: ANTLR grammar/lexer/parser, AstBuilder visitor
- [[project_spark_sql_planning_explained]] — Next phase: how LogicalPlan becomes physical plan
- [[project_spark_sql_architecture]] — Overall SQL architecture and query execution pipeline
- [[project_sparksession_internals]] — Where sqlParser lives in SessionState
- [[project_spark_optimizer_rule_execution]] — How optimizer transforms the LogicalPlan after parsing
- [[project_spark_physical_planning_walkthrough]] — How the optimized LogicalPlan becomes a physical plan
