---
name: SparkSession Internals
description: Complete breakdown of SparkSession — constructor params, SharedState, SessionState, and user-facing APIs with types and purposes
type: project
---

# SparkSession — What's Inside

SparkSession has **two levels of state**: shared (across sessions) and per-session.

## Constructor Parameters
```scala
class SparkSession(
  sparkContext: SparkContext,          // the underlying core engine
  existingSharedState: Option[SharedState],
  parentSessionState: Option[SessionState],
  extensions: SparkSessionExtensions,  // custom rules, parsers, etc.
  initialSessionOptions: Map[String, String],
  parentManagedJobTags: Map[String, String]
)
```

## SharedState (shared across sessions on the same SparkContext)

| Field | Type | What it does |
|-------|------|-------------|
| `cacheManager` | `CacheManager` | Tracks cached DataFrames/tables |
| `statusStore` | `SQLAppStatusStore` | SQL execution metrics for UI |
| `externalCatalog` | `ExternalCatalogWithListener` | Persistent metadata (Hive metastore or in-memory) |
| `globalTempViewManager` | `GlobalTempViewManager` | Cross-session temp views |
| `jarClassLoader` | `MutableURLClassLoader` | Dynamic JAR loading |
| `streamingQueryStatusListener` | `StreamingQueryStatusListener` | Streaming query metrics |

## SessionState (per-session, isolated)

| Field | Type | What it does |
|-------|------|-------------|
| `conf` | `SQLConf` | SQL config (e.g., `spark.sql.shuffle.partitions`) |
| `catalog` | `SessionCatalog` | Tables, views, functions for this session |
| `catalogManager` | `CatalogManager` | V2 catalog routing |
| `sqlParser` | `ParserInterface` | SQL string → LogicalPlan |
| `analyzer` | `Analyzer` | Resolves unresolved plans (column names, tables) |
| `optimizer` | `Optimizer` | Logical plan optimization rules |
| `planner` | `SparkPlanner` | Logical → Physical plan |
| `streamingQueryManager` | `StreamingQueryManager` | Manages active streaming queries |
| `listenerManager` | `ExecutionListenerManager` | Query execution listeners |
| `resourceLoader` | `SessionResourceLoader` | Load JARs/files |
| `artifactManager` | `ArtifactManager` | Session-scoped JARs/classes |
| `experimentalMethods` | `ExperimentalMethods` | Hook into planner with custom strategies |
| `udfRegistration` | `UDFRegistration` | Register UDFs |
| `udtfRegistration` | `UDTFRegistration` | Register UDTFs |

## User-Facing APIs (methods on SparkSession)

| Method | Returns | Purpose |
|--------|---------|---------|
| `sql("...")` | `DataFrame` | Run SQL |
| `read` | `DataFrameReader` | Read from data sources |
| `readStream` | `DataStreamReader` | Read streaming sources |
| `table("name")` | `DataFrame` | Load a table by name |
| `catalog` | `Catalog` | Browse databases/tables/functions |
| `createDataFrame(...)` | `DataFrame` | From RDD, Seq, or Java List |
| `range(...)` | `DataFrame` | Generate sequence of numbers |
| `udf` | `UDFRegistration` | Register UDFs |
| `streams` | `StreamingQueryManager` | Manage streaming queries |
| `conf` | `RuntimeConfig` | Get/set config |
| `newSession()` | `SparkSession` | New session sharing same SparkContext |

## Visual Summary

```
SparkSession
├── sparkContext (SparkContext)          ← the core engine (scheduler, cluster manager)
├── sharedState (shared across sessions)
│   ├── externalCatalog                 ← persistent metadata (Hive metastore)
│   ├── globalTempViewManager           ← cross-session temp views
│   ├── cacheManager                    ← cached tables
│   └── jarClassLoader                  ← dynamic JARs
├── sessionState (per-session)
│   ├── conf (SQLConf)                  ← SQL settings
│   ├── catalog (SessionCatalog)        ← tables, views, functions
│   ├── sqlParser                       ← SQL → LogicalPlan
│   ├── analyzer                        ← resolve names, types
│   ├── optimizer                       ← optimize logical plan
│   ├── planner                         ← logical → physical plan
│   ├── streamingQueryManager           ← streaming queries
│   └── udfRegistration                 ← user-defined functions
└── user APIs: sql(), read, table(), catalog, range(), etc.
```

The key insight: `SharedState` holds things that are **expensive and shared** (metastore connection, cache), while `SessionState` holds things that are **per-session and isolated** (configs, temp views, the full query compilation pipeline).
