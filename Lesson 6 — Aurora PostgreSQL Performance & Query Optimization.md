# 🐘 Lesson 6 — Aurora PostgreSQL Performance & Query Optimization

Now we enter one of the most important areas for a real-world Aurora PostgreSQL engineer:

> **How do you find out why a query or database is slow, and how do you fix it?**

The key principle is:

**Don't guess. Measure → diagnose → optimize → verify.**

---

# 1. What does "performance" mean?

Database performance isn't just CPU.

You need to think about:

```text id="3k7x1a"
Query latency
Throughput
CPU
Memory
I/O
Locks
Connections
Concurrency
Replication lag
Cache efficiency
Storage activity
```

For example:

```text
Query takes 5 seconds
        ↓
Why?
        ↓
CPU?
I/O?
Lock?
Bad plan?
Missing index?
Too many connections?
```

Performance engineering starts with finding the actual bottleneck.

---

# 2. The performance investigation flow

Use this mental model:

```text id="g7s8h2"
User reports slowness
        ↓
Measure
        ↓
Identify problematic query
        ↓
EXPLAIN
        ↓
Understand execution plan
        ↓
Find bottleneck
        ↓
Optimize
        ↓
Test again
        ↓
Measure improvement
```

Never jump directly to:

> "Add an index."

An index may help—or may make things worse.

---

# 3. EXPLAIN

PostgreSQL provides `EXPLAIN` to show how it intends to execute a query.

Example:

```sql id="j8q4w3"
EXPLAIN
SELECT *
FROM customers
WHERE customer_id = 100;
```

Conceptually:

```text id="p7n2s4"
SQL
 ↓
EXPLAIN
 ↓
Query Plan
```

The plan tells you what PostgreSQL expects to do.

---

# 4. EXPLAIN ANALYZE

`EXPLAIN ANALYZE` actually executes the query and reports runtime information.

Example:

```sql id="5x9m2v"
EXPLAIN ANALYZE
SELECT *
FROM customers
WHERE customer_id = 100;
```

This can show things such as:

```text id="h8r4q6"
Actual execution time
Rows returned
Rows processed
Actual loops
Plan operations
```

This is much more useful when diagnosing real performance.

---

# 5. EXPLAIN vs EXPLAIN ANALYZE

Remember:

```text id="x2q8j5"
EXPLAIN
   ↓
Shows planned execution


EXPLAIN ANALYZE
   ↓
Executes + measures actual execution
```

Be careful with `EXPLAIN ANALYZE` on:

```text id="z5p1r9"
UPDATE
DELETE
INSERT
```

because the statement actually executes unless you use an appropriate transaction/rollback strategy.

---

# 6. Sequential scan

Suppose PostgreSQL examines every row:

```text id="k3w7m1"
Table: customers

Row 1  → check
Row 2  → check
Row 3  → check
...
Row 1,000,000 → check
```

This is a **sequential scan**.

It isn't automatically bad.

For a small table:

```text id="s6d2p8"
10 rows
 ↓
Sequential scan
 ↓
Very cheap
```

For a huge table:

```text id="r4v9c2"
100 million rows
 ↓
Sequential scan
 ↓
Potentially expensive
```

Context matters.

---

# 7. Index scan

Suppose we have:

```sql id="e7h3c1"
CREATE INDEX idx_customers_email
ON customers(email);
```

Then:

```sql id="w4p9x7"
SELECT *
FROM customers
WHERE email = 'alex@example.com';
```

PostgreSQL may use the index to find matching rows efficiently.

Conceptually:

```text id="m3k8q2"
Query
 ↓
Index
 ↓
Matching row locations
 ↓
Table data
```

---

# 8. Indexes are not free

A common beginner mistake:

> "If indexes make queries faster, create indexes on everything."

No.

Indexes have costs.

They consume:

```text id="f5t7n1"
Storage
Memory/cache
Write overhead
Maintenance resources
```

When a row changes, relevant indexes may also need updating.

So:

```text id="v6x2j9"
More indexes
   ↓
Potentially faster reads
   +
Potentially more expensive writes
```

---

# 9. Selectivity

Indexes are especially useful when a condition narrows the result set significantly.

Suppose:

```text id="w8p2r5"
1,000,000 rows
```

Query:

```sql id="9k4f2m"
WHERE customer_id = 12345
```

Maybe only one row matches.

That's highly selective.

But:

```sql id="q2m7x4"
WHERE country = 'USA'
```

might return hundreds of thousands of rows.

In that case, PostgreSQL may decide that a sequential scan is cheaper.

---

# 10. PostgreSQL optimizer

PostgreSQL has a cost-based optimizer.

Conceptually:

```text id="c7n3v5"
SQL
 ↓
Optimizer
 ↓
Statistics
 ↓
Candidate plans
 ↓
Estimated costs
 ↓
Chosen plan
```

The optimizer tries to choose an efficient execution strategy.

---

