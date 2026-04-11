---
name: Spark and Consensus Algorithms
description: Spark does NOT use Raft/Paxos — single driver architecture, no leader election needed, driver dies = app fails, consensus only in surrounding infra (YARN/HDFS/ZK)
type: project
tags: [spark, distributed-systems, consensus, fault-tolerance]
---

#spark-core #spark-distributed

## Spark Does NOT Use Consensus Algorithms

### Why Not?

Single driver architecture — no leader election needed:

```
Driver (single point of control)
  ├── DAGScheduler       ← one instance
  ├── TaskScheduler      ← one instance
  ├── BlockManagerMaster ← one instance
  └── MapOutputTracker   ← one instance

Executors (stateless workers)
  ← just run tasks, report back, no peer coordination
```

Nothing to reach consensus on — driver is the single authority.

### When Driver Dies

Entire application fails. No failover, no election, no recovery. Restart the whole job.

Design trade-off: simple + fast > resilient. Spark jobs are batch/finite, not long-lived services.

### Where Consensus Exists in the Ecosystem

| Component | Consensus? | How |
|-----------|-----------|-----|
| Spark itself | No | Single driver |
| YARN ResourceManager | Yes (ZooKeeper) | RM HA leader election |
| HDFS NameNode | Yes (ZooKeeper) | NameNode HA, QJM (Paxos variant) |
| Spark Standalone Master | Yes (ZooKeeper) | Optional HA with ZK |
| Kafka | Yes (KRaft/ZK) | Broker leader election |

### One Exception: Standalone HA

Spark Standalone mode can use ZK for Master HA — but this is the cluster manager, not the execution engine. Driver still has no HA.

## Related Notes
- [[project_spark_distributed_systems_topics]] — Key distributed systems topics for Spark
- [[project_spark_rpc_inbox_dispatcher]] — Spark RPC internals (driver↔executor communication)
- [[project_spark_heartbeat_contents]] — Executor heartbeat for liveness detection
- [[project_spark_task_completion_tracking]] — How driver tracks task/stage/job completion
- [[project_spark_listener_bus]] — Driver-only event bus
