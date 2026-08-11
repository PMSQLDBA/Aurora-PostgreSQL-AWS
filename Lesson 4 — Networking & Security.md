# 🐘 Lesson 4 — Aurora PostgreSQL Networking & Security

Now we're moving from **deployment** into one of the most important production areas: **how applications securely connect to Aurora PostgreSQL**.

---

## 1. The networking mental model

Think about the connection path like this:

```text
Application
    ↓
VPC
    ↓
Subnet / Route
    ↓
Security Group
    ↓
Aurora Endpoint
    ↓
Aurora PostgreSQL
```

Every layer has a responsibility.

A database connection can fail before PostgreSQL even receives the request.

---

# 2. Aurora lives inside a VPC

Aurora PostgreSQL is deployed into an Amazon VPC.

A simplified architecture:

```text
AWS Region
│
└── VPC
    │
    ├── Application Subnet
    │
    ├── Private DB Subnet
    │
    └── Aurora PostgreSQL
```

For production databases, Aurora is normally kept on private network paths rather than exposing the database directly to the public internet.

---

# 3. Private database architecture

A typical production design is:

```text
                    Internet
                       │
                       ↓
                  Load Balancer
                       │
                       ↓
                 Application
                       │
                 Private Network
                       │
                       ↓
              Aurora PostgreSQL
```

The database doesn't need to be directly accessible from the internet.

This significantly reduces the attack surface.

---

# 4. Security Groups

Security groups are one of the most important network-security mechanisms for Aurora.

Think of them as virtual firewalls attached to AWS resources.

For example:

```text
Application Security Group
          │
          │ TCP 5432
          ↓
Aurora Security Group
```

Instead of allowing:

```text
0.0.0.0/0 → 5432
```

you can allow the application security group to connect to the database security group.

---

# 5. Why security-group references are useful

Suppose your application servers change IP addresses.

If you use individual IP rules:

```text
10.10.1.15
10.10.1.16
10.10.1.17
```

you may need to continually update database rules.

With security-group-based access:

```text
Application SG
      ↓
Aurora SG
```

the security relationship follows the application resources.

This is generally cleaner and easier to operate.

---

# 6. PostgreSQL port

The standard PostgreSQL port is:

```text
5432
```

So a typical connection is:

```text
Application
    │
    │ TCP 5432
    ↓
Aurora PostgreSQL
```

But remember:

> Port 5432 being open does not mean PostgreSQL authentication will succeed.

Networking and authentication are separate layers.

---

# 7. Network troubleshooting layers

When an application cannot connect, don't immediately assume the password is wrong.

Troubleshoot in this order:

```text
1. DNS resolution
       ↓
2. Network route
       ↓
3. Security group
       ↓
4. Network ACL / routing controls
       ↓
5. Port reachability
       ↓
6. TLS
       ↓
7. PostgreSQL authentication
       ↓
8. Authorization
```

This saves enormous troubleshooting time.

---

# 8. Aurora endpoints

From Lesson 2, we know Aurora provides endpoints.

The important ones are:

### Cluster endpoint

Usually points to the current writer.

```text
Application
     ↓
Cluster Endpoint
     ↓
Writer
```

### Reader endpoint

Used for read workloads.

```text
Application
     ↓
Reader Endpoint
     ↓
Readers
```

### Instance endpoint

Connects directly to one particular DB instance.

---

# 9. Don't hard-code the writer

Avoid application configuration such as:

```text
writer-instance-1.example...
```

because the writer can change during failover.

Instead:

```text
Application
     ↓
Aurora Cluster Endpoint
     ↓
Current Writer
```

This allows the database infrastructure to change underneath the application.

---

# 10. DNS

Aurora endpoints depend on DNS.

Conceptually:

```text
Application
     ↓
DNS lookup
     ↓
Aurora endpoint
     ↓
Current database target
```

Therefore, DNS problems can appear as database connectivity problems.

Useful troubleshooting questions:

```text
Can the client resolve the hostname?
What IP does it resolve to?
Is the client using the expected DNS resolver?
Can the client reach the resulting address?
```

