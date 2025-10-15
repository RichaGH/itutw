# Вопрос

To analyze the operation of the hasTestResult() function. You need to check which queries are being run to the database server at this moment. How can I enable logging of all queries and their execution time in postgresql11, or view this information in the database?

# Рассуждения

We are using PostgreSQL 11. There are several ways to log queries and their execution time.

Option 1: Enable logging in postgresql.conf

We can set the following parameters in the postgresql.conf file:

log_statement: Controls which SQL statements are logged.

log_min_duration_statement: Logs the statement and its duration if it runs longer than the specified time (in milliseconds).

However, note that we want to capture all queries (especially the ones from the hasTestResult() function) and their execution time, so we can use:

log_statement = 'all' # Log all statements, but this can be very verbose.
log_min_duration_statement = 0 # Log every statement and its duration.

But logging every statement can have performance overhead and fill up disk space quickly.

Alternatively, we can set log_min_duration_statement to a threshold (e.g., 100 milliseconds) to log only the slow ones, but we are not sure if the problematic queries are slow or just numerous.

Since we are troubleshooting a specific function, we might want to capture all queries for a short period.

Option 2: Use the pg_stat_statements extension (if installed) to get statistics about executed queries.

This is less invasive and can be used to see which queries are taking the most time and are called most frequently.

Option 3: Use the auto_explain extension to get execution plans of slow queries.

Let's go with Option 1 for detailed logging and Option 2 for a broader view without too much overhead.

Steps for Option 1 (temporary detailed logging):

Locate the postgresql.conf file. It is usually in the data directory (can be found by running SHOW config_file; in psql).

Make a backup of the current configuration.

Modify the following settings:

log_statement = 'all' # Optional: if we want every statement, but be cautious.
log_min_duration_statement = 0 # Log every query with its duration.

Alternatively, to avoid too much log, we can set:

log_min_duration_statement = 100 # Log queries that take longer than 100ms.

Also, set the logging output to a file:

logging_collector = on
log_destination = 'stderr' # or 'csvlog' for CSV format which can be easier to parse.
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'

Restart PostgreSQL for the changes to take effect (if we change log_statement or logging_collector may require restart).

Reproduce the issue by running the JQL with hasTestResult().

After reproducing, revert the changes and restart PostgreSQL to avoid excessive logging.

However, note that restarting the database might not be acceptable in production. Alternatively, we can change some settings without restart by using:
~~~
text
ALTER SYSTEM SET ...;
~~~
And then reload with:
~~~
text
SELECT pg_reload_conf();
~~~
But note: log_statement and log_min_duration_statement require a restart to change. So we might have to choose a different approach if we cannot restart.

Option 2: Using pg_stat_statements

If the extension is installed, we can use it to get query statistics without restarting.

Steps:

Check if pg_stat_statements is installed:
~~~
SELECT * FROM pg_available_extensions WHERE name = 'pg_stat_statements';

If not, install it:

CREATE EXTENSION pg_stat_statements;
~~~
Then, we can query the pg_stat_statements view to see the top queries by total time or number of calls.
~~~
Example query:

SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
~~~
We can reset the statistics before reproducing the issue to get a clean slate:
~~~
SELECT pg_stat_statements_reset();
~~~
Then, run the JQL search with hasTestResult() and then check the pg_stat_statements view.

Option 3: Use auto_explain

This module can automatically log execution plans of slow queries. We can set it in the configuration:
~~~
text
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 100   # Log plans for queries slower than 100ms
auto_explain.log_analyze = true
This requires a restart.
~~~
Given the constraints, I would recommend:

If we can restart, use Option 1 with log_min_duration_statement = 0 for a short period to capture all queries.

If we cannot restart, use Option 2 (pg_stat_statements) to analyze the queries.

