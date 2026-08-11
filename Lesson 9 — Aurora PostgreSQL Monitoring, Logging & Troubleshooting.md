# 🐘 Lesson 9 — Aurora PostgreSQL Monitoring, Logging & Troubleshooting

This lesson is about answering the question:

> **"What is happening inside my Aurora PostgreSQL environment right now, and how do I prove the root cause?"**

A production DBA should be able to move from **symptom → evidence → root cause → fix**.

---

# 1. Monitoring vs troubleshooting

These are related but different.

### Monitoring

Continuously watching the system:

```text
CPU
Memory
Connections
Latency
Database load
Storage
Replication
Errors
```

### Troubleshooting

Investigating a specific problem:

```text
Application is slow
      ↓
Collect evidence
      ↓
Find bottleneck
      ↓
Fix
```

Think:

```text
Monitoring = "What is happening?"
Troubleshooting = "Why is it happening?"
```

---

# 2. The Aurora monitoring stack

A useful mental model is:

```text
                    Aurora
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
   CloudWatch     Performance       PostgreSQL
                    Insights          Views
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                  DBA Diagnosis
```

Each layer answers different questions.

---

# 3. Amazon CloudWatch

Amazon CloudWatch provides infrastructure/service-level metrics.

For Aurora, you can monitor metrics such as:

```text
CPU utilization
Database connections
Freeable memory
Network activity
Storage-related metrics
Replication-related metrics
```

The exact metrics available depend on the Aurora configuration and engine.

---

# 4. CPU monitoring

Suppose you see:

```text
CPU
████████████████████ 95%
```

Don't immediately conclude:

> "The database needs a bigger instance."

First ask:

```text
What SQL is consuming CPU?
Are queries inefficient?
Did workload suddenly increase?
Are joins/aggregations expensive?
Did an application deployment change query behavior?
```

Metrics tell you **there is a problem**.

They don't necessarily tell you **why**.

---

# 5. Database connections

Monitor connection counts.

Conceptually:

```text
Connections
    │
    │       ███
    │     █████
    │   ███████
    │██████████
    └────────────────→ Time
```

A sudden increase might indicate:

```text
Connection leak
Application deployment
Traffic spike
Poor connection pooling
Retry storm
```

Connection metrics should therefore be correlated with application events.

---

# 6. Freeable memory

Memory pressure can affect database behavior.

Watch for sustained changes rather than reacting to one isolated datapoint.

Potential causes:

```text
Higher concurrency
Larger workload
Large sorts
Hash operations
More connections
Changed query patterns
```

Again:

> **Metric → hypothesis → investigation.**

---

# 7. Performance Insights

Performance Insights is particularly useful for answering:

> **Which database activity is consuming the most database time?**

Conceptually:

```text
Application
    ↓
SQL workload
    ↓
Database load
    ↓
Wait events / SQL
    ↓
Top contributors
```

This is much more actionable than simply knowing:

```text
CPU = 85%
```

---

# 8. Database load

Think of database load as:

```text
How much work is the database handling?
```

High load can come from:

```text
CPU-bound queries
I/O waits
Locks
Concurrency
Other database waits
```

So:

```text
High load
   ≠
High CPU only
```

This distinction is important.

---

# 9. Wait events

Suppose users complain:

> "Everything is slow."

You investigate Performance Insights and discover many sessions waiting on locks.

That changes the diagnosis:

```text
❌ "Need more CPU"

Correct direction:
"Investigate blocking transactions."
```

Wait-event analysis helps distinguish different bottlenecks.

---

# 10. PostgreSQL `pg_stat_activity`

PostgreSQL provides runtime information through system views.

A commonly used view is:

```sql id="1gq8k2"
SELECT
    pid,
    usename,
    state,
    query_start,
    query
FROM pg_stat_activity;
```

This helps you understand active sessions.

You can investigate:

```text
Who is connected?
What are they doing?
How long has the query been running?
Which application/user is involved?
```

---

# 11. Finding long-running queries

For example:

```sql id="j4m7x8"
SELECT
    pid,
    now() - query_start AS duration,
    usename,
    state,
    query
FROM pg_stat_activity
WHERE query_start IS NOT NULL
ORDER BY duration DESC;
```

This is a useful starting point.

But remember:

> A long-running query isn't automatically a bad query.

A large analytical query may legitimately run for several minutes.

Context matters.

---

# 12. Finding idle transactions

An especially important state is:

```text
idle in transaction
```

Conceptually:

```text
BEGIN
 ↓
SQL
 ↓
Application stops doing work
 ↓
Transaction remains open
```

This can cause:

```text
Old row versions retained
Locks held longer
Vacuum complications
Blocking
Resource consumption
```

Idle transactions deserve attention in production systems.

---

# 13. Blocking sessions

Suppose:

```text
Session A
   ↓
Holding lock
   ↓
Session B
   ↓
Waiting
```

The application might report:

> "Query is taking 30 seconds."

But the query itself may only need 10 milliseconds.

The remaining time is lock waiting.

Therefore:

```text
Query latency
=
Execution time
+
Waiting time
```

This is a crucial troubleshooting concept.

---

# 14. PostgreSQL locks

When troubleshooting locking problems, investigate PostgreSQL's lock-related system information.

Your goal is to determine:

```text
Who is waiting?
Who is blocking?
What object is involved?
How long has the wait existed?
```

Then decide whether the blocking transaction is legitimate or problematic.

---

# 15. Logs

Database logs are another major source of evidence.

Logs can help identify:

```text
Errors
Connection problems
Authentication failures
Slow operations
Checkpoint activity
Database events
Application/database interaction problems
```

Logging must be balanced carefully.

Too little logging:

```text
Not enough evidence
```

Too much logging:

```text
Large log volume
Noise
Additional overhead
Harder investigation
```

---

# 16. Slow query logging

For persistent performance problems, slow-query information can be extremely useful.

The goal is:

```text
Find queries taking unusually long
        ↓
Identify SQL
        ↓
EXPLAIN ANALYZE
        ↓
Optimize
```

Don't simply collect logs forever.

Define what you want to learn from them.

---

# 17. Error investigation workflow

Suppose an application starts returning:

```text
Database connection errors
```

Don't immediately restart Aurora.

Use:

```text
1. Check application logs
2. Check Aurora/CloudWatch metrics
3. Check connection count
4. Check database sessions
5. Check recent deployments
6. Check network/security changes
7. Check database events
8. Determine scope
```

Then act.

---

# 18. Correlation is powerful

Suppose:

```text
14:00
Application deployment

14:03
Connections increase

14:05
CPU increases

14:07
Latency increases
```

That sequence gives you a strong lead.

Monitoring becomes much more powerful when you correlate:

```text
Database metrics
+
SQL activity
+
Application events
+
Infrastructure changes
```

---

# 19. Baselines

You need to know what "normal" looks like.

Example:

```text
Normal CPU:         30–50%
Normal connections: 100–200
Normal latency:     50–100 ms
```

Then:

```text
Today:
CPU:                90%
Connections:        800
Latency:            2 seconds
```

The deviation becomes obvious.

Without a baseline, every metric can look meaningless.

---

# 20. Alerts

Good alerts should be:

```text
Actionable
Specific
Meaningful
```

Bad alert:

```text
CPU > 70%
```

for 30 seconds.

It may produce endless noise.

Better:

```text
Sustained high CPU
+
Elevated database load
+
Increased query latency
```

The exact thresholds should be based on your workload.

---

# 21. Alert fatigue

If engineers receive:

```text
100 alerts/day
```

and 95 are harmless, they will eventually ignore alerts.

Monitoring design should therefore prioritize:

```text
Signal > noise
```

Critical alerts should correspond to situations requiring human action.

---

# 22. Troubleshooting methodology

Use this framework:

```text
             SYMPTOM
                ↓
          Define the scope
                ↓
          Collect metrics
                ↓
        Identify database load
                ↓
         Identify wait events
                ↓
          Find top SQL
                ↓
        Inspect query plans
                ↓
        Check locks/sessions
                ↓
          Form hypothesis
                ↓
              FIX
                ↓
          Verify improvement
```

This prevents random changes.

---

# 23. Example incident

Imagine users report:

> "The application became extremely slow."

You discover:

```text
CPU = 45%
```

So CPU isn't obviously the problem.

Performance Insights shows:

```text
High database load
   ↓
Lock waits
```

Then `pg_stat_activity` reveals:

```text
Transaction A
   ↓
Open for 25 minutes
   ↓
Blocking many sessions
```

Root cause:

> **Long-running transaction causing lock contention.**

Not:

> "Aurora needs a larger instance."

---

# 24. Another example

Suppose:

```text
CPU = 90%
```

Performance Insights:

```text
Top SQL
   ↓
Large aggregation query
```

`EXPLAIN ANALYZE`:

```text
Large sequential scan
   ↓
Millions of rows
   ↓
Expensive aggregation
```

Now you can investigate:

```text
Query design
Indexes
Filtering
Data model
Aggregation strategy
Statistics
```

This is evidence-driven optimization.

---

# 25. Operational dashboard

A practical Aurora dashboard might include:

```text
┌───────────────────────────────────┐
│ CPU                               │
├───────────────────────────────────┤
│ Database Connections              │
├───────────────────────────────────┤
│ Freeable Memory                   │
├───────────────────────────────────┤
│ Database Load                     │
├───────────────────────────────────┤
│ Query Latency                     │
├───────────────────────────────────┤
│ Replication / Reader Metrics      │
├───────────────────────────────────┤
│ Errors                            │
└───────────────────────────────────┘
```

Then drill into Performance Insights and PostgreSQL views when something changes.

---

# 26. Monitoring architecture

A mature architecture might look like:

```text
                Aurora PostgreSQL
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
   CloudWatch     Performance     DB Views
                    Insights
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                   Dashboards
                       ↓
                    Alerts
                       ↓
                  DBA / SRE
                       ↓
                Incident response
```

---

# 27. Common troubleshooting mistakes

### ❌ Restarting before investigating

You may temporarily hide the symptom without fixing the root cause.

### ❌ Looking only at CPU

A database can be slow because of locks, I/O, or other waits.

### ❌ Looking only at application logs

The database may contain the evidence you need.

### ❌ Looking only at database metrics

Metrics often tell you **what**, while SQL/session analysis helps explain **why**.

### ❌ Killing sessions blindly

Terminate sessions only when you understand the consequences.

### ❌ Changing multiple things simultaneously

You won't know which change fixed—or caused—the problem.

---

# 28. The golden troubleshooting rule

When something is wrong:

> **Change as little as possible until you understand the problem.**

Use evidence.

```text
Observe
  ↓
Hypothesize
  ↓
Test
  ↓
Change
  ↓
Verify
```

Not:

```text
Problem
 ↓
Change 10 settings
 ↓
Hope
```

---

# 🧠 Interview Questions

### Q1. What is the difference between CloudWatch and Performance Insights?

CloudWatch provides broad AWS resource/service metrics, while Performance Insights focuses on database workload and helps identify database load, SQL activity, and wait behavior.

### Q2. What is `pg_stat_activity` useful for?

Inspecting PostgreSQL sessions, query activity, states, users, and query timing.

### Q3. Why can a query appear slow even when its execution plan is efficient?

It may be waiting on locks, resources, or other contention before or during execution.

### Q4. Why are baselines important?

They establish what normal behavior looks like so abnormal patterns can be detected accurately.

### Q5. What is the correct troubleshooting approach?

**Measure → isolate → investigate → form a hypothesis → make a targeted change → verify.**

---

# 🎯 Lesson 9 Summary

Your monitoring mental model:

```text
                    Aurora
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
    CloudWatch    Performance    PostgreSQL
                   Insights        Views
        │             │             │
        └─────────────┼─────────────┘
                      ↓
                  Evidence
                      ↓
                 Root Cause
                      ↓
                     Fix
                      ↓
                  Verify
```

### Five things to remember

**1.** Metrics tell you **what is happening**.

**2.** Performance Insights helps identify **database workload and waits**.

**3.** `pg_stat_activity` is extremely useful for investigating sessions and queries.

**4.** Lock contention and long-running transactions can create major performance problems.

**5.** **Never troubleshoot by guessing—collect evidence first.**

---

**Next → Lesson 10: Aurora PostgreSQL Production Architecture — Security, Scaling, HA, DR & Final Blueprint**
