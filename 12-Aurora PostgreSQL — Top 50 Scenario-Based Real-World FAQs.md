**Top 50 real-world, scenario-based Aurora PostgreSQL questions**

# 🐘 Aurora PostgreSQL — Top 50 Scenario-Based Real-World FAQs

## 1. Production application suddenly becomes slow. What do you check first?

1. Confirm whether the slowdown is application-wide or limited to specific APIs/queries.
2. Check CloudWatch metrics such as CPU, connections, memory, and relevant database metrics.
3. Check Performance Insights for database load, top SQL, and wait events.
4. Inspect `pg_stat_activity` for long-running or blocked sessions.
5. Check recent deployments, schema changes, traffic spikes, and configuration changes.
6. Fix the identified bottleneck rather than immediately scaling the DB instance.

---

## 2. CPU suddenly reaches 95%. What would you do?

1. Don't immediately resize the Aurora instance.
2. Use Performance Insights to identify SQL contributing to database load.
3. Check whether workload volume or concurrency suddenly increased.
4. Examine execution plans for expensive queries.
5. Check for missing/ineffective indexes or inefficient joins.
6. Scale only after determining that additional compute capacity is actually required.

---

## 3. CPU is normal, but queries are extremely slow. Why?

1. High CPU is only one possible database bottleneck.
2. Check Performance Insights for wait events.
3. Investigate lock contention and blocked transactions.
4. Check I/O-related waits and database load.
5. Inspect `pg_stat_activity` for long-running or waiting sessions.
6. Determine whether the problem is SQL execution or waiting for another resource.

---

## 4. A query suddenly went from 100 ms to 10 seconds. How do you investigate?

1. Compare the current execution plan with the previous known-good behavior.
2. Run `EXPLAIN ANALYZE` carefully in an appropriate environment.
3. Check whether table statistics changed or became stale.
4. Look for data-volume or data-distribution changes.
5. Check whether indexes or schema changes occurred.
6. Investigate plan changes before simply adding hardware.

---

## 5. `EXPLAIN` shows a sequential scan. Is that automatically a problem?

1. No, sequential scans can be perfectly valid.
2. Check the table size and percentage of rows being returned.
3. Compare estimated and actual rows.
4. Determine whether an index would actually be selective.
5. Check the overall execution plan rather than one node.
6. Optimize only if the sequential scan is actually contributing to the problem.

---

## 6. The application reports "too many connections." What do you check?

1. Check current database connections.
2. Identify which applications/users are creating them.
3. Look for connection leaks.
4. Check whether connection pooling is configured correctly.
5. Investigate application deployments or traffic spikes.
6. Don't simply increase the connection limit without understanding resource impact.

---

## 7. After a deployment, database connections suddenly double. What could be wrong?

1. Check whether the application's connection-pool configuration changed.
2. Look for additional application instances/pods.
3. Check whether every application instance creates its own large pool.
4. Investigate connection leaks or retry storms.
5. Compare connection counts before and after deployment.
6. Correct the application/pooling behavior before increasing database capacity.

---

## 8. Aurora writer fails during business hours. What happens?

1. Aurora detects the writer failure.
2. An eligible reader can be promoted.
3. The cluster endpoint can point to the new writer.
4. Existing connections to the failed writer may be lost.
5. Applications need reconnect and retry handling.
6. Validate that the application recovered and that writes are functioning normally.

---

## 9. Aurora failed over, but the application is still down. What do you investigate?

1. Check whether the application uses the correct cluster endpoint.
2. Determine whether connection pools are still holding stale connections.
3. Check application retry/reconnect behavior.
4. Verify network/security-group connectivity.
5. Check whether transactions were interrupted.
6. Fix application resilience rather than assuming Aurora failover failed.

---

## 10. An application uses an instance endpoint instead of the cluster endpoint. Why is that risky?

1. The instance may no longer be the writer after failover.
2. The application can continue trying to connect to the old writer.
3. Failover abstraction is lost.
4. Connection management becomes more complicated.
5. The cluster endpoint normally provides better writer abstraction.
6. Review endpoint usage as part of HA design.

