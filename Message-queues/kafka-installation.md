
## 1. Setting Up a Kafka Cluster (Modern KRaft, 3-node Example)

Let’s assume:

* 3 Linux VMs / bare metal nodes:

  * `kafka-1`, `kafka-2`, `kafka-3`
* Each with:

  * Java 11+
  * 4+ vCPUs, 16+ GB RAM, fast SSDs
  * Ports open for Kafka traffic (default 9092, and controller ports like 9093)

### 1.1 Plan the Cluster

**Decide:**

* **Replication factor** (RF): 3 (recommended for prod).
* **Min in-sync replicas**: 2 (with `acks=all` gives good durability).
* **Number of partitions** per topic → depending on throughput (start small, you can increase).

**Disk layout:**

* Use dedicated disks (e.g. `/data/kafka1`, `/data/kafka2`).
* One or more `log.dirs`, not root filesystem.

---

### 1.2 Download & Unpack Kafka

On each node:

```bash
wget https://downloads.apache.org/kafka/<version>/kafka_2.13-<version>.tgz
tar -xzf kafka_2.13-<version>.tgz -C /opt
ln -s /opt/kafka_2.13-<version> /opt/kafka
```

Create data directories:

```bash
sudo mkdir -p /data/kafka
sudo chown -R kafka:kafka /data/kafka
```

(Assuming a `kafka` user.)

---

### 1.3 Generate a Cluster ID (once)

On **one** node:

```bash
/opt/kafka/bin/kafka-storage.sh random-uuid
```

You’ll get something like:

```text
dE5iJPuTSby8xgzlC1kT3w
```

Use **the same cluster ID** on all nodes.

---

### 1.4 Configure KRaft (server.properties)

For a combined **controller+broker** node, edit `/opt/kafka/config/kraft/server.properties` (path varies with version). Do something like:

**On kafka-1:**

```properties
process.roles=broker,controller
node.id=1

controller.quorum.voters=1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093

listeners=PLAINTEXT://kafka-1:9092,CONTROLLER://kafka-1:9093
advertised.listeners=PLAINTEXT://kafka-1:9092

log.dirs=/data/kafka
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Topic defaults (tune as needed)
num.partitions=8
default.replication.factor=3
min.insync.replicas=2

# Internal topic replication
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

# Log retention
log.retention.hours=168
log.segment.bytes=1073741824

# Quotas, etc. left as defaults for now
```

Repeat on **kafka-2** and **kafka-3** with:

```properties
node.id=2
listeners=PLAINTEXT://kafka-2:9092,CONTROLLER://kafka-2:9093
advertised.listeners=PLAINTEXT://kafka-2:9092
```

```properties
node.id=3
listeners=PLAINTEXT://kafka-3:9092,CONTROLLER://kafka-3:9093
advertised.listeners=PLAINTEXT://kafka-3:9092
```

The important bits:

* `process.roles=broker,controller` → node is both.
* `node.id` must be unique per node.
* `controller.quorum.voters` same on all nodes.
* `listeners` & `advertised.listeners` must be resolvable by clients.

---

### 1.5 Format Storage with Cluster ID

On each node:

```bash
/opt/kafka/bin/kafka-storage.sh format \
  -t dE5iJPuTSby8xgzlC1kT3w \
  -c /opt/kafka/config/kraft/server.properties
```

This “binds” the log dirs to the cluster ID and config.

---

### 1.6 Start the Brokers

For quick testing:

```bash
/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
```

For prod, you’d create a `systemd` unit:

```ini
[Unit]
Description=Apache Kafka broker
After=network.target

[Service]
User=kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
Restart=on-failure
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable kafka
sudo systemctl start kafka
```

Do this on all three nodes.

---

### 1.7 Verify Cluster Health

On any node:

* **Check metadata quorum:**

```bash
/opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server kafka-1:9092 describe
```

You should see:

* A current **leader** controller

* All voters as “current”

* **Create a test topic:**

```bash
/opt/kafka/bin/kafka-topics.sh --bootstrap-server kafka-1:9092 \
  --create --topic test-topic \
  --partitions 3 --replication-factor 3
```

* **Describe topic:**

```bash
/opt/kafka/bin/kafka-topics.sh --bootstrap-server kafka-1:9092 \
  --describe --topic test-topic
```

Check that:

* Partitions are spread across brokers.
* Each has a leader and ISR size = 3.

---

### 1.8 Test Produce / Consume

Producer:

```bash
/opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server kafka-1:9092 \
  --topic test-topic
```

Type some messages, Ctrl+C to exit.

Consumer:

```bash
/opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server kafka-1:9092 \
  --topic test-topic \
  --from-beginning
```

You should see all the messages.