---

# 11. TLS / SSL

Network security isn't complete simply because the database is private.

You should also consider:

> **Is the database traffic encrypted while traveling across the network?**

Conceptually:

```text
Application
      │
      │ TLS
      ↓
Aurora PostgreSQL
```

TLS helps protect credentials and database traffic from network interception.

---

# 12. Encryption at rest vs in transit

These are different.

### Encryption at rest

Protects stored data.

```text
Aurora Storage
      ↓
Encrypted
```

### Encryption in transit

Protects network traffic.

```text
Application
      ↓
Encrypted connection
      ↓
Aurora
```

Production systems commonly need both.

---

# 13. PostgreSQL SSL connection

A PostgreSQL client can be configured to use SSL/TLS.

For example, connection configuration can specify an SSL mode.

A simplified example:

```text
sslmode=require
```

For stronger certificate validation, applications should use an appropriate certificate-validation mode and trusted CA configuration rather than simply relying on encryption without identity verification.

---

# 14. Authentication vs authorization

This distinction is fundamental.

### Authentication

> Who are you?

Examples:

```text
Username/password
IAM authentication
```

### Authorization

> What are you allowed to do?

Examples:

```text
SELECT
INSERT
UPDATE
DELETE
CREATE
```

So:

```text
Authentication
      ↓
Who?
      ↓
Authorization
      ↓
What can they do?
```

---

# 15. PostgreSQL users and roles

PostgreSQL uses roles to represent identities and permissions.

Conceptually:

```text
Role
 │
 ├── LOGIN
 ├── Password
 ├── Database access
 └── Object privileges
```

Example:

```sql id="b9q7hk"
CREATE ROLE app_user
LOGIN
PASSWORD '...';
```

In production, don't embed real passwords in application source code.

---

# 16. Least privilege

Suppose an application only needs:

```text
SELECT
INSERT
UPDATE
```

It shouldn't automatically receive:

```text
DROP DATABASE
CREATE ROLE
SUPERUSER
```

A safer model is:

```text
Application
     ↓
Limited PostgreSQL role
     ↓
Required tables/functions only
```

This reduces the blast radius if credentials are compromised.

---

# 17. IAM database authentication

Aurora PostgreSQL can integrate with AWS IAM database authentication.

Conceptually:

```text
Application
     ↓
AWS IAM identity
     ↓
Temporary authentication token
     ↓
Aurora PostgreSQL
```

This can reduce reliance on long-lived database passwords.

It is especially useful when applications already use IAM-based AWS identities.

---

# 18. Secrets Manager

Another common architecture uses AWS Secrets Manager.

```text
Application
     ↓
Secrets Manager
     ↓
Database credential
     ↓
Aurora
```

Benefits include:

* Centralized secret storage
* Credential rotation
* Reduced hard-coded secrets
* Better operational control

---

# 19. IAM vs Secrets Manager

They solve related but different problems.

### Secrets Manager

Stores secrets.

```text
Password
Username
Connection information
```

### IAM authentication

Uses IAM identity and temporary authentication mechanisms.

```text
IAM identity
      ↓
Authentication token
      ↓
Database
```

Choose based on your application architecture and operational requirements.

---

# 20. Network isolation

A strong production architecture often looks like:

```text
                 Internet
                    │
             ┌──────┴──────┐
             │ Application │
             └──────┬──────┘
                    │
                Private VPC
                    │
             Security Groups
                    │
                    ↓
             Aurora PostgreSQL
```

The database is not directly exposed to the public internet.

---

# 21. Application-to-database security group

A common pattern:

```text
        Application
       App-SG: sg-app
              │
              │ allows TCP 5432
              ↓
       Database
       DB-SG: sg-db
```

The DB security group can specify:

```text
Inbound:
Source = sg-app
Port = 5432
```

This is preferable to broadly allowing database access from arbitrary networks.

---

# 22. Public accessibility

During deployment, Aurora provides settings related to whether the DB instance can receive public connectivity.

