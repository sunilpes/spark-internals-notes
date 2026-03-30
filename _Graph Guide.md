---
tags: [spark-sql]
---

# Graph View Guide

This vault is structured for Obsidian's graph view to show how Spark internals connect.

## Opening the Graph

- **Global graph**: Press `Ctrl+G` (or `Cmd+G` on Mac)
- **Local graph**: Open any note, then click the "Open local graph" icon in the right sidebar

## Color Coding

The graph uses tag-based coloring (pre-configured in `.obsidian/graph.json`):

| Color | Tag | Topic Area |
|-------|-----|-----------|
| Blue | `#spark-sql` | SQL/Catalyst (parsing, planning, optimizer, codegen) |
| Orange | `#spark-core` | Core internals (RPC, scheduler, heartbeat, tasks) |
| Green | `#spark-storage` | Storage (BlockManager, memory, disk, caching) |
| Magenta | `#spark-aqe` | Adaptive Query Execution |
| Yellow | `#spark-joins` | Join strategy, skew handling |
| Cyan | `#spark-optimization` | Optimizer rules, CBO, format-aware |
| Red-orange | `#spark-distributed` | Distributed systems concepts |
| Purple | `#spark-execution` | Physical execution, UDF, codegen |

Notes can have multiple tags, so they may appear in overlapping clusters.

## Key Clusters to Explore

### SQL Pipeline (longest chain)
Start from `project_spark_sql_parsing_explained` and follow the links forward through planning, optimization, physical planning, and codegen. This is the complete query lifecycle.

### AQE Chain
Start from `project_spark_aqe_stats_flow` and follow through stage splitting, skew handling, constraints, and workarounds.

### Storage Cluster
`project_spark_blockmanager_internals` is the hub -- it connects to RDD blocks, memory unrolling, and file processing.

### RPC Cluster
`project_spark_rpc_inbox_dispatcher` connects to message format, heartbeats, and task tracking.

## Tips

- **Filter by tag**: In graph view, type `tag:#spark-aqe` in the search box to isolate AQE notes
- **Local graph depth**: Increase depth to 2 or 3 to see connections-of-connections
- **Use the MOC**: [[_MOC]] is the central hub -- it links to everything by topic
- **Hover for preview**: Hover over nodes in the graph to see note previews
