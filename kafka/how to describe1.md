## Вопрос
How can i lookup topics in kafka. For example by name i want to see disk space used, how long topic will store data before delete it, how many consumer hopic has, any problems on consumer group

## Ответ

To look up and manage topics in Apache Kafka, you can use a combination of Kafka command-line tools, monitoring tools, and administrative APIs. Below are some common tasks and how to achieve them:

---

### 1. **Lookup Topic Details by Name**
   - Use the `kafka-topics.sh` script to get basic information about a topic:
     ```bash
     kafka-topics.sh --describe --topic <topic_name> --bootstrap-server <broker_address>
     ```
     This will show details like partition count, replication factor, and partition leaders.

---

### 2. **Check Disk Space Used by a Topic**
   - Kafka does not provide a direct command to check disk usage per topic. However, you can:
     - Manually calculate disk usage by inspecting the data logs on the broker's filesystem. The data is stored in the `log.dirs` directory (configured in `server.properties`).
     - Use monitoring tools like **Kafka Manager**, **Confluent Control Center**, or **Prometheus** with Kafka exporters to track disk usage.

---

### 3. **Check Retention Policy (How Long Data is Stored)**
   - Kafka topics have retention policies defined by configurations like `retention.ms` or `retention.bytes`. To check these settings:
     ```bash
     kafka-configs.sh --describe --entity-type topics --entity-name <topic_name> --bootstrap-server <broker_address>
     ```
     Look for `retention.ms` (time-based retention) or `retention.bytes` (size-based retention).

---

### 4. **Check Number of Consumers for a Topic**
   - To see how many consumer groups are consuming from a topic:
     ```bash
     kafka-consumer-groups.sh --list --bootstrap-server <broker_address>
     ```
     Then, describe a specific consumer group to see its members and lag:
     ```bash
     kafka-consumer-groups.sh --describe --group <group_name> --bootstrap-server <broker_address>
     ```

---

### 5. **Check for Consumer Group Problems**
   - Use the `kafka-consumer-groups.sh` tool to check for consumer lag or other issues:
     ```bash
     kafka-consumer-groups.sh --describe --group <group_name> --bootstrap-server <broker_address>
     ```
     Look for columns like `LAG` (number of messages the consumer is behind) and `CURRENT-OFFSET` (consumer's current position).

---

### 6. **Use Monitoring Tools**
   - For a more comprehensive view, use monitoring tools like:
     - **Kafka Manager**: Provides a web UI for managing and monitoring Kafka clusters.
     - **Confluent Control Center**: A commercial tool for monitoring and managing Kafka.
     - **Prometheus + Grafana**: Use Kafka exporters (e.g., Kafka Exporter or JMX Exporter) to collect metrics and visualize them in Grafana.
     - **Burrow**: A tool specifically for monitoring consumer lag.

---

### 7. **Check Broker and Topic Logs**
   - If you suspect issues, check the Kafka broker logs (located in the `logs` directory) and consumer application logs for errors or warnings.

---

### Example Workflow
1. List all topics:
   ```bash
   kafka-topics.sh --list --bootstrap-server <broker_address>
   ```
2. Describe a specific topic:
   ```bash
   kafka-topics.sh --describe --topic <topic_name> --bootstrap-server <broker_address>
   ```
3. Check consumer groups:
   ```bash
   kafka-consumer-groups.sh --list --bootstrap-server <broker_address>
   ```
4. Describe a consumer group:
   ```bash
   kafka-consumer-groups.sh --describe --group <group_name> --bootstrap-server <broker_address>
   ```

---

By combining these tools and commands, you can effectively manage and monitor Kafka topics, consumer groups, and their associated configurations.
