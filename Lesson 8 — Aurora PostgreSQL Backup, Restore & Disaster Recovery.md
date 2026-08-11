# 🐘 Lesson 8 — Aurora PostgreSQL Backup, Restore & Disaster Recovery

Now we move from **High Availability** to **Backup, Restore, and Disaster Recovery (DR)**.

The key distinction:

> **HA keeps the database available during infrastructure failures. Backup/DR helps you recover when data or an entire environment must be restored.**

---

## 1. The recovery hierarchy

Think about recovery in layers:

```text
Aurora PostgreSQL
       │
       ├── HA / Failover
       │      ↓
       │   Instance failure
       │
       ├── Automated Backups
       │      ↓
       │   Point-in-time recovery
       │
       ├── Snapshots
       │      ↓
       │   Long-term restore
       │
       └── DR Architecture
              ↓
           Regional / major failure
```

---

# 2. Automated backups

Aurora provides automated backup capabilities.

These are designed to support recovery to a point in time within the configured backup retention period.

Conceptually:

```text
Monday
  │
  ├── Backup history
  ├── Tuesday
  ├── Wednesday
  ├── Thursday
  └── Friday
```

The important concept is:

> **You can recover to a specific point in time rather than only restoring yesterday's full backup.**

---

# 3. Point-in-time recovery

Suppose:

```text
10:00 AM → Normal
11:00 AM → Normal
12:00 PM → Accidental DELETE
```

You may want:

```text
Restore → 11:59 AM
```

Conceptually:

```text
Database history
──────────────────────────────→
10:00   11:00   11:59   12:00
                         ❌ DELETE

                    ↑
             Recovery target
```

This is **point-in-time recovery (PITR)**.

---

# 4. Why PITR is powerful

Imagine someone accidentally executes:

```sql
DELETE FROM customers;
```

and commits it.

Aurora HA won't automatically fix this.

Instead:

```text
Accidental change
      ↓
Identify recovery point
      ↓
Point-in-time restore
      ↓
Recovered Aurora environment
```

You can then inspect and determine how to safely recover the required data/application state.

---

# 5. Backup vs snapshot

These concepts are different.

### Automated backups

Designed for ongoing recovery and point-in-time restoration.

### Manual snapshots

Explicit snapshots you create and retain for longer-term purposes.

Think:

```text
Automated backup
      ↓
Operational recovery


Manual snapshot
      ↓
Explicit retained copy
```

---

# 6. When would you use a snapshot?

Examples:

```text
Before major migration
Before major schema change
Before risky application deployment
Long-term retention
Creating a copy of a database environment
```

For example:

```text
Production
    ↓
Manual snapshot
    ↓
Major upgrade
```

If something goes seriously wrong, the snapshot provides another recovery option.

---

# 7. Restore doesn't mean "overwrite production"

A very important concept:

When you restore an Aurora database, you generally create a **new DB cluster/environment** from the recovery point or snapshot.

Conceptually:

```text
Original Cluster
       │
       │ Restore
       ↓
New Cluster
```

This is useful because you can:

```text
Inspect recovered data
Validate application behavior
Compare environments
Perform controlled recovery
```

---

# 8. Recovery workflow

Suppose production data is corrupted.

A simplified workflow:

```text
Incident
   ↓
Stop further damage
   ↓
Determine recovery target
   ↓
Restore Aurora cluster
   ↓
Validate database
   ↓
Validate application
   ↓
Decide cutover strategy
   ↓
Resume production
```

Don't rush directly into a restore without understanding what happened.

---

# 9. RPO

Remember from Lesson 7:

**RPO = Recovery Point Objective**

It asks:

> How much data can the business afford to lose?

Example:

```text
RPO = 15 minutes
```

means the recovery architecture should aim to limit potential data loss to roughly that level.

Your backup and DR strategy must satisfy the business's actual RPO.

---

# 10. RTO

**RTO = Recovery Time Objective**

It asks:

> How long can the application be unavailable?

Example:

```text
RTO = 30 minutes
```

means the recovery process should target restoring service within approximately 30 minutes.

---

# 11. RPO + RTO

Put them together:

```text
RPO
 ↓
How much data can we lose?

RTO
 ↓
How long can we be down?
```

Example:

```text
Business requirement

RPO = 5 minutes
RTO = 30 minutes
```

Now your architecture must be designed to meet those requirements.

---

# 12. HA vs Backup vs DR

This is an important interview topic.

| Capability        | Primary purpose                           |
| ----------------- | ----------------------------------------- |
| Aurora failover   | Instance availability                     |
| Automated backups | Recovery/PITR                             |
| Manual snapshots  | Retained recovery copy                    |
| Cross-Region DR   | Regional disaster                         |
| Application retry | Recover from transient connection failure |

Think:

```text
Instance failure
      ↓
HA


Accidental deletion
      ↓
Backup / PITR


Regional disaster
      ↓
DR
```

---

# 13. Cross-Region disaster recovery

For stronger DR requirements, you may maintain Aurora infrastructure in another AWS Region.

Conceptually:

```text
               Application
                    │
                    ↓
              Primary Region
                    │
                 Aurora
                    │
              DR mechanism
                    ↓
              Secondary Region
                    │
                 Aurora
```

The exact Aurora feature used depends on your engine/version and DR requirements.

---

# 14. Why another Region?

Suppose:

```text
Region A
    ❌
```

Multi-AZ infrastructure inside Region A doesn't protect you from a full Regional outage.

