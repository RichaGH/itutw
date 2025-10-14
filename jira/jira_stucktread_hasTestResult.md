  # Вопрос

well, log analysis led us to the JQL hasTestResult() function. Each access to this function increases the CPU load on the application server by 300% of the CPU. What needs to be checked and done to reduce the load? The Zephyr Scale 9.0.0 plugin from smartbear for jira server is installed in the system.

# Рассуждения

We have identified that the JQL function hasTestResult() is causing high CPU load on the Jira server. This function is likely provided by the Zephyr Scale plugin (from SmartBear) for Jira Server. The function is used in JQL queries to filter issues that have test results.

The high CPU load indicates that the function is performing a resource-intensive operation, possibly by scanning many records or performing complex calculations each time it is called.

Here are the steps to investigate and mitigate the issue:

Understand the Usage of hasTestResult():

The hasTestResult() JQL function is used in JQL queries to find issues that have test results (e.g., test executions). This function is provided by the Zephyr Scale plugin.

Check the Frequency of Queries Using hasTestResult():

If there are many JQL queries using hasTestResult() (e.g., in dashboards, filters, or by users), then each call is triggering the expensive operation. We can check the Jira logs for JQL queries that use this function.

Check the Implementation of hasTestResult():

The Zephyr Scale plugin may have a custom implementation for this JQL function. The high CPU could be due to an inefficient implementation. Since we cannot change the plugin code directly, we must look for workarounds or optimization from the plugin side.

Check for Database Indexes:

The function might be performing a database query that lacks proper indexes. We should check the database for slow queries related to the Zephyr Scale plugin and ensure that the necessary indexes are in place.

Consider Caching:

If the function's results don't change frequently, caching could be a solution. However, since the function is about test results which might change often, caching might not be straightforward.

Reduce the Use of hasTestResult():

If possible, try to reduce the number of JQL queries that use this function. For example, if there are dashboards or filters that use this function frequently, consider if they can be replaced with a less expensive alternative.

Contact the Plugin Vendor:

Since the plugin is from SmartBear, we can contact their support and report the performance issue. They might have a patch or recommendations for optimizing the use of hasTestResult().

Monitor the Database During the Query:

While the JQL function is running, monitor the database to see what queries are being executed and how long they take. This can help identify if the issue is in the database layer.

Check for Plugin Updates:

There might be a newer version of the Zephyr Scale plugin that addresses this performance issue.

Consider Hardware/Resource Upgrades:

If the issue cannot be resolved immediately, and the function is critical, temporarily increasing the CPU resources might help, but this is not a long-term solution.

Steps to Take Immediately:
A. Identify the Queries Using hasTestResult()
Check the Jira logs for JQL queries that contain hasTestResult. You can use:
~~~
bash
grep -i "hasTestResult" atlassian-jira.log
~~~
Also, check the slow JQL queries in the Jira logs.

B. Check Database Performance
Enable slow query logging in the database if not already enabled.

Look for slow queries that are related to the Zephyr Scale plugin. The table names might include zephyr_ or similar.

C. Check for Indexes on Zephyr Scale Tables
The function hasTestResult() likely queries one or more tables in the database. Check the database for indexes on the columns used by this function. You may need to consult Zephyr Scale documentation or the vendor for the required indexes.

D. Review Zephyr Scale Documentation
Look for any known performance issues with hasTestResult() in the Zephyr Scale documentation.

E. Contact SmartBear Support
Raise a support ticket with SmartBear, providing them with the details of the performance issue, including the Jira and database logs.
Example of How to Check for Indexes in MySQL:
~~~
sql
-- List indexes on a table (replace with the actual table name)
SHOW INDEX FROM zephyr_scale_table;
~~~
Example for PostgreSQL:
~~~
sql
-- List indexes on a table
SELECT
    tablename,
    indexname,
    indexdef
FROM
    pg_indexes
WHERE
    tablename LIKE 'zephyr_%';
~~~
Long-term Solution:
If the issue is with the plugin's implementation, the vendor must provide a fix. In the meantime, you can:

Avoid using the hasTestResult() function in JQL queries where possible.

If you must use it, try to combine it with other conditions to limit the number of issues being checked.

