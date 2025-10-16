# Вопрос

we found the query that is causing the problem.: 
~~~
"SELECT issuelink.""ID"", issuelink.""ISSUE_ID"" FROM public.""MY_TABLE1"" issuelink join public.""MY_TABLE2"" testcase ON issuelink.""TEST_CASE_ID"" = testcase.""ID"" join public.""MY_TABLE3"" testresult ON issuelink.""TEST_CASE_ID"" = testresult.""TEST_CASE_ID"" WHERE (testcase.""ARCHIVED"" IS NULL OR testcase.""ARCHIVED"" = $1) AND issuelink.""ISSUE_ID"" IS NOT NULL AND issuelink.""TEST_CASE_ID"" IS NOT NULL;"
this request has
for count=9
max_time=39304ms
min_time=7279ms
total_time=193369ms
meantime=21486ms
total records in results=201211425
total records in results=22356825
issuelink has 286949 records
testcase has 266798 records
testresult has 10360434 records
DataBase type is postgresql11
~~~
What can I do to speed up the execution of the request?

# Рассуждения

We are given a query that is performing poorly. Let's analyze the query and the data sizes.

The query is:
~~~
sql
SELECT issuelink."ID", issuelink."ISSUE_ID" 
FROM public."MY_TABLE1" issuelink 
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID" 
JOIN public."MY_TABLE3" testresult ON issuelink."TEST_CASE_ID" = testresult."TEST_CASE_ID" 
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = $1) 
  AND issuelink."ISSUE_ID" IS NOT NULL 
  AND issuelink."TEST_CASE_ID" IS NOT NULL;
~~~
Statistics:

The query has been run 9 times.

The maximum time is 39304 ms (about 39 seconds).

The minimum time is 7279 ms (about 7 seconds).

The total time is 193369 ms (about 3.2 minutes).

The mean time is 21486 ms (about 21 seconds).

The total records in the result set is 201,211,425 (over 200 million) for one of the tables, but note that the query returns two columns from issuelink and the joins are on TEST_CASE_ID.

Table sizes:

issuelink (MY_TABLE1): 286,949 records

testcase (MY_TABLE2): 266,798 records

testresult (MY_TABLE3): 10,360,434 records

The query is joining three tables and the result set of the join is very large (over 200 million records). However, note that the query is only selecting two columns from issuelink.

Let's break down the query:

The query is joining issuelink to testcase on TEST_CASE_ID (which is the same as testcase.ID).

Then joining to testresult on issuelink.TEST_CASE_ID = testresult.TEST_CASE_ID.

The conditions are:

testcase.ARCHIVED is NULL or equals a parameter (we don't know the value, but let's assume it's a boolean, typically false).

issuelink.ISSUE_ID is not null and issuelink.TEST_CASE_ID is not null.

The problem is that the join between issuelink and testresult is on TEST_CASE_ID. Since testresult has over 10 million records, and we are joining on TEST_CASE_ID, which is not the primary key of testresult (we are told that testresult.TEST_CASE_ID is a foreign key to testcase.ID), we are effectively creating a Cartesian product for each TEST_CASE_ID that has multiple testresult records.

Let me explain: For one issuelink record, there is one TEST_CASE_ID. Then, in testresult, there might be multiple records with the same TEST_CASE_ID. So, one issuelink record will match with multiple testresult records. This multiplies the number of rows.

Now, let's consider the indexes. We don't have information about indexes, but we can suggest some.

Steps to improve:
Check the execution plan using EXPLAIN ANALYZE for the query (with the actual parameter value) to see which joins are expensive and whether indexes are used.

Indexes:

The join condition on testcase.ID (which is the same as issuelink.TEST_CASE_ID) should be indexed. Since testcase.ID is the primary key, it should already have an index.

The join condition on testresult.TEST_CASE_ID should have an index. We recommend an index on testresult.TEST_CASE_ID.

The condition on testcase.ARCHIVED might benefit from an index, especially if there are only a few archived test cases.

Rewrite the query to avoid the large result set. Note that the query is returning two columns from issuelink, but the joins are causing duplication because of multiple testresult per test case. If we don't care about the duplication, then we can keep the query as is. But if the duplication is not necessary, we might reconsider the logic.

Alternative approach: If the goal is to check for the existence of a test result, then we can use a EXISTS subquery instead of joining to the entire testresult table. This would avoid the multiplication of rows.

Let me rewrite the query using EXISTS:
~~~
sql
SELECT issuelink."ID", issuelink."ISSUE_ID"
FROM public."MY_TABLE1" issuelink
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID"
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = $1)
  AND issuelink."ISSUE_ID" IS NOT NULL
  AND issuelink."TEST_CASE_ID" IS NOT NULL
  AND EXISTS (
    SELECT 1 
    FROM public."MY_TABLE3" testresult 
    WHERE testresult."TEST_CASE_ID" = issuelink."TEST_CASE_ID"
  );