However, note that pg_stat_statements does not log every single query by default (it's a sample-based view) and it aggregates data. But it can show us which queries are the most frequent and taking the most time.

Let's assume we cannot restart and use pg_stat_statements.

Steps for using pg_stat_statements:

Ensure pg_stat_statements is installed and enabled.

Reset the statistics:
~~~
SELECT pg_stat_statements_reset();
~~~
Reproduce the issue by running the JQL with hasTestResult() multiple times to get a good sample.

Run the following query to get the top queries:
~~~
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;
~~~
Look for queries that are called many times (high calls) and/or have a high total_time. The query text might be truncated, but we can look for patterns (like joins on the test result tables).

We can also look for queries that are run by the Jira user (if we know the username) by joining with pg_stat_activity but pg_stat_statements does not store user information by default.

Alternatively, we can use the following to get the current queries in real-time from pg_stat_activity:
~~~
text
SELECT datname, usename, application_name, state, query, query_start 
FROM pg_stat_activity 
WHERE state = 'active' AND query NOT LIKE '%pg_stat_activity%';
But this only shows currently running queries.
~~~
Given the nature of the problem (many quick queries), I think pg_stat_statements is the best bet without restarting.

If we can restart, we can get more detailed logs by setting:
~~~
text
log_min_duration_statement = 0
~~~
and then reproduce the issue and then check the log.

Let me know which option is feasible for you.

# Ответ


Perfect! Let's get detailed visibility into what `hasTestResult()` is actually doing to the database. Here are several methods to capture the queries in PostgreSQL 11:

## **Method 1: Enable Comprehensive Query Logging in PostgreSQL**

### **Step 1: Modify PostgreSQL Configuration**

Edit `postgresql.conf` (usually located in `/var/lib/pgsql/11/data/` or `/etc/postgresql/11/main/`):

```bash
# Find and edit your postgresql.conf
sudo nano /var/lib/pgsql/11/data/postgresql.conf
```

Add or modify these settings:
```ini
# Basic logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'

# Log ALL queries and their duration
log_statement = 'all'                    # Log all statements
log_min_duration_statement = 0          # Log ALL queries regardless of duration

# Detailed timing information
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0

# For even more detail
log_autovacuum_min_duration = 0
```

### **Step 2: Reload PostgreSQL Configuration**

You don't need to restart the entire service:
```bash
# Reload configuration
sudo systemctl reload postgresql-11

# Or using SQL
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

### **Step 3: Reproduce the Issue and Monitor**

Now run your JQL search with `hasTestResult()` and watch the logs:
```bash
# Tail the PostgreSQL log in real-time
sudo tail -f /var/lib/pgsql/11/data/pg_log/postgresql-*.log

# Or check recent logs
sudo find /var/lib/pgsql/11/data/pg_log/ -name "*.log" -exec ls -la {} \;
```

## **Method 2: Use pg_stat_statements (Recommended for Production)**

This method is less intrusive and provides aggregated query statistics.

### **Step 1: Enable pg_stat_statements**

Edit `postgresql.conf`:
```ini
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000
```

### **Step 2: Restart PostgreSQL and Create Extension**
```bash
sudo systemctl restart postgresql-11

# Connect to your Jira database
sudo -u postgres psql -d jiradb

-- Create the extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### **Step 3: Analyze Query Performance**

Run these diagnostic queries:

```sql
-- Top 10 slowest queries
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;

-- Top 10 most frequently called queries
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements 
ORDER BY calls DESC 
LIMIT 10;

-- Queries with the highest total execution time
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements 
ORDER BY total_time DESC 
LIMIT 10;

-- Look for Zephyr/test-related queries
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements 
WHERE query ILIKE '%test%' OR query ILIKE '%result%'
ORDER BY total_time DESC;
```

## **Method 3: Real-time Query Monitoring**

### **Option A: Current Activity View**
```sql
-- See currently running queries
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query,
    query_start,
    now() - query_start as duration
FROM pg_stat_activity 
WHERE state = 'active' 
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration DESC;
```

### **Option B: Lock Monitoring**
```sql
-- Check for query locks (common in performance issues)
SELECT 
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked ON blocked_locks.pid = blocked.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.GRANTED;
```

## **Method 4: Jira-Specific SQL Logging**

You can also enable SQL logging directly in Jira:

### **Step 1: Enable Jira SQL Logging**
Edit `log4j.properties` in your Jira home directory:
```properties
# SQL query logging
log4j.logger.com.atlassian.jira=INFO
log4j.logger.com.atlassian.jira.jdbc=DEBUG
log4j.logger.com.opensymphony.module.propertyset.hibernate=WARN

# Zephyr-specific logging
log4j.logger.com.smartbear = DEBUG
```

### **Step 2: Correlate with Thread Dumps**
While the issue is occurring, take thread dumps to see what the stuck threads are doing:
```bash
# Get Jira process ID
jps -l

# Take thread dump
jstack <jira_pid> > thread_dump.txt
```

## **Recommended Approach for Your Situation**

1. **Start with Method 2 (pg_stat_statements)** - it's production-safe and will show you the problematic queries without overwhelming the logs.

2. **If you need more detail, use Method 1** with a short logging window during a maintenance period.

3. **Combine with Method 4** to correlate database queries with Jira application logs.

## **What to Look For**

When you run a JQL search with `hasTestResult()`, watch for:

- **High-frequency queries** - the same query running thousands of times
- **N+1 query patterns** - one query to get issue IDs, then individual queries for each issue
- **Queries on Zephyr tables** - look for `testresult`, `stepresult`, `execution` tables
- **Missing indexes** - queries with sequential scans (SEQ SCAN) instead of index scans

## **Quick Safety Note**

For production systems, I'd recommend:
- Using `pg_stat_statements` first (Method 2)
- If you enable full query logging (Method 1), do it during off-hours and revert quickly
- Monitor disk space as query logs can grow rapidly

Would you like me to help you interpret the query patterns once you capture them? This will help us understand exactly why `hasTestResult()` is so expensive and how to optimize it.
