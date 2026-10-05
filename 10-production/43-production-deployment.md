# Lesson 43 — PostgreSQL Production Deployment

## Goal of This Lesson

This lesson connects the PostgreSQL concepts you have learned into a practical production deployment strategy.

A production database deployment is not simply:

~~~text
Install PostgreSQL
      ↓
Run application
~~~

A reliable deployment considers:

~~~text
Infrastructure
Configuration
Security
Secrets
Schema migrations
Connection pooling
Backups / PITR
Monitoring
High availability
Application deployment
Rollback / recovery
~~~

---

# 1. Development vs Production

Local development may look like:

~~~text
Laptop
  ├─ Next.js / Node.js
  └─ PostgreSQL localhost
~~~

Production usually separates responsibilities:

~~~text
Users
  ↓ HTTPS
Application platform
  ↓ secure DB connection
PostgreSQL infrastructure
  ↓
Backups / monitoring / replicas
~~~

Production must survive failures, restarts, deployments, traffic spikes, and human mistakes.

---

# 2. Two Main Deployment Models

PostgreSQL can broadly be deployed as:

~~~text
Self-managed PostgreSQL
or
Managed PostgreSQL service
~~~

Both run PostgreSQL, but operational responsibility differs.

---

# 3. Self-Managed PostgreSQL

You operate PostgreSQL on infrastructure such as:

~~~text
VM
bare-metal server
container platform
Kubernetes
cloud compute instance
~~~

You are responsible for areas such as:

~~~text
installation
upgrades
configuration
storage
security
TLS
backups
PITR
replication
HA
monitoring
failover
OS maintenance
~~~

This provides control but requires significant operational expertise.

---

# 4. Managed PostgreSQL

A managed provider handles some infrastructure/database operations for you.

Depending on provider and plan, features may include:

~~~text
automatic backups
PITR
TLS
monitoring
replicas
high availability
storage management
maintenance
upgrades
~~~

You still remain responsible for:

~~~text
schema design
queries
indexes
application security
connection usage
migrations
data access
cost/capacity decisions
recovery requirements
~~~

Managed does not mean maintenance-free.

---

# 5. Which Should a Full-Stack Developer Choose?

For most portfolio projects, startups, and small teams:

~~~text
Managed PostgreSQL
~~~

is often the practical starting point.

Why:

~~~text
less operational overhead
built-in backup options
simpler deployment
easier TLS/network setup
managed upgrades/monitoring options
~~~

Self-managed PostgreSQL becomes appropriate when requirements, expertise, control, cost, or infrastructure constraints justify it.

---

# 6. CareerLoop Deployment Architecture

A practical initial architecture for CareerLoop:

~~~text
Browser
   ↓ HTTPS
Next.js on Vercel
   ↓
pooled / managed PostgreSQL endpoint
   ↓
Managed PostgreSQL
   ├─ backups / PITR according to plan
   └─ provider monitoring
~~~

This is simpler than operating Jenkins + Docker + PostgreSQL infrastructure immediately.

You can add infrastructure complexity later when there is a real requirement.

---

# 7. ShopHub Production Architecture

A larger application could evolve toward:

~~~text
Users
  ↓
CDN / application platform
  ↓
Next.js / Node.js instances
  ↓
Connection pool / pooled endpoint
  ↓
PostgreSQL Primary
  ├─→ Read replica(s) if required
  └─→ Backups + PITR
         ↓
Monitoring / alerts
~~~

Architecture should evolve with workload rather than beginning with maximum complexity.

---

# 8. Production Database Creation

Do not treat the default superuser as your application identity.

Conceptually create separate roles:

~~~text
postgres/admin
→ administration only

migration role
→ schema changes

application role
→ runtime CRUD only
~~~

This follows the least-privilege model from Lesson 35.

---

# 9. Runtime Application Role

Example concept:

~~~sql
CREATE ROLE shophub_app
LOGIN
PASSWORD 'use-a-secret-manager-generated-password';
~~~

Then grant only required privileges.

Do not copy a literal password like this into source code or Git.

Managed services may provision roles differently.

---

# 10. Migration Role

Schema migrations may require permissions such as:

~~~text
CREATE TABLE
ALTER TABLE
CREATE INDEX
CREATE TYPE
~~~

Your normal runtime application usually should not need all of these.

Architecture:

~~~text
CI/CD migration job
      ↓
Migration Role
      ↓
schema changes

Application
      ↓
Runtime Role
      ↓
normal queries
~~~

This reduces the damage possible from a compromised runtime credential.

---

# 11. DATABASE_URL

A common application connection string looks conceptually like:

~~~text
postgresql://USER:PASSWORD@HOST:PORT/DATABASE
~~~

Store it as a secret environment variable:

~~~text
DATABASE_URL
~~~

Never commit production credentials to Git.

---

# 12. Server-Only Environment Variables

For Next.js:

~~~text
DATABASE_URL
→ server only
~~~

Never:

~~~text
NEXT_PUBLIC_DATABASE_URL
~~~

`NEXT_PUBLIC_*` values are intended for browser exposure.

Database credentials must remain server-side.

---

# 13. Secret Management

Production secrets may be stored in:

~~~text
deployment platform secret manager
cloud secret manager
CI/CD secret store
Kubernetes Secret with appropriate protections
~~~

Do not store them in:

~~~text
Git repository
Docker image
frontend JavaScript
README
public CI logs
~~~

Rotate credentials if exposure is suspected.

---

# 14. TLS

Application-to-database traffic should use TLS when required by the deployment/network threat model and provider.

Conceptually:

~~~text
Application
   ↓ encrypted PostgreSQL connection
PostgreSQL
~~~

Do not blindly disable certificate verification just to make a connection error disappear.

Follow your PostgreSQL/provider certificate instructions.

---

# 15. Network Exposure

Prefer:

~~~text
Application
   ↓ private/restricted network
PostgreSQL
~~~

over:

~~~text
Entire Internet
      ↓
public PostgreSQL port
~~~

If public access is necessary, restrict it appropriately using provider/firewall/network controls and strong authentication/TLS.

The database should not be more exposed than the architecture requires.

---

# 16. Connection Pooling

From Lesson 38:

~~~text
HTTP requests
    ↓
Application instances
    ↓
connection pools / pooled endpoint
    ↓
controlled PostgreSQL connections
~~~

Do not create a new physical database connection for every HTTP request.

---

# 17. Node.js Pool

Typical `pg` configuration:

~~~js
import { Pool } from "pg";

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
});
~~~

The correct `max` depends on:

~~~text
PostgreSQL connection budget
number of application instances
other database clients
workload
provider limits
~~~

Do not copy `max: 10` blindly into every architecture.

---

# 18. Multiple Instances Multiply Connections

If:

~~~text
10 application instances
×
10 pool connections
=
up to 100 application-side DB connections
~~~

Then also account for:

~~~text
admin sessions
workers
migration jobs
monitoring
replication
provider processes/reserved connections
~~~

Pool sizing is a system-level calculation.

---

# 19. Serverless Connection Bursts

Serverless/auto-scaling environments can create many application instances rapidly.

~~~text
Traffic spike
    ↓
many function instances
    ↓
many connection pools/connections
    ↓
PostgreSQL connection pressure
~~~

Use provider-recommended pooling/connection mechanisms for serverless workloads.

A managed pooled endpoint or PgBouncer-style layer may be appropriate.

---

# 20. Schema Migrations

Production schema changes should be version-controlled.

Typical tools:

~~~text
Drizzle migrations
Prisma Migrate
node-pg-migrate
Flyway
Liquibase
custom SQL migrations
~~~

Core principle:

~~~text
Schema change
    ↓
