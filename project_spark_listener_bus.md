---
name: Spark Listener Bus
description: How SparkListenerBus works — driver-only async event bus, event producers (DAGScheduler, TaskScheduler), consumers (SparkUI, EventLogging), routing via pattern matching
type: project
tags: [spark-core, spark-distributed]
---

## SparkListenerBus — Driver-Only Event Bus

`SparkListenerBus` runs **only on the driver**. It dispatches lifecycle events to registered listeners.

### How It Works

Simple **pattern-match dispatcher** — receives events and routes them to the right listener method:

```
Event arrives → doPostEvent() → match event type → call listener.onXxx()

SparkListenerJobStart       → listener.onJobStart()
SparkListenerTaskEnd        → listener.onTaskEnd()
SparkListenerStageCompleted → listener.onStageCompleted()
```

### Who Produces Events?

Driver's internal components fire events:

```
DAGScheduler        → SparkListenerJobStart, SparkListenerStageSubmitted
TaskScheduler       → SparkListenerTaskStart, SparkListenerTaskEnd
BlockManager        → SparkListenerBlockManagerAdded
SparkContext        → SparkListenerApplicationStart/End
ExecutorAllocation  → SparkListenerExecutorAdded/Removed
```

### Who Consumes Them? (Listeners)

```
SparkUI / Web UI           → renders the Spark UI tabs (jobs, stages, SQL)
EventLoggingListener       → writes event log for Spark History Server
SQLAppStatusListener       → tracks SQL execution metrics
StreamingQueryListener     → streaming progress updates
Your custom listener       → spark.addSparkListener(new MyListener)
```

### Architecture

```
Driver only:

DAGScheduler ─────┐
TaskScheduler ────┤
BlockManager ─────┤  events    ┌──────────────────┐    ┌── SparkUI
SparkContext ─────┼──────────→│ SparkListenerBus   │───┤── EventLoggingListener
ExecutorAlloc ────┤           │ (async queue +     │   ├── SQLAppStatusListener
                  │           │  dispatch thread)  │   └── YourCustomListener
                  └           └──────────────────┘
```

### Async Dispatch

The actual queuing comes from `ListenerBus` and `AsyncEventQueue`:

```
Event → AsyncEventQueue (LinkedBlockingQueue) → dispatch thread → doPostEvent() → listener.onXxx()
```

Events posted **asynchronously** so they don't block the scheduler. Queue fills up at 10,000 events → drops with warning.

### Why Only Driver?

Executors don't need lifecycle events — they just run tasks. Executor metrics flow to driver via RPC (StatusUpdate, Heartbeat), and the driver converts them into SparkListenerEvents for the bus.

### Key Source File

`core/src/main/scala/org/apache/spark/scheduler/SparkListenerBus.scala`

## Related Notes

- [[project_spark_heartbeat_contents]] — Executor metrics forwarded to listener bus
- [[project_spark_task_completion_tracking]] — Task/stage events posted to listener bus
- [[project_spark_rpc_inbox_dispatcher]] — RPC system that delivers executor updates to driver
