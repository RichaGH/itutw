# Вопрос

You are an IT engineer with 20 years of experience. Your task is to understand the reason for the high CPU usage on the jira server host.  You have a log in catalina.out in which, as the download grows, entries like "org.apache.catalina.valves" appear en masse.Stuck.threadDetectionValve" . 
At the same time, the load on the database increases. it is necessary to check the database logs for slow queries, and also identify, based on the jira logs, which queries lead to slow queries. You should check the logs and explain to a young colleague what you were watching and why.

# Расуждения

We are facing high CPU usage on the Jira server host, and in the catalina.out log we see many entries of "org.apache.catalina.valves.StuckThreadDetectionValve". This valve is used to detect threads that are taking a long time to complete, which can indicate a stuck thread. Additionally, the database load is increasing.

Steps to diagnose:

Check the Jira logs (catalina.out and other application logs) for the stuck thread entries and try to identify the operations that are getting stuck.

Check the database logs for slow queries. Since the database is under load, there might be slow queries that are causing the threads to block.

Cross-reference the time of the stuck threads in Jira with the time of the slow queries in the database logs.

Let's break down the steps:

Step 1: Analyze Jira logs for stuck threads.

In the catalina.out, look for entries that contain "StuckThreadDetectionValve". These entries will typically show the thread ID and the time the thread has been stuck. Also, note the stack trace or the operation that the thread was performing.

Example log entry:
"org.apache.catalina.valves.StuckThreadDetectionValve notifyStuckThreadDetected Thread [http-nio-8080-exec-123] (id=[123]) has been active for [60,000] milliseconds (since [2023-01-01 10:00:00]) to serve the same request. There is a probability that the thread is stuck."

Step 2: Check database logs for slow queries.

Depending on the database (MySQL, PostgreSQL, etc.), the slow query log will contain queries that take longer than a defined threshold. Look for queries that are taking a long time and note the time they occurred and the query itself.

Step 3: Correlate the times.

If the stuck threads in Jira are occurring at the same time as the slow queries in the database, it is likely that the slow queries are causing the threads to block.

Step 4: Identify the problematic queries in Jira.

From the Jira logs, we can often see the SQL queries that are being executed. Alternatively, we can enable more detailed logging in Jira for SQL queries (if not already enabled) to see which queries are taking a long time.

Step 5: Analyze the slow queries.

Once we have the slow queries, we can analyze them to see why they are slow. Common reasons include missing indexes, table locks, or inefficient query structure.

Step 6: Take corrective action.

This might involve adding indexes, optimizing the query, or increasing database resources.

Now, let's explain to a young colleague:

Hey there,

We are seeing high CPU on the Jira server and many stuck thread messages in the catalina.out. This means that some threads in the Jira application are taking a very long time to complete, and the StuckThreadDetectionValve is detecting them.

At the same time, the database is under high load, so we suspect that the stuck threads are waiting for the database to respond.

Here's what we are going to do:

Look at the catalina.out and find the stuck thread entries. We note the time and the thread details.

We then check the database slow query logs around the same time to see if there are any queries that were running slowly.

We try to find which Jira operations (and hence which database queries) are causing the issue.

Once we identify the slow queries, we can work on optimizing them, for example by adding indexes or rewriting the query.

Let me show you how we can do this:

First, let's look at the catalina.out for stuck thread entries:

We can use grep to filter the stuck thread messages:

grep "StuckThreadDetectionValve" catalina.out

This will give us the lines that indicate stuck threads. We can then look at the context around these lines to see if there is any additional information, such as the request URL or the stack trace.

Second, we check the database slow query logs. For example, in MySQL, the slow query log is defined in the my.cnf file and we can look at the log file to see the slow queries.

We can use the following command to see the slow queries (if the log is in a file):

cat /path/to/mysql-slow.log

But often, we want to look at a specific time range. We can use tools like mysqldumpslow or pt-query-digest to analyze the slow query log.

Third, we try to correlate the times. If we see a stuck thread at 10:00:00 and a slow query that started at 10:00:00 and ran for 60 seconds, that's a good candidate.

Fourth, we identify the query in the Jira logs. We might need to enable SQL logging in Jira to see the exact query. This can be done by adjusting the log4j properties.

However, note that Jira might not log the exact SQL by default. We can enable it by:

Editing the log4j.properties file (or via the Jira administration UI if available) to set the log level for com.atlassian.jira to DEBUG for SQL.

But be cautious: this can generate a lot of logs.

Alternatively, we can look at the database slow query log and see the query and then try to find which part of Jira is issuing that query.

Once we have the slow query, we can analyze it. For example, we can run an EXPLAIN on the query to see the execution plan and check for missing indexes.

Let me know if you have any questions as we go through the logs.

We are going to need to work together to fix this because it might require changes to the database or the Jira configuration.

Let's start by checking the catalina.out for the stuck thread messages and then move to the database logs.

# Ответ

Of course. Let's break this down as if I'm walking a junior colleague through the investigation. This is a classic "firefighting" scenario that teaches you how to connect the dots between different system components.

### **The Situation: A Perfect Storm**

Hello! Let's put on our detective hats. We have two primary symptoms that are almost certainly linked:

1.  **Jira Symptom:** High CPU on the Jira host and a flood of `StuckThreadDetectionValve` messages in `catalina.out`.
2.  **Database Symptom:** High load on the database server.