---

# 🔒 Security Scenarios

## 11. A developer asks for unrestricted database access. What do you do?

1. Determine what operations they actually need.
2. Create/use an appropriate role.
3. Grant only required privileges.
4. Avoid unnecessary administrative or superuser-level access.
5. Separate development, application, and DBA permissions.
6. Follow least-privilege principles.

---

## 12. Database credentials are hard-coded in application code. What is the problem?

1. Credentials can leak through source repositories or logs.
2. Rotation becomes difficult.
3. Multiple applications may share credentials.
4. Compromised credentials can create a large blast radius.
5. Move credentials into an appropriate secrets-management mechanism.
6. Rotate exposed credentials immediately according to incident procedures.

---

## 13. Someone asks to open PostgreSQL port 5432 to `0.0.0.0/0`. Would you approve it?

1. Normally, no.
2. It exposes the database to potentially unnecessary network access.
3. Restrict access to known application/network sources.
4. Use security groups and private networking appropriately.
5. Provide controlled administrative access for DBAs.
6. Any exception should have a strong documented justification and compensating controls.

---

## 14. An application suddenly receives authentication failures. What do you check?

1. Check whether credentials were changed or rotated.
2. Verify the application's configured secret.
3. Confirm the database role still exists and can log in.
4. Check whether the application is connecting to the expected cluster/database.
5. Review relevant logs and recent deployments.
6. Check whether network connectivity is being confused with authentication failure.

---

## 15. A security audit finds that every application uses the same DB user. What should you do?

1. Identify the applications and their actual permissions.
2. Create appropriately scoped roles.
3. Separate application identities where practical.
4. Apply least privilege.
5. Rotate the shared credential.
6. Establish a role/credential lifecycle and ownership process.

---

# 🔐 Backup & Recovery Scenarios

## 16. Someone accidentally deletes millions of rows. What do you do?

1. Stop or isolate the operation causing further damage.
2. Preserve evidence and determine the exact incident time.
3. Identify an appropriate recovery point.
4. Use PITR or another approved recovery mechanism to create a recovered environment.
5. Validate the recovered data.
6. Develop a controlled procedure for restoring/reconciling required data into production.

---

## 17. The business asks, "Can we recover the database to 2:37 PM?" What do you need to know?

1. Confirm the available backup/PITR recovery window.
2. Verify the exact time and timezone.
3. Determine the required database recovery target.
4. Restore to an appropriate environment.
5. Validate data around the target timestamp.
6. Only then decide how to perform the production recovery.

---

## 18. Your backup exists, but nobody has tested restoring it. Is the system DR-ready?

1. No.
2. Backup existence doesn't prove recoverability.
3. Perform a restore test.
4. Validate database integrity and application compatibility.
5. Measure actual recovery time.
6. Document and regularly rehearse the recovery procedure.

---

## 19. The primary AWS Region becomes unavailable. What do you do?

1. Activate the documented DR process.
2. Determine whether the configured DR architecture meets the incident requirements.
3. Promote/use the appropriate secondary-region environment.
4. Redirect application traffic.
5. Validate database and application functionality.
6. Monitor recovery and document the incident afterward.

---

## 20. Management says RTO is 15 minutes but your restore process takes 2 hours. What is wrong?

1. The recovery architecture does not meet the business requirement.
2. Backup capability alone isn't sufficient.
3. Review the current RTO and actual recovery steps.
4. Evaluate a faster DR architecture.
5. Automate recovery and traffic redirection where appropriate.
6. Test the redesigned process against the 15-minute requirement.

---

# 🔄 Performance Scenarios

## 21. A table has 500 million rows and a query scans the entire table. What do you investigate?

1. Understand the query's filtering and access pattern.
2. Check whether an appropriate index can reduce the scanned data.
3. Review `EXPLAIN ANALYZE`.
4. Check selectivity and statistics.
5. Consider partitioning only if it genuinely matches the workload.
6. Validate improvements using realistic production-like data.

---

