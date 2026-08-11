# 🐘 Lesson 3 — Aurora PostgreSQL Deployment & Configuration

In Lesson 1 we learned **what Aurora PostgreSQL is**.
In Lesson 2 we learned **how the cluster is architected**.

Now we'll learn how to **deploy and configure an Aurora PostgreSQL cluster correctly**.

---

## 1. What are we deploying?

An Aurora PostgreSQL environment has two major concepts:

```text
Aurora DB Cluster
│
├── DB Cluster
│    └── Shared Aurora storage
│
├── Writer DB Instance
│
└── Reader DB Instance(s)
```

You don't simply create "a PostgreSQL server."

You create an **Aurora DB cluster** and then associate one or more database instances with it.

---

# 2. Deployment prerequisites

Before creating Aurora PostgreSQL, think about:

* AWS Region
* VPC
* Subnets
* DB subnet group
* Security groups
* Aurora PostgreSQL engine version
* Instance class
* Storage requirements
* Encryption
* Authentication
* Backup retention
* Monitoring
* Parameter groups

A good deployment starts with architecture rather than clicking **Create** immediately.

---

# 3. Choose the AWS Region

Your first major decision is:

> **Where should the Aurora cluster live?**

For example:

```text
Application → us-east-1
Aurora     → us-east-1
```

Keeping the application and database geographically close generally reduces network latency.

Consider:

```text
Latency
Data residency
Compliance
Availability
DR strategy
AWS service availability
Cost
```

Don't select a Region purely because it is geographically closest.

---

# 4. VPC

Aurora PostgreSQL is deployed into an AWS **VPC**.

Conceptually:

```text
AWS Region
    │
    └── VPC
         │
         ├── Subnet AZ-A
         ├── Subnet AZ-B
         └── Subnet AZ-C
                 │
            Aurora Cluster
```

The VPC provides the network boundary around your database environment.

---

# 5. DB subnet group

A **DB subnet group** tells AWS which subnets can be used for database resources.

Conceptually:

```text
DB Subnet Group
│
├── Private Subnet — AZ-A
├── Private Subnet — AZ-B
└── Private Subnet — AZ-C
```

For production systems, use appropriate private subnets and distribute them across Availability Zones.

---

# 6. Why multiple Availability Zones?

Suppose your database environment uses:

```text
AZ-A
AZ-B
AZ-C
```

If infrastructure in one AZ experiences a failure, the architecture has additional locations available for resilience.

Remember from Lesson 2:

> Aurora's storage architecture is distributed across Availability Zones.

Availability Zone planning is therefore part of your HA design.

---

# 7. Security groups

A security group controls network traffic reaching the Aurora instances.

For example:

```text
Application Security Group
          │
          │ PostgreSQL traffic
          ↓
Aurora Security Group
```

You might permit:

```text
Source: Application SG
Port:   PostgreSQL
Protocol: TCP
```

Avoid:

```text
0.0.0.0/0
```

for database access unless there is a very specific, controlled reason.

---

# 8. Aurora PostgreSQL port

The standard PostgreSQL port is:

```text
5432
```

So a typical rule might conceptually be:

```text
Application
    │
 TCP 5432
    ↓
Aurora
```

The application should be allowed to connect through the security-group relationship.

---

# 9. Choose the Aurora PostgreSQL engine version

Aurora PostgreSQL supports specific PostgreSQL-compatible engine versions.

Your decision should consider:

```text
Application compatibility
PostgreSQL features
Extensions
AWS support lifecycle
Migration requirements
Performance
Security
```

Don't automatically choose the newest version without testing your application.

---

# 10. DB cluster vs DB instance settings

This is extremely important.

Some configuration belongs to the:

### Cluster

```text
Aurora DB Cluster
```

Other configuration belongs to:

### DB instance

```text
Writer
Reader
```

Think:

```text
Cluster-level
    ↓
Shared cluster behavior


Instance-level
    ↓
Compute-specific behavior
```