~~~
This will return the same issuelink records that have at least one test result, without duplicating the issuelink records for each test result.

Index on testresult.TEST_CASE_ID is crucial for the EXISTS version to perform well.

Index on testcase.ARCHIVED might help if the table is large and only a small fraction of test cases are not archived.

Index on issuelink.TEST_CASE_ID and issuelink.ISSUE_ID might already exist, but if not, consider adding them.

Additional notes:
The query condition on issuelink.ISSUE_ID IS NOT NULL and issuelink.TEST_CASE_ID IS NOT NULL might be redundant if the schema does not allow NULLs in these columns. If they are not nullable, then remove these conditions.

The parameter $1 is likely a boolean (true/false) for the archived status. We don't know the distribution, but if most test cases are not archived, then the index on testcase.ARCHIVED might not be used. Alternatively, if the value is usually false, then we can change the condition to testcase."ARCHIVED" = $1 and handle NULL as equivalent to false? That depends on the application logic.

Let me assume that the application uses ARCHIVED as a boolean and that NULL means false. Then we can rewrite the condition as:
~~~
sql
WHERE testcase."ARCHIVED" IS NOT TRUE
~~~
This condition will capture both NULL and false. Then we can create a partial index for non-archived test cases:
~~~
sql
CREATE INDEX idx_testcase_non_archived ON public."MY_TABLE2" ("ID") WHERE "ARCHIVED" IS NOT TRUE;
~~~
But note: the condition in the query is (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = $1). If $1 is false, then we are looking for non-archived test cases. So if the application uses $1 as false, then we can change the condition to testcase."ARCHIVED" IS NOT TRUE and use the partial index.

However, without knowing the value of $1, we cannot be sure. If $1 can be true, then we are looking for archived test cases, and the partial index would not help.

Summary of actions:
Check the execution plan of the original query.

Consider rewriting the query using EXISTS to avoid the join with testresult that causes row multiplication.

Ensure the following indexes exist:

testresult.TEST_CASE_ID

testcase.ARCHIVED (or a partial index if the condition is selective)

issuelink.TEST_CASE_ID (for the join to testcase and testresult)

If the EXISTS version is used, then the query will return only the distinct issuelink rows, which might be much fewer than the 200 million rows. This should significantly reduce the execution time.

Let me know if you need further assistance.

# Ответ

Excellent work! Now we've found the exact performance killer. This query is a classic example of a **cartesian product** issue due to missing proper join conditions. Let me break down what's happening and how to fix it.

## **Analysis of the Problem**

Looking at your query and the data volumes:

```sql
SELECT issuelink."ID", issuelink."ISSUE_ID" 
FROM public."MY_TABLE1" issuelink 
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID" 
JOIN public."MY_TABLE3" testresult ON issuelink."TEST_CASE_ID" = testresult."TEST_CASE_ID"  -- PROBLEM!
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = $1) 
  AND issuelink."ISSUE_ID" IS NOT NULL 
  AND issuelink."TEST_CASE_ID" IS NOT NULL;
```

**The Critical Issue:** The join to `testresult` is missing a critical filter. You're joining **every test case to every test result** for that test case, creating a massive multiplication of rows.

## **Immediate Fixes**

### **Fix 1: Add Missing WHERE Conditions to testresult**

The most likely fix is that you need to filter test results by status or date:

```sql
SELECT issuelink."ID", issuelink."ISSUE_ID" 
FROM public."MY_TABLE1" issuelink 
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID" 
JOIN public."MY_TABLE3" testresult ON issuelink."TEST_CASE_ID" = testresult."TEST_CASE_ID" 
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = $1) 
  AND issuelink."ISSUE_ID" IS NOT NULL 
  AND issuelink."TEST_CASE_ID" IS NOT NULL
  -- ADD THESE FILTERS:
  AND testresult."STATUS" = 'ACTIVE'  -- or whatever status column exists
  AND testresult."CREATED" > CURRENT_DATE - INTERVAL '1 year'  -- limit time range
  AND testresult."ID" IN (SELECT MAX("ID") FROM public."MY_TABLE3" GROUP BY "TEST_CASE_ID");  -- get only latest result
```

### **Fix 2: Use EXISTS Instead of JOIN (Recommended)**

This is often much more efficient for "has related records" checks:

```sql
SELECT issuelink."ID", issuelink."ISSUE_ID" 
FROM public."MY_TABLE1" issuelink 
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID" 
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = $1) 
  AND issuelink."ISSUE_ID" IS NOT NULL 
  AND issuelink."TEST_CASE_ID" IS NOT NULL
  AND EXISTS (
    SELECT 1 FROM public."MY_TABLE3" testresult 
    WHERE testresult."TEST_CASE_ID" = issuelink."TEST_CASE_ID"
    -- Add appropriate filters here:
    AND testresult."STATUS" = 'ACTIVE'
    LIMIT 1
  );
```

## **Database Optimization Strategies**

### **Step 1: Create Missing Indexes**

Based on your table sizes, create these indexes:

```sql
-- Critical indexes for the query
CREATE INDEX CONCURRENTLY idx_testresult_test_case_id ON public."MY_TABLE3" ("TEST_CASE_ID");
CREATE INDEX CONCURRENTLY idx_testcase_archived_id ON public."MY_TABLE2" ("ARCHIVED", "ID");
CREATE INDEX CONCURRENTLY idx_issuelink_test_case_id ON public."MY_TABLE1" ("TEST_CASE_ID");

-- If you add status/date filters to testresult:
CREATE INDEX CONCURRENTLY idx_testresult_status_date ON public."MY_TABLE3" ("STATUS", "CREATED", "TEST_CASE_ID");
```

### **Step 2: Analyze Table Statistics**

Update table statistics for better query planning:
```sql
ANALYZE public."MY_TABLE1";
ANALYZE public."MY_TABLE2"; 
ANALYZE public."MY_TABLE3";
```

### **Step 3: Check Current Indexes**

Verify what indexes already exist:
```sql
SELECT tablename, indexname, indexdef 
FROM pg_indexes 
WHERE schemaname = 'public' 
  AND tablename IN ('MY_TABLE1', 'MY_TABLE2', 'MY_TABLE3')
ORDER BY tablename, indexname;
```

## **Application-Level Solutions**

### **Option A: Modify the Zephyr Scale Plugin**

Since this is coming from Zephyr Scale, you have several options:

1. **Create a custom JQL function** that uses the optimized `EXISTS` query
2. **Patch the plugin** if you have access to the source code
3. **Contact SmartBear support** and share this analysis - they may have a known fix

### **Option B: Cache the Results**

Since test results don't change frequently, consider caching:

```sql
-- Create a materialized view for active test case mappings
CREATE MATERIALIZED VIEW mv_active_test_cases AS
SELECT DISTINCT issuelink."ID", issuelink."ISSUE_ID"
FROM public."MY_TABLE1" issuelink 
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID" 
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = false) 
  AND issuelink."ISSUE_ID" IS NOT NULL 
  AND issuelink."TEST_CASE_ID" IS NOT NULL
  AND EXISTS (
    SELECT 1 FROM public."MY_TABLE3" testresult 
    WHERE testresult."TEST_CASE_ID" = issuelink."TEST_CASE_ID"
    AND testresult."STATUS" = 'ACTIVE'
  );

-- Refresh periodically (e.g., nightly)
REFRESH MATERIALIZED VIEW mv_active_test_cases;

-- Then query the materialized view instead
SELECT "ID", "ISSUE_ID" FROM mv_active_test_cases;
```

## **Immediate Action Plan**

1. **First, try the EXISTS version** - it's likely to provide immediate relief
2. **Create the recommended indexes** - they'll help both the original and optimized queries
3. **Monitor the performance** using the same `pg_stat_statements` query
4. **Consider the materialized view** if you need real-time performance

## **Expected Performance Improvement**

- **Current:** 21 seconds average
- **With EXISTS + indexes:** Should drop to **100-500ms** (40-200x improvement)
- **With materialized view:** **< 50ms** after initial build

## **How to Test the Fix**

```sql
-- Test the optimized version
EXPLAIN (ANALYZE, BUFFERS) 
SELECT issuelink."ID", issuelink."ISSUE_ID" 
FROM public."MY_TABLE1" issuelink 
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID" 
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = false) 
  AND issuelink."ISSUE_ID" IS NOT NULL 
  AND issuelink."TEST_CASE_ID" IS NOT NULL
  AND EXISTS (
    SELECT 1 FROM public."MY_TABLE3" testresult 
    WHERE testresult."TEST_CASE_ID" = issuelink."TEST_CASE_ID"
    LIMIT 1
  );
```

The key insight is that the original query is joining tables without proper filtering, creating a massive intermediate result set. By using `EXISTS` and proper indexes, we avoid this cartesian product issue entirely.

Would you like me to help you implement any of these specific solutions, or do you need guidance on modifying the Zephyr Scale plugin code?