## 22. A query has an index but PostgreSQL still chooses a sequential scan. Why?

1. The optimizer may consider the sequential scan cheaper.
2. The query may return a large percentage of rows.
3. The index may have poor selectivity.
4. Statistics may be inaccurate.
5. Data distribution may have changed.
6. Trust the plan only after checking actual performance with appropriate testing.

---

## 23. An index fixed reads but writes became slower. Why?

1. Every index must be maintained during relevant writes.
2. Additional indexes increase write overhead.
3. Indexes consume storage.
4. More indexes can increase maintenance work.
5. Measure read benefit versus write cost.
6. Keep indexes that provide meaningful workload value.

---

## 24. A reporting query is hurting production transactions. What would you do?

1. Determine whether the reporting query can be routed to readers.
2. Use a dedicated reader or custom endpoint where appropriate.
3. Optimize the reporting SQL.
4. Review its indexes and execution plan.
5. Consider workload isolation if requirements justify it.
6. Ensure critical transactional workloads remain protected.

---

## 25. A query suddenly starts using a different execution plan. What could cause it?

1. Statistics may have changed.
2. Data distribution may have changed.
3. Table size may have increased significantly.
4. PostgreSQL configuration may have changed.
5. Available indexes may have changed.
6. Compare old and new plans to identify the specific decision that changed.

---

## 26. Query estimates say 100 rows but actual results are 10 million. Why is that dangerous?

1. The optimizer is making a severely incorrect assumption.
2. It may choose an inappropriate join strategy.
3. It may allocate resources incorrectly.
4. It can produce dramatically slower plans.
5. Investigate statistics and data distribution.
6. Use `EXPLAIN ANALYZE` to confirm the estimation problem.

---

## 27. A query is fast normally but extremely slow at month-end. Why?

1. Month-end may produce a much larger dataset.
2. Data distribution may change.
3. Concurrency may increase.
4. Reporting workloads may compete with transactions.
5. Query plans may behave differently at larger volumes.
6. Reproduce the month-end workload and optimize for realistic production conditions.

---

# 🔒 Locking & Transactions

## 28. Users report that UPDATE statements are hanging. What do you check?

1. Inspect active sessions.
2. Check for blocking transactions.
3. Identify the session holding the relevant lock.
4. Check transaction duration.
5. Investigate idle-in-transaction sessions.
6. Resolve the underlying transaction/locking issue rather than killing sessions blindly.

---

## 29. A transaction has been open for 30 minutes. Why is that concerning?

1. It may retain locks.
2. It can delay cleanup of old row versions.
3. It can contribute to bloat.
4. It can block other workloads.
5. Investigate why the application left the transaction open.
6. Fix transaction management rather than simply terminating sessions repeatedly.

---

## 30. One query is blocking hundreds of sessions. What do you do?

1. Identify the blocking session.
2. Determine what transaction it is performing.
3. Determine whether it is legitimate or stuck.
4. Assess business impact before terminating it.
5. If safe, terminate/resolve the blocking transaction according to the incident procedure.
6. Fix the application or SQL behavior that caused the blocking.

---

## 31. An application opens a transaction and then performs several API calls before committing. What is the risk?

1. The transaction remains open unnecessarily long.
2. Locks may be held longer.
3. Old row versions may remain visible.
4. Other transactions can be blocked.
5. Database resource usage can increase.
6. Keep transaction boundaries as short as the business operation permits.

---

# 📈 Scaling Scenarios

## 32. CPU is consistently 85% during peak hours. Should you scale up?

1. First establish whether CPU is actually the bottleneck.
2. Identify top SQL and workload patterns.
3. Determine whether query optimization can reduce CPU.
4. Check whether traffic growth is expected.
5. If workload genuinely requires more capacity, consider scaling.
6. Monitor after the change to verify the expected improvement.

---

## 33. Read traffic is growing rapidly while writes remain manageable. What architecture would you consider?

1. Add appropriate Aurora reader capacity.
2. Route eligible read traffic to readers.
3. Use appropriate endpoints/routing.
4. Verify consistency requirements.
5. Monitor reader load and connection distribution.
6. Continue optimizing expensive read queries.