migration file in Git
    ↓
review
    ↓
CI/CD migration step
    ↓
production database
~~~

Do not manually modify production schema with undocumented ad-hoc SQL unless handling an incident under a controlled process.

---

# 21. Migration History

Example:

~~~text
001_create_users.sql
002_create_jobs.sql
003_add_status_to_jobs.sql
004_create_application_index.sql
~~~

Migrations create a reproducible history of database evolution.

This lets development, staging, and production reach compatible schemas predictably.

---

# 22. Application Code and Schema Must Be Compatible

Deployment problem:

~~~text
New application deployed
      ↓
expects new column
      ↓
migration has not run
      ↓
application crashes
~~~

Or the reverse:

~~~text
migration removes old column
      ↓
old application instances still running
      ↓
old code crashes
~~~

Production deployments must coordinate code and schema changes.

---

# 23. Expand-and-Contract Migration

For risky schema changes, use backward-compatible stages.

Example: rename `name` to `full_name`.

Unsafe idea:

~~~text
DROP name
ADD full_name
deploy new code
~~~

Safer conceptual approach:

~~~text
1. EXPAND
   add full_name while old name still exists

2. DEPLOY COMPATIBLE CODE
   code works during transition

3. BACKFILL
   copy required historical data

4. SWITCH
   use new column

5. CONTRACT
   remove old column after old code is gone
~~~

This reduces downtime and version incompatibility.

---

# 24. Adding a Column

A simple additive migration is often easier to deploy safely:

~~~sql
ALTER TABLE users
ADD COLUMN display_name TEXT;
~~~

Then application code can begin using it after the schema exists.

But large-table operations and defaults/constraints can still have performance/locking implications.

Always understand the actual PostgreSQL version and operation behavior.

---

# 25. Adding NOT NULL Safely — Conceptual

On a large populated table, immediately introducing a constraint plus data transformation may be risky.

A staged pattern can be:

~~~text
add nullable column
      ↓
deploy code writing new values
      ↓
backfill existing rows in controlled batches
      ↓
validate data
      ↓
add/enforce constraint
~~~

The exact safest method depends on the schema and PostgreSQL version.

---

# 26. Creating Indexes in Production

Normal:

~~~sql
CREATE INDEX idx_orders_user_id
ON orders(user_id);
~~~

can block writes while building depending on operation/workload.

For a busy production table, PostgreSQL provides:

~~~sql
CREATE INDEX CONCURRENTLY idx_orders_user_id
ON orders(user_id);
~~~

This reduces blocking of normal writes, but has important restrictions and generally takes more work/time.

---

# 27. CREATE INDEX CONCURRENTLY Caveats

Important points:

~~~text
cannot run inside a normal transaction block
can take longer
can consume significant resources
failure can leave an invalid index that must be handled
still requires operational monitoring
~~~

Do not add `CONCURRENTLY` everywhere without understanding its behavior.

---

# 28. Avoid Long Blocking DDL

A migration may wait for a lock while application traffic continues accumulating behind it.

Example:

~~~text
migration wants strong table lock
      ↓
long transaction already holds conflicting lock
      ↓
migration waits
      ↓
new application queries queue behind migration
      ↓
incident
~~~

Use timeouts, monitoring, staged changes, and maintenance planning for risky DDL.

---

# 29. lock_timeout for Migrations

A migration can set a limited lock wait:

~~~sql
SET lock_timeout = '5s';
~~~

Then a migration can fail rather than waiting indefinitely for a conflicting lock.

This is often safer than unexpectedly freezing production traffic.

Choose thresholds deliberately.

---

# 30. statement_timeout for Migration Sessions

Some migration operations should have bounded execution time.

Example concept:

~~~sql
SET statement_timeout = '10min';
~~~

But long legitimate operations may need larger values.

Use migration-specific session settings rather than globally changing production behavior without reason.

---

# 31. Backups Before Risky Changes

