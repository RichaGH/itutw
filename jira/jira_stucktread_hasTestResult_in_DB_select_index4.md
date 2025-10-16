# Вопрос

You cant change text of request. But you can add indexes. how do I figure out which index or several indexes to build in order to get the maximum query performance? How do I use the scheduler and statistics correctly to select the best index configuration?

# Рассуждения

We are given a query that we cannot change, but we can add indexes.
The query is:
~~~
SELECT issuelink."ID", issuelink."ISSUE_ID"
FROM public."MY_TABLE1" issuelink
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID"
JOIN public."MY_TABLE3" testresult ON issuelink."TEST_CASE_ID" = testresult."TEST_CASE_ID"
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = $1)
AND issuelink."ISSUE_ID" IS NOT NULL
AND issuelink."TEST_CASE_ID" IS NOT NULL;
~~~
We know the table sizes:
issuelink (MY_TABLE1): 286,949 records
testcase (MY_TABLE2): 266,798 records
testresult (MY_TABLE3): 10,360,434 records

The problem is that the join to testresult is causing a cartesian product (or at least a many-to-many) without any additional filters.

Since we cannot change the query, we must rely on indexes.

Steps to determine the best indexes:

Analyze the query and the join conditions and where clauses.

Consider the selectivity of each condition.

Use PostgreSQL's query planner (EXPLAIN) to see the current plan and then try to improve it.

Let's break down the query:

Tables and their conditions:

issuelink (MY_TABLE1):
- Conditions: "ISSUE_ID" IS NOT NULL, "TEST_CASE_ID" IS NOT NULL
- Join on: "TEST_CASE_ID" = testcase."ID"

testcase (MY_TABLE2):
- Conditions: (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = $1)
- Join on: testcase."ID" = issuelink."TEST_CASE_ID"

testresult (MY_TABLE3):
- Join on: issuelink."TEST_CASE_ID" = testresult."TEST_CASE_ID"

We note that the join between issuelink and testresult is on "TEST_CASE_ID", and there are no additional conditions on testresult.

The massive number of records in testresult (10 million) and the fact that we are joining without any additional filters means that we are likely generating a huge intermediate result.

However, note that the query is only selecting two columns from issuelink.

How can we speed this up?

Approach:

Make sure that the joins are efficient by having indexes on the join columns.

Consider composite indexes that include the where clause conditions and the join columns.

Let's consider the following indexes:

For issuelink (MY_TABLE1):

We are using "TEST_CASE_ID" for joining to both testcase and testresult.

We have conditions on "ISSUE_ID" and "TEST_CASE_ID" (both are NOT NULL). Since these are conditions on the table, we can consider an index that includes these columns, but note that the condition is just IS NOT NULL, which is not very selective.

However, the join condition is on "TEST_CASE_ID", so we want to have an index on "TEST_CASE_ID" to speed up the joins.

Also, we are selecting "ID" and "ISSUE_ID". We don't need to include these in the index for the join, but if we create a composite index we might be able to use an index-only scan.

But note: the table issuelink is the smallest (286k records). So an index on "TEST_CASE_ID" might be enough.

Let's consider:
Index on issuelink ("TEST_CASE_ID")

But wait, we are also joining testresult on the same column. So this index will be used for both joins.

For testcase (MY_TABLE2):

We are joining on "ID", so an index on "ID" is essential (but if it's the primary key, it's already indexed).

We have a condition on "ARCHIVED". We can consider a composite index on ("ARCHIVED", "ID") to help with the condition and the join.

Since the condition on "ARCHIVED" is an OR condition, it might be better to have an index on "ARCHIVED" and then use the primary key index for the join. However, let's see:

The condition is: (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = $1)

We can create a partial index for the condition, but note that the condition is an OR. Alternatively, we can create an index on "ARCHIVED" and then let the planner use the index on "ID" for the join.

But note: the table testcase has 266k records. The condition on "ARCHIVED" might be selective? We don't know the distribution.

Let's consider:
Index on testcase ("ARCHIVED", "ID")

Why? This index can be used to filter by "ARCHIVED" and then join by "ID". But note: the condition is OR, so it might not use the index efficiently.

Alternatively, we can create two indexes and let the planner choose:
~~~
text
 Index on testcase ("ARCHIVED")   [for the condition]
 Index on testcase ("ID")         [for the join, but probably already exists as primary key]
