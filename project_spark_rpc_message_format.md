---
name: Spark RPC Message Format
description: Wire format of Spark Netty RPC messages — RequestMessage structure, serialization, actual case class messages (LaunchTask, StatusUpdate, Heartbeat), and concrete flow example
type: project
tags: [spark-core, spark-distributed]
---

## RPC Message Format

### Wire Format (what goes over the network)

`RequestMessage` (`NettyRpcEnv.scala:592`) — the envelope:

```
┌─────────────────────────────────────────────────┐
│ Sender Address                                   │
│   boolean (exists?) + UTF host + int port        │
├─────────────────────────────────────────────────┤
│ Receiver Address                                 │
│   boolean (exists?) + UTF host + int port        │
├─────────────────────────────────────────────────┤
│ Receiver Endpoint Name (UTF string)              │
│   e.g. "CoarseGrainedScheduler"                  │
├─────────────────────────────────────────────────┤
│ Content (Java-serialized case class)             │
│   e.g. LaunchTask, StatusUpdate, Heartbeat       │
└─────────────────────────────────────────────────┘
```

The header (addresses + name) is manually serialized for efficiency. The content uses Java serialization.

### The Actual Messages (case classes)

**Driver → Executor** (`CoarseGrainedClusterMessage.scala`):

```scala
case class LaunchTask(data: SerializableBuffer)

case class KillTask(taskId: Long, executor: String,
                    interruptThread: Boolean, reason: String)

case class UpdateExecutorsLogLevel(logLevel: String)
```

**Executor → Driver**:

```scala
case class RegisterExecutor(
    executorId: String,
    executorRef: RpcEndpointRef,
    hostname: String,
    cores: Int,
    logUrls: Map[String, String],
    attributes: Map[String, String],
    resources: Map[String, ResourceInformation],
    resourceProfileId: Int)

case class StatusUpdate(
    executorId: String,
    taskId: Long,
    state: TaskState,
    data: SerializableBuffer,
    taskCpus: Int,
    resources: Map[String, Map[String, Long]])
```

**Heartbeat** (`HeartbeatReceiver.scala:43`):

```scala
case class Heartbeat(
    executorId: String,
    accumUpdates: Array[(Long, Seq[AccumulatorV2[_, _]])],
    blockManagerId: BlockManagerId,
    executorUpdates: Map[(Int, Int), ExecutorMetrics])

case class HeartbeatResponse(reregisterBlockManager: Boolean)
```

### Concrete Example: LaunchTask flow

```
Driver sends LaunchTask to Executor-1:

1. RequestMessage created:
   senderAddress = RpcAddress("driver-host", 7077)
   receiver = NettyRpcEndpointRef("CoarseGrainedExecutorBackend" @ Executor-1)
   content = LaunchTask(serializedTaskBytes)

2. Serialized to ByteBuffer:
   [true]["driver-host"][7077]          ← sender address
   [true]["executor1-host"][54321]      ← receiver address
   ["CoarseGrainedExecutorBackend"]     ← endpoint name
   [Java-serialized LaunchTask object]  ← content

3. Wrapped in OutboxMessage, sent via Netty TransportClient

4. Executor-1 receives bytes:
   NettyRpcHandler.receive() → deserialize → RequestMessage
   → Dispatcher.postRemoteMessage() → wraps as RpcMessage
   → Inbox["CoarseGrainedExecutorBackend"].post(msg)
   → inbox.process() → endpoint.receive(LaunchTask(...))
   → CoarseGrainedExecutorBackend runs the task
```

### Two Outbox Message Types

```scala
// Fire-and-forget (one-way)
case class OneWayOutboxMessage(content: ByteBuffer)

// Expects a reply (two-way)
case class RpcOutboxMessage(
    content: ByteBuffer,
    _onFailure: (Throwable) => Unit,
    _onSuccess: (TransportClient, ByteBuffer) => Unit)
```

All messages extend `CoarseGrainedClusterMessage` which is just a sealed trait — the actual data is in the case class fields, serialized via Java serialization.

## Related Notes

- [[project_spark_rpc_inbox_dispatcher]] — How messages are dispatched after deserialization
- [[project_spark_heartbeat_contents]] — Heartbeat message structure
- [[project_spark_task_completion_tracking]] — StatusUpdate and LaunchTask message flows