Before a high-risk migration:

~~~text
verify recent backup/PITR
      ↓
verify recovery procedure
      ↓
run migration
~~~

But remember:

> A backup is not a substitute for a safe migration.

Recovery can take significant time.

---

# 32. Staging Environment

Before production:

~~~text
Migration
   ↓
Staging database
   ↓
application tests
   ↓
performance/lock observations
   ↓
Production
~~~

Staging should resemble production enough to reveal meaningful problems.

A migration that takes 100 ms on 100 local rows may take much longer on millions of production rows.

---

# 33. Production-Like Data Volume

You usually do not need real sensitive production data in staging.

But performance testing should use realistic:

~~~text
row counts
data distribution
index sizes
query patterns
concurrency
~~~

Use sanitized/synthetic data where appropriate.

---

# 34. Deployment Order

A common safe deployment sequence:

~~~text
1. Validate backups / database health
2. Apply backward-compatible schema migration
3. Deploy application code
4. Run smoke tests
5. Monitor errors/latency
6. Backfill if required
7. Later remove deprecated schema
~~~

Not every deployment needs every step, but compatibility should be deliberate.

---

# 35. CI/CD Pipeline

Conceptual pipeline:

~~~text
Developer push
    ↓
Pull Request
    ↓
Tests / lint / build
    ↓
Merge
    ↓
Deploy pipeline
    ├─ database migration
    ├─ application deployment
    ├─ smoke tests
    └─ monitoring
~~~

Database migration is part of application delivery, not an unrelated manual activity.

---

# 36. Jenkins Example — Conceptual

Your later self-hosted flow might be:

~~~text
GitHub
  ↓
Jenkins
  ├─ install dependencies
  ├─ test
  ├─ build Docker image
  ├─ run DB migrations
  ├─ deploy application
  └─ health/smoke checks
       ↓
PostgreSQL managed/self-hosted infrastructure
~~~

Be careful with ordering and rollback because database changes are stateful.

---

# 37. Vercel Example

For CareerLoop:

~~~text
GitHub push/merge
      ↓
Vercel build/deploy
      ↓
Next.js application
      ↓
Managed PostgreSQL
~~~

Schema migrations still need a deliberate production mechanism.

Do not assume application deployment automatically makes every database migration safe.

---

# 38. Migration Concurrency

Problem:

~~~text
Instance A starts migration
Instance B starts same migration
~~~

Your migration tooling/deployment pipeline should prevent unsafe concurrent migration execution.

Common approach:

~~~text
one dedicated migration job
      ↓
then deploy/activate application instances
~~~

Use the migration tool's supported locking/history mechanism where applicable.

---

# 39. ORM Migrations vs db push

Development convenience commands that synchronize schema directly may not provide the same controlled history as versioned migrations.

For production, prefer:

~~~text
reviewable migration files
version history
repeatable CI/CD process
~~~

Use the specific production workflow recommended by your ORM/tool.

---

# 40. Seed Data

Separate:

~~~text
schema migration
from
application seed/demo data
~~~

Production seeds should be idempotent or tightly controlled.

Never let a deployment accidentally recreate demo users/products or overwrite real production data.

---

# 41. Configuration

From Lesson 37, important PostgreSQL configuration areas include:

~~~text
memory
connections
WAL/checkpoints
autovacuum
timeouts
logging
planner/statistics
~~~

For managed PostgreSQL, some parameters are controlled or limited by the provider.

Start from provider/PostgreSQL defaults and tune based on measured workload.

---

# 42. Security Checklist

Before launch:

~~~text
✓ application does not use superuser
✓ least-privilege runtime role
✓ production secrets outside Git
✓ TLS/network protection configured
✓ database access restricted
✓ parameterized queries
✓ authorization enforced
✓ sensitive logs controlled
✓ credentials rotatable
~~~

Security is part of deployment, not a later feature.

---