We'll explore parameter groups shortly.

---

# 11. Instance class

Aurora database instances require a compute class.

The instance class determines resources such as:

```text
CPU
Memory
Network capacity
```

Conceptually:

```text
Small workload
    ↓
Smaller instance

Large workload
    ↓
Larger instance
```

Do not size solely by database storage.

Remember:

> **Aurora storage and compute are separate.**

A small database can still require significant CPU or memory.

---

# 12. Initial writer

When you create a new Aurora cluster, you create the initial DB instance.

That instance becomes the:

```text
Writer
```

Conceptually:

```text
New Aurora Cluster
       │
       ↓
Writer Instance
       │
       ↓
Aurora Storage
```

Additional readers can be added later.

---

# 13. Adding readers

You can add additional Aurora PostgreSQL DB instances to the cluster.

For example:

```text
Aurora Cluster
│
├── Writer
├── Reader 1
└── Reader 2
```

Readers can provide:

* Read capacity
* Additional availability options
* Failover targets

The application can use the reader endpoint for eligible read workloads.

---

# 14. Parameter groups

Aurora PostgreSQL uses parameter groups to manage database configuration.

There are two important concepts:

### DB cluster parameter group

Controls parameters associated with the Aurora cluster.

### DB parameter group

Controls parameters associated with individual DB instances.

Conceptually:

```text
Aurora Cluster
│
├── Cluster Parameter Group
│
├── Writer
│    └── DB Parameter Group
│
└── Reader
     └── DB Parameter Group
```

This distinction is frequently important during configuration and troubleshooting.

---

# 15. Don't change parameters blindly

PostgreSQL has many configuration parameters.

For example:

```text
work_mem
shared_buffers
max_connections
log settings
planner settings
```

But Aurora is a managed platform.

You should not assume:

> "I used this parameter on my self-managed PostgreSQL server, therefore I should set it identically on Aurora."

Some parameters may be:

* Managed by AWS
* Restricted
* Cluster-level
* Instance-level
* Dynamic
* Static

Always understand the parameter's scope and restart requirements before changing it.

---

# 16. Encryption

Aurora PostgreSQL supports encryption at rest using AWS KMS.

Conceptually:

```text
Application
     ↓
Encrypted connection
     ↓
Aurora
     ↓
Encrypted storage
```

Encryption protects stored database data and related storage infrastructure.

You should decide encryption requirements **before production deployment**.

---

# 17. Encryption in transit

For network communication, use TLS/SSL where required.

Conceptually:

```text
Application
      │
      │ TLS
      ↓
Aurora PostgreSQL
```

Encryption at rest and encryption in transit solve different problems.

```text
At rest
   ↓
Protect stored data

In transit
   ↓
Protect network communication
```

---

# 18. Authentication

Aurora PostgreSQL can support database authentication mechanisms such as:

```text
PostgreSQL username/password
IAM database authentication
```

A production architecture should consider:

* Credential storage
* Rotation
* Application identity
* Least privilege
* Secret exposure

Avoid hard-coding database passwords inside application source code.

---

# 19. Secrets Manager

AWS Secrets Manager can be used to securely manage database credentials.

Instead of:

```text
Application code
    ↓
hard-coded password
```

use:

```text
Application
    ↓
Secrets Manager
    ↓
Database credentials
    ↓
Aurora
```

This makes credential rotation and secret management easier.

---

# 20. Backup configuration

Aurora provides automated backups.

During deployment, consider:

```text
Backup retention
Backup window
Maintenance window
Recovery requirements
```

Don't think:

> "Aurora is highly available, therefore I don't need backups."

HA and backup serve different purposes.

### HA

Protects against infrastructure/instance failures.

### Backup

Protects against situations such as:

```text
Accidental deletion
Logical corruption
Bad application changes
Point-in-time recovery requirements
```

---

# 21. Maintenance window

