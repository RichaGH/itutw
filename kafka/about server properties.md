## Запрос
describe server.properties: sections and parameters

## Ответ

The `server.properties` file is the main configuration file for an Apache Kafka broker. It contains various sections and parameters that define how the broker operates. Below is a breakdown of the key sections and parameters in the `server.properties` file:

---

### **1. Broker Basics**
These parameters define the broker's identity and network settings.

- **`broker.id`**: 
  - A unique integer identifier for the broker in the Kafka cluster.
  - Example: `broker.id=1`

- **`listeners`**:
  - Comma-separated list of URIs and protocols the broker will listen on.
  - Example: `listeners=PLAINTEXT://:9092`

- **`advertised.listeners`**:
  - The address clients should use to connect to the broker.
  - Example: `advertised.listeners=PLAINTEXT://broker1.example.com:9092`

- **`zookeeper.connect`**:
  - Specifies the Zookeeper connection string (if using Zookeeper).
  - Example: `zookeeper.connect=localhost:2181`

- **`log.dirs`**:
  - Directory where Kafka stores its data logs.
  - Example: `log.dirs=/var/lib/kafka/data`

---

### **2. Topic Defaults**
These parameters define default settings for topics created on the broker.

- **`num.partitions`**:
  - Default number of partitions for a topic.
  - Example: `num.partitions=1`

- **`default.replication.factor`**:
  - Default replication factor for topics.
  - Example: `default.replication.factor=3`

- **`auto.create.topics.enable`**:
  - Whether to automatically create topics when a producer starts writing to them.
  - Example: `auto.create.topics.enable=true`

---

### **3. Log Management**
These parameters control how Kafka manages log files.

- **`log.retention.hours`**:
  - Number of hours to retain logs before deletion (time-based retention).
  - Example: `log.retention.hours=168` (7 days)

- **`log.retention.bytes`**:
  - Maximum size of a log segment before deletion (size-based retention).
  - Example: `log.retention.bytes=1073741824` (1 GB)

- **`log.segment.bytes`**:
  - Size of a single log segment file.
  - Example: `log.segment.bytes=1073741824` (1 GB)

- **`log.cleanup.policy`**:
  - Policy for log cleanup: `delete` (default) or `compact`.
  - Example: `log.cleanup.policy=delete`

---

### **4. Network and Threads**
These parameters control network and thread behavior.

- **`num.network.threads`**:
  - Number of threads handling network requests.
  - Example: `num.network.threads=3`

- **`num.io.threads`**:
  - Number of threads performing disk I/O.
  - Example: `num.io.threads=8`

- **`socket.send.buffer.bytes`**:
  - Socket send buffer size.
  - Example: `socket.send.buffer.bytes=102400`

- **`socket.receive.buffer.bytes`**:
  - Socket receive buffer size.
  - Example: `socket.receive.buffer.bytes=102400`

---

### **5. Replication**
These parameters control replication behavior.

- **`replica.lag.time.max.ms`**:
  - Maximum time a replica can lag behind the leader.
  - Example: `replica.lag.time.max.ms=10000`

- **`replica.fetch.max.bytes`**:
  - Maximum size of data a replica can fetch.
  - Example: `replica.fetch.max.bytes=1048576`

- **`unclean.leader.election.enable`**:
  - Whether to allow out-of-sync replicas to become leaders.
  - Example: `unclean.leader.election.enable=false`

---

### **6. Security**
These parameters configure security settings.

- **`security.inter.broker.protocol`**:
  - Security protocol for inter-broker communication.
  - Example: `security.inter.broker.protocol=PLAINTEXT`

- **`ssl.keystore.location`**:
  - Location of the keystore file for SSL.
  - Example: `ssl.keystore.location=/path/to/keystore`

- **`ssl.keystore.password`**:
  - Password for the keystore file.
  - Example: `ssl.keystore.password=changeit`

- **`sasl.enabled.mechanisms`**:
  - SASL mechanisms enabled for authentication.
  - Example: `sasl.enabled.mechanisms=PLAIN`

---

### **7. Performance Tuning**
These parameters help optimize Kafka performance.

- **`message.max.bytes`**:
  - Maximum size of a message the broker can accept.
  - Example: `message.max.bytes=1000012`

- **`queued.max.requests`**:
  - Maximum number of requests queued before blocking.
  - Example: `queued.max.requests=500`

- **`fetch.max.bytes`**:
  - Maximum amount of data a consumer can fetch in one request.
  - Example: `fetch.max.bytes=52428800`

---

### **8. Advanced Settings**
These are less commonly modified but important for advanced use cases.

- **`controlled.shutdown.enable`**:
  - Whether to allow controlled shutdown of the broker.
  - Example: `controlled.shutdown.enable=true`

- **`delete.topic.enable`**:
  - Whether to allow topic deletion.
  - Example: `delete.topic.enable=true`

- **`compression.type`**:
  - Compression type for messages (`none`, `gzip`, `snappy`, `lz4`, `zstd`).
  - Example: `compression.type=snappy`

---

### **9. Metrics and Monitoring**
These parameters configure metrics and monitoring.

- **`metric.reporters`**:
  - List of classes for reporting metrics.
  - Example: `metric.reporters=com.example.MyMetricsReporter`

- **`metrics.sample.window.ms`**:
  - Time window for metrics sampling.
  - Example: `metrics.sample.window.ms=30000`

---

### **10. Kafka Raft Metadata (KRaft Mode)**
If using Kafka without Zookeeper (KRaft mode), these parameters are relevant.

- **`process.roles`**:
  - Roles of the broker: `broker`, `controller`, or both.
  - Example: `process.roles=broker,controller`

- **`node.id`**:
  - Unique ID for the node in KRaft mode.
  - Example: `node.id=1`

- **`controller.quorum.voters`**:
  - List of controller nodes in the cluster.
  - Example: `controller.quorum.voters=1@localhost:9093`

---

### **Example `server.properties` File**
```properties
broker.id=1
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://broker1.example.com:9092
log.dirs=/var/lib/kafka/data
num.partitions=1
default.replication.factor=3
log.retention.hours=168
log.retention.bytes=1073741824
log.segment.bytes=1073741824
log.cleanup.policy=delete
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
replica.lag.time.max.ms=10000
replica.fetch.max.bytes=1048576
unclean.leader.election.enable=false
security.inter.broker.protocol=PLAINTEXT
message.max.bytes=1000012
queued.max.requests=500
fetch.max.bytes=52428800
controlled.shutdown.enable=true
delete.topic.enable=true
compression.type=snappy
```

---

This configuration file is highly customizable, and the parameters can be adjusted based on your cluster's requirements. Always refer to the official [Apache Kafka documentation](https://kafka.apache.org/documentation/) for the latest details and best practices.