# 43. Backup Checklist

Verify:

~~~text
✓ automated backups enabled
✓ retention understood
✓ PITR window understood
✓ backup failures monitored
✓ restore procedure documented
✓ restore tested
~~~

Do not discover your provider's backup limitations during an incident.

---

# 44. Monitoring Checklist

Monitor:

~~~text
✓ availability
✓ connections
✓ query latency
✓ expensive queries
✓ lock waits
✓ long transactions
✓ CPU/memory/storage
✓ autovacuum
✓ WAL
✓ replication if used
✓ backups
~~~

Set alerts before users become your monitoring system.

---

# 45. High Availability Checklist

If HA is required:

~~~text
✓ standby exists
✓ replication health monitored
✓ failover mechanism defined
✓ stable endpoint/routing
✓ split-brain/fencing strategy
✓ RTO/RPO understood
✓ application reconnect behavior tested
✓ failover tested
✓ redundancy restored after failover
~~~

Do not advertise HA because a replica exists.

---

# 46. Health Checks

Application health should distinguish where useful:

~~~text
process alive
application ready
database reachable
critical dependency healthy
~~~

A health endpoint should not run an expensive database query.

Example simple DB check:

~~~sql
SELECT 1;
~~~

But production readiness logic should reflect the deployment platform and failure policy.

---

# 47. Application Startup

Do not require every application instance to perform schema migrations automatically during startup.

Why?

~~~text
10 instances start
      ↓
10 instances attempt migration
      ↓
race / lock / deployment complexity
~~~

Prefer a dedicated controlled migration phase/job for production.

---

# 48. Graceful Shutdown

When an application instance is terminated:

~~~text
stop accepting new requests
      ↓
finish/cancel in-flight work appropriately
      ↓
close database pool
      ↓
process exits
~~~

For Node `pg`:

~~~js
await pool.end();
~~~

This helps avoid abruptly abandoning work during controlled shutdowns.

---

# 49. Deployment During Active Transactions

Do not kill application processes instantly during deployments if avoidable.

Use graceful deployment mechanisms so active requests/transactions can finish within a bounded period.

Otherwise users may see:

~~~text
connection reset
transaction rollback
ambiguous operation result
~~~

Idempotency remains important for retryable business operations.

---

# 50. Rollback Is Different for Code and Database

Application rollback:

~~~text
deploy previous image/version
~~~

Database rollback is harder because:

~~~text
new data may already use new schema
migration may be destructive
old code may not understand new data
~~~

Therefore prefer backward-compatible migrations and **roll-forward** fixes where practical.

---

# 51. Dangerous Destructive Migration

Example:

~~~sql
DROP COLUMN legacy_status;
~~~

If old application instances still use it, rollback becomes impossible without restoring/recreating data.

Safer:

~~~text
stop writing old field
      ↓
observe
      ↓
remove old code dependency
      ↓
later drop column
~~~

Separate destructive cleanup from the initial feature deployment.

---

# 52. Data Backfills

Large backfills should not necessarily run as one huge transaction.

Bad:

~~~text
UPDATE 100 million rows in one transaction
~~~

Potential consequences:

~~~text
large WAL volume
long locks/transaction
replication lag
dead tuples
rollback cost
I/O pressure
~~~

Prefer controlled batches when appropriate.

---

# 53. Batched Backfill — Conceptual

~~~text
Batch 1: rows 1–10,000
commit
Batch 2: next rows
commit
...
~~~

Monitor:

~~~text
database load
WAL rate
replication lag
lock contention
API latency
~~~

Pause/throttle if production health degrades.

---

# 54. Production Index Strategy

Before launch, index actual access patterns:

~~~text
WHERE filters
JOIN keys
ORDER BY + pagination
foreign-key lookup paths
unique business constraints
~~~

Do not index every column.

Use:

~~~text
pg_stat_statements
EXPLAIN ANALYZE
real workload
~~~