# 11. Statistics

PostgreSQL uses statistics about table data.

For example:

```text id="g5y1r8"
Number of rows
Value distribution
Most common values
Data correlation
```

These statistics help estimate:

```text id="d3k6q2"
How many rows will match?
Which plan is cheaper?
Should I use an index?
Which join strategy should I use?
```

This is why `ANALYZE` matters.

---

# 12. Stale statistics

Suppose a table changes dramatically:

```text id="p8m4x1"
Before:
100,000 rows

After:
50,000,000 rows
```

If statistics aren't updated appropriately, the optimizer may make poor assumptions.

That can lead to a bad execution plan.

---

# 13. Query plans

A plan might conceptually look like:

```text id="k2x7m4"
Nested Loop
   │
   ├── Index Scan
   │
   └── Index Scan
```

Or:

```text id="f8v1p3"
Hash Join
   │
   ├── Seq Scan
   └── Seq Scan
```

Understanding these operations is fundamental to PostgreSQL performance tuning.

---

# 14. Join strategies

PostgreSQL commonly uses strategies such as:

### Nested Loop

```text id="a8j2n6"
Outer rows
   ↓
Repeated lookup
   ↓
Inner table
```

Can be excellent when the outer side is small and the inner side is efficiently indexed.

---

### Hash Join

```text id="w4q8p1"
Build hash table
       ↓
Probe hash table
       ↓
Matching rows
```

Often useful for larger joins where hashing is efficient.

---

### Merge Join

```text id="j7v3x5"
Sorted input A
      +
Sorted input B
      ↓
Merge
```

Can be useful when inputs are already sorted or sorting is efficient.

---

# 15. Query example

Imagine:

```sql id="q1v9z6"
SELECT o.order_id, c.name
FROM orders o
JOIN customers c
  ON o.customer_id = c.customer_id
WHERE c.country = 'USA';
```

Don't immediately optimize it.

First:

```sql id="p4k8m2"
EXPLAIN ANALYZE
SELECT ...
```

Then ask:

```text id="x6n2r9"
How many rows?
Which scan?
Which join?
Estimated rows vs actual rows?
Execution time?
Loops?
```

---

# 16. Estimated vs actual rows

One of the most useful clues is:

```text id="g3r7k5"
Estimated rows: 10
Actual rows:    500,000
```

That's a huge mismatch.

It suggests the optimizer's assumptions about the data may be wrong.

Possible causes include:

```text id="w1x6c4"
Stale statistics
Data distribution changes
Correlated columns
Complex predicates
```

This is why reading `EXPLAIN ANALYZE` is a critical DBA skill.

---

# 17. Locks can look like slow queries

Suppose your query itself is fast:

```text id="v5n8r2"
Query execution = 20 ms
```

But the application waits:

```text id="p9m3x7"
Total latency = 10 seconds
```

Maybe the query is waiting on a lock.

Conceptually:

```text id="y4k8c1"
Transaction A
      ↓
holds lock
      ↓
Transaction B
      ↓
WAITING
```

So performance analysis must include concurrency and locking—not just SQL execution plans.

---

# 18. Connection pressure

Suppose your application creates:

```text id="k5r2p9"
5,000 database connections
```

Even if individual queries are fast, the database may experience resource pressure.

This is why connection pooling matters.

A healthy architecture might look like:

```text id="d8m4v6"
Thousands of requests
        ↓
Connection pool
        ↓
Controlled DB connections
        ↓
Aurora
```

---

# 19. CPU bottleneck

Suppose CPU is consistently high:

```text id="m3x7q1"
CPU
████████████████████ 95%
```

Potential causes include:

* Expensive queries
* Large joins
* Sorts
* Aggregations
* Excessive concurrency
* Inefficient application behavior

Don't immediately increase the instance size.

First determine **why CPU is high**.

---

# 20. I/O bottleneck

A workload may be constrained by database I/O.

Potential causes:

```text id="r6p2w9"
Large scans
Poor indexing
High read workload
Large sorts
Working set doesn't fit efficiently in memory
```

Aurora's storage architecture changes the underlying I/O model compared with traditional PostgreSQL, but **I/O remains an important performance dimension**.

---

# 21. Memory

Memory is important for:

```text id="b4n7k2"
Database caching
Sorts
Hash operations
Connections
Working set
```

Too little effective memory can result in more work against storage.

But memory parameters such as `work_mem` must be configured carefully.

Remember:

> `work_mem` is not simply "memory allocated to PostgreSQL."

It can apply to individual operations and potentially many concurrent sessions.

---

# 22. Performance Insights

AWS provides **Amazon RDS Performance Insights** for database performance analysis, including Aurora.

The goal is to answer:

> **What is the database spending its time waiting on?**

Conceptually:

```text id="z7m4p1"
Application
     ↓
Database workload
     ↓
Performance Insights
     ↓
Top SQL / waits
     ↓
Root cause
```

