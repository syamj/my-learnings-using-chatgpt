Great — let’s go **deep into RabbitMQ internals**:
**Exchanges**, **Queues**, **Bindings**, **Channels**, **Storage**, **Routing**, and **how messages travel inside the broker**.

This will be a **low-level, engineer-focused breakdown**.

---

# 🧬 **1. RabbitMQ Internal Architecture Overview**

At a high level:

```
Producer → Connection → Channel → Exchange → Binding → Queue → Consumer
```

Inside RabbitMQ, every component is an Erlang process (actor model).
So exchanges, queues, connections, consumers — each runs in its own lightweight process.

---

# 🔁 **2. Exchanges – The Routing Brain**

An **Exchange** is where producers publish messages.

### Key responsibilities:

* Accept messages from channels
* Apply routing logic
* Deliver messages to bound queues
* Handle unroutable messages

### Exchange types and how routing works internally:

---

## **2.1 Direct Exchange**

Routing key **must match** the binding key exactly.

```
routing_key = "order.created"
binding_key = "order.created" → match
```

⚙️ Internally:

* Exchange keeps a hash table of `binding_key -> list of queues`
* Lookup is O(1), extremely fast

---

## **2.2 Topic Exchange**

Pattern-based routing:
Supports `*` (one word) and `#` (zero or more words)

Example:

```
order.*.created        # matches 3-word patterns
order.#                # matches anything starting with order.
```

⚙️ Internally:

* Exchange stores trie-like routing structures for wildcard patterns
* Matching is slower than direct exchange but very flexible

---

## **2.3 Fanout Exchange**

Broadcast.

No routing key check.
Exchange simply sends the message to **all bound queues**.

⚙️ Internally:

* Keeps a list of queue references
* Iterates and writes message to each queue

---

## **2.4 Headers Exchange**

Routes based on message headers.

---

## **2.5 Default Exchange ("")**

Every queue is automatically bound as:

```
binding key = queue name
```

Producers using:

```
exchange = ""
routing_key = queue_name
```

push messages **directly into a queue**.

---

# 📦 **3. Queues – The Storage Engine**

Queues are the most complex component.

Each queue has:

* Its own Erlang process
* Its own internal message index
* Zero or more backing stores (RAM, disk)
* Configuration: durable, auto-delete, exclusive, quorum/stream/classic

---

# 🧱 **3.1 Classic Queues (Old Yet Common)**

Internally, classic queues maintain:

### **In-memory buffer**

* Holds recently-published messages
* Provides fast delivery to consumers

### **Message store**

Disk-based persistence.

Messages can be in:

1. **RAM only** (transient)
2. **Disk only** (when queue grows large)
3. **Both** (persistent + cached)

### How messages flow:

```
Publish → In-memory queue (RAM) → Disk segments → Consumer ACK → Delete from disk
```

---

# 🧩 **3.2 Quorum Queues (Modern, Recommended)**

Built on **Raft consensus**.

Architecture:

* Leader + Followers (replicas)
* Messages must be written to a majority before ACKing producer

Benefits:

* Strong durability
* Better failure handling
* Eliminates mirrored queues

Drawbacks:

* More IOPS
* Higher CPU usage

---

# 🚀 **3.3 Stream Queues (Kafka-Like)**

Messages stored in append-only log segments:

```
segment1.log
segment2.log
segment3.log
```

Consumers read using **offsets**, not message deletion.

Perfect for:

* telemetry
* metrics
* event streams

---

# 🔗 **4. Bindings – Links Between Exchange & Queue**

A **binding** is a rule:

```
exchange → queue
```

with:

* binding key (direct/topic)
* headers (headers exchange)
* arguments

Internally:

* Exchange keeps binding tables (depending on type)
* Queues don’t store bindings — exchanges do

---

# 🧵 **5. Channels – Virtual Sessions Inside Connections**

Producers/consumers don’t speak to RabbitMQ directly via TCP — they first create **channels**.

### Why channels?

* A single TCP connection can host 1000s of channels
* Reduces connection overhead
* Lightweight Erlang processes

Internally:

* Each channel has its own inbox/outbox buffers
* Each channel tracks unacked messages

---

# 🧮 **6. How Message Routing Happens Internally**

Let's trace a message from producer → consumer.

---

# 🔬 **Step-by-Step Internal Flow**

### **1. Producer publishes message**

Producer sends:

* exchange name
* routing key
* payload
* properties (persistent, headers, TTL)
* delivery mode

Over AMQP.

---

### **2. Channel receives message**

Channel performs:

* basic.validate
* check ACL/vhost permissions
* locate exchange

---

### **3. Exchange applies routing logic**

Lookup queues:

* direct → hash lookup
* topic → pattern matching
* fanout → list iteration

Produces a list of target queues.

---

### **4. Message enters Queue**

Queue receives message:

* Assigns internal sequence ID
* Write to RAM buffer
* If persistent: append to log file on disk

Disk writes happen in batches to optimize fsync.

---

### **5. Message ready for delivery**

If a consumer is waiting:

* Queue sends message to the consumer channel immediately
  Else:
* It stays in RAM/disk

---

### **6. Consumer processes message**

Consumer sends:

* ACK → remove message
* NACK → requeue or dead-letter
* Reject → discard or DLX

Queue updates its internal index.

---

# 🗄 **7. Storage Internals (Disk)**

RabbitMQ uses:

### **Write-ahead log**

For persistent messages:

```
msg_store_persistent/queue_name/
   00000001.sto
   00000002.sto
   ...
```

Messages are stored in segments (like Kafka log segments).

Index maps:

```
message_sequence → file_offset
```

### **Memory vs Disk**

* RabbitMQ tries to keep hot messages in RAM
* Cold messages are paged out to disk
* Memory alarms prevent OOM situations

---

# 🔐 **8. Dead Lettering**

If a message:

* expires (TTL)
* is rejected
* queue is full

It is routed to the **dead-letter exchange (DLX)**.

RabbitMQ stores:

* x-death header
* retry count

---

# 📉 **9. Flow Control & Backpressure**

RabbitMQ applies:

### **TCP backpressure**

Slows down producers if consumers are slow.

### **Memory alarms**

Pause publishing if broker RAM is below threshold.

### **Disk alarms**

Pause writes when disk usage exceeds limit.

---

# 🧯 **10. Failure Recovery**

RabbitMQ stores metadata in:

* Mnesia (Erlang distributed DB)

Data includes:

* queue declarations
* exchange declarations
* bindings
* users & permissions
* policies

On restart:

* Metadata restored from Mnesia
* Queue messages reconstructed from disk

---