to guide optimization.

---

# 55. Query Safety

Production SQL should use parameterization:

~~~js
await pool.query(
  "SELECT id, email FROM users WHERE email = $1",
  [email]
 );
~~~

Never construct SQL syntax from untrusted values.

Dynamic identifiers require allowlisting.

This is the deployment-level application of Lesson 36.

---

# 56. Timeouts

Production applications should not wait forever.

Relevant layers may include:

~~~text
HTTP request timeout
pool acquisition timeout
database connection timeout
statement_timeout
lock_timeout
idle_in_transaction_session_timeout
~~~

Timeouts should align across layers.

A database timeout longer than the user's HTTP request timeout may waste work after the client has already given up.

---

# 57. Capacity Planning

Watch trends:

~~~text
database size
connections
query throughput
CPU
memory
disk I/O
WAL generation
replica lag
backup duration
~~~

Ask:

~~~text
At current growth rate, when will we hit the next limit?
~~~

Production operations should be proactive, not only reactive.

---

# 58. Vertical Scaling

Vertical scaling means giving one database server more resources:

~~~text
more CPU
more RAM
faster/larger storage
~~~

This is often the simplest scaling step.

PostgreSQL can handle substantial workloads on a well-sized single primary.

---

# 59. Read Scaling

When reads become the bottleneck:

~~~text
Primary
   ├─→ Read Replica A
   └─→ Read Replica B
~~~

Remember:

~~~text
replica lag
read-after-write consistency
routing complexity
~~~

Do not add replicas if query/index optimization solves the problem more simply.

---

# 60. Write Scaling

Scaling PostgreSQL writes is more difficult than adding read replicas.

Before complex distributed architecture, optimize:

~~~text
queries
indexes
transactions
connection count
batching
schema
hardware
contention
~~~

Sharding/distributed databases are advanced solutions for requirements that justify their complexity.

---

# 61. Docker and PostgreSQL Production

You can run PostgreSQL in Docker, but:

~~~text
Docker container
≠
production database architecture
~~~

You still need:

~~~text
persistent storage
backups
resource limits
monitoring
security
upgrades
HA if required
recovery
~~~

For small teams, managed PostgreSQL is often simpler than operating a production DB container yourself.

---

# 62. Kubernetes and PostgreSQL

Running stateful PostgreSQL on Kubernetes requires more than a Deployment and PVC.

Production concerns include:

~~~text
stateful identity
persistent storage
replication
failover
backups
operator lifecycle
upgrades
scheduling/failure domains
~~~

PostgreSQL operators can automate much of this, but they add operational complexity.

Do not run PostgreSQL on Kubernetes merely to say you use Kubernetes.

---

# 63. Application Containers vs Database

A common practical architecture:

~~~text
Docker/Kubernetes
→ application services

Managed PostgreSQL
→ database
~~~

This lets the team learn/use container orchestration without taking on unnecessary database operations.

---

# 64. Disaster Recovery

Production deployment must answer:

~~~text
What if primary fails?
What if region fails?
What if data is accidentally deleted?
What if a migration corrupts data?
What if credentials are compromised?
~~~

Solutions can include:

~~~text
HA
replication
backups
PITR
cross-region recovery
credential rotation
incident runbooks
~~~

No single mechanism solves every failure.

---

# 65. Production Deployment Flow

~~~text
Developer
   ↓
Git commit / Pull Request
   ↓
CI tests
   ↓
Build artifact/image
   ↓
Pre-deployment checks
   ├─ DB health
   ├─ backup/recovery status
   └─ migration review
   ↓
Run safe migration
   ↓
Deploy application
   ↓
Health / smoke tests
   ↓
Monitor DB + app metrics
   ↓
Complete deployment
~~~

This is the flow you should remember for interviews.

---

# 66. Failed Deployment Flow

~~~text
Deployment
   ↓
error rate / latency rises
   ↓
