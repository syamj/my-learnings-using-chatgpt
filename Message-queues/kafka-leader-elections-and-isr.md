
---

# **1. Key Concepts Recap**

Before diving into failure scenarios, you need to be crystal clear on these terms:

### **Partition**

A single ordered log. One partition = one leader + zero or more followers.

### **Leader**

Broker that handles:

* All writes (produce requests)
* All reads (fetch requests) unless follower fetching is enabled

### **Replicas**

Copies of a partition on different brokers.

### **ISR (In-Sync Replicas)**

Replicas that are “caught up enough” to the leader.

Rules for ISR:

* Replicas must be within `replica.lag.time.max.ms` threshold
* Leader is always part of ISR
* ISR ⊆ AR (assigned replicas)

### **AR (Assigned Replicas)**

All replicas assigned to a partition (leader + followers).

Example:

```
AR = [1,2,3]
ISR = [1,2,3]
Leader = 1
```

---

# **2. Normal Write Flow (healthy cluster)**

1. Producer sends messages to leader of partition.
2. Leader writes to the log.
3. Followers pull data asynchronously.
4. Replicas stay in ISR as long as replication lag is small.
5. If producer uses:

   * `acks=1` → leader-only ack
   * `acks=all` → wait for ISR acknowledgment (durability)

---

# **3. Failure Scenario 1: **Leader Fails** (Most Common)**

Let’s say:

```
Partition X
Leader = Broker-1
ISR = [1,2,3]
```

### **3.1 Broker-1 goes down**

What happens?

1. **Followers stop receiving heartbeats from the leader.**
2. KRaft metadata controller marks the leader as **failed**.
3. **Controller triggers a leader election.**

### **3.2 Leader Election Logic**

Kafka tries to pick a new leader **only from ISR**, not from AR.

Example:

```
AR  = [1,2,3]
ISR = [1,2,3]  → all replicas good
```

A new leader is chosen from ISR using a deterministic, reproducible order (usually the next ISR replica).

### **Why only ISR?**

Because only ISR replicas are guaranteed to be “safe enough” — they have the most recent acknowledged data.

Using a non-ISR replica could cause **data loss**.

### **3.3 After Election**

Example:

```
Old leader = 1
New leader = 2
ISR = [2,3]
```

Broker-1 is removed from ISR.

**Messages are still available.
No data loss.
Cluster continues smoothly.**

---

# **4. Failure Scenario 2: Follower Falls Behind (Lagging Replica)**

ISR evaluation happens continuously.

### Case:

```
Leader = 1
Follower 2 is lagging too far (network slow, overloaded)
Follower 3 is caught up
```

Kafka checks:

```
replication.lag.time.max.ms = 10000 (10 seconds default)
```

If follower-2 is behind >10 seconds:

```
ISR = [1,3]
Follower 2 removed from ISR
```

### What does this mean?

* It is **still a replication factor of 3**, but effective durability = 2 (ISR count)
* If leader fails now → only follower-3 is eligible to become leader
* Follower-2 cannot become leader until it catches up and rejoins ISR

---

# **5. Failure Scenario 3: Leader Fails When ISR < RF (danger)**

Most interesting scenario:

```
AR  = [1,2,3]
ISR = [1,3]    (2 has fallen behind)
Leader = 1
```

Now if leader (1) fails:

* **Only replicas in ISR can become leader**
* ISR = [1,3]
* New leader must be **3**

Follower-2 cannot become leader → unsafe.

This is still okay — no data loss.

**UNLESS ISR becomes size=1...**

---

# **6. Failure Scenario 4: ISR shrinks to 1 and Leader dies**

Bad case:

```
AR  = [1,2,3]
ISR = [1]      (2 & 3 lag behind)
Leader = 1
```

If leader (1) fails:

* No other replica is in ISR.
* Controller sees **ISR empty** (except failed leader).
* Kafka must choose between:

  * **Wait for ISR to come back** (unavailability)
  * **Elect a non-ISR replica** (risk data loss)

Controlled by this setting:

```
unclean.leader.election.enable = false  (default, safe)
```

### **Case 1: unclean.leader.election = false**

Kafka waits until an ISR member comes back alive.

✔ Data consistency preserved
✘ Partition unavailable (no reads/writes)

### **Case 2: unclean.leader.election = true**

Kafka chooses a non-ISR replica as leader.

✔ Partition becomes available
✘ **Messages may be lost** (from the old leader that never replicated)

This is rarely used except in:

* test environments
* log aggregation pipelines where loss is acceptable

---

# **7. Rejoining ISR**

A follower rejoins ISR if:

1. It catches up to the current leader within lag threshold.
2. Follower and leader logs match on high watermark.

Simplified steps:

1. Follower pulls data aggressively (replication fetcher).
2. Catches up to leader’s “high watermark”.
3. Controller marks the follower as part of ISR again.

---

# **8. High Watermark: The Most Important Internal Concept**

### Leader tracks two offsets:

| Name                     | Meaning                                      |
| ------------------------ | -------------------------------------------- |
| **Log End Offset (LEO)** | Last written message                         |
| **High Watermark (HW)**  | Highest offset replicated to all ISR members |

Consumers are **never allowed to read data > HW**, ensuring no one reads uncommitted data.

So:

* Producer writes → increases LEO
* Followers replicate → HW increases as they catch up

If ISR shrinks, HW freezes until followers return.

---

# **9. Summary Diagram**

### Normal Case:

```
Leader: 1 (ISR: 1,2,3)
      ↓ append
Follower-2 (caught up)
Follower-3 (caught up)
```

### Follower Lag:

```
Leader: 1 (ISR: 1,3)
Follower-2 (removed)
```

### Leader Fail:

```
1 dies → 3 becomes leader
ISR = [3]
```

### All Follower Lag → Bad Case:

```
ISR = [1]
1 dies → NO ISR → no leader → partition unavailable
```

---

# **10. Final Summary**

### **Kafka always tries to elect leader from ISR.**

Why?
To avoid data loss.

### **If ISR has ≥ 2 replicas → safe failover**

No downtime, no loss.

### **If ISR drops to 1 → leader failure = downtime**

Unless unclean leader election is enabled (unsafe).

### **Replication lag plays a huge role**

Monitoring ISR shrinkage is critical.

---
