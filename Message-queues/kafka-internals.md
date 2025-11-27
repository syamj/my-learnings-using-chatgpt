
---

## 1. Kafka Internals – What’s Inside a Cluster

### 1.1 Core Roles in a Modern Kafka Cluster

In current Kafka (2.8+), ZooKeeper is deprecated; Kafka can self-manage metadata using **KRaft** (Kafka Raft).

You’ll see three logical roles:

* **Broker**

  * Stores partitions (logs), handles client requests (produce/fetch).
* **Controller** (KRaft)

  * Manages cluster metadata: who is leader for which partition, topic configs, broker membership, etc.
  * Multiple controllers exist, but one is **active**, others are **standby** in a Raft quorum.
* **Clients**

  * Producers, consumers, admin tools (kafka-topics.sh, etc.)

In small clusters, the same nodes usually act as **broker+controller**.

---

### 1.2 Topic, Partition, Replica, ISR

* **Topic**: logical stream (`payments`, `orders`, `logs`).
* **Partition**: an ordered, append-only log.
* **Replica**: copy of a partition on a different broker.
* **Leader replica**: handles all reads/writes.
* **Follower replicas**: async copy from leader.
* **ISR (In-Sync Replicas)**: replicas that are caught up to the leader within a threshold (`replica.lag.time.max.ms`).

**Write path**:

1. Producer sends record → topic, partition (leader).
2. Leader appends to local log segment (sequential write).
3. Followers fetch new data and append.
4. Depending on `acks`:

   * `acks=1`: leader only.
   * `acks=all`: waits for ISR to acknowledge.

If leader fails → controller elects a **new leader** from ISR.

---

### 1.3 Log, Segments, Indexes

Each partition on disk:

```text
/logs/payments-0/
  00000000000000000000.log
  00000000000000000000.index
  00000000000000000000.timeindex
  00000000000000100000.log
  ...
```

* **.log**: actual messages.
* **.index**: maps logical offsets → physical file positions.
* **.timeindex**: maps timestamps → file positions (for time-based lookups and retention).

Segments are rolled by size (`log.segment.bytes`) or time. Retention works by deleting old segments (`log.retention.hours` or `log.retention.bytes`).

---

### 1.4 Request Flow (Produce / Fetch)

**Produce path:**

1. Client resolves **bootstrap server** → gets metadata → knows which broker is leader for which partition.
2. Producer batches records per partition.
3. Sends `ProduceRequest` to the leader broker.
4. Broker appends to log, updates indexes.
5. Broker waits for follower replication (if `acks=all`) → responds.

**Fetch path (consumer):**

1. Consumer joins **consumer group**.
2. Group coordinator assigns partitions.
3. Consumer sends `FetchRequest` to broker leader with offset.
4. Broker reads from log/OS page cache and returns records.
5. Consumer periodically commits offsets (Kafka topic `__consumer_offsets` or external store).

---

### 1.5 Metadata & KRaft

In KRaft mode:

* **Metadata quorum** is a Raft cluster (odd number of controllers).
* Metadata includes:

  * Topics, partitions, replication factor
  * Broker registrations and heartbeats
  * ACLs, configs, etc.
* Changes (create topic, add broker, elect leader) → written as entries in Raft log → replicated to controllers.

If active controller dies → Raft elects a new leader.

---

### 1.6 Offsets & Consumer Groups

* Each consumer group stores its offsets in an internal topic `__consumer_offsets`.
* Group coordinator manages:

  * Joins/leaves of consumers.
  * Partition assignment.
  * Rebalances when membership or partition count changes.

Offsets are just **Kafka messages** in this internal topic, so they benefit from the same durability and replication.

---

## 2. Internals You’ll Care About in Ops

Since you’re DevOps/SRE, a few key things:

### 2.1 High Availability

* Use **RF=3**, `min.insync.replicas=2`, `acks=all`.
* Spread brokers across AZs / racks (use `broker.rack` for rack-aware placement).
* Monitor ISR size → if dropping, you have replication lag or unhealthy nodes.

### 2.2 Backpressure & Load

* Too many partitions → more file handles, memory, CPU.
* Too few → bottlenecks.
* Use metrics to tune:

  * Request/response latency
  * Under-replicated partitions
  * Bytes in/out per broker
  * Disk usage per broker

### 2.3 Security (later)

* TLS for clients and inter-broker.
* SASL (SCRAM/OAuth) for auth.
* ACLs for topic access.

---

## 3. Kafka on Kubernetes / Managed Options (Quick Note)

Since you live in K8s land:

* **On Kubernetes**:

  * Use operators like **Strimzi** or **Confluent for Kubernetes**.
  * They handle KRaft setup, storage, configs, rolling upgrades.
* **Managed**:

  * Confluent Cloud, AWS MSK, Azure Event Hubs (Kafka-compatible), GCP Pub/Sub (not Kafka but similar idea).

Ops complexity drops a lot, but you lose some low-level tuning flexibility.

---