---

## 34. Adding readers didn't improve performance. Why?

1. The workload may be primarily write-heavy.
2. Reads may still be routed to the writer.
3. The bottleneck may be a single expensive query.
4. Lock contention may remain.
5. Application connection routing may be incorrect.
6. Check actual workload distribution before adding more readers.

---

## 35. Database connections are high even after adding readers. Why?

1. Readers don't automatically solve connection-management problems.
2. Each application instance may still create excessive connections.
3. Connection pools may be oversized.
4. Connection leaks may exist.
5. Check connections per application instance and per reader.
6. Fix pooling before increasing database capacity further.

---

# 🧹 PostgreSQL Maintenance Scenarios

## 36. A frequently updated table is growing rapidly. What do you investigate?

1. Check update/delete patterns.
2. Investigate dead tuples and vacuum behavior.
3. Check autovacuum activity.
4. Look for long-running transactions.
5. Examine table/index bloat indicators.
6. Tune maintenance only after identifying the underlying cause.

---

## 37. Autovacuum isn't keeping up. What could be happening?

1. The table may have extremely high update/delete activity.
2. Autovacuum thresholds may not suit the workload.
3. Long-running transactions may delay cleanup.
4. Resource contention may slow maintenance.
5. Table-specific settings may need review.
6. Measure first rather than globally increasing aggressive vacuum settings.

---

## 38. A table has many dead tuples. What could be causing it?

1. Heavy UPDATE/DELETE activity.
2. Long-running transactions.
3. Inadequate vacuum frequency.
4. Maintenance contention.
5. Application transaction design issues.
6. Investigate the complete MVCC/vacuum lifecycle.

---

# 🚨 Incident Scenarios

## 39. Latency increased immediately after a schema migration. What do you do?

1. Correlate the incident timeline with the migration.
2. Identify which SQL/query behavior changed.
3. Check locks and database load.
4. Compare execution plans before and after the migration.
5. If necessary, use the documented rollback/mitigation procedure.
6. Determine the root cause before reapplying the migration.

---

## 40. A deployment caused a massive increase in database load. How do you investigate?

1. Identify queries introduced or changed by the deployment.
2. Compare Performance Insights before and after deployment.
3. Check connection count and concurrency.
4. Inspect execution plans.
5. Determine whether the deployment changed query frequency or efficiency.
6. Roll back or mitigate if the deployment is confirmed as the cause.

---

## 41. Application retries caused even more database load during an outage. What happened?

1. The application likely created a retry storm.
2. Every failed request generated additional database attempts.
3. This increased pressure on an already unhealthy system.
4. Add exponential backoff and jitter.
5. Limit retry counts.
6. Design retries with database and business-operation idempotency in mind.

---

## 42. Database health looks normal, but users still report failures. What next?

1. Check application logs.
2. Verify connection and authentication errors.
3. Check network paths and security groups.
4. Check application timeouts and retry behavior.
5. Determine whether only one application component is affected.
6. Don't assume every user-facing database error is a database-capacity problem.

---

# 🏗️ Architecture Scenarios

## 43. Your company requires RPO close to zero and very low RTO. What should you investigate?

1. Precisely define the business RPO/RTO.
2. Evaluate Aurora HA capabilities.
3. Evaluate cross-Region DR requirements.
4. Consider architectures designed for rapid Regional recovery.
5. Automate traffic redirection and recovery procedures where appropriate.
6. Test the design against the actual RPO/RTO targets.

---

## 44. Your company says downtime of several hours is acceptable. Do you need an expensive multi-Region architecture?

1. Not necessarily.
2. Start with the actual business RTO/RPO.
3. Evaluate backup/PITR-based recovery.
4. Compare recovery time and operational complexity.
5. Choose the least complex architecture that satisfies requirements.
6. Revisit the design if business requirements change.

---

## 45. You need a separate workload for analytics. What would you consider?

