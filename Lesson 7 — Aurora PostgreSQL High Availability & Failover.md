# 🐘 Lesson 7 — Aurora PostgreSQL High Availability & Failover

This lesson is about one of Aurora PostgreSQL's biggest strengths:

> **How Aurora continues operating when database infrastructure fails.**

We'll focus strictly on **HA, failover, writer/reader roles, and recovery behavior**.

---

# 1. High Availability vs Disaster Recovery

First, don't confuse these two.

### High Availability (HA)

Protects against failures such as:

```text
DB instance failure
Infrastructure failure
Availability Zone problems
```

Goal:

> **Keep the application available with minimal interruption.**

### Disaster Recovery (DR)

Deals with larger events such as:

```text
Region failure
Major data-center event
Large-scale corruption
```

Goal:

> **Recover the application/database in another environment or Region.**

So:

```text
HA
 ↓
Keep operating


DR
 ↓
Recover after major disaster
```

---

# 2. Aurora's HA architecture

A simplified Aurora architecture:

```text id="q4m8s2"
                 Aurora Cluster
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
       Writer      Reader 1     Reader 2
          │           │           │
          └───────────┼───────────┘
                      ↓
             Aurora Distributed
                   Storage
                      │
              Multiple AZs
```

The important point:

> **The compute instances share the Aurora cluster's distributed storage architecture.**

This makes failover fundamentally different from simply restoring an entire database from scratch.

---

# 3. What happens when the writer fails?

Suppose:

```text id="m6x2p9"
Writer A
   ❌
   ↓
Failure
```

If suitable reader instances exist, Aurora can promote one of them.

Conceptually:

```text id="z8k4v1"
BEFORE

Writer A
Reader B
Reader C


Writer A fails


AFTER

Writer B
Reader C
```

The cluster continues with a new writer.

---

# 4. Why readers matter for HA

Readers aren't only for scaling reads.

They can also provide potential failover targets.

Think:

```text id="g2r7m5"
Reader
  │
  ├── Read workload
  │
  └── Potential failover target
```

This is why production Aurora clusters commonly have more than one DB instance.

---

# 5. Single-instance Aurora

You can have a cluster with only one DB instance.

Conceptually:

```text id="c8n4x1"
Aurora Cluster
      │
    Writer
      │
  Distributed
    Storage
```

This provides Aurora's storage architecture, but it doesn't provide the same instance-level failover capability as a cluster with additional instances.

If the only instance fails:

```text id="p6m2k9"
Writer fails
   ↓
No reader available
   ↓
Recovery requires replacement/restart
```

Therefore:

> **Aurora storage durability does not automatically mean your application has zero-downtime compute HA.**

---

# 6. Multi-instance architecture

A stronger HA architecture:

```text id="v5r8k3"
Aurora Cluster
│
├── Writer
├── Reader 1
└── Reader 2
```

Now:

```text id="x7n3m6"
Writer fails
      ↓
Reader selected
      ↓
Reader promoted
      ↓
New Writer
```

This significantly improves failover capability.

---

# 7. Failover priority

Aurora can use configured instance priorities to influence which instance should be promoted during failover.

Think:

```text id="b9k4p2"
Reader 1 → preferred promotion target
Reader 2 → secondary target
```

This allows you to influence the failover order.

For example, you might want:

```text id="q2m7x8"
Large reader
   ↓
Preferred promotion


Small reader
   ↓
Lower priority
```

Why?

Because promoting a very small instance might result in insufficient capacity for the production workload.

---

# 8. Instance size matters during failover

Imagine:

```text id="r8x2m5"
Writer = large
Reader 1 = large
Reader 2 = small
```

If the large writer fails, promoting the small reader may restore availability but create a serious performance problem.

So HA design must consider:

```text id="w4n7k2"
Failover capability
+
Post-failover capacity
```

A system isn't truly resilient if failover leaves it severely underpowered.

---

# 9. Cluster endpoint during failover

This is why the cluster endpoint is so important.

Before:

```text id="h6p3m8"
Cluster Endpoint
       ↓
Writer A
```

