# 🐘 Lesson 5 — PostgreSQL Administration on Aurora

Now we move into the **database-engineering layer**.

Aurora PostgreSQL is managed by AWS, but you are still responsible for understanding and administering the **PostgreSQL database itself**.

---

## 1. What does "administration" mean on Aurora?

On a self-managed PostgreSQL server, a DBA may manage:

```text
Operating system
PostgreSQL installation
Filesystem
Storage
PostgreSQL configuration
Database
Users
Backups
Monitoring
```

On Aurora, AWS manages much of the underlying infrastructure.

Your focus shifts toward:

```text
Databases
Schemas
Roles
Permissions
SQL
Indexes
Extensions
Transactions
Sessions
VACUUM
ANALYZE
Configuration
Performance
Security
```

Think:

```text
AWS manages the platform
          ↓
You manage the database workload
```

---

# 2. Connecting to Aurora PostgreSQL

A PostgreSQL client can connect using an Aurora endpoint.

For example:

```bash id="y6n1kz"
psql -h <aurora-endpoint> -U admin -d postgres
```

Conceptually:

```text id="8yy4zo"
psql
 ↓
DNS
 ↓
Aurora Endpoint
 ↓
Aurora PostgreSQL
```

The exact endpoint, username, database, and authentication method depend on your environment.

---

# 3. PostgreSQL databases

An Aurora PostgreSQL cluster can contain multiple databases.

Conceptually:

```text id="3gajlq"
Aurora Cluster
│
├── postgres
├── application_db
├── reporting_db
└── analytics_db
```

You can connect to a specific database:

```bash id="tq74dl"
psql -h <endpoint> -U app_user -d application_db
```

---

# 4. Schemas

A PostgreSQL database can contain multiple schemas.

Example:

```text id="y7d1o3"
application_db
│
├── public
├── sales
├── inventory
└── reporting
```

Schemas provide logical organization and can also help establish security boundaries.

For example:

```sql id="8p9fkr"
CREATE SCHEMA sales;
```

Then:

```sql id="8rj5pc"
CREATE TABLE sales.orders (
    order_id bigint,
    customer_id bigint
);
```

---

# 5. Database vs schema

Don't confuse them.

```text id="i0pkq7"
Cluster
   │
   └── Database
          │
          ├── Schema
          │     ├── Table
          │     └── View
          │
          └── Schema
                └── Table
```

A **database** is a larger logical container.

A **schema** organizes objects inside a database.

---

# 6. PostgreSQL roles

PostgreSQL uses **roles** for identities and permissions.

A role can represent:

* A human administrator
* An application
* A reporting user
* A service identity

Example:

```sql id="b2wqv1"
CREATE ROLE app_user
LOGIN;
```

A role can then receive permissions.

---

# 7. Roles and users

In modern PostgreSQL terminology:

> A user is essentially a role that has the LOGIN attribute.

For example:

```sql id="2xj8sg"
CREATE ROLE reporting_user
LOGIN;
```

A role doesn't necessarily need to log in.

You can create a group-like role:

```sql id="o0n6uh"
CREATE ROLE reporting_role;
```

Then grant permissions to it.

---

# 8. Role hierarchy

A strong security pattern is:

```text id="1s6w0w"
Permissions
     ↓
Database Role
     ↓
Application User
```

For example:

```sql id="d7x9vz"
CREATE ROLE app_readwrite;
CREATE ROLE app_user LOGIN;

GRANT app_readwrite TO app_user;
```

Now permissions can be managed through the role rather than individually for every user.

---

# 9. PostgreSQL privileges

Common privileges include:

```text id="pf0p3j"
SELECT
INSERT
UPDATE
DELETE
EXECUTE
USAGE
CREATE
```

For example:

```sql id="x5z2vk"
GRANT SELECT ON sales.orders TO reporting_role;
```

Now members of that role can read the table.

---

# 10. Least privilege

Suppose an application needs only:

```text id="g78wqs"
SELECT
INSERT
UPDATE
```

Don't give it:

```text id="3g5vpe"
SUPERUSER
DROP DATABASE
CREATE ROLE
```

Instead:

```text id="s5g2df"
Application
     ↓
app_user
     ↓
app_readwrite
     ↓
Required objects
```

This limits damage if application credentials are compromised.

---

# 11. Public schema

PostgreSQL traditionally has a `public` schema.

Don't automatically assume every application should have unrestricted access to it.

In a production system, explicitly define:

```text id="e1yr19"
Who can USAGE the schema?
Who can CREATE objects?
Who can SELECT?
Who can modify objects?
```

