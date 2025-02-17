## Question: What can i do for optimize logstash by CPU usage?


Optimizing Logstash for CPU usage is crucial, especially when dealing with high-throughput systems like Kafka and OpenSearch. Here are several strategies to reduce CPU usage and improve performance:

---

### 1. **Tune Logstash Worker Threads**
   - Logstash uses a pipeline with multiple worker threads to process events. By default, the number of workers is set to the number of CPU cores. You can adjust this based on your CPU capacity.
   - Add the following setting to your `logstash.yml` configuration file:
     ```yaml
     pipeline.workers: 2  # Adjust based on your CPU cores and workload
     ```
   - If you have a small number of CPU cores, reducing the number of workers can help reduce CPU contention.

---

### 2. **Increase Batch Size**
   - Logstash processes events in batches. Increasing the batch size can reduce the overhead of processing individual events.
   - Add the following setting to your `logstash.yml`:
     ```yaml
     pipeline.batch.size: 125  # Default is 125, increase if you have sufficient memory
     pipeline.batch.delay: 50   # Milliseconds to wait before processing a batch
     ```
   - Larger batch sizes reduce the frequency of batch processing, which can lower CPU usage.

---

### 3. **Optimize Filters**
   - Filters are often the most CPU-intensive part of Logstash. Optimize or remove unnecessary filters.
   - Use **conditionals** to apply filters only when necessary:
     ```ruby
     filter {
       if [field_name] == "value" {
         # Apply filter only for specific events
       }
     }
     ```
   - Avoid using heavy filters like `grok` or `ruby` unless absolutely necessary. If you must use `grok`, precompile your patterns for better performance.

---

### 4. **Use Persistent Queues**
   - Enable persistent queues to buffer events on disk instead of keeping them in memory. This reduces memory pressure and can indirectly lower CPU usage.
   - Add the following to your `logstash.yml`:
     ```yaml
     queue.type: persisted
     queue.max_bytes: 4gb  # Adjust based on your disk space
     ```

---

### 5. **Disable Debugging Output**
   - If you are using `stdout { codec => rubydebug }` for debugging, disable it in production. Writing to the console can be CPU-intensive.

---

### 6. **Optimize Kafka Input**
   - Use the `fetch_max_bytes` and `fetch_max_wait_ms` settings in the Kafka input plugin to control how much data is fetched in each poll:
     ```ruby
     input {
       kafka {
         bootstrap_servers => "kafka1:9092,kafka2:9092,kafka3:9092"
         topics => ["mytopic1"]
         fetch_max_bytes => 1048576  # 1 MB per fetch
         fetch_max_wait_ms => 500    # Wait up to 500ms for data
       }
     }
     ```
   - Avoid setting `decorate_events` to `true` unless you need Kafka metadata, as it adds overhead.

---

### 7. **Optimize OpenSearch Output**
   - Use the `flush_size` and `idle_flush_time` settings in the OpenSearch output plugin to control how often data is sent to OpenSearch:
     ```ruby
     output {
       opensearch {
         hosts => ["https://myopensearch.mydomain:9200"]
         index => "myindex1"
         flush_size => 500  # Send data in batches of 500 events
         idle_flush_time => 5  # Flush every 5 seconds if no new events
       }
     }
     ```
   - Larger `flush_size` values reduce the frequency of HTTP requests, which can lower CPU usage.

---

### 8. **Monitor and Profile**
   - Use the **Logstash Monitoring API** to identify bottlenecks:
     ```bash
     curl -XGET 'http://localhost:9600/_node/stats/pipelines?pretty'
     ```
   - Look for plugins or stages that are consuming excessive CPU or time.

---

### 9. **Upgrade Logstash**
   - Ensure you are using the latest version of Logstash, as newer versions often include performance improvements and bug fixes.

---

### 10. **Scale Horizontally**
   - If a single Logstash instance cannot handle the load, consider running multiple Logstash instances in parallel. Distribute the Kafka partitions across multiple Logstash consumers.

---

### 11. **Use a Dedicated Filter Worker**
   - If you have a heavy filter pipeline, consider offloading filtering to a separate Logstash instance. For example:
     - **Instance 1**: Kafka input ? Output to an intermediate Kafka topic.
     - **Instance 2**: Kafka input (from intermediate topic) ? Filters ? OpenSearch output.

---

### 12. **Reduce Event Size**
   - If your events are large, consider trimming unnecessary fields before processing them in Logstash. Smaller events require less CPU to process.

---

### 13. **Use JVM Tuning**
   - Logstash runs on the JVM, so tuning JVM settings can help reduce CPU usage. Edit the `jvm.options` file (located in the Logstash config directory) and adjust the following:
     ```bash
     -Xms2g  # Initial heap size
     -Xmx2g  # Maximum heap size
     ```
   - Avoid setting the heap size too large, as it can lead to long garbage collection pauses.

---

### 14. **Disable Unnecessary Plugins**
   - Disable any unused plugins or outputs to reduce overhead.

---

By implementing these optimizations, you should see a significant reduction in CPU usage while maintaining or even improving throughput. Always test changes in a staging environment before applying them to production.