~~~
For testresult (MY_TABLE3):

We are joining on "TEST_CASE_ID". This is the most critical index because testresult is the largest table (10 million records).

We must have an index on testresult ("TEST_CASE_ID"). Without this, the join will be very slow.

Also, note that we are not using any other conditions on testresult. So a simple index on "TEST_CASE_ID" should help.

However, consider: if there are multiple test results per test case, then the join will multiply the rows. We cannot avoid that without changing the query, but we can make the join faster.

We can also consider composite indexes that cover the entire where clause and the join, but note that the query is only selecting from issuelink, so we don't need to cover columns from testresult or testcase in the index for those tables.

Let's summarize the indexes to create:

issuelink: index on "TEST_CASE_ID"

testcase: index on "ARCHIVED" (if the condition is selective) and ensure primary key index on "ID" exists.

testresult: index on "TEST_CASE_ID"

But wait, the query is joining testresult on "TEST_CASE_ID", and then we are not using any other condition. So the index on testresult."TEST_CASE_ID" is crucial.

However, let's think about the order of joins. The query planner might choose to start from testcase, then join to issuelink, then join to testresult. Or it might start from testresult. We don't know.

We can use the following steps to test:

Step 1: Check the current execution plan without indexes (if there are no indexes now) or with current indexes.

Step 2: Create the indexes and then check the execution plan again.

Step 3: Use the statistics to see if the indexes are being used.

How to check the execution plan:

Use EXPLAIN (ANALYZE, BUFFERS) with the query. But note: the query takes 20+ seconds, so we might want to run it in a controlled environment.

Step 4: If the performance is still not acceptable, we might need to consider more advanced strategies, such as:

Creating a composite index on testresult that includes the join and then including the testresult columns that are used in the query (but none are used in the select, so maybe not).

Alternatively, we can try to create a materialized view that pre-joins the tables with the necessary conditions and then query the materialized view. But note: we cannot change the query. So unless we can replace the table with a materialized view, this is not an option.

We can also try to cluster the tables by the join keys.

Step 5: Use PostgreSQL's statistics to see if the indexes are being used.

We can query pg_stat_user_indexes to see if the indexes are being scanned.

Step 6: Consider the possibility of partitioning the testresult table by TEST_CASE_ID if the data is too large.

However, let's start with the indexes.

Proposed indexes:

CREATE INDEX CONCURRENTLY idx_issuelink_test_case_id ON public."MY_TABLE1" ("TEST_CASE_ID");
CREATE INDEX CONCURRENTLY idx_testcase_archived ON public."MY_TABLE2" ("ARCHIVED");
CREATE INDEX CONCURRENTLY idx_testresult_test_case_id ON public."MY_TABLE3" ("TEST_CASE_ID");

But note: the condition on testcase is an OR of two conditions. The index on "ARCHIVED" might be used for the condition, but then the join on "ID" might use the primary key index.

Alternatively, we can create a composite index on testcase that includes both the condition and the join key? Let's see:

We have two conditions:
testcase."ARCHIVED" IS NULL
testcase."ARCHIVED" = $1

And we are joining on testcase."ID"

We can create an index on testcase ("ARCHIVED", "ID"). This index can be used to filter by "ARCHIVED" and then by "ID". But note: the condition is OR, so the planner might not use the index for both conditions.

Alternatively, we can create two partial indexes:
~~~
text
CREATE INDEX CONCURRENTLY idx_testcase_archived_null ON public."MY_TABLE2" ("ID") WHERE "ARCHIVED" IS NULL;
CREATE INDEX CONCURRENTLY idx_testcase_archived_value ON public."MY_TABLE2" ("ID") WHERE "ARCHIVED" = $1;
~~~
But we don't know the value of $1. And if $1 can change, then we cannot create a partial index for the value.

So, let's stick to the index on "ARCHIVED" and rely on the primary key index for the join.

However, if the condition on "ARCHIVED" is not very selective, then the planner might choose to do a sequential scan on testcase and then use the primary key index for the join. We'll let the planner decide.

Step 7: After creating the indexes, we must update the statistics:
~~~
text
ANALYZE public."MY_TABLE1";
ANALYZE public."MY_TABLE2";
ANALYZE public."MY_TABLE3";
~~~
Step 8: Test the query again.