1. Determine whether the workload can run against Aurora readers.
2. Consider a dedicated reader for workload isolation.
3. Use suitable routing mechanisms.
4. Optimize analytical SQL.
5. Monitor impact on transactional workloads.
6. If analytics outgrow the operational database, consider a dedicated analytics platform.

---

## 46. You need zero-downtime application deployments. How should database changes be handled?

1. Make schema changes backward compatible where possible.
2. Use an expand-and-contract migration strategy.
3. Deploy application compatibility before removing old structures.
4. Avoid risky long-running blocking migrations.
5. Test migrations against production-scale data.
6. Maintain a rollback/mitigation plan.

---

# 💰 Cost & Operations

## 47. AWS costs for Aurora suddenly increased. How do you investigate?

1. Compare current and previous DB instance capacity.
2. Check whether additional readers were added.
3. Review storage and backup usage.
4. Investigate cross-Region/data-transfer changes.
5. Check workload growth.
6. Optimize unnecessary capacity without compromising required availability.

---

## 48. The team wants to remove a reader to save money. What should you ask first?

1. Is that reader required for HA/failover?
2. What is its failover priority?
3. What happens to performance if it is removed?
4. Does it serve reporting or read workloads?
5. What are the RTO requirements?
6. Make the decision based on workload and business requirements, not cost alone.

---

# 🧠 Senior-Level Scenarios

## 49. You have 30 minutes to troubleshoot a major Aurora outage. What is your approach?

1. **0–5 min:** Confirm scope, impact, and whether there was a recent change.
2. **5–10 min:** Check CloudWatch, connections, database load, and Aurora events.
3. **10–15 min:** Check Performance Insights, waits, top SQL, and active sessions.
4. **15–20 min:** Determine whether the issue is capacity, SQL, locks, connectivity, or infrastructure.
5. **20–25 min:** Apply the safest targeted mitigation available.
6. **25–30 min:** Verify recovery, communicate status, and preserve evidence for root-cause analysis.

---

# 🏆 50. You are asked to design a production Aurora PostgreSQL environment from scratch. What is your approach?

1. **Security:** Design VPC/subnets, security groups, TLS, credentials, and least-privilege roles.
2. **HA:** Deploy appropriate writer/readers across AZs and configure failover priorities/endpoints.
3. **Performance:** Establish instance sizing, connection pooling, indexing, query monitoring, and baselines.
4. **Recovery:** Define backup retention, PITR, snapshots, RPO/RTO, and DR architecture.
5. **Operations:** Build CloudWatch/Performance Insights dashboards, alerts, logs, runbooks, and change procedures.
6. **Validation:** Test failover, restore, application reconnection, migrations, scaling, and disaster recovery **before** calling the platform production-ready.

---

# 🔥 The 10 Scenarios You Should Master First

If you're preparing for a **real Aurora PostgreSQL production/DBA interview**, prioritize these:

| Priority | Scenario                                   |
| -------- | ------------------------------------------ |
| ⭐ 1      | Production database suddenly slow          |
| ⭐ 2      | CPU reaches 95%                            |
| ⭐ 3      | Lock blocking hundreds of sessions         |
| ⭐ 4      | Writer fails / Aurora failover             |
| ⭐ 5      | Application doesn't recover after failover |
| ⭐ 6      | Accidental DELETE → PITR                   |
| ⭐ 7      | Query plan suddenly changes                |
| ⭐ 8      | Connection storm                           |
| ⭐ 9      | Cross-Region DR / RPO / RTO                |
| ⭐ 10     | Design Aurora production architecture      |

### The golden production troubleshooting pattern

```text
SYMPTOM
   ↓
SCOPE
   ↓
METRICS
   ↓
DATABASE LOAD
   ↓
WAIT EVENTS
   ↓
TOP SQL
   ↓
SESSIONS / LOCKS
   ↓
EXPLAIN ANALYZE
   ↓
ROOT CAUSE
   ↓
TARGETED FIX
   ↓
VERIFY
   ↓
DOCUMENT
```

**This is the pattern I would use in almost every Aurora PostgreSQL production scenario.**
