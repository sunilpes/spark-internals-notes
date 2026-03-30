---
name: WholeStageCodegenExec Explained
description: How WholeStageCodegenExec works — the produce/consume protocol, how operators contribute Java code, InputAdapter boundaries, CollapseCodegenStages rule, and generated Java code examples with source file references
type: project
tags: [spark-sql, spark-execution]
---

## What WholeStageCodegenExec Actually Does

**Short answer**: No, it doesn't "contain" the query plan. It **wraps** a subtree of operators and **fuses them into a single Java function** by asking each operator to contribute Java code. The operators still exist as a tree — codegen just eliminates the overhead of calling between them.

### The Problem Codegen Solves

Without codegen, each operator is a separate Java object. Data flows between them via virtual method calls:

```
Without codegen (Volcano model):
  ProjectExec.next()
    -> calls FilterExec.next()
      -> calls ScanExec.next()
        -> returns Row
      -> evaluates condition on Row    <- virtual dispatch
      -> returns Row
    -> projects columns from Row        <- virtual dispatch
    -> returns Row

Per row: 3 virtual calls, 3 object boundaries, no JIT optimization across them
```

### What Codegen Does: Fuses Everything Into One Function

At `WholeStageCodegenExec.scala:665`, `doCodeGen()` generates a **single Java class**:

```scala
def doCodeGen(): (CodegenContext, CodeAndComment) = {
  val ctx = new CodegenContext
  val code = child.asInstanceOf[CodegenSupport].produce(ctx, this)  // <- ask child tree to emit code

  // Wrap all the generated code in ONE function
  ctx.addNewFunction("processNext",
    s"""
      protected void processNext() throws java.io.IOException {
        ${code.trim}       // <- ALL operators fused into this single method
      }
     """, inlineToOuterClass = true)

  // Generate a single Java class
  val source = s"""
    final class $className extends BufferedRowIterator {
      private Object[] references;
      private scala.collection.Iterator[] inputs;
      ${ctx.declareMutableStates()}        // <- fields for all operators combined

      public void init(int index, scala.collection.Iterator[] inputs) { ... }

      ${ctx.declareAddedFunctions()}       // <- helper functions
    }
  """
}
```

### The produce/consume Protocol — How Operators Contribute Code

Each operator implements two methods that **generate Java source code as strings** (not execute logic):

```
+-------------------------------------------------------------+
|  WholeStageCodegenExec calls child.produce()                |
|                                                             |
|  produce() goes DOWN the tree (top -> bottom):              |
|    ProjectExec.produce()                                    |
|      -> FilterExec.produce()                                |
|        -> InputAdapter.produce()   (reads from RDD)         |
|                                                             |
|  consume() goes UP the tree (bottom -> top):                |
|    InputAdapter: "here's a row variable"                    |
|      -> FilterExec.doConsume(): "if (condition) { ... }"    |
|        -> ProjectExec.doConsume(): "compute new columns"    |
|          -> WholeStageCodegen.doConsume(): "append to output"|
+-------------------------------------------------------------+
```

#### Concrete Example: `ProjectExec` at `basicPhysicalOperators.scala:54`

```scala
// produce: "I don't read data -- just ask my child to produce"
protected override def doProduce(ctx: CodegenContext): String = {
  child.asInstanceOf[CodegenSupport].produce(ctx, this)  // delegate downward
}

// consume: "When a row arrives from my child, compute the projected columns"
override def doConsume(ctx: CodegenContext, input: Seq[ExprCode], row: ExprCode): String = {
  val exprs = bindReferences[Expression](projectList, child.output)
  val resultVars = exprs.map(_.genCode(ctx))   // generate Java code for each expression
  // ... then call parent.consume() with new column variables
}
```

#### Concrete Example: `FilterExec` at `basicPhysicalOperators.scala:253`

```scala
// produce: "I don't read data -- just ask my child"
protected override def doProduce(ctx: CodegenContext): String = {
  child.asInstanceOf[CodegenSupport].produce(ctx, this)
}

// consume: "When a row arrives, check the condition, only pass it up if true"
override def doConsume(ctx: CodegenContext, input: Seq[ExprCode], row: ExprCode): String = {
  val predicateCode = generatePredicateCode(ctx, ...)
  s"""
     |do {
     |  $predicateCode                 // <- generated Java: if (age <= 25) continue;
     |  $numOutput.add(1);
     |  ${consume(ctx, resultVars)}    // <- call parent (ProjectExec) to consume
     |} while (false);
   """
}
```

### What the Generated Java Code Looks Like

For `SELECT name FROM users WHERE age > 25`, the generated `processNext()` method looks roughly like:

```java
// This is ONE function -- no virtual calls between operators!
protected void processNext() throws java.io.IOException {
    while (scan_input.hasNext()) {
        InternalRow scan_row = (InternalRow) scan_input.next();

        // --- FilterExec's generated code ---
        int scan_age = scan_row.getInt(2);          // read age column
        if (!(scan_age > 25)) continue;             // filter condition

        // --- ProjectExec's generated code ---
        UTF8String project_name = scan_row.getUTF8String(1);  // read name column

        // --- WholeStageCodegenExec's generated code ---
        currentRows.add(unsafeRow(project_name));   // append to output
        if (shouldStop()) return;
    }
}
```

**Everything is inlined into one tight loop.** No virtual dispatch, no object boundaries, JIT can optimize the whole thing as one unit.

### doExecute() — Compilation and Execution at `WholeStageCodegenExec.scala:730`

