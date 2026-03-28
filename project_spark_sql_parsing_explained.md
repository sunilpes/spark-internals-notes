---
name: Spark SQL Parsing Explained
description: Complete walkthrough of how SQL string becomes a LogicalPlan — ANTLR grammar, Lexer, Parser, AstBuilder, UnresolvedRelation, and the parse tree to LogicalPlan conversion with source file references
type: project
---

## How Spark SQL Parsing Works — From SQL String to LogicalPlan

### What is ANTLR?

ANTLR (**AN**other **T**ool for **L**anguage **R**ecognition) is a parser generator. You give it a grammar file (`.g4`), and it generates Java/Scala code that can parse text matching that grammar.

Spark needs to turn SQL strings into tree structures. Writing a parser by hand for all of SQL is extremely complex. ANTLR automates this.

```
You write:   SqlBaseParser.g4  (grammar rules)
ANTLR generates:  SqlBaseParser.java  (parser code)
                   SqlBaseLexer.java   (tokenizer code)
```

ANTLR works in two stages:

**1. Lexer** — string to tokens (defined in `SqlBaseLexer.g4`):
```antlr
SELECT : 'SELECT';
FROM   : 'FROM';
WHERE  : 'WHERE';
NUMBER : [0-9]+;
IDENTIFIER : [a-zA-Z_][a-zA-Z0-9_]*;
```

**2. Parser** — tokens to parse tree (defined in `SqlBaseParser.g4`):
```antlr
querySpecification
    : selectClause fromClause? whereClause? ;

selectClause
    : SELECT namedExpressionSeq ;

fromClause
    : FROM relation ;

whereClause
    : WHERE booleanExpression ;
```

ANTLR's job **ends** at the parse tree. It knows nothing about databases, tables, or execution. Spark's **AstBuilder** then walks this parse tree and creates LogicalPlan nodes.

---

### Key Source Files

| Component | Path |
|-----------|------|
| SparkSqlParser (main entry) | `sql/core/src/main/scala/org/apache/spark/sql/execution/SparkSqlParser.scala` |
| CatalystSqlParser | `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/parser/CatalystSqlParser.scala` |
| AbstractSqlParser | `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/parser/AbstractSqlParser.scala` |
| AbstractParser (ANTLR base) | `sql/api/src/main/scala/org/apache/spark/sql/catalyst/parser/parsers.scala` |
| AstBuilder (visitor) | `sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/parser/AstBuilder.scala` |
| Grammar (parser) | `sql/api/src/main/antlr4/org/apache/spark/sql/catalyst/parser/SqlBaseParser.g4` |
| Grammar (lexer) | `sql/api/src/main/antlr4/org/apache/spark/sql/catalyst/parser/SqlBaseLexer.g4` |

---

### Full Walkthrough: `SELECT name FROM users WHERE age > 25`

#### Stage 1: SQL String to ANTLR Tokens (Lexer)

```
SELECT  name  FROM  users  WHERE  age  >  25
  |      |     |     |      |     |   |   |
[KW]  [IDENT] [KW] [IDENT] [KW] [IDENT][>][NUM]
```

The `SqlBaseLexer` (ANTLR-generated) tokenizes the raw string. Keywords like `SELECT`, `FROM`, `WHERE` become keyword tokens. `name`, `users`, `age` become identifiers.

#### Stage 2: Tokens to Parse Tree (ANTLR Grammar)

The `SqlBaseParser` matches tokens against the grammar rules in `SqlBaseParser.g4`:

```
compoundOrSingleStatement
  └─ singleStatement
       └─ query
            └─ queryTerm
                 └─ queryPrimary
                      └─ querySpecification          <- this is the key rule
                           ├─ selectClause: SELECT name
                           ├─ fromClause:   FROM users
                           └─ whereClause:  WHERE age > 25
```

The grammar rule that matches:
```antlr
querySpecification
    : selectClause fromClause? whereClause? aggregationClause? ...
```

This is still just an ANTLR parse tree (syntax) — not a LogicalPlan yet.

#### Stage 3: Parse Tree to LogicalPlan (AstBuilder)

The **AstBuilder** (`AstBuilder.scala`, 7000+ lines) is an ANTLR visitor that walks the parse tree and creates LogicalPlan nodes. The exact call chain:

```
visitCompoundOrSingleStatement()
  -> visitSingleStatement()
    -> visitQuery()                            <- line 725
      -> plan(ctx.queryTerm)
        -> visitQuerySpecification()
          -> visitCommonSelectQueryClausePlan()   <- line 1590, the core method
```