Think of it like this: Jira (the application) is a team of workers (the threads). The database is the central library where they have to go to get information. Right now, our workers are getting stuck waiting for books from the library, and because they're stuck, they're piling up and causing a traffic jam that uses a lot of CPU.

---

### **Step 1: Understanding the Jira Logs - The "Stuck Thread" Valve**

**What we're looking at:**
The `org.apache.catalina.valves.StuckThreadDetectionValve` is a safety valve in Tomcat (the application server Jira runs on). Its job is to detect when an HTTP request is taking an unusually long time to complete.

**What a typical log entry means:**
You'll see entries that look something like this:

```
WARNING: [Stuck Thread Detection Valve] Thread "http-nio-8080-exec-123" (id=123) has been active for 60,005 milliseconds and may be stuck.
org.apache.catalina.valves.StuckThreadDetectionValve.notifyStuckThreadDetected(StuckThreadDetectionValve.java:220)
...
<<Full Stack Trace here showing what the thread was doing>>
```

*   **`http-nio-8080-exec-123`**: This is the name of the Tomcat thread that's stuck.
*   **`60,005 milliseconds` (60 seconds)**: This is how long the thread has been processing a single request. The default threshold is often 60 seconds.
*   **The Stack Trace**: This is the **most important part**. It tells us exactly what code Jira was executing when the thread got stuck.

**Why I'm looking at this:**
I need to find the common pattern in the stack traces of all these stuck threads. If 90% of them are stuck in the same method or, even better, while executing a database query, we've found our prime suspect.

---

### **Step 2: Investigating the Database - The "Slow Queries"**

The Jira logs point to threads being stuck, and the high database load suggests the database is the bottleneck. Now we need to prove it.

**Where to Look:**
We need to check the database's slow query log. This is configured in the database configuration file (e.g., `my.cnf` for MySQL, `postgresql.conf` for PostgreSQL).

*   **MySQL:** Look for the `slow_query_log` and `long_query_time` settings.
*   **PostgreSQL:** Look for the `log_min_duration_statement` setting.

**What I'm looking for:**
Queries that take longer than the `long_query_time` (e.g., 10 seconds) to execute. The log will show the exact SQL statement and its execution time.

**Example of a Slow Query Log Entry (MySQL):**
```
# Time: 2023-10-25T10:34:12.123456Z
# User@Host: jira_user[@jira_app_host] []
# Query_time: 45.123  Lock_time: 0.001 Rows_sent: 10  Rows_examined: 8500000
SELECT `ID`, `PROJECT`, `SUMMARY` FROM `jiraissue` WHERE `PROJECT` = 10000 AND `UPDATED` > '2023-01-01' ORDER BY `UPDATED` DESC;
```

*   **`Query_time: 45.123`**: This query took 45 seconds! That's a problem.
*   **`Rows_examined: 8500000`**: It had to look at 8.5 million rows to return only 10. This is a huge red flag for a missing index.

---

### **Step 3: Connecting the Dots - The "Smoking Gun"**

Now, we bring the two pieces of evidence together.

1.  From the **Jira Stuck Thread stack trace**, I will see lines that point to database activity. It might mention a specific Jira class or plugin, but the key is it will show the thread is in a "WAITING" or "RUNNABLE" state on a line related to JDBC (Java Database Connectivity), like `java.sql.Statement.executeQuery()`.

2.  From the **Database Slow Query Log**, I have a list of specific, slow SQL statements.

**The "Aha!" Moment:**
I will take the slow SQL queries from the database log and search for them (or parts of them) in the Jira application logs. Jira often logs the SQL it's about to execute, especially under high load or error conditions (look for `atlassian-jira-sql-logger` in your `log4j.properties`).

If I find that the same slow query from the database log appears in the Jira logs just before a thread gets stuck, we have our smoking gun. **That specific query is the cause of the stuck threads, which in turn is causing the high CPU on the Jira server** (as Tomcat uses more CPU to manage all these stuck threads and the operating system context-switches like crazy).

---

### **Summary for the Young Colleague**

So, to recap the investigation plan:

1.  **Jira Logs (`catalina.out`):** Find the `StuckThreadDetectionValve` entries. The stack traces tell us *what Jira was doing* when it got stuck. The common pattern points to the type of operation causing the issue.
2.  **Database Logs (Slow Query Log):** Find the queries that are taking a very long time to run. These are the direct cause of the database load and are likely the reason the Jira threads are stuck.
3.  **Correlation:** Match the slow queries from the database to the operations in the Jira stack traces. This confirms the root cause.

**The Fix:**
The solution will almost always be **database-centric**:

*   **Add an Index:** The most common fix. If a query is slow because it's doing a full table scan (high `Rows_examined`), adding an index on the `WHERE` or `ORDER BY` columns can make it thousands of times faster.
*   **Optimize the Query:** Sometimes, the query itself is poorly written and can be rewritten for better performance.
*   **Analyze the Data Model:** In rare cases, the issue might be a poorly designed table or a missing foreign key.
*   **Check for Locking:** The database logs might also show other queries locking tables, causing our queries to wait.

You've just learned a critical skill: tracing a performance problem from the application server all the way down to a specific, poorly performing line of code (the SQL query). This is the essence of full-stack troubleshooting. Great work asking the right questions