stop rollout
   ↓
identify code vs DB issue
   ↓
rollback compatible application change
or
roll forward database fix
   ↓
validate data and service
   ↓
monitor recovery
~~~

Database changes make rollback planning especially important.

---

# 67. Smoke Tests

After deployment, test critical flows.

For ShopHub:

~~~text
login
product listing
cart
order creation
inventory update
payment callback processing
admin query
~~~

For CareerLoop:

~~~text
signup/login
create application
update status
list applications
filter/search
~~~

Smoke tests should catch obvious integration/database failures quickly.

---

# 68. Observe Immediately After Deployment

Watch:

~~~text
HTTP error rate
database error rate
connection count
query latency
lock waits
CPU/I/O
WAL rate
replication lag
~~~

A migration can succeed syntactically yet still cause production performance problems.

---

# 69. Release Gradually Where Possible

For large systems, deployment techniques may include:

~~~text
rolling deployment
canary release
blue-green deployment
feature flags
~~~

Database schema must remain compatible with whichever application versions may run simultaneously.

---

# 70. Blue-Green and Database Reality

Application environments can be duplicated:

~~~text
Blue application
Green application
~~~

But both may share one stateful production database.

Therefore:

> **Blue-green application deployment does not automatically make database schema changes reversible.**

Backward-compatible database changes are still important.

---

# 71. Feature Flags

Feature flags can separate deployment from feature activation.

~~~text
Deploy schema
   ↓
Deploy compatible code
   ↓
Feature remains OFF
   ↓
validate production
   ↓
turn feature ON
~~~

This can reduce deployment risk for complex changes.

---

# 72. Production Documentation

Document:

~~~text
database provider/cluster
connection mechanism
roles
migration procedure
backup/PITR policy
restore procedure
monitoring dashboards
alerts
failover behavior
incident contacts/runbooks
~~~

Documentation is part of operational reliability.

---

# Common Mistakes

## 73. Using the PostgreSQL Superuser From the Application

Use least-privilege runtime credentials.

## 74. Committing DATABASE_URL to Git

Store secrets in deployment/secret-management systems.

## 75. Opening PostgreSQL to the Entire Internet Without Need

Restrict network access.

## 76. Creating a New DB Connection per Request

Use connection pooling/managed pooling.

## 77. Running Schema Migrations From Every App Instance

Use a controlled migration job/stage.

## 78. Deploying Destructive Schema Changes With New Code at the Same Time

Prefer staged expand-and-contract migrations.

## 79. Running Huge Backfills in One Transaction

Batch and monitor when appropriate.

## 80. Assuming Application Rollback Automatically Rolls Back Database State

Database changes/data may not be reversible.

## 81. Running PostgreSQL in Docker Without Backup/Persistence Planning

A container alone does not provide database reliability.

## 82. Assuming Managed PostgreSQL Removes All Responsibility

You still own queries, schema, application access, migrations, capacity, and recovery requirements.

---

# Interview Revision

## Managed vs self-hosted PostgreSQL?

Managed PostgreSQL delegates much of infrastructure, backup, maintenance, and HA operation to a provider. Self-hosted PostgreSQL gives more control but requires the team to operate those capabilities.

## Should an application connect as PostgreSQL superuser?

No. Use a least-privilege runtime role and separate administrative/migration privileges.

## Where should DATABASE_URL be stored?

In a secure server-side secret/environment management system, never committed to Git or exposed to browser code.

## Why use connection pooling?

PostgreSQL connections are finite and relatively expensive; pooling lets many application requests reuse a controlled number of database sessions.

## How should production migrations be managed?

Keep versioned migration files in source control, review/test them, execute them through a controlled deployment step, and design schema changes for compatibility.

## What is expand-and-contract?

A migration strategy where you first add backward-compatible schema, deploy compatible code/backfill/switch usage, then later remove obsolete schema.

