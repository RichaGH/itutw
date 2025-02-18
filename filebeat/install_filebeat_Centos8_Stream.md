## Вопрос
if i need install filebeat in the Centos8 Stream. What i must to do?
## Ответ
To install **Filebeat** on **CentOS 8 Stream**, follow these steps. This guide assumes you have root or sudo access to the system.

---

### Step 1: Update the System
Before installing any new software, ensure your system is up to date:
```bash
sudo dnf update -y
```

---

### Step 2: Add the Elastic Stack Repository
Filebeat is part of the Elastic Stack, so you need to add the Elastic repository to your system.

1. **Import the Elastic GPG Key**:
   ```bash
   sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
   ```

2. **Create the Elastic Repository File**:
   Create a new repository file for Elastic:
   ```bash
   sudo nano /etc/yum.repos.d/elastic.repo
   ```

3. **Add the Following Content**:
   Paste the following configuration into the file:
   ```ini
   [elastic-8.x]
   name=Elastic repository for 8.x packages
   baseurl=https://artifacts.elastic.co/packages/8.x/yum
   gpgcheck=1
   gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
   enabled=1
   autorefresh=1
   type=rpm-md
   ```

4. **Save and Exit**:
   Save the file and exit the editor (`Ctrl+O`, `Enter`, `Ctrl+X` in `nano`).

---

### Step 3: Install Filebeat
Now that the Elastic repository is configured, you can install Filebeat using `dnf`:
```bash
sudo dnf install filebeat -y
```

---

### Step 4: Enable and Start Filebeat
After installation, enable and start the Filebeat service:

1. **Enable Filebeat to Start on Boot**:
   ```bash
   sudo systemctl enable filebeat
   ```

2. **Start Filebeat**:
   ```bash
   sudo systemctl start filebeat
   ```

3. **Check the Status**:
   Verify that Filebeat is running:
   ```bash
   sudo systemctl status filebeat
   ```

---

### Step 5: Configure Filebeat
Edit the Filebeat configuration file (`filebeat.yml`) to set up your inputs and outputs.

1. **Open the Configuration File**:
   ```bash
   sudo nano /etc/filebeat/filebeat.yml
   ```

2. **Configure Inputs**:
   Set up Filebeat to read from your `mylog.log` file. For example:
   ```yaml
   filebeat.inputs:
   - type: log
     enabled: true
     paths:
       - /path/to/mylog.log  # Replace with the actual path to your log file
     json.keys_under_root: true
     json.add_error_key: true
   ```

3. **Configure Kafka Output**:
   Set up Filebeat to send logs to Kafka. For example:
   ```yaml
   output.kafka:
     enabled: true
     hosts: ["mykaf1:9092", "mykaf2:9092", "mykaf3:9092"]
     topic: "mytopik1"
     partition.round_robin:
       reachable_only: false
     required_acks: 1
     compression: none
     max_message_bytes: 1000000
     codec.json:
       pretty: false
   ```

4. **Save and Exit**:
   Save the file and exit the editor.

---

### Step 6: Test the Configuration
Before starting Filebeat, test the configuration to ensure there are no syntax errors:
```bash
sudo filebeat test config
```

---

### Step 7: Restart Filebeat
After making changes to the configuration, restart Filebeat to apply the new settings:
```bash
sudo systemctl restart filebeat
```

---

### Step 8: Verify Logs in Kafka
Use a Kafka consumer tool to verify that logs are being sent to the `mytopik1` topic:
```bash
kafka-console-consumer --bootstrap-server mykaf1:9092 --topic mytopik1 --from-beginning
```

---

### Step 9: (Optional) Enable Logging for Debugging
If you encounter issues, enable detailed logging for debugging:

1. **Edit the Logging Configuration**:
   Open the Filebeat configuration file:
   ```bash
   sudo nano /etc/filebeat/filebeat.yml
   ```

2. **Add Logging Configuration**:
   Add the following under the `logging` section:
   ```yaml
   logging.level: debug
   logging.to_files: true
   logging.files:
     path: /var/log/filebeat
     name: filebeat.log
     keepfiles: 7
     permissions: 0644
   ```

3. **Restart Filebeat**:
   Restart Filebeat to apply the logging changes:
   ```bash
   sudo systemctl restart filebeat
   ```

4. **Check Logs**:
   View the Filebeat logs for debugging:
   ```bash
   tail -f /var/log/filebeat/filebeat.log
   ```

---

### Step 10: (Optional) Secure Filebeat
If your Kafka cluster requires authentication (e.g., SASL/SSL), update the `output.kafka` section in the configuration file with the appropriate settings. For example:
```yaml
output.kafka:
  username: "your_username"
  password: "your_password"
  ssl.enabled: true
```

---

### Summary
You have now installed and configured Filebeat on CentOS 8 Stream to read logs from `mylog.log` and send them to a Kafka topic (`mytopik1`). If you encounter any issues, check the Filebeat logs for detailed error messages. Let me know if you need further assistance!
