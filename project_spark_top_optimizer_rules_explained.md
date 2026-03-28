---
name: Spark Top Optimizer Rules Explained
description: Top 6 most impactful Catalyst optimizer rules with before/after tree node examples — PushDownPredicates, ColumnPruning, ConstantFolding, CollapseProject, BooleanSimplification, PruneFilters
type: project
---

## Top 6 Most Widely Used Optimizer Rules

### 1. PushDownPredicates (line 108)

Pushes filters **as close to the data source as possible** — fewer rows flow through the plan.

```
BEFORE:                              AFTER:
Project [name]                       Project [name]
└── Filter (age > 25)                └── Filter (age > 25)
    └── Project [name, age, id]          └── Relation users [name, age, id]
        └── Relation users                   ↑ filter pushed past Project
```

With joins — this is where it really matters:

```
BEFORE:                              AFTER:
Filter (users.age > 25)             Join (users.id = orders.uid)
└── Join (users.id = orders.uid)    ├── Filter (age > 25)        ← pushed down
    ├── Relation users              │   └── Relation users
    └── Relation orders             └── Relation orders
```

**Why it matters**: Filtering 1B rows down to 1M **before** a join vs after is the difference between minutes and hours.

---

### 2. ColumnPruning (line 1064)

Removes columns that **no one downstream needs**.

```
BEFORE:                              AFTER:
Project [name]                       Project [name]
└── Filter (age > 25)                └── Filter (age > 25)
    └── Relation users                   └── Project [name, age]     ← NEW: prunes "id"
         [id, name, age]                     └── Relation users [id, name, age]
```

**Why it matters**: For Parquet/ORC columnar formats, pruning columns means Spark **skips reading entire column chunks from disk**. If a table has 200 columns and you need 3, this is ~98% less I/O.

---

### 3. ConstantFolding (line 137)

Evaluates **constant expressions at compile time** instead of per-row at runtime.

```
BEFORE:                              AFTER:
Project [name, 1 + 2 AS x]          Project [name, 3 AS x]
└── Filter (age > 10 * 2 + 5)       └── Filter (age > 25)
    └── Relation users                   └── Relation users
```

**Why it matters**: Avoids computing the same constant expression for every row (millions/billions of times).

---

### 4. CollapseProject (line 118)

Merges **adjacent Project nodes** into one.

```
BEFORE:                              AFTER:
Project [name]                       Project [name]
└── Project [name, age]              └── Relation users
    └── Project [name, age, id]
        └── Relation users
```

**Why it matters**: Each Project node is an extra iterator in the execution pipeline. Collapsing them reduces overhead.

---

### 5. BooleanSimplification (line 142)

Simplifies **redundant boolean logic**.

```
BEFORE:                              AFTER:
Filter (a > 5 AND true)             Filter (a > 5)
Filter (a > 5 OR true)              Relation t              ← filter removed entirely!
Filter (a > 5 AND a > 5)            Filter (a > 5)          ← deduplicated
Filter (NOT (NOT (a > 5)))          Filter (a > 5)          ← double negation
Filter (a > 5 AND a > 3)            Filter (a > 5)          ← a>5 implies a>3
```

**Why it matters**: ORMs and code-generated SQL often produce redundant conditions. This cleans them up.

---

### 6. PruneFilters (line 146)

Removes **filters that are always true or always false** based on constraints.

```
BEFORE:                              AFTER:
Filter (id IS NOT NULL)              Relation users          ← filter removed!
└── Relation users                       (id is already NOT NULL per schema)

Filter (1 = 0)                       LocalRelation []       ← entire scan eliminated!
└── Join                                  (no rows possible)
    ├── Relation users
    └── Relation orders
```

**Why it matters**: Can eliminate entire subtrees of the plan when it proves no rows can match.

---

## All 6 Together — Example Query

```sql
SELECT name FROM users WHERE age > 25
```

```
PARSED:
Project [name]
└── Filter (age > 25)
    └── UnresolvedRelation [users]

AFTER ANALYSIS:
Project [name#10]
└── Filter (age#11 > 25)
    └── Relation users [id#9, name#10, age#11]

RULE 1 - PushDownPredicates: (nothing to push past — already at bottom)
RULE 2 - ColumnPruning: prune "id" — nobody needs it
Project [name#10]
└── Filter (age#11 > 25)
    └── Project [name#10, age#11]        ← NEW
        └── Relation users [id#9, name#10, age#11]

RULE 3 - CollapseProject: nothing to collapse yet
RULE 4 - ConstantFolding: "25" already a constant — nothing to fold
RULE 5 - BooleanSimplification: single condition — nothing to simplify
RULE 6 - PruneFilters: age can be > 25 — can't prune

FINAL OPTIMIZED:
Project [name#10]
└── Filter (age#11 > 25)
    └── Project [name#10, age#11]
        └── Relation users [id#9, name#10, age#11]
```

For a simple query, most rules are no-ops. The rules really shine on **complex queries** with joins, subqueries, and many columns.