## Why CREATE INDEX CONCURRENTLY?

It allows index creation with less blocking of normal table writes on a busy production table, though it has restrictions and additional operational cost.

## Why should migrations have lock awareness?

A DDL statement waiting for a lock can cause application traffic to queue behind it. `lock_timeout`, monitoring, and staged changes can reduce risk.

## Why is DB rollback harder than application rollback?

Schema and data may already have changed, and destructive migrations may not be reversible. Backward-compatible migrations and roll-forward fixes are often safer.

## Should PostgreSQL run inside Kubernetes?

It can, but production operation requires persistent storage, replication, failover, backups, monitoring, upgrades, and stateful orchestration. Managed PostgreSQL is often simpler.

## What should be checked after deployment?

Application/database errors, query latency, connections, locks, CPU/I/O, WAL/replication, critical smoke tests, and data correctness.

---

# Complete Production Architecture

~~~text
                         USERS
                           ↓
                        HTTPS
                           ↓
                 Application Platform
              ┌────────────┴────────────┐
              ↓                         ↓
        App Instance A            App Instance B
              │                         │
              └──────────┬──────────────┘
                         ↓
                Pool / DB Endpoint
                         ↓
                 PostgreSQL Primary
                  │       │       │
                  │       │       └→ Monitoring
                  │       │
                  │       └→ Backup / PITR
                  │
                  └→ Standby / Replica
                      if required
~~~

Supporting controls:

~~~text
Secrets
TLS
Least privilege
Migrations
Timeouts
Alerts
Restore tests
Failover tests
~~~

---

# Production Deployment Checklist

~~~text
DATABASE
✓ correct schema/migrations
✓ constraints
✓ indexes for real access patterns
✓ statistics/maintenance healthy

SECURITY
✓ least-privilege role
✓ secrets protected
✓ TLS/network restrictions
✓ parameterized SQL

CONNECTIONS
✓ pooling configured
✓ total connection budget calculated
✓ serverless pooling strategy if required

RECOVERY
✓ automated backups
✓ PITR if required
✓ retention understood
✓ restore tested

OBSERVABILITY
✓ metrics
✓ slow-query visibility
✓ logs
✓ alerts

HA
✓ only if business requires it
✓ replica/failover/routing tested
✓ RTO/RPO documented

DEPLOYMENT
✓ versioned migrations
✓ backward-compatible changes
✓ controlled migration job
✓ smoke tests
✓ rollback/roll-forward plan
~~~

---

# Quick Revision

~~~text
CODE
 ↓
Git
 ↓
CI/CD
 ├─ tests
 ├─ migration
 └─ deployment
 ↓
Application
 ↓
Connection Pool
 ↓
PostgreSQL
 ├─ security
 ├─ monitoring
 ├─ backups/PITR
 └─ HA/replicas if required
~~~

### Deployment Rule

~~~text
Schema first when additive/backward-compatible
      ↓
Compatible application
      ↓
Backfill / transition
      ↓
Destructive cleanup later
~~~

### Golden Production Principle

~~~text
Do not optimize for complexity.
Optimize for:
Reliability + Security + Recoverability + Observability
~~~

---

## Key Takeaway

> **A production PostgreSQL deployment is a complete operating system around the database, not just a running server. Use least-privilege roles and protected secrets, secure the network/TLS path, control connections with pooling, deploy schema through reviewed backward-compatible migrations, automate backups and test restores, monitor real workload behavior, and add replication/HA only when RPO/RTO requirements justify it. For most full-stack applications, managed PostgreSQL is the simplest reliable starting point; your application still owns schema quality, SQL performance, migration safety, and correct use of the database.**

---

[← Previous: Lesson 42 — Monitoring & Logging](./42-monitoring-and-logging.md) | [Back to Roadmap](../README.md) | [Next: Lesson 44 — Production E-Commerce Database →](../11-production-project/44-production-ecommerce-database.md)