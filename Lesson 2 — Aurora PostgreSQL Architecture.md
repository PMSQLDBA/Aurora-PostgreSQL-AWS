# 🐘 Lesson 2 — Aurora PostgreSQL Architecture

Lesson 1 gave us the basic picture. Now we go one level deeper and understand **how Aurora PostgreSQL is actually organized**.

---

## 1. Aurora's architecture at a glance

The most important mental model is:

```text
                    AWS REGION
                        │
                 Aurora DB Cluster
                        │
          ┌─────────────┴─────────────┐
          │                           │
       COMPUTE                     STORAGE
          │                           │
   ┌──────┼──────┐                    │
   ↓      ↓      ↓                    ↓
 Writer Reader Reader       Distributed Aurora
                            Storage Volume
                                  │
                         ┌────────┼────────┐
                         ↓        ↓        ↓
                        AZ-1     AZ-2     AZ-3
```

Aurora deliberately separates **database compute** from **database storage**.

That's the foundation for much of Aurora's behavior.

---

# 2. Aurora DB Cluster

An Aurora PostgreSQL deployment is organized around a **DB cluster**.

A cluster contains:

* One writer DB instance
* Zero or more reader DB instances
* Shared Aurora storage
* Cluster endpoints
* Cluster-level configuration

Conceptually:

```text
Aurora PostgreSQL Cluster
│
├── Writer
│
├── Reader
│
├── Reader
│
└── Shared Storage
```

The important word here is **shared**.

---

# 3. Compute instances

Aurora database instances provide the compute layer.

They run the PostgreSQL-compatible database engine and provide:

```text
CPU
Memory
Database processes
Connections
Query execution
Buffer/cache
Background processes
```

There are generally two roles:

```text
Writer
Reader
```

---

# 4. Writer instance

The writer is the primary database instance.

It handles:

```text
INSERT
UPDATE
DELETE
DDL
Transactions
```

For example:

```sql
INSERT INTO customers
(customer_id, name)
VALUES
(101, 'Alex');
```

The write is processed by the writer.

Conceptually:

```text
Application
     ↓
Cluster Endpoint
     ↓
Writer
     ↓
Aurora Storage
```

---

# 5. Reader instances

Reader instances provide additional compute capacity for read workloads.

For example:

```sql
SELECT *
FROM customers
WHERE customer_id = 101;
```

A reader can execute read workloads without becoming the writer.

A cluster might look like:

```text
                 Cluster
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
    Writer       Reader 1     Reader 2
       │            │            │
       └────────────┼────────────┘
                    ↓
             Shared Storage
```

---

# 6. Readers are not separate databases

This is a critical concept.

With Aurora, you don't normally think:

```text
Database A
Database B
Database C
```

Instead:

```text
       Writer
          │
          │
    Shared storage
       /       \
 Reader 1     Reader 2
```

The instances operate against the cluster's shared storage architecture.

This is very different from simply creating independent PostgreSQL servers and replicating complete database copies between them.

---

# 7. Aurora storage

Aurora uses a distributed storage subsystem rather than treating storage as a conventional disk attached to each database instance.

Conceptually:

```text
                 Database instances
                /       |       \
               /        |        \
              ↓         ↓         ↓
        ┌─────────────────────────────┐
        │   Aurora Distributed Store  │
        └─────────────────────────────┘
                  /    |    \
                 ↓     ↓     ↓
                AZ-1  AZ-2  AZ-3
```

This architecture is one of Aurora's defining characteristics.

---

# 8. Storage replication

Aurora's storage layer is designed for high durability by maintaining multiple copies of storage across Availability Zones.

A simplified mental model is:

```text
                    Storage
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
      AZ-1            AZ-2           AZ-3
        │              │              │
      Copies         Copies         Copies
```

The exact implementation is managed by AWS, so you don't manually manage individual storage replicas.

---

# 9. Compute failure vs storage failure