Aurora has maintenance mechanisms for managed database infrastructure.

You should understand:

```text
Preferred maintenance window
Preferred backup window
Potential restart implications
Application impact
```

For production environments, coordinate maintenance with the application's operating schedule.

---

# 22. Monitoring configuration

At deployment time, think about:

```text
CloudWatch
Enhanced Monitoring
Performance Insights
Database logs
Alerts
```

Monitoring should not be added only after the database becomes slow.

Production databases should be observable from day one.

---

# 23. Deployment flow

A simplified deployment process looks like:

```text
1. Choose Region
       ↓
2. Select/create VPC
       ↓
3. Create DB subnet group
       ↓
4. Configure security groups
       ↓
5. Choose Aurora PostgreSQL version
       ↓
6. Choose instance class
       ↓
7. Configure authentication
       ↓
8. Enable encryption
       ↓
9. Configure backups
       ↓
10. Configure monitoring
       ↓
11. Create cluster
       ↓
12. Create writer
       ↓
13. Add readers if required
       ↓
14. Test connectivity
```

---

# 24. Production deployment example

Imagine an application with:

```text
3 Availability Zones
1 Writer
2 Readers
Private networking
KMS encryption
Secrets Manager
CloudWatch
Performance monitoring
Automated backups
```

Architecture:

```text
                 Application
                     │
              Private Network
                     │
             ┌───────┴───────┐
             ↓               ↓
       Cluster Endpoint   Reader Endpoint
             ↓               ↓
          Writer        ┌─────┴─────┐
                        ↓           ↓
                     Reader 1    Reader 2
             └──────────┬───────────┘
                        ↓
               Aurora Storage
```

This is a much stronger starting point than simply creating a database with default settings.

---

# 25. Common deployment mistakes

### ❌ Mistake 1 — Publicly exposing the database

Avoid unnecessary internet exposure.

### ❌ Mistake 2 — Using broad security-group rules

Don't allow unrestricted database access.

### ❌ Mistake 3 — Choosing compute based only on storage size

A 100 GB database can still have a heavy workload.

### ❌ Mistake 4 — Ignoring parameter-group scope

Cluster parameters and instance parameters are different.

### ❌ Mistake 5 — No monitoring

You can't troubleshoot what you can't observe.

### ❌ Mistake 6 — No restore testing

Backups without restore testing are an unverified assumption.

### ❌ Mistake 7 — Hard-coded credentials

Use appropriate secret-management mechanisms.

---

# 🧠 Interview Questions

### Q1. What is a DB subnet group?

A collection of subnets that can be used by database resources in a VPC, normally spanning multiple Availability Zones.

### Q2. What is the difference between a cluster parameter group and DB parameter group?

A cluster parameter group controls parameters associated with the Aurora cluster, while a DB parameter group applies instance-level database configuration.

### Q3. Why shouldn't you size Aurora based only on database size?

Because Aurora separates storage from compute. CPU, memory, connections, query complexity, concurrency, and I/O requirements determine compute needs independently of storage capacity.

### Q4. Why use a security group instead of exposing port 5432 publicly?

A security group allows you to restrict database connectivity to known application sources, reducing the attack surface.

---

# 🎯 Lesson 3 Summary

Remember this deployment model:

```text
                 AWS REGION
                     │
                    VPC
                     │
              DB Subnet Group
                     │
             Aurora DB Cluster
                     │
        ┌────────────┴────────────┐
        ↓                         ↓
     Writer                    Readers
        │                         │
        └──────────┬──────────────┘
                   ↓
          Aurora Distributed
                Storage
```

And configure:

```text
Networking
     ↓
Security
     ↓
Engine version
     ↓
Compute
     ↓
Parameter groups
     ↓
Encryption
     ↓
Authentication
     ↓
Backups
     ↓
Monitoring
```

**Next → Lesson 4: Aurora PostgreSQL Networking & Security**