A second Region gives you a separate failure domain.

```text
Region A                Region B
─────────               ─────────
Aurora                  Aurora
   │                       │
Primary                  DR
```

---

# 15. Global Database concept

Aurora Global Database is designed for globally distributed Aurora deployments, with a primary Region and secondary Regions.

Conceptually:

```text
             Aurora Global Database
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
   Primary Region          Secondary Region
          │                     │
       Writer                Readers
```

This can provide:

* Cross-Region disaster recovery
* Global read locality
* Faster recovery options for Regional failures

The exact behavior and supported capabilities depend on the Aurora engine/version and configuration.

---

# 16. DR isn't just a database problem

Suppose Aurora is successfully recovered.

But your application infrastructure is gone.

You still have:

```text
Database ✅
Application ❌
DNS ❌
Secrets ❌
Networking ❌
Monitoring ❌
```

Therefore a real DR strategy must consider the entire application stack:

```text
Application
Database
Networking
DNS
IAM
Secrets
Infrastructure
Monitoring
Deployment
```

---

# 17. Infrastructure as Code

DR becomes much easier when infrastructure is reproducible.

Conceptually:

```text
Infrastructure Definition
        ↓
VPC
        ↓
Security Groups
        ↓
Aurora
        ↓
Application
```

Instead of manually rebuilding everything during an emergency, infrastructure can be recreated consistently.

Common IaC approaches include AWS CloudFormation and Terraform.

---

# 18. Backup retention strategy

Think beyond:

> "Do we have backups?"

Ask:

```text
How long are backups retained?
Who can delete them?
Can they be restored?
Have restores been tested?
What is the required RPO?
What is the required RTO?
```

Backup existence isn't the same as recoverability.

---

# 19. Restore testing

This is one of the most important DBA practices.

Don't merely say:

> "Our backups are enabled."

Test:

```text
Backup
 ↓
Restore
 ↓
Connect
 ↓
Validate tables
 ↓
Validate data
 ↓
Validate application
```

A backup that has never been restored is an **untested recovery assumption**.

---

# 20. Recovery validation

After restoring, validate:

```text
Database exists
 ↓
Expected schemas exist
 ↓
Expected tables exist
 ↓
Expected row counts
 ↓
Critical queries work
 ↓
Users/roles work
 ↓
Application connects
 ↓
Application transactions work
```

Don't stop at:

> "The database started."

---

# 21. Logical corruption

Not all failures are hardware failures.

Consider:

```text
Application bug
     ↓
UPDATE customers
SET balance = 0;
```

The database may be completely healthy from an infrastructure perspective.

Aurora won't automatically know the data is logically wrong.

This is why backups and point-in-time recovery are essential.

---

# 22. Snapshot-based migration

Snapshots can also be useful for migration workflows.

Conceptually:

```text
Source Aurora
      ↓
Snapshot / copy
      ↓
Target environment
      ↓
Validation
```

Always verify compatibility, downtime requirements, networking, security, and application behavior before treating this as a complete migration strategy.

---

# 23. DR runbook

A good organization should have a written DR runbook.

Example:

```text
1. Declare incident
2. Identify failure scope
3. Determine primary/DR status
4. Validate database state
5. Activate DR environment
6. Update application configuration
7. Validate connectivity
8. Validate critical transactions
9. Redirect traffic
10. Monitor
11. Document incident
```

People should not be inventing the recovery process during a disaster.

---

# 24. Common backup/DR mistakes

### ❌ Mistake 1 — Confusing HA with backup

Failover doesn't protect against accidental data changes.

### ❌ Mistake 2 — Never testing restores

Backup ≠ proven recovery.

### ❌ Mistake 3 — No RPO/RTO definition

Without business targets, you can't design the right DR architecture.

### ❌ Mistake 4 — DR database but no application DR

The database alone isn't the application.

### ❌ Mistake 5 — No documented runbook

People make mistakes under pressure.

### ❌ Mistake 6 — Assuming snapshots are a complete DR strategy

A snapshot is only one component of recovery architecture.

---

# 🧠 Interview Questions

### Q1. What is point-in-time recovery?

The ability to restore a database to a selected point in its recovery history within the available recovery window.

### Q2. What is the difference between RPO and RTO?

**RPO** = acceptable amount of data loss.
**RTO** = acceptable recovery time.

### Q3. Does Aurora failover protect against accidental DELETE statements?

No. Failover protects availability; backup/PITR mechanisms address data recovery.

### Q4. Why should restore testing be performed?

To verify that backups are actually usable and that the organization can meet its recovery objectives.

### Q5. Why might you use another AWS Region?

To provide protection against a major failure affecting the primary Region and to support a cross-Region DR strategy.

---

# 🎯 Lesson 8 Summary

Remember:

```text
                 RECOVERY
                    │
       ┌────────────┼────────────┐
       ↓            ↓            ↓
      HA          Backup          DR
       │            │             │
   Instance       PITR        Other Region
    failure      Restore       / Recovery
```

### Five things to remember

**1.** **HA ≠ backup ≠ DR.**

**2.** PITR is extremely useful for recovering from accidental or logical data changes.

**3.** **RPO** tells you how much data loss is acceptable.

**4.** **RTO** tells you how quickly recovery must happen.

**5.** Always **test restores and DR procedures**—don't merely enable backups.

---

**Next → Lesson 9: Aurora PostgreSQL Monitoring, Logging & Troubleshooting**
