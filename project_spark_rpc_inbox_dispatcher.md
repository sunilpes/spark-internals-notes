---
name: Spark RPC Inbox and Dispatcher Flow
description: How Spark's Netty RPC system processes messages — Dispatcher, MessageLoop, Inbox, thread pools, and end-to-end flow from network to RpcEndpoint
type: project
---

## Who Processes Inbox Messages — The Complete Flow

The **`Dispatcher`** and **`MessageLoop`** drive the processing. Here's the chain:

### End-to-End Flow

```
Network (Netty I/O thread)
  ↓
NettyRpcHandler.receive()          ← deserializes bytes → RequestMessage
  ↓
Dispatcher.postRemoteMessage()     ← wraps as RpcMessage/OneWayMessage
  ↓
Dispatcher.postMessage()           ← looks up endpoint's MessageLoop
  ↓
MessageLoop.post()                 ← adds message to Inbox, puts Inbox on active queue
  ↓
receiveLoop() [thread pool]        ← blocks on active.take(), calls inbox.process()
  ↓
Inbox.process()                    ← pattern-matches message, calls endpoint method
  ↓
RpcEndpoint.receive() / receiveAndReply()   ← your actual business logic
```

### The Key Players

**1. `NettyRpcHandler`** (`NettyRpcEnv.scala:677`)
- Netty I/O thread calls this when bytes arrive on the wire
- Deserializes → `RequestMessage`, then hands off to Dispatcher

**2. `Dispatcher`** (`Dispatcher.scala`)
- The **router** — maps endpoint names to their `MessageLoop`
- `endpoints: ConcurrentMap[String, MessageLoop]` (line 41)
- `postMessage()` (line 172) — finds the right MessageLoop and calls `loop.post()`

**3. `MessageLoop`** (`MessageLoop.scala`) — **this is the engine**

Two variants:

| Type | Used by | Thread pool |
|------|---------|-------------|
| `SharedMessageLoop` (line 103) | Most endpoints | `dispatcher-event-loop` pool, `max(2, numCores)` threads |
| `DedicatedMessageLoop` (line 160) | `IsolatedRpcEndpoint` | Own pool, `endpoint.threadCount()` threads |

The core loop (line 66):
```scala
private def receiveLoop(): Unit = {
  while (true) {
    val inbox = active.take()      // BLOCKS here waiting for work
    inbox.process(dispatcher)      // processes one batch of messages
  }
}
```

`active` is a `LinkedBlockingQueue[Inbox]` — thread pool threads block on `take()` until an Inbox has messages.

**4. `Inbox`** — the per-endpoint queue

### Visual Architecture

```
Dispatcher
├── endpoints: Map[String → MessageLoop]
│
├── SharedMessageLoop (most endpoints share this)
│   ├── thread pool: "dispatcher-event-loop" (N threads)
│   │   ├── Thread-1 → receiveLoop() → active.take() → inbox.process()
│   │   ├── Thread-2 → receiveLoop() → active.take() → inbox.process()
│   │   └── Thread-N → ...
│   ├── active: BlockingQueue[Inbox]  ← inboxes with pending messages
│   └── endpoints:
│       ├── "HeartbeatReceiver" → Inbox → [msg, msg, ...]
│       ├── "MapOutputTracker" → Inbox → [msg, ...]
│       └── "BlockManagerMaster" → Inbox → [msg, ...]
│
└── DedicatedMessageLoop (isolated endpoints)
    ├── "CoarseGrainedScheduler" → own thread pool → own Inbox
    └── etc.
```

### How control flows for a typical RPC

Say an executor sends a `StatusUpdate` to the driver:

```
1. Executor's Outbox serializes & sends over network
2. Driver's Netty I/O thread → NettyRpcHandler.receive()
3. Deserialize → RequestMessage(sender, "CoarseGrainedScheduler", StatusUpdate)
4. Dispatcher.postRemoteMessage() → wraps as OneWayMessage
5. Dispatcher.postMessage("CoarseGrainedScheduler", msg)
6. endpoints.get("CoarseGrainedScheduler") → DedicatedMessageLoop
7. loop.post() → inbox.post(msg) + setActive(inbox)
8. Thread in dedicated pool unblocks from active.take()
9. inbox.process() → pattern match OneWayMessage
10. CoarseGrainedSchedulerBackend.receive(StatusUpdate)
    → updates task status, launches new tasks, etc.
```

### Key Source Files

| Component | File | Key Lines |
|-----------|------|-----------|
| Network Input | `core/.../rpc/netty/NettyRpcEnv.scala` | 677-717 |
| Message Dispatch | `core/.../rpc/netty/Dispatcher.scala` | 134-189 |
| Endpoint Registry | `core/.../rpc/netty/Dispatcher.scala` | 41-47 |
| Shared Loop | `core/.../rpc/netty/MessageLoop.scala` | 103-155 |
| Dedicated Loop | `core/.../rpc/netty/MessageLoop.scala` | 160-199 |
| Inbox Processing | `core/.../rpc/netty/Inbox.scala` | 86-164 |
| Outbox (sending) | `core/.../rpc/netty/Outbox.scala` | 95-283 |

### Outbox (sending side)

`Outbox.scala` handles the **reverse direction** — buffering outbound messages until a network connection is established. Each remote address gets one `Outbox`. It's not involved in receiving/processing.
