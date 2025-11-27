
---

# **1. What Is a Consumer Group?**

A **consumer group** is a *set of consumers* that work together to consume data from a topic.

Kafka guarantees:

### **Each partition is consumed by exactly ONE consumer within a group.**

Meaning:

* Consumers in the same group **divide partitions among themselves**.
* Each message is processed **exactly once per group**.

**Consumer Group = horizontal scaling + fault tolerance**
Not multiple readers — it’s **one logical reader** scaled across multiple processes.

---

# **2. How Many Consumers Can Consume from a Topic?**

Depends on:

## **A. Within ONE consumer group**

**Max consumers = number of partitions in the topic**

Example:

```
Topic T: 6 partitions
Consumer Group G:
  - Consumer A
  - Consumer B
  - Consumer C
```

Kafka will assign:

```
C-A → 2 partitions
C-B → 2 partitions
C-C → 2 partitions
```

If you add **7th consumer**, it will stay **idle** — no partitions left.

### **Rule:

Consumers ≤ partitions.
More consumers than partitions = waste.**

---

## **B. Across MULTIPLE consumer groups**

There is **no limit**.

You can have:

* 10 consumer groups
* 100 consumer groups
* 10,000 consumer groups

Each consumer group reads **independently**, at its own speed.

### Kafka’s built-in fan-out:

```
Topic T
 ├── Group A (microservices)
 ├── Group B (analytics)
 ├── Group C (fraud detection)
 └── Group D (DLQ processor)
```

Each group receives its own copy of the data.

---

# **3. What Happens When a Consumer Dies?**

Let’s say:

```
Topic: 6 partitions
Group G: 3 consumers (C1, C2, C3)
```

Partitions assigned:

```
C1 → P0, P1
C2 → P2, P3
C3 → P4, P5
```

### Step-by-step when C2 dies:

### **Step 1 — Heartbeats stop**

Kafka detects missing heartbeats:

* `session.timeout.ms` (defaults ~10s)
* `heartbeat.interval.ms` (~3s)

### **Step 2 — Group Coordinator triggers rebalance**

Only partitions from the dead consumer (C2) get redistributed.

```
C1 → P0, P1, P2
C3 → P4, P5, P3
```

### **Step 3 — New consumers restart reading from last committed offsets**

Kafka stores offsets in the internal topic:

```
__consumer_offsets
```

Offsets are committed by the group periodically (default every 5s).

The **new consumer reads from the last committed offset**, so processing resumes exactly where the previous one left off.

---

# **4. How Does a New Consumer Read From “Where the Old One Left”?**

Because Kafka stores offsets **per consumer group**, per partition.

Stored like:

```
<topic, partition, group-id> → committed_offset
```

When a new consumer joins:

1. It gets partition P2
2. It loads last committed offset from `__consumer_offsets`
3. It starts fetching from that offset

Exactly same place, no duplication if processing is idempotent.

---

# **5. Can We Attach Multiple Consumer Groups to the Same Topic?**

### **YES — this is extremely common.**

Each consumer group receives its **own independent stream** of data.

Example:

```
Topic: transactions

Group: fraud-service      → real-time fraud checks
Group: analytics-service  → store into data lake
Group: notification-svc   → send SMS/Email
Group: ml-training        → build models
```

Each group:

* Has independent offset tracking
* Consumes at its own pace
* Can scale independently
* Can lag without affecting others

### **This is a major difference from RabbitMQ**

(Where consuming removes the message.)

Kafka does **not** delete messages when read.

---

# **6. Can Different Consumer Groups Consume at Different Speeds?**

### **Yes — absolutely.**

* One group may consume in real-time with low lag
* Another group may lag hours behind
* Another may stop for a day and resume later
* None of them affect each other

Kafka is designed for **independent readers**.

---

# **7. Summary (Key Points You Must Remember)**

### ✔ **Consumer Group = 1 logical subscriber**

Messages are load-balanced across consumers in the group.

### ✔ **Max consumers per group = number of partitions**

Extra consumers remain idle.

### ✔ **If a consumer dies → rebalance → new consumer reads from last committed offset**

Fault-tolerance built in.

### ✔ **Many consumer groups can read from the same topic**

Complete fan-out, each group gets its own copy.

### ✔ **Different groups can consume at different pace**

Offsets are independent; lag doesn’t interfere.

---