Step 9: If the performance is still not acceptable, we can try to create a composite index on issuelink that includes both the join and the conditions. But note: the conditions on issuelink are only IS NOT NULL, which are not selective.

Alternatively, we can try to create a composite index on testresult that includes the join column and then includes the testresult's primary key? But note: the query does not use any column from testresult, so we don't need to include any other columns.

The join is only on TEST_CASE_ID, so the index on TEST_CASE_ID is sufficient.

Step 10: If the query is still slow, we can try to adjust the configuration parameters, such as work_mem, to see if that helps.

However, the main problem is the massive join between issuelink and testresult. Without changing the query, we are limited.

Another idea: we can try to create a multi-column index on testresult that includes both TEST_CASE_ID and the primary key (if the primary key is used for something else, but note the query doesn't use it). But I don't think that will help.

Let's recap the expected execution with indexes:
~~~
text
We want the planner to use the index on testresult (TEST_CASE_ID) to quickly find the test results for each test case, and then use the index on issuelink (TEST_CASE_ID) to quickly find the issuelink records for the test case.

We also want the planner to use the index on testcase (ARCHIVED) to quickly filter testcase records, and then use the primary key index to join with issuelink.
~~~
But note: the query is joining testresult to issuelink directly on TEST_CASE_ID, and then also joining testcase to issuelink on TEST_CASE_ID (which is the same as testcase.ID). So the plan might be:
~~~
text
Hash Join between issuelink and testresult on TEST_CASE_ID
Then Hash Join with testcase on TEST_CASE_ID (or ID)
~~~
Alternatively, the planner might start with testcase, then join to issuelink, then join to testresult.

Without the indexes, the planner might be doing sequential scans and hash joins, which are expensive.

With the indexes, we hope for index scans and nested loops if the number of rows is small, but if the number of rows is large, then hash joins might still be used.

We must let the planner choose.

Step 11: If the indexes do not help, we can try to use the pg_stat_statements view to see if the query is still slow, and then try to adjust the configuration.

Step 12: Consider hardware and configuration: enough memory? enough work_mem?

Given the table sizes, the join between issuelink and testresult could produce a huge number of rows (if one test case has many test results). We are only selecting two columns from issuelink, but the intermediate result might be too large.

If the indexes do not help, we might have to consider:
~~~
text
- Partitioning the testresult table by TEST_CASE_ID.
- Creating a summary table that pre-joins issuelink and testresult and then updating it periodically.
~~~
But again, we cannot change the query.

Let's try the indexes and see.

Summary of indexes to create:

ON public."MY_TABLE1" ("TEST_CASE_ID")

ON public."MY_TABLE2" ("ARCHIVED")

ON public."MY_TABLE3" ("TEST_CASE_ID")

And then analyze the tables.

We can also try to set enable_seqscan to off temporarily to force index usage, but that is not recommended for production.

Step 13: Monitor the query performance after index creation.

We can use the following query to check the performance of the query from pg_stat_statements:
~~~
text
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
WHERE query LIKE '%issuelink%MY_TABLE1%testcase%MY_TABLE2%testresult%MY_TABLE3%'
~~~
But note: the query text might be normalized.

Alternatively, we can look for the slowest queries and see if our query is still there.

Step 14: If the performance is still not acceptable, we might need to consider more drastic measures, such as:
~~~
text
- Talking to the developers to change the query (even though we said we can't, maybe we can show them the performance issue)
- Considering a trigger to maintain a materialized view that pre-computes the existence of test results for each test case and then using that in the query (but again, we cannot change the query).
~~~
However, let's not get ahead of ourselves. Let's try the indexes first.

Let me know if you need help with the exact commands to create the indexes and analyze the tables.

# Ответ

Perfect! Since we can't change the query text, we need to use strategic indexing. Let me show you a systematic approach to find the optimal indexes.

## **Step 1: Analyze the Current Query Plan**

First, let's see what PostgreSQL is actually doing:

```sql
-- Get the detailed execution plan
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) 
SELECT issuelink."ID", issuelink."ISSUE_ID" 
FROM public."MY_TABLE1" issuelink 
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID" 
JOIN public."MY_TABLE3" testresult ON issuelink."TEST_CASE_ID" = testresult."TEST_CASE_ID" 
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = false) 
  AND issuelink."ISSUE_ID" IS NOT NULL 
  AND issuelink."TEST_CASE_ID" IS NOT NULL;

-- Simpler version for quick analysis
EXPLAIN (ANALYZE) 
SELECT issuelink."ID", issuelink."ISSUE_ID" 
FROM public."MY_TABLE1" issuelink 
JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID" 
JOIN public."MY_TABLE3" testresult ON issuelink."TEST_CASE_ID" = testresult."TEST_CASE_ID" 
WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = false) 
  AND issuelink."ISSUE_ID" IS NOT NULL 
  AND issuelink."TEST_CASE_ID" IS NOT NULL;
```

## **Step 2: Check Current Indexes**

```sql
-- Check existing indexes on these tables
SELECT 
    tablename,
    indexname,
    indexdef
FROM pg_indexes 
WHERE schemaname = 'public' 
  AND tablename IN ('MY_TABLE1', 'MY_TABLE2', 'MY_TABLE3')
ORDER BY tablename, indexname;

-- Check index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as index_scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes 
WHERE tablename IN ('MY_TABLE1', 'MY_TABLE2', 'MY_TABLE3');
```

## **Step 3: Systematic Index Strategy**

Based on your query pattern, here are the candidate indexes to test:

### **Priority 1: Join Condition Indexes**
```sql
-- Most critical: Indexes on join columns
CREATE INDEX CONCURRENTLY idx_mytable1_test_case_id ON public."MY_TABLE1" ("TEST_CASE_ID");
CREATE INDEX CONCURRENTLY idx_mytable3_test_case_id ON public."MY_TABLE3" ("TEST_CASE_ID");

-- For MY_TABLE2, if "ID" is not the primary key
CREATE INDEX CONCURRENTLY idx_mytable2_id ON public."MY_TABLE2" ("ID");
```

### **Priority 2: Composite Indexes for WHERE clauses**
```sql
-- For the ARCHIVED filter on MY_TABLE2
CREATE INDEX CONCURRENTLY idx_mytable2_archived_id ON public."MY_TABLE2" ("ARCHIVED", "ID");

-- For the NOT NULL filters on MY_TABLE1  
CREATE INDEX CONCURRENTLY idx_mytable1_test_case_issue ON public."MY_TABLE1" ("TEST_CASE_ID", "ISSUE_ID") 
WHERE "ISSUE_ID" IS NOT NULL AND "TEST_CASE_ID" IS NOT NULL;
```

### **Priority 3: Covering Indexes**
```sql
-- Covering index that includes all selected columns
CREATE INDEX CONCURRENTLY idx_mytable1_covering ON public."MY_TABLE1" ("TEST_CASE_ID") 
INCLUDE ("ID", "ISSUE_ID")
WHERE "ISSUE_ID" IS NOT NULL AND "TEST_CASE_ID" IS NOT NULL;
```

## **Step 4: Use the Index Advisor Approach**

Let's test different index combinations systematically:

```sql
-- Create a test schema to experiment safely
CREATE SCHEMA IF NOT EXISTS index_testing;

-- Copy a subset of data for testing (optional, for large tables)
CREATE TABLE index_testing.my_table1 AS 
SELECT * FROM public."MY_TABLE1" WHERE random() < 0.1; -- 10% sample

-- Test different index combinations and measure performance
```

## **Step 5: Automated Index Testing Script**

Create a script to test different index configurations:

```sql
-- Script to test index combinations
DO $$
DECLARE
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    duration INTERVAL;
    test_case TEXT;
BEGIN
    -- Test case 1: No additional indexes (baseline)
    test_case := 'Baseline (existing indexes)';
    start_time := clock_timestamp();
    PERFORM COUNT(*) FROM (
        SELECT issuelink."ID", issuelink."ISSUE_ID" 
        FROM public."MY_TABLE1" issuelink 
        JOIN public."MY_TABLE2" testcase ON issuelink."TEST_CASE_ID" = testcase."ID" 
        JOIN public."MY_TABLE3" testresult ON issuelink."TEST_CASE_ID" = testresult."TEST_CASE_ID" 
        WHERE (testcase."ARCHIVED" IS NULL OR testcase."ARCHIVED" = false) 
          AND issuelink."ISSUE_ID" IS NOT NULL 
          AND issuelink."TEST_CASE_ID" IS NOT NULL
    ) sub;
    end_time := clock_timestamp();
    duration := end_time - start_time;
    RAISE NOTICE 'Test %: %', test_case, duration;

    -- Create index combination 1
    CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_test1 ON public."MY_TABLE1" ("TEST_CASE_ID");
    CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_test2 ON public."MY_TABLE3" ("TEST_CASE_ID");
    
    -- Test case 2: With basic join indexes
    test_case := 'Basic join indexes';
    start_time := clock_timestamp();
    PERFORM COUNT(*) FROM (/* same query */) sub;
    end_time := clock_timestamp();
    duration := end_time - start_time;
    RAISE NOTICE 'Test %: %', test_case, duration;
    
    -- Continue with other index combinations...
    
    -- Cleanup test indexes
    DROP INDEX CONCURRENTLY IF EXISTS idx_test1;
    DROP INDEX CONCURRENTLY IF EXISTS idx_test2;
END $$;
```

## **Step 6: Use PostgreSQL Statistics**

```sql
-- Check table statistics
SELECT 
    schemaname,
    tablename,
    attname AS column_name,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats 
WHERE tablename IN ('MY_TABLE1', 'MY_TABLE2', 'MY_TABLE3')
  AND attname IN ('TEST_CASE_ID', 'ARCHIVED', 'ISSUE_ID', 'ID');

-- Check for correlation (how well physical order matches logical order)
SELECT 
    tablename,
    attname,
    correlation
FROM pg_stats 
WHERE tablename IN ('MY_TABLE1', 'MY_TABLE2', 'MY_TABLE3')
  AND attname IN ('TEST_CASE_ID', 'ARCHIVED', 'ISSUE_ID', 'ID');
```

## **Step 7: Recommended Index Testing Order**

Based on your data sizes, test in this order:

### **Phase 1: Critical Join Indexes**
```sql
-- These will likely give the biggest bang for buck
CREATE INDEX CONCURRENTLY idx_mytable3_test_case_id ON public."MY_TABLE3" ("TEST_CASE_ID");
CREATE INDEX CONCURRENTLY idx_mytable1_test_case_id ON public."MY_TABLE1" ("TEST_CASE_ID");
```

### **Phase 2: Filter Optimization**
```sql
-- Optimize the ARCHIVED filter
CREATE INDEX CONCURRENTLY idx_mytable2_archived_filter ON public."MY_TABLE2" ("ARCHIVED") 
WHERE "ARCHIVED" IS NULL OR "ARCHIVED" = false;
```

### **Phase 3: Covering Indexes**
```sql
-- If still needed, create covering indexes
CREATE INDEX CONCURRENTLY idx_mytable1_covering ON public."MY_TABLE1" ("TEST_CASE_ID") 
INCLUDE ("ID", "ISSUE_ID");
```

## **Step 8: Monitor Index Usage**

After creating indexes, monitor their usage:

```sql
-- Check which indexes are being used
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes 
WHERE tablename IN ('MY_TABLE1', 'MY_TABLE2', 'MY_TABLE3')
ORDER BY idx_scan DESC;

-- Check index size
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as size
FROM pg_indexes 
WHERE tablename IN ('MY_TABLE1', 'MY_TABLE2', 'MY_TABLE3');
```

## **Step 9: Expected Performance Targets**

With proper indexing, you should aim for:

- **Initial goal:** Reduce from 21 seconds to 1-2 seconds
- **Good performance:** 200-500ms  
- **Excellent performance:** < 100ms

## **Immediate Action Plan**

1. **Run the EXPLAIN ANALYZE** to see current query plan
2. **Check existing indexes** to avoid duplicates
3. **Start with Phase 1 indexes** - these alone might solve 80% of the problem
4. **Test after each index** using the query timing
5. **Monitor index usage** to ensure they're being used

The key insight is that the massive `testresult` table (10M+ records) needs an index on the join column `TEST_CASE_ID`, and the other tables need indexes to support their roles in the query.

Would you like me to help you interpret the EXPLAIN ANALYZE output once you run it? That will tell us exactly where the time is being spent and which indexes will be most effective.