After:

```text id="z4q7n1"
Cluster Endpoint
       ↓
Writer B
```

The application continues using the same logical endpoint.

It doesn't need to know:

> "Writer A died; connect to Writer B."

Aurora handles the cluster role transition.

---

# 10. Application reconnect behavior

Failover doesn't mean an existing TCP connection magically survives.

A connection to the failed writer can be broken.

Therefore applications need appropriate:

```text id="j8m2v5"
Connection retry
Connection pool recovery
Timeouts
Transaction retry
Error handling
```

The database can fail over successfully while an application still fails if its connection-handling logic is poor.

---

# 11. Retry logic

Suppose:

```text id="n4x7c2"
Application
    ↓
Database connection
    ↓
Writer fails
    ↓
Connection breaks
```

The application should be capable of:

```text id="m6p2r8"
Detect failure
    ↓
Wait / backoff
    ↓
Reconnect
    ↓
Resolve cluster endpoint
    ↓
Connect to new writer
```

Use sensible retry strategies rather than an aggressive infinite retry loop.

---

# 12. Transactions during failover

Suppose a transaction is running:

```text id="k7m3x9"
BEGIN
  ↓
UPDATE
  ↓
Writer fails
  ↓
Connection lost
```

That transaction should be considered failed from the application's perspective.

The application may need to:

```text id="q5v8n2"
Reconnect
   ↓
Determine transaction outcome
   ↓
Retry safely if appropriate
```

This is why **idempotency** and transaction design matter.

---

# 13. Don't blindly retry writes

Imagine:

```text id="w3p9m6"
INSERT payment
       ↓
Network interruption
       ↓
Application doesn't know:
Did INSERT commit?
```

If the application blindly retries:

```text id="f8k2x4"
INSERT payment again
```

you could create a duplicate.

Better application design can use:

```text id="m5n7q1"
Idempotency keys
Unique constraints
Transaction design
Application-level request IDs
```

HA therefore involves **application engineering**, not just database configuration.

---

# 14. Reader endpoint and HA

The reader endpoint is primarily intended for read traffic.

During a topology change, eligible readers can change.

Conceptually:

```text id="k4m8x2"
Reader Endpoint
      │
 ┌────┴────┐
 ↓         ↓
Reader A  Reader B
```

Applications should not assume the endpoint permanently represents one particular server.

---

# 15. Availability Zones

Aurora's storage layer is distributed across multiple AZs.

Conceptually:

```text id="p8n3v6"
              Aurora Storage
             /      |      \
            ↓       ↓       ↓
          AZ-A    AZ-B    AZ-C
```

This provides resilience against failures affecting a single Availability Zone.

---

# 16. Regional failure

Now consider a much bigger problem:

```text id="y7m2k4"
Entire AWS Region
       ❌
```

Multi-AZ HA does **not** automatically mean your application is protected against an entire Region becoming unavailable.

That's where DR architectures come in.

```text id="n5x8p3"
Region A
Aurora
   │
   │ DR strategy
   ↓
Region B
Aurora
```

We'll distinguish HA and DR more deeply in Lesson 8.

---

# 17. RTO

**Recovery Time Objective (RTO)** asks:

> How long can the application be unavailable?

Example:

```text id="f3k7m9"
RTO = 60 seconds
```

means the target is to recover service within approximately that timeframe.

---

# 18. RPO

**Recovery Point Objective (RPO)** asks:

> How much data loss can the business tolerate?

For example:

```text id="q9m4x2"
RPO = 5 minutes
```

means the business could potentially tolerate losing up to about five minutes of data, depending on the architecture.

HA and DR strategies should be designed around actual RTO/RPO requirements.

---

# 19. HA isn't the same as backup

This distinction is extremely important.

### HA

```text id="v4n8p1"
Writer fails
   ↓
Reader promoted
```

### Backup recovery

```text id="z6m2q7"
Database corruption
       ↓
Restore/recovery
```

A failover mechanism doesn't protect you from every logical problem.

For example:

```text id="j8x4k5"
DELETE FROM customers;
```