```scala
override def doExecute(): RDD[InternalRow] = {
  val (ctx, cleanedSource) = doCodeGen()              // 1. Generate Java source code
  val (_, compiledCodeStats) = try {
    CodeGenerator.compile(cleanedSource)               // 2. Compile to JVM bytecode
  } catch {
    case NonFatal(_) if !Utils.isTesting && conf.codegenFallback =>
      return child.execute()                           // 3. Fallback if compilation fails
  }

  // 4. If method too large for JIT, fallback
  if (compiledCodeStats.maxMethodCodeSize > conf.hugeMethodLimit) {
    return child.execute()
  }

  val references = ctx.references.toArray
  val rdds = child.asInstanceOf[CodegenSupport].inputRDDs()
  // 5. Run the compiled code on each partition
  rdds.head.mapPartitionsWithIndex { (index, iter) =>
    val evaluator = evaluatorFactory.createEvaluator()
    evaluator.eval(index, iter)                        // runs processNext() in a loop
  }
}
```

### Where Does the Boundary Fall? (`InputAdapter`)

Not every operator supports codegen. The `CollapseCodegenStages` rule at `WholeStageCodegenExec.scala:906` walks the physical plan and:

1. **Wraps codegen-capable subtrees** in `WholeStageCodegenExec` (line 970-974)
2. **Wraps non-codegen operators** in `InputAdapter` (line 932-936) -- this is the **boundary** where codegen stops and the Volcano iterator model takes over

```
WholeStageCodegenExec    <- codegen boundary START
  ProjectExec            <- contributes Java code (doConsume)
    FilterExec           <- contributes Java code (doConsume)
      InputAdapter       <- codegen boundary END (reads from RDD iterator)
        ShuffleExchange  <- NOT codegen'd (runs as normal Volcano iterator)
          ...
```

`InputAdapter` at `WholeStageCodegenExec.scala:504` simply reads rows from the child's RDD:
```scala
case class InputAdapter(child: SparkPlan) extends UnaryExecNode with InputRDDCodegen {
  override def inputRDD: RDD[InternalRow] = child.execute()  // just call the child normally
}
```

### CollapseCodegenStages Rule at `WholeStageCodegenExec.scala:906`

```scala
case class CollapseCodegenStages(...) extends Rule[SparkPlan] {

  private def supportCodegen(plan: SparkPlan): Boolean = plan match {
    case plan: CodegenSupport if plan.supportCodegen =>
      val willFallback = plan.expressions.exists(_.exists(e => !supportCodegen(e)))
      val hasTooManyOutputFields = WholeStageCodegenExec.isTooManyFields(conf, plan.schema)
      val hasTooManyInputFields = plan.children.exists(p =>
        WholeStageCodegenExec.isTooManyFields(conf, p.schema))
      !willFallback && !hasTooManyOutputFields && !hasTooManyInputFields
    case _ => false
  }

  // Wraps non-codegen operators in InputAdapter
  private def insertInputAdapter(plan: SparkPlan): SparkPlan = plan match {
    case p if !supportCodegen(p) => InputAdapter(insertWholeStageCodegen(p))
    case j: SortMergeJoinExec =>
      j.withNewChildren(j.children.map(child => InputAdapter(insertWholeStageCodegen(child))))
    case p => p.withNewChildren(p.children.map(insertInputAdapter))
  }

  // Wraps codegen-capable subtrees in WholeStageCodegenExec
  private def insertWholeStageCodegen(plan: SparkPlan): SparkPlan = plan match {
    case plan: CodegenSupport if supportCodegen(plan) =>
      WholeStageCodegenExec(insertInputAdapter(plan))(codegenStageCounter.incrementAndGet())
    case other =>
      other.withNewChildren(other.children.map(insertWholeStageCodegen))
  }
}
```

### Visual Summary

```
Physical Plan (before codegen):

  ProjectExec(name)
      |
  FilterExec(age > 25)
      |
  ShuffleExchange           <- can't codegen (network shuffle)
      |
  FileSourceScanExec


After CollapseCodegenStages:

  WholeStageCodegenExec --------------------------+
    ProjectExec           } fused into ONE         |  single generated
      FilterExec          } Java function          |  Java class
        InputAdapter -----------------------------+
          ShuffleExchange                           <- separate stage
            WholeStageCodegenExec ----------------+
              FileSourceScanExec   } another      |  another generated
            --------------------------------------|  Java class
```

So `WholeStageCodegenExec` doesn't "have" the plan -- it **compiles its subtree** into a single Java function by having each operator contribute code snippets via `produce()`/`consume()`. The operators that can't codegen (shuffles, sorts with spill, etc.) become `InputAdapter` boundaries that feed rows into the codegen'd function via a normal iterator.

### Key Source Files

| Component | File | Lines |
|-----------|------|-------|
| CodegenSupport trait | WholeStageCodegenExec.scala | 47-422 |
| produce() | WholeStageCodegenExec.scala | 94-101 |
| consume() | WholeStageCodegenExec.scala | 153-207 |
| WholeStageCodegenExec | WholeStageCodegenExec.scala | 635-863 |
| doCodeGen() | WholeStageCodegenExec.scala | 665-722 |
| doExecute() | WholeStageCodegenExec.scala | 730-792 |
| InputAdapter | WholeStageCodegenExec.scala | 504-569 |
| CollapseCodegenStages | WholeStageCodegenExec.scala | 906-987 |
| ProjectExec codegen | basicPhysicalOperators.scala | 54-80 |
| FilterExec codegen | basicPhysicalOperators.scala | 249-276 |
| CodegenContext | CodeGenerator.scala | 137+ |

## Related Notes

- [[project_spark_physical_planning_walkthrough]] — Physical plan that codegen wraps
- [[project_spark_sql_planning_explained]] — How the physical plan was created
- [[project_spark_udf_physical_planning]] — Python UDFs break codegen boundaries
- [[project_spark_query_planning_caveats]] — Codegen limitations and fallbacks