This is extremely useful in production troubleshooting.

---

# 23. Wait events

A database may spend time waiting for different resources.

Conceptually:

```text id="x9c3v7"
CPU
I/O
Locks
Client activity
Storage
Other database resources
```

Instead of simply asking:

> "Why is CPU high?"

ask:

> **"What is the database waiting on?"**

This produces much better diagnosis.

---

# 24. Query optimization process

Use this repeatable process:

```text id="s4n8k2"
1. Identify slow query
        ↓
2. Measure latency
        ↓
3. EXPLAIN ANALYZE
        ↓
4. Examine plan
        ↓
5. Compare estimated vs actual rows
        ↓
6. Check indexes
        ↓
7. Check statistics
        ↓
8. Check locks / waits
        ↓
9. Optimize
        ↓
10. Measure again
```

---

# 25. Example: missing index

Suppose:

```sql id="u3m7q8"
SELECT *
FROM orders
WHERE customer_id = 5001;
```

There are:

```text id="x5k2r9"
50 million orders
```

If PostgreSQL repeatedly performs large sequential scans, an appropriate index may help:

```sql id="b7n4v2"
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

Then compare:

```text id="h8p3m6"
Before
↓
Sequential scan
↓
8 seconds

After
↓
Index-based plan
↓
120 ms
```

But the actual result must be measured—not assumed.

---

# 26. Composite indexes

Suppose queries frequently use:

```sql id="m4k9x2"
WHERE customer_id = ?
AND order_date >= ?
```

A composite index may be appropriate:

```sql id="j2p7w5"
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, order_date);
```

Index design must follow actual query patterns.

---

# 27. Query optimization isn't always about indexes

A slow query can result from:

```text id="v8n2q6"
Bad join
Poor filtering
Unnecessary columns
Large result sets
Functions preventing efficient access
Stale statistics
Lock contention
Excessive sorting
Poor application query patterns
```

Therefore:

> **Index tuning is only one part of query optimization.**

---

# 28. SELECT *

Avoid blindly doing:

```sql id="r5x8m1"
SELECT *
FROM customers;
```

If you only need:

```text id="q2n7v4"
customer_id
name
```

prefer:

```sql id="f8m3k6"
SELECT customer_id, name
FROM customers;
```

This can reduce:

* Data transfer
* Processing
* Unnecessary column access

Especially important for large queries.

---

# 29. Pagination

Bad pagination can become expensive.

For example:

```sql id="d6q4p8"
SELECT *
FROM orders
ORDER BY order_id
OFFSET 1000000
LIMIT 50;
```

The database may need to process a huge number of rows before returning the desired page.

For large datasets, **keyset/seek pagination** can often be more efficient.

Conceptually:

```text id="n8m2x5"
Last seen ID = 1000000
       ↓
WHERE order_id > 1000000
       ↓
LIMIT 50
```

---

# 30. Production performance mindset

A good Aurora PostgreSQL engineer doesn't ask:

> "What setting should I change?"

They ask:

```text id="k7x4m2"
What is slow?
        ↓
Where is the time going?
        ↓
What evidence proves the bottleneck?
        ↓
What change addresses it?
        ↓
Did the change actually improve performance?
```

That mindset is more important than memorizing tuning parameters.

---

# 🧠 Interview Questions

### Q1. What does `EXPLAIN` do?

It shows the execution plan PostgreSQL expects to use for a query.

### Q2. What does `EXPLAIN ANALYZE` do?

It executes the query and provides actual execution information alongside the plan.

### Q3. Is a sequential scan always bad?

No. For small tables or queries returning a large percentage of rows, a sequential scan can be the most efficient strategy.

### Q4. Why aren't indexes free?

Indexes consume storage and require maintenance during writes and updates.

### Q5. What can cause a query to be slow besides a bad execution plan?

Locks, I/O, CPU pressure, memory pressure, connection contention, stale statistics, and application behavior can all contribute.

---

# 🎯 Lesson 6 Summary

Your performance mental model:

```text id="r2m8v6"
                 Query
                   ↓
             EXPLAIN ANALYZE
                   ↓
             Execution Plan
                   ↓
       ┌───────────┼───────────┐
       ↓           ↓           ↓
     Scan         Join       Sort
       │           │           │
       └───────────┼───────────┘
                   ↓
              Performance
                   ↓
        ┌──────────┼──────────┐
        ↓          ↓          ↓
       CPU        I/O       Locks
                   │
              Statistics
                   │
                Indexes
```

### Five things to remember

**1.** Always **measure before optimizing**.

**2.** Learn to read `EXPLAIN ANALYZE`.

**3.** Sequential scans aren't automatically bad.

**4.** Indexes improve some reads but have write/storage costs.

**5.** Performance troubleshooting must consider **CPU, I/O, locks, connections, statistics, and query plans**.

---

**Next → Lesson 7: Aurora PostgreSQL High Availability & Failover**