Security should be intentional.

---

# 12. Extensions

PostgreSQL supports extensions that add functionality.

For example:

```sql id="kz7qrm"
CREATE EXTENSION extension_name;
```

Aurora PostgreSQL supports a range of PostgreSQL extensions, but **not every PostgreSQL extension is necessarily available or appropriate on Aurora**.

Therefore:

> Always verify extension compatibility with the specific Aurora PostgreSQL engine version.

This becomes especially important during migrations.

---

# 13. Why extensions matter during migration

Suppose your existing PostgreSQL environment uses:

```text id="o0gnq4"
Extension A
Extension B
Extension C
```

Before migrating to Aurora:

```text id="xw2xj1"
Inventory extensions
       ↓
Check Aurora support
       ↓
Test application
       ↓
Plan alternatives
```

Never assume every extension from self-managed PostgreSQL will work identically.

---

# 14. PostgreSQL configuration

PostgreSQL has many configuration parameters.

Examples include:

```text id="5b8n4y"
max_connections
work_mem
log settings
planner settings
statement timeout
```

On Aurora, configuration is managed through AWS-supported parameter groups.

Remember from Lesson 3:

```text id="x0bbj8"
Cluster Parameter Group
        +
DB Parameter Group
```

---

# 15. Static vs dynamic parameters

Some parameters can change while the database is running.

Others require a restart.

Conceptually:

```text id="n1m3tj"
Dynamic
   ↓
Change
   ↓
Take effect without restart


Static
   ↓
Change
   ↓
Restart may be required
```

Before modifying a parameter, determine:

* Its scope
* Whether it is dynamic
* Whether a reboot is required
* Whether it affects all instances

---

# 16. Don't copy PostgreSQL tuning blindly

Suppose you find a blog recommending:

```text id="vpln7o"
work_mem = 256MB
```

Don't immediately apply it.

Why?

Because the correct value depends on:

```text id="zj3l5s"
Workload
Concurrency
Query complexity
Memory
Connections
Instance size
```

A setting that works for one environment can cause memory pressure in another.

---

# 17. Connections

Every application connection consumes resources.

Conceptually:

```text id="6z4t7b"
Application
 ├── Connection 1
 ├── Connection 2
 ├── Connection 3
 └── Connection 4
          ↓
       Aurora
```

Too many connections can create resource pressure.

This is why connection pooling is important for many applications.

---

# 18. Connection pooling

Instead of creating a new database connection for every request:

```text id="2z1f55"
Request
 ↓
New DB connection
 ↓
Query
 ↓
Disconnect
```

a pool can reuse connections:

```text id="m6iz4n"
Application
      ↓
 Connection Pool
  ┌───┼───┐
  ↓   ↓   ↓
 DB  DB  DB
```

This can reduce connection overhead and improve application scalability.

---

# 19. Transactions

PostgreSQL transactions provide atomicity and consistency.

Example:

```sql id="1xkq3d"
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE account_id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE account_id = 2;

COMMIT;
```

If something goes wrong:

```sql id="q2b6bw"
ROLLBACK;
```

Transactions are fundamental to correct database applications.

---

# 20. Long-running transactions

Long transactions can cause operational problems.

For example:

```text id="j8c7v1"
Transaction starts
       ↓
10 minutes
       ↓
Still open
```

Potential consequences include:

* Old row versions remaining longer
* Vacuum delays
* Lock contention
* Increased storage/resource pressure

Therefore, monitor long-running transactions.

---

# 21. VACUUM

PostgreSQL uses MVCC.

When rows are updated or deleted, old row versions can remain until PostgreSQL can clean them up.

`VACUUM` helps reclaim/manage this space and maintain database health.

Conceptually:

```text id="zj9y8s"
UPDATE / DELETE
      ↓
Old row versions
      ↓
VACUUM
      ↓
Cleanup
```

Aurora PostgreSQL still uses PostgreSQL's vacuum mechanisms.

---

# 22. Autovacuum

PostgreSQL provides **autovacuum** to automate vacuuming and analyze operations.

Conceptually:

```text id="3d2g5x"
Database workload
       ↓
Dead tuples
       ↓
Autovacuum
       ↓
Cleanup + statistics maintenance
```

Don't assume:

> "Aurora is managed, so vacuum doesn't matter."

It absolutely matters for PostgreSQL workload health.

---

# 23. ANALYZE

`ANALYZE` collects statistics about data distribution.

The optimizer uses those statistics when generating execution plans.