If the statement succeeds, HA won't say:

> "That was accidental. Let's undo it."

You need backup/recovery mechanisms for that.

---

# 20. Testing failover

Never assume your HA design works.

Test it.

A failover test should verify:

```text id="c7n2m5"
Database detects failure
        ↓
New writer selected
        ↓
Cluster endpoint moves
        ↓
Application reconnects
        ↓
Transactions recover appropriately
        ↓
Application resumes
```

Measure:

```text id="x5k8q3"
Detection time
Failover time
Reconnect time
Application recovery time
```

---

# 21. HA testing checklist

Before declaring an Aurora environment production-ready:

```text id="r4m7x1"
☐ Multiple DB instances
☐ Appropriate AZ distribution
☐ Failover priority configured
☐ Cluster endpoint used
☐ Connection retry implemented
☐ Connection pooling tested
☐ Transaction retry behavior tested
☐ Monitoring configured
☐ Alerts configured
☐ Failover tested
☐ Recovery time measured
```

---

# 22. Common HA mistakes

### ❌ Mistake 1 — Only one DB instance

You have fewer compute-level failover options.

### ❌ Mistake 2 — Small reader used as failover target

The database may recover but have inadequate capacity.

### ❌ Mistake 3 — Hard-coded writer endpoint

Application may fail after topology changes.

### ❌ Mistake 4 — No connection retry logic

Aurora fails over, but the application remains disconnected.

### ❌ Mistake 5 — Blind transaction retries

Can cause duplicate writes.

### ❌ Mistake 6 — Assuming HA protects against data corruption

It doesn't.

### ❌ Mistake 7 — Never testing failover

An untested HA architecture is an assumption.

---

# 23. Production HA architecture

A strong example:

```text id="s9x4k2"
                 Application
                     │
                     ↓
             Cluster Endpoint
                     │
                     ↓
                ┌─────────┐
                │ Writer  │
                └────┬────┘
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
      Reader 1              Reader 2
       Priority 0             Priority 1
          │                     │
          └──────────┬──────────┘
                     ↓
            Aurora Distributed
                  Storage
               /    |    \
             AZ-A  AZ-B  AZ-C
```

If the writer fails:

```text id="h4p7m2"
Writer
  ❌
  ↓
Reader 1
  ↓
Promoted
  ↓
New Writer
```

---

# 🧠 Interview Questions

### Q1. What happens when an Aurora writer fails?

Aurora can promote an eligible reader instance to become the new writer, depending on the cluster's configuration and available instances.

### Q2. Why should applications use the cluster endpoint?

Because the active writer can change during failover. The cluster endpoint abstracts the current writer.

### Q3. Does failover preserve existing database connections?

No. Connections to the failed instance can be broken, so applications need appropriate reconnect and retry handling.

### Q4. Is Aurora Multi-AZ the same as DR?

No. Multi-AZ primarily addresses availability within a Region. DR addresses larger failures, including potentially an entire Region.

### Q5. Why is failover priority important?

It influences which reader is preferred for promotion, helping you choose an instance with suitable capacity.

---

# 🎯 Lesson 7 Summary

Your HA mental model:

```text id="m7x3q8"
                  AURORA
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
       Writer                 Readers
          │                     │
          │                 Failover
          │                  Targets
          └──────────┬──────────┘
                     ↓
            Distributed Storage
                     │
                Multiple AZs
```

When the writer fails:

```text id="v2k8m4"
Writer failure
     ↓
Eligible reader
     ↓
Promotion
     ↓
New writer
     ↓
Cluster endpoint
     ↓
Application reconnects
```

### Five things to remember

**1.** Aurora HA relies on **multiple DB instances + distributed storage**.

**2.** Readers can serve both **read workloads and failover roles**.

**3.** The **cluster endpoint** is critical during writer failover.

**4.** Applications must handle **connection failures and safe transaction retries**.

**5.** **HA ≠ backup ≠ DR**.

---

**Next → Lesson 8: Aurora PostgreSQL Backup, Restore & Disaster Recovery**