Conclusion:
The root cause is likely an inefficient implementation of the hasTestResult() JQL function by the Zephyr Scale plugin. The immediate steps are to identify the frequency of use, check for missing database indexes, and contact the vendor for a fix.

Let me know if you need further assistance with any of these steps.

# Ответ

Excellent detective work! You've narrowed it down to the exact culprit. This is a common but serious performance issue with Jira plugins, especially those that add custom JQL functions. Let me explain this to you step by step.

## **What's Happening: The Root Cause**

The `hasTestResult()` JQL function from Zephyr Scale is performing what we call a **"table scan" or "N+1 query problem"** for every issue it evaluates.

Think of it this way:
- When you run a JQL search with `hasTestResult()`, Jira first finds all issues matching your basic criteria
- Then, for **EACH** of those issues, it calls the `hasTestResult()` function
- Each function call makes one or more database queries to check if that specific issue has test results
- If you're searching across 1,000 issues, that's 1,000+ separate database queries!

This explains the 300% CPU spike - the Jira server is making thousands of rapid-fire database calls.

## **Immediate Action Plan**

### **Step 1: Identify the Heavy Users**
Check which dashboards, filters, or users are using `hasTestResult()` most frequently:

```sql
-- Check for recent searches with hasTestResult()
SELECT * FROM searchrequest WHERE LOWER(querystring) LIKE '%hastestresult%';

-- Check dashboard usage (you may need to check application logs for this)
grep -i "hasTestResult" atlassian-jira.log
```

### **Step 2: Immediate Mitigations**

**A. Educate Users:**
- Ask teams to avoid using `hasTestResult()` in board filters or dashboard gadgets
- Suggest alternatives like searching by issue type (Test, Test Case) or specific test-related labels

**B. Create Saved Filters:**
Instead of everyone running `project = ABC AND hasTestResult()`, create a saved filter and update it periodically.

**C. Adjust Board Configurations:**
Check if any active boards use this function in their filters.

### **Step 3: Technical Investigation**

**Check Zephyr Scale Version Compatibility:**
```bash
# Check your exact Zephyr Scale version and Jira compatibility
ls /path/to/jira/atlassian-jira/plugins/ | grep zephyr
```

**Review Zephyr Scale Configuration:**
- Check if there are performance settings in Zephyr Scale administration
- Look for any caching options for test results

### **Step 4: Database Optimization**

Even while we work on the root cause, we can help the database handle the load better:

```sql
-- Check for missing indexes on Zephyr tables
-- Common Zephyr tables that might need indexes:
-- "testresult", "stepresult", "teststep", "execution"

-- Example: Check if testresult table has indexes on issue_id
SELECT * FROM pg_indexes WHERE tablename LIKE '%test%' AND tablename LIKE '%result%';
```

### **Step 5: Monitor and Validate**

**Create a monitoring baseline:**
```bash
# Monitor CPU during known heavy usage periods
top -p $(pgrep -f jira)

# Check active threads in Jira
grep "StuckThreadDetectionValve" catalina.out | wc -l
```

## **Long-Term Solutions**

### **Option A: Zephyr Scale Upgrade**
Check if SmartBear has addressed this in newer versions. This is often the best solution.

### **Option B: Custom Cached JQL Function**
If you have development resources, you could create a custom JQL function that caches results, but this requires careful implementation.

### **Option C: Alternative Workflows**
- Use issue properties or custom fields to mark issues with test results
- Create periodic jobs to update a "has_test_results" custom field
- Use Jira Automation to maintain this field when test results are added

## **How to Explain This to Your Team**

"Team, we've identified that the `hasTestResult()` search function is causing significant performance issues. Each time this function runs, it makes individual database calls for every issue in your search results. 

**Immediate ask:** Please avoid using `hasTestResult()` in active boards, dashboard gadgets, or frequent searches. 

**Alternative:** Use issue types like 'Test Case' or specific labels to filter test-related issues instead."

## **Expected Outcome**

After implementing these changes, you should see:
- CPU usage drop significantly during peak hours
- Fewer or no `StuckThreadDetectionValve` warnings
- Reduced database load
- Faster JQL search performance

The key insight here is that custom JQL functions can be deceptively expensive. What looks like a simple search filter to users can actually generate massive database load behind the scenes.

Would you like me to help you implement any of these specific steps, or do you need help crafting the communication to your user community?