Inside `visitCommonSelectQueryClausePlan`, three things happen **bottom-up**:

**3a. FROM clause -> UnresolvedRelation**

```scala
// AstBuilder.scala line ~4006
private def createUnresolvedRelation(ctx, ident, ...): UnresolvedRelation = {
  new UnresolvedRelation(ident)   // ident = Seq("users")
}
```

Result: `UnresolvedRelation(Seq("users"))`

**3b. WHERE clause -> wraps in Filter**

```scala
// AstBuilder.scala line 1491
private def withWhereClause(ctx: WhereClauseContext, plan: LogicalPlan): LogicalPlan = {
  Filter(expression(ctx.booleanExpression), plan)
}
```

`expression(ctx.booleanExpression)` parses `age > 25` into:
```
GreaterThan(UnresolvedAttribute("age"), Literal(25))
```

Result: `Filter(age > 25, UnresolvedRelation("users"))`

**3c. SELECT clause -> wraps in Project**

```scala
// AstBuilder.scala line ~1632
Project(namedExpressions, withFilter)
```

`namedExpressions` = `Seq(UnresolvedAttribute("name"))`

Result: `Project([name], Filter(age > 25, UnresolvedRelation("users")))`

#### Final Parsed Tree

```
Project([UnresolvedAttribute("name")])
    |
Filter(GreaterThan(UnresolvedAttribute("age"), Literal(25)))
    |
UnresolvedRelation(Seq("users"))
```

Everything is **unresolved** — column names are just strings, the table is just a name. The Analyzer will resolve them next.

---

### Visual Summary of the Full Pipeline

```
"SELECT name FROM users WHERE age > 25"
        |
        v  SqlBaseLexer
   [SELECT][name][FROM][users][WHERE][age][>][25]
        |
        v  SqlBaseParser (ANTLR grammar rules)
   querySpecification
     +- selectClause(name)
     +- fromClause(users)
     +- whereClause(age > 25)
        |
        v  AstBuilder (visitor pattern)
   1. FROM   -> UnresolvedRelation("users")
   2. WHERE  -> Filter(age > 25, ^)
   3. SELECT -> Project([name], ^)
        |
        v  Output: Unresolved LogicalPlan tree
   Project([name])
       |
   Filter(age > 25)
       |
   UnresolvedRelation("users")
```

The tree is built **inside-out** — FROM first (bottom), then WHERE wraps it, then SELECT wraps that. This matches how data flows at execution time: read table -> filter rows -> pick columns.

---

### What is UnresolvedRelation?

`UnresolvedRelation` is a **placeholder** LeafNode. At parse time, Spark doesn't know:
- Whether the table actually exists
- What columns it has
- Where the data is stored (Parquet, CSV, Hive, etc.)

It's literally just: "the user mentioned a table called 'users'".

From `unresolved.scala:116`:
```scala
case class UnresolvedRelation(
    multipartIdentifier: Seq[String],       // supports db.table, catalog.db.table
    options: CaseInsensitiveStringMap,
    override val isStreaming: Boolean = false)
  extends UnresolvedLeafNode with NamedRelation
```

It handles any named dataset reference:
- Tables: `FROM users` -> `Seq("users")`
- Qualified tables: `FROM mydb.users` -> `Seq("mydb", "users")`
- Views: `FROM active_users_view`
- CTEs: `WITH cte AS (...) SELECT * FROM cte`
- Temp views: created via `df.createTempView("myview")`
- Streaming sources: when `isStreaming = true`

#### Why not skip straight to LogicalRelation?

1. **Parser doesn't know what "users" is** — could be a Parquet table, CSV table, Hive table, temp view, CTE, or something that doesn't exist
2. **Separation of concerns** — Parser handles syntax (is this valid SQL?), Analyzer handles semantics (do these tables/columns exist?)
3. **Same SQL resolves differently** — depending on session context, "users" could be a temp view, Hive table, or Parquet table
4. **Analysis rules run in order** — CTE resolution, temp view resolution, and catalog lookup take turns matching the same UnresolvedRelation

During the Analyze phase, the Analyzer looks up "users" in the catalog and replaces it:
```
UnresolvedRelation("users")
        |  Analyzer resolves it
        v
LogicalRelation(ParquetTable("users"), schema=[id, name, age])
```

---

### Class Hierarchy Diagram

A Mermaid class diagram is saved separately at:
`spark-sql-planning-classes.mmd` (in this same memory directory)

It shows: TreeNode -> QueryPlan -> LogicalPlan/SparkPlan -> LeafNode/UnaryNode/BinaryNode -> concrete operators, and how Strategy classes bridge logical to physical nodes.
