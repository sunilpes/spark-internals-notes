---
name: Spark UDF/Lambda Physical Planning
description: How Scala and Python UDFs integrate into physical plans — Scala inlined in codegen, Python extracted to separate operators (BatchEvalPythonExec/ArrowEvalPythonExec), serialization paths, performance comparison
type: project
tags: [spark-execution, spark-sql]
---

## How Lambda/UDF Functions Get Into Physical Plans

### Scala UDFs — Inline in JVM, No Separate Operator

```
1. udf() wraps lambda as ScalaUDF expression
2. ScalaUDF stays INSIDE Project/Filter nodes (not a separate operator)
3. WholeStageCodegen inlines via ScalaUDF.doGenCode()
   → function reference cached in CodegenContext
   → no serialization, no process spawning
```

Physical plan:
```
WholeStageCodegenExec
└── ProjectExec [ScalaUDF(name) AS upper_name]  ← UDF is INSIDE the Project
    └── FileSourceScanExec
```

### Python UDFs — Separate Operator, External Process

```
1. Python pickles function → sends bytecode to JVM as PythonUDF
2. ExtractPythonUDFs rule extracts into SEPARATE logical node:
   Project [PythonUDF(name)] → Project + BatchEvalPython node
3. PythonEvals strategy converts to physical:
   BatchEvalPython → BatchEvalPythonExec (Pickle)
   ArrowEvalPython → ArrowEvalPythonExec (Arrow)
4. Each partition spawns Python worker, data flows via socket
```

Physical plan:
```
WholeStageCodegenExec
└── ProjectExec [upper_name]
    └── BatchEvalPythonExec [PythonUDF(name)]  ← SEPARATE operator
        └── InputAdapter                        ← codegen breaks here
            └── WholeStageCodegenExec
                └── FileSourceScanExec
```

### Python Execution Architecture

```
JVM Task                          Python Worker (per partition)
  Batch rows                        ↑
  Serialize (Pickle/Arrow)          │
  Socket ──────────────────→  Deserialize → Execute UDF → Serialize
  Socket ←──────────────────  Send back
  Deserialize, continue pipeline
```

### Python UDF Types

| Type | Physical Node | Serialization | Speed |
|------|--------------|---------------|-------|
| Regular `@udf` | BatchEvalPythonExec | Pickle (row-at-a-time) | Slowest |
| `@pandas_udf` | ArrowEvalPythonExec | Arrow (columnar batches) | ~100x faster |
| `@udf(useArrow=True)` | ArrowEvalPythonExec | Arrow (columnar batches) | ~100x faster |

### Performance Comparison

| | Scala UDF | Python UDF | Python Pandas UDF |
|--|-----------|-----------|-------------------|
| Physical operator | None (inlined) | BatchEvalPythonExec | ArrowEvalPythonExec |
| Process | Same JVM | Separate Python | Separate Python |
| Serialization | None | Pickle (per row) | Arrow (per batch) |
| Codegen | Yes (inlined) | No (breaks codegen) | No (breaks codegen) |
| Cost per row | ~10ns | ~100μs | ~1μs |

### Key Source Files

| File | What |
|------|------|
| `catalyst/.../expressions/ScalaUDF.scala` | Scala UDF expression + codegen |
| `execution/python/ExtractPythonUDFs.scala:162` | Extracts Python UDFs into separate nodes |
| `execution/SparkStrategies.scala:860` | PythonEvals strategy |
| `execution/python/BatchEvalPythonExec.scala` | Pickle-based execution |
| `execution/python/ArrowEvalPythonExec.scala` | Arrow-based execution |
| `execution/python/PythonUDFRunner.scala` | Python worker management |

## Related Notes

- [[project_spark_physical_planning_walkthrough]] — Physical planning context
- [[project_whole_stage_codegen_explained]] — Python UDFs break codegen boundaries
- [[project_spark_query_planning_caveats]] — UDF pushdown blocking, Python codegen breaks
- [[project_spark_sql_planning_explained]] — Where UDF planning fits in the pipeline