For production environments, carefully evaluate whether public accessibility is actually required.

Usually:

```text
Public application
       ↓
Application tier
       ↓
Private Aurora
```

is safer than:

```text
Internet
       ↓
Aurora
```

---

# 23. Network ACLs

Security groups aren't the only networking control.

AWS VPC networking can also involve:

* Route tables
* Network ACLs
* DNS configuration
* Subnet routing

Therefore, when connectivity fails, don't look only at the Aurora security group.

---

# 24. On-premises connectivity

Suppose your company has:

```text
Corporate Data Center
        │
        ↓
AWS
        │
        ↓
Aurora PostgreSQL
```

Connectivity may use mechanisms such as:

```text
VPN
ExpressRoute-like AWS networking equivalent: Direct Connect
```

AWS Direct Connect provides dedicated network connectivity between on-premises environments and AWS.

The architecture becomes:

```text
On-Prem
   │
Direct Connect / VPN
   │
   ↓
AWS VPC
   │
   ↓
Aurora
```

---

# 25. Security architecture example

A production application might use:

```text
                   Users
                     │
                     ↓
               Application
                     │
                Private VPC
                     │
             Application SG
                     │
               TCP 5432
                     │
                Database SG
                     │
                     ↓
             Aurora PostgreSQL
                     │
              Encrypted Storage
```

Authentication:

```text
Application
    ↓
IAM / Secrets Manager
    ↓
PostgreSQL identity
```

---

# 26. Common security mistakes

### ❌ Allowing 0.0.0.0/0 on port 5432

Huge attack surface.

### ❌ Hard-coding passwords

Secrets can leak through source code, logs, or configuration.

### ❌ Using administrator credentials for applications

Violates least privilege.

### ❌ Assuming private networking means secure by itself

You still need authentication, authorization, encryption, monitoring, and good network controls.

### ❌ Ignoring TLS

Private network traffic can still require encryption.

---

# 27. Troubleshooting example

Suppose the application reports:

```text
connection timed out
```

Don't start by changing the PostgreSQL password.

Check:

```text
DNS
 ↓
Route
 ↓
Security Group
 ↓
Network ACL
 ↓
Port 5432
 ↓
TLS
 ↓
Authentication
```

Now suppose the error is:

```text
password authentication failed
```

Networking probably worked.

Move to:

```text
Username
Password
Authentication method
PostgreSQL role
Secret configuration
```

This distinction is critical.

---

# 🧠 Interview Questions

### Q1. What is the standard PostgreSQL port?

**5432.**

### Q2. Should Aurora PostgreSQL normally be publicly accessible?

For most production architectures, unnecessary public exposure should be avoided. Prefer private networking and tightly controlled application access.

### Q3. What is the difference between authentication and authorization?

**Authentication** verifies identity.
**Authorization** determines what that identity can access or perform.

### Q4. Why use Aurora endpoints?

They prevent applications from depending on a particular database instance and support Aurora's failover and read-routing architecture.

### Q5. What does a security group do?

It controls network traffic allowed to and from AWS resources at the network interface level.

---

# 🎯 Lesson 4 Summary

Remember this architecture:

```text
                 APPLICATION
                      │
                      ↓
                    VPC
                      │
              ┌───────┴───────┐
              ↓               ↓
         DNS / Route      Security Group
              │               │
              └───────┬───────┘
                      ↓
                 Port 5432
                      ↓
              Aurora Endpoint
                      ↓
             Aurora PostgreSQL
                      │
             ┌────────┴────────┐
             ↓                 ↓
        Authentication    Authorization
             │                 │
             └────────┬────────┘
                      ↓
                   Database
```

### Five things to remember

**1.** Aurora PostgreSQL should normally be placed in a controlled VPC network.
**2.** Security groups should restrict database access to trusted sources.
**3.** Use **Aurora endpoints**, not hard-coded instance addresses.
**4.** Protect connections with **TLS** where required.
**5.** Separate **authentication** from **authorization** and apply least privilege.

---

**Next → Lesson 5: PostgreSQL Administration on Aurora**
