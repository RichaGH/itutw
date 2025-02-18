## Что хотим
i need filebeat config to send strings from mylog.log to kafka (servers mykaf1,mykaf2,mykaf3) to topik = mytopik1

## Как предлагают сделать

To configure **Filebeat** to send log entries from `mylog.log` to a Kafka topic (`mytopik1`) on Kafka servers (`mykaf1`, `mykaf2`, `mykaf3`), you need to create a Filebeat configuration file (`filebeat.yml`). Below is an example configuration:

### Filebeat Configuration (`filebeat.yml`)

```yaml
# Filebeat configuration

# Input configuration (read from mylog.log)
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /path/to/mylog.log  # Replace with the full path to your mylog.log file
  json.keys_under_root: true  # Parse JSON logs and add fields to the root
  json.add_error_key: true    # Add an error key if JSON parsing fails

# Kafka output configuration
output.kafka:
  enabled: true
  hosts: ["mykaf1:9092", "mykaf2:9092", "mykaf3:9092"]  # Kafka broker addresses
  topic: "mytopik1"  # Kafka topic to send logs to
  partition.round_robin:
    reachable_only: false
  required_acks: 1
  compression: none
  max_message_bytes: 1000000  # Maximum message size (adjust as needed)
  codec.json:
    pretty: false

# Optional: Logging configuration (for debugging)
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat.log
  keepfiles: 7
  permissions: 0644
```

---

### Key Configuration Details:

1. **Input Section**:
   - `type: log`: Specifies that Filebeat should read log files.
   - `paths`: Path to your `mylog.log` file. Replace `/path/to/mylog.log` with the actual path.
   - `json.keys_under_root`: Parses JSON logs and adds fields to the root of the event.
   - `json.add_error_key`: Adds an error key if JSON parsing fails.

2. **Kafka Output Section**:
   - `hosts`: List of Kafka broker addresses (`mykaf1`, `mykaf2`, `mykaf3`). Replace `9092` with the correct port if your Kafka brokers use a different port.
   - `topic`: Kafka topic to send logs to (`mytopik1`).
   - `partition.round_robin`: Distributes messages across Kafka partitions in a round-robin fashion.
   - `required_acks`: Ensures that the Kafka broker acknowledges the message (set to `1` for at least one broker acknowledgment).
   - `compression`: Disables compression (`none`). You can enable `gzip`, `snappy`, or `lz4` if needed.
   - `max_message_bytes`: Maximum size of a Kafka message (adjust based on your log size).
   - `codec.json`: Ensures the output is formatted as JSON.

3. **Logging Section**:
   - Configures Filebeat's logging for debugging purposes. Logs are saved to `/var/log/filebeat/filebeat.log`.

---

### Steps to Use This Configuration:

1. **Install Filebeat**:
   - Download and install Filebeat on your system. Follow the official [Filebeat installation guide](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation-configuration.html).

2. **Save the Configuration**:
   - Save the above configuration to `/etc/filebeat/filebeat.yml` (default location for Filebeat configuration on Linux).

3. **Test the Configuration**:
   - Run Filebeat in debug mode to test the configuration:
     ```bash
     filebeat -e -d "*"
     ```
   - Check the logs for any errors or warnings.

4. **Start Filebeat**:
   - Start Filebeat as a service:
     ```bash
     sudo systemctl start filebeat
     ```
   - Enable Filebeat to start on boot:
     ```bash
     sudo systemctl enable filebeat
     ```

5. **Verify Logs in Kafka**:
   - Use a Kafka consumer tool (e.g., `kafka-console-consumer`) to verify that logs are being sent to the `mytopik1` topic:
     ```bash
     kafka-console-consumer --bootstrap-server mykaf1:9092 --topic mytopik1 --from-beginning
     ```

---

### Example Log Entry Sent to Kafka:
Each line in `mylog.log` will be sent as a JSON message to Kafka. For example:

```json
{
  "id": 1,
  "curDateVal": "2023-10-25 16:45:30.123",
  "idata1": 7,
  "sdata1": "abcd",
  "ddata1": 1234567.89,
  "idata2": 42,
  "sdata2": "abcdefghijklmnopqrst",
  "ddata2": 56.78,
  "data3": 789,
  "sdata3": "Example String 123 [Brackets]",
  "ddata3": -12345.67
}
```

---

### Additional Notes:
- **Kafka Authentication**: If your Kafka cluster requires authentication (e.g., SASL/SSL), you need to add the appropriate configuration under the `output.kafka` section. For example:
  ```yaml
  output.kafka:
    username: "your_username"
    password: "your_password"
    ssl.enabled: true
  ```
- **Filebeat Modules**: If you want to use Filebeat modules for additional processing (e.g., parsing, filtering), enable them in the configuration.
- **Performance Tuning**: Adjust `max_message_bytes`, `compression`, and `required_acks` based on your Kafka cluster's performance and requirements.

This configuration should work seamlessly to send your `mylog.log` entries to Kafka. Let me know if you need further assistance!