Conceptually:

```text id="pxj9fa"
Table data
    ↓
ANALYZE
    ↓
Statistics
    ↓
Query optimizer
    ↓
Execution plan
```

Poor statistics can contribute to poor query plans.

---

# 24. VACUUM vs ANALYZE

Easy way to remember:

### VACUUM

Helps manage dead row versions and storage cleanup.

### ANALYZE

Updates optimizer statistics.

```text id="49d2l7"
VACUUM
   ↓
Housekeeping


ANALYZE
   ↓
Optimizer knowledge
```

Autovacuum can perform both types of maintenance.

---

# 25. Monitoring sessions

When troubleshooting Aurora PostgreSQL, you may need to examine active sessions.

For example:

```sql id="9k8e4f"
SELECT
    pid,
    usename,
    state,
    query
FROM pg_stat_activity;
```

This can help identify:

```text id="d8s8r6"
Active sessions
Idle sessions
Long-running queries
Blocked sessions
Applications
```

---

# 26. Finding long-running queries

A DBA may inspect query duration:

```sql id="o3r6k1"
SELECT
    pid,
    now() - query_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE query_start IS NOT NULL
ORDER BY duration DESC;
```

This is useful during performance investigations.

---

# 27. Locks and blocking

Suppose:

```text id="zv6xqa"
Transaction A
     ↓
Locks row

Transaction B
     ↓
Needs same row
```

B may become blocked.

Conceptually:

```text id="f4b7xk"
A ────────→ Lock
             ↑
             │
B ───────────┘
```

Blocking is not automatically a database failure.

But excessive blocking can severely affect application performance.

---

# 28. Database maintenance mindset

A good Aurora PostgreSQL administrator continuously considers:

```text id="i9p4l0"
Connections
Transactions
Locks
Vacuum
Statistics
Indexes
Permissions
Logs
Extensions
Parameters
Performance
```

The fact that AWS manages infrastructure doesn't eliminate these responsibilities.

---

# 29. Example production administration model

```text id="c8j6kl"
                 Aurora
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
     Security    Database    Performance
        │           │           │
      Roles       Schemas      Queries
      Grants      Tables       Indexes
      IAM         Functions    Statistics
      TLS         Extensions   Plans
                    │
                 Maintenance
                    │
              VACUUM / ANALYZE
```

---

# 30. Common administration mistakes

### ❌ Giving applications excessive privileges

Use dedicated roles.

### ❌ Ignoring autovacuum

It is fundamental to PostgreSQL health.

### ❌ Allowing unlimited connections

Use sensible limits and connection pooling.

### ❌ Changing parameters without understanding them

Test configuration changes.

### ❌ Installing extensions without checking support

Verify Aurora compatibility.

### ❌ Ignoring long-running transactions

They can cause cascading operational problems.

### ❌ Treating Aurora like a normal EC2 PostgreSQL server

Aurora has its own managed-service architecture and operational boundaries.

---

# 🧠 Interview Questions

### Q1. What is the difference between a PostgreSQL database and schema?

A database is a higher-level logical container; schemas organize objects within a database.

### Q2. What is a PostgreSQL role?

A security identity that can own objects, receive privileges, and optionally authenticate with the LOGIN attribute.

### Q3. Why is autovacuum important?

It helps clean up obsolete row versions and maintain statistics, supporting PostgreSQL's MVCC model and query optimization.

### Q4. What does ANALYZE do?

It collects statistics about table data distribution so the PostgreSQL optimizer can make better execution-plan decisions.

### Q5. Why should applications use dedicated roles?

To enforce least privilege and reduce the impact of compromised application credentials.

---

# 🎯 Lesson 5 Summary

The Aurora DBA mental model is:

```text id="x7c3k8"
             AURORA
                │
        ┌───────┴───────┐
        ↓               ↓
    AWS Platform     PostgreSQL
        │               │
    Infrastructure    Databases
    Storage           Schemas
    HA                Roles
    Backups           Grants
                      SQL
                      Extensions
                      VACUUM
                      ANALYZE
                      Transactions
```

### Remember these five:

**1.** Aurora is managed infrastructure, **not managed DBA responsibility**.

**2.** PostgreSQL roles and privileges are central to security.

**3.** Use **least privilege** for applications.

**4.** **VACUUM and ANALYZE** remain essential PostgreSQL maintenance concepts.

**5.** Monitor **connections, transactions, locks, and long-running queries**.

---

**Next → Lesson 6: Aurora PostgreSQL Performance & Query Optimization**
