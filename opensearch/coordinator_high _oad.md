## Вопрос
I have an opensearch cluster. There is a high CPU usage on nodes with the coordinator role. Recommendations are needed to identify and eliminate the causes.

## Ответ

High CPU usage on coordinator nodes in an OpenSearch cluster can be caused by several factors, such as inefficient queries, heavy indexing load, or misconfiguration. Below are steps to identify and resolve the root cause:

---

### **1. Identify the Root Cause**
Start by diagnosing the source of the high CPU usage.

#### **a. Check Cluster Health and Performance Metrics**
- Use the OpenSearch Dashboards or API to monitor cluster health and performance:
  - `GET _cluster/health`
  - `GET _nodes/stats`
  - `GET _cat/nodes?v` (to see CPU usage per node)
- Look for:
  - High search or indexing rates.
  - Long query execution times.
  - High JVM memory pressure or GC activity.

#### **b. Analyze Query Performance**
- Use the OpenSearch `_search` profiling API to identify slow or resource-intensive queries:
  ```json
  POST /<index>/_search
  {
    "profile": true,
    "query": {
      "match_all": {}
    }
  }
  ```
- Look for:
  - Complex aggregations.
  - Wildcard or regex queries.
  - Large result sets or deep pagination.

#### **c. Check for Hot Threads**
- Use the `_nodes/hot_threads` API to identify threads consuming high CPU:
  ```bash
  GET /_nodes/hot_threads
  ```
- Look for:
  - Long-running tasks or threads stuck in loops.

#### **d. Review Coordinator Node Logs**
- Check the logs of the coordinator nodes for errors or warnings:
  - Look for:
    - Query timeouts.
    - Circuit breaker exceptions.
    - Resource exhaustion warnings.

---

### **2. Optimize Queries**
Inefficient queries are a common cause of high CPU usage.

#### **a. Simplify Queries**
- Avoid expensive operations like:
  - Wildcard (`*`) or regex queries.
  - Nested aggregations.
  - `script` fields or painless scripts.
- Use filters instead of queries where possible (filters are cached).

#### **b. Paginate Large Result Sets**
- Avoid deep pagination with `from` and `size`. Use `search_after` instead.
- Example:
  ```json
  POST /<index>/_search
  {
    "size": 10,
    "query": {
      "match_all": {}
    },
    "sort": [
      { "timestamp": "asc" }
    ],
    "search_after": [ "2023-10-01T00:00:00Z" ]
  }
  ```

#### **c. Limit Aggregations**
- Use `sampler` or `terms` aggregations with smaller sizes to reduce CPU load.

---

### **3. Optimize Indexing**
Heavy indexing can also strain coordinator nodes.

#### **a. Batch Indexing Requests**
- Use bulk indexing instead of individual document indexing:
  ```bash
  POST /_bulk
  { "index": { "_index": "my_index", "_id": "1" } }
  { "field1": "value1" }
  { "index": { "_index": "my_index", "_id": "2" } }
  { "field1": "value2" }
  ```

#### **b. Reduce Refresh Interval**
- Increase the refresh interval to reduce the frequency of segment creation:
  ```json
  PUT /<index>/_settings
  {
    "index.refresh_interval": "30s"
  }
  ```

#### **c. Disable Unnecessary Features**
- Disable features like `_source`, `_all`, or dynamic mapping if not needed.

---

### **4. Scale the Cluster**
If the coordinator nodes are overloaded, consider scaling the cluster.

#### **a. Add More Coordinator Nodes**
- Add more nodes with the coordinator role to distribute the load.

#### **b. Separate Roles**
- Dedicate nodes to specific roles (e.g., data, coordinator, ingest) to isolate workloads.

#### **c. Use Dedicated Master Nodes**
- Ensure master-eligible nodes are not handling data or coordinator tasks.

---

### **5. Tune OpenSearch Configuration**
Adjust OpenSearch settings to optimize performance.

#### **a. Increase Thread Pools**
- Increase thread pool sizes for search and indexing:
  ```yaml
  thread_pool:
    search:
      size: 20
      queue_size: 1000
    write:
      size: 20
      queue_size: 1000
  ```

#### **b. Adjust Circuit Breakers**
- Increase circuit breaker limits if queries are failing due to memory pressure:
  ```yaml
  indices.breaker.total.limit: 70%
  ```

#### **c. Enable Caching**
- Enable query and request caching to reduce repeated computation:
  ```json
  PUT /<index>/_settings
  {
    "index.requests.cache.enable": true
  }
  ```

---

### **6. Monitor and Iterate**
- Continuously monitor the cluster using OpenSearch Dashboards or external tools like Prometheus and Grafana.
- Iterate on optimizations based on observed performance.

---

### **7. Consider External Factors**
- Check for external factors like network latency or client-side issues.
- Ensure the cluster is not under a DDoS attack or experiencing excessive traffic.

---

By following these steps, you should be able to identify and mitigate the causes of high CPU usage on coordinator nodes in your OpenSearch cluster.