This architecture creates an important distinction.

Suppose:

```text
Writer
  ↓
FAILS
```

The underlying cluster storage doesn't disappear simply because that compute instance failed.

Aurora can use another database instance during recovery/failover.

Conceptually:

```text
Writer A
   ❌
   ↓
Existing Aurora storage
   ↓
Reader B
   ↓
Promoted writer
```

This is one reason Aurora can provide rapid database recovery.

---

# 10. Aurora endpoints

Applications should normally use **endpoints**, rather than depending on a particular DB instance.

The major endpoint concepts are:

### Cluster endpoint

Used for the primary/writer connection.

```text
Application
      ↓
Cluster Endpoint
      ↓
Current Writer
```

### Reader endpoint

Used for read connections.

```text
Application
      ↓
Reader Endpoint
      ↓
Eligible Readers
```

### Instance endpoint

Connects to a particular instance.

```text
Application
      ↓
Instance Endpoint
      ↓
Specific DB Instance
```

---

# 11. Why cluster endpoints are important

Imagine:

```text
Writer A
```

is currently primary.

Your application connects to:

```text
cluster-endpoint
```

Now Writer A fails.

Aurora promotes another suitable instance.

The application can continue using the cluster endpoint rather than needing to know:

> "Which machine is currently the writer?"

Conceptually:

```text
Before:

Cluster Endpoint → Writer A


After failover:

Cluster Endpoint → Writer B
```

The endpoint abstracts the underlying compute instance.

---

# 12. Reader endpoint

The reader endpoint is designed for read workloads.

Conceptually:

```text
                Reader Endpoint
                       │
             ┌─────────┴─────────┐
             ↓                   ↓
          Reader 1             Reader 2
```

This can provide read scaling.

However, your application needs to understand the consistency characteristics of the workload.

For example:

```text
WRITE customer
   ↓
IMMEDIATELY READ customer
```

Depending on where that read goes and the replication state, blindly routing it to a reader may not provide the behavior the application expects.

---

# 13. Instance endpoints

Sometimes you deliberately need to connect to one particular instance.

For example:

```text
Writer instance
Reader instance
```

Instance endpoints are useful for targeted administrative or diagnostic connections.

But application architecture should generally avoid unnecessarily coupling itself to a particular instance.

---

# 14. Aurora storage growth

Aurora storage is designed to grow automatically as the database consumes more space, subject to the service's current limits and configuration.

This differs from traditional environments where a DBA might manually:

```text
Create disk
Extend volume
Resize filesystem
Add LUN
Resize data file
```

Aurora abstracts much of the underlying storage management.

But:

> **Automatic storage growth does not mean storage is free or unlimited.**

You still need capacity and cost monitoring.

---

# 15. Compute scaling vs storage scaling

This distinction is extremely important.

### Compute scaling

Changes the resources available to the database instance:

```text
CPU
Memory
Network capacity
```

### Storage scaling

Changes the amount of database storage available.

Aurora separates these concerns.

Conceptually:

```text
                Aurora
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
     Compute             Storage
        │                   │
  CPU / Memory       Database capacity
```

This separation provides flexibility in managing workloads.

---

# 16. Availability Zones

Aurora's architecture uses multiple Availability Zones for storage durability and availability.

Imagine:

```text
              AWS Region
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
       AZ-1      AZ-2      AZ-3
        │         │         │
        └──── Storage ──────┘
                  │
              DB Cluster
```

The purpose is resilience against failures affecting a single Availability Zone.

---

# 17. Aurora cluster vs instance

This distinction is frequently tested.

### DB Cluster

Logical database system containing:

```text
Storage
Endpoints
Instances
Cluster configuration
```

### DB Instance

Compute resource running the database engine.

Think:

```text
Cluster = database environment

Instance = compute member of that environment
```

---

# 18. Writer and reader roles

A useful way to remember:

```text
WRITER
  │
  ├── Writes
  ├── Reads
  └── Primary transactional workload


READER
  │
  └── Primarily read workload
```

The writer can also process reads.

A reader is intended primarily to scale read workloads and provide additional capacity/HA options.

---

# 19. Failover architecture

Suppose:

```text
Writer
   ↓
Failure
```

Aurora can promote an appropriate reader instance.

Conceptually:

```text
             Before
               │
          Writer A
          Reader B
          Reader C

               ↓
          Writer A fails

               ↓

             After
               │
          Writer B
          Reader C
```

The cluster endpoint is then associated with the current writer.

This is why endpoint-based application design is so important.

---

# 20. Aurora's key architectural advantage

Traditional architecture might look like:

```text
Server
  +
Storage
```

Aurora:

```text
Compute
   +
Distributed shared storage
```

That architectural separation supports:

* Faster recovery
* Multiple compute instances
* Read scaling
* Storage durability
* Simplified storage management
* Managed failover

---

# 21. What Aurora does NOT mean

Don't make these assumptions:

❌ Every query automatically becomes faster.

❌ Readers eliminate all replication-lag considerations.

❌ You never need performance tuning.

❌ You never need backups.

❌ AWS automatically designs your application's HA strategy.

❌ Storage growth means you don't need capacity planning.

Aurora removes significant infrastructure management, but **database engineering still matters**.

---

# 22. Architecture example

Imagine an e-commerce application:

```text
                 E-Commerce App
                       │
             ┌─────────┴─────────┐
             ↓                   ↓
       Write operations     Read operations
             ↓                   ↓
      Cluster Endpoint      Reader Endpoint
             ↓                   ↓
          Writer          Reader 1 / Reader 2
             │                   │
             └─────────┬─────────┘
                       ↓
              Aurora Distributed
                    Storage
```

This architecture separates:

```text
Write workload
Read workload
Compute
Storage
```

while keeping them inside one Aurora cluster.

---

# 23. Architecture decision checklist

When designing Aurora PostgreSQL, ask:

```text
1. How many instances do I need?
2. Which instance is the writer?
3. How much read capacity is required?
4. What endpoint should the application use?
5. What happens during writer failure?
6. What is the RTO?
7. What is the RPO?
8. Which AWS Region?
9. Which Availability Zones?
10. How will the workload be monitored?
```

These questions will come back throughout the remaining lessons.

---

# 🧠 Interview Questions

### Q1. What is an Aurora DB cluster?

A logical Aurora database environment containing database instances, shared distributed storage, endpoints, and cluster-level configuration.

### Q2. What is the difference between a writer and reader?

The writer is the primary instance responsible for write operations. Readers primarily serve read workloads and can provide read scaling and failover targets.

### Q3. Why is Aurora storage different?

Aurora separates compute from a distributed storage layer that is replicated across Availability Zones and managed by AWS.

### Q4. Why should applications use Aurora endpoints?

Endpoints abstract the underlying database instance, allowing Aurora to change the active writer or distribute reads without requiring the application to hard-code individual instances.

---

# 🎯 Lesson 2 — What You Should Remember

```text
             AURORA CLUSTER
                   │
       ┌───────────┴───────────┐
       ↓                       ↓
    COMPUTE                 STORAGE
       │                       │
 ┌─────┴─────┐          Distributed
 ↓           ↓             Storage
Writer     Readers             │
 │           │          Multiple AZs
 └─────┬─────┘
       ↓
   Endpoints
```

### Five key takeaways

**1.** Aurora is organized into **DB clusters**.
**2.** The cluster has a **writer and potentially multiple readers**.
**3.** Compute and storage are **separated**.
**4.** Aurora uses a **distributed storage architecture across Availability Zones**.
**5.** Applications should use **Aurora endpoints** to avoid coupling to a specific instance.

---

**Next → Lesson 3: Aurora PostgreSQL Deployment & Configuration**
