# Lesson 46 — PostgreSQL Production Checklist

## Final Lesson

This is the final lesson of the PostgreSQL curriculum.

It is designed as a practical checklist you can use before deploying a PostgreSQL-backed application and as a fast revision sheet before interviews.

Do not treat every item as something every small project must implement. The goal is to know what to evaluate and why.

---

# 1. The Production Mental Model

~~~text
Application
    ↓
Connection management
    ↓
PostgreSQL
    ├─ Schema & constraints
    ├─ Queries & indexes
    ├─ Transactions & concurrency
    ├─ Security
    ├─ MVCC / VACUUM / WAL
    ├─ Backups / recovery
    ├─ Replication / HA
    └─ Monitoring / alerts
~~~

A production database is not only tables and SQL. It is a complete reliability system.

---

# 2. Database Design Checklist

Before production, verify:

~~~text
□ Entities match business requirements
□ Relationships are correctly modeled
□ Many-to-many relationships use junction tables
□ Core data is normalized appropriately
□ Intentional denormalization has a clear reason
□ Historical business data is preserved where required
□ JSONB is used for flexible data, not to avoid relational design
□ Naming conventions are consistent
~~~

Example ShopHub relationships:

~~~text
users 1:N orders
orders 1:N order_items
products 1:N order_items
orders N:M products through order_items
categories 1:N products
~~~

---

# 3. Primary Key Checklist

Every important entity should have a stable identifier.

Common choices:

~~~text
BIGINT GENERATED ... AS IDENTITY
UUID
~~~

Verify:

~~~text
□ Every entity has an appropriate primary key
□ Composite keys are used where the relationship itself is the identity
□ Public identifiers do not replace authorization
□ ID strategy is consistent with application requirements
~~~

---

# 4. Foreign Key Checklist

Verify:

~~~text
□ Relationships use foreign keys where appropriate
□ ON DELETE behavior is intentional
□ CASCADE is not used blindly
□ SET NULL columns are nullable
□ Referencing columns are indexed when workload requires it
~~~

Remember:

> PostgreSQL automatically creates an index for PRIMARY KEY/UNIQUE constraints, but it does not automatically create an index on the referencing side of every foreign key.

---

# 5. Constraint Checklist

Use the database as the final integrity layer.

Verify appropriate use of:

~~~text
□ NOT NULL
□ UNIQUE
□ CHECK
□ PRIMARY KEY
□ FOREIGN KEY
□ DEFAULT
~~~

Examples:

~~~sql
CHECK (price >= 0)
CHECK (quantity > 0)
CHECK (stock >= 0)
~~~

Defense in depth:

~~~text
Frontend validation
      ↓
Server validation
      ↓
Authorization
      ↓
Database constraints
~~~

---

# 6. Data Type Checklist

Verify:

~~~text
□ INTEGER/BIGINT used for integer quantities
□ NUMERIC or integer minor units used for money
□ BOOLEAN used for true/false state
□ DATE used for calendar dates
□ TIMESTAMPTZ used for real-world instants
□ UUID used only when its tradeoffs fit
□ JSONB used intentionally
□ TEXT/VARCHAR chosen intentionally
~~~

Avoid storing numbers, timestamps, and booleans as arbitrary text.

---

# 7. Money Checklist

For financial/e-commerce systems:

~~~text
□ Do not use floating-point types for authoritative money
□ Currency is stored explicitly when needed
□ Server calculates authoritative totals
□ Client-submitted price is never trusted
□ Historical charged values are preserved
□ Payment-provider minor-unit conversion is consistent
~~~

Example:

~~~text
₹499.99
→ 49999 paise
~~~

---

# 8. Timestamp Checklist

Verify:

~~~text
□ created_at exists where useful
□ updated_at strategy is consistent
□ TIMESTAMPTZ is used for real-world instants
□ application converts timestamps for user display timezone
□ expiration/payment/shipping times are modeled explicitly
~~~

`DEFAULT now()` sets an INSERT default; it does not automatically maintain `updated_at` on UPDATE.

---

# 9. SQL Query Checklist

Verify:

~~~text
□ Queries select only needed columns
□ Filtering happens in SQL when appropriate
□ Large result sets are paginated
□ N+1 patterns are avoided
□ JOIN conditions are correct
□ Aggregations use WHERE/HAVING correctly
□ NULL semantics are understood
□ Dynamic SQL identifiers are allowlisted
~~~

Prefer:

~~~sql
SELECT id, name, price
FROM products
WHERE category_id = $1
LIMIT 20;
~~~

over unnecessary `SELECT *`.

---

# 10. Parameterized Query Checklist

Never build SQL from untrusted values using string concatenation.

Safe Node.js example:

~~~js
await pool.query(
  "SELECT id, email FROM users WHERE email = $1",
  [email]
 );
~~~

Verify:

~~~text
□ User input is parameterized
□ Search values are parameterized
□ LIMIT/OFFSET values are validated
□ Dynamic columns/order directions are allowlisted
□ Raw ORM SQL is reviewed for injection risks
~~~

---

# 11. Index Checklist

Index real access patterns.

Look at:

~~~text
□ WHERE filters
□ JOIN keys
□ ORDER BY
□ pagination cursors
□ uniqueness requirements
□ foreign-key lookup paths
□ frequently queried expressions
~~~

Example:

~~~sql
CREATE INDEX orders_user_created_idx
ON orders(user_id, created_at DESC, id DESC);
~~~

Do not index every column.

---

# 12. Composite Index Checklist

Ask:

~~~text
□ What columns use equality filters?
□ What columns use ranges?
□ What ordering is required?
□ Does the query use the leftmost index prefix effectively?
~~~

Column order matters.

Example access pattern:

~~~text
WHERE user_id = ?
ORDER BY created_at DESC, id DESC
~~~

Potential index:

~~~text
(user_id, created_at DESC, id DESC)
~~~

---

# 13. Advanced Index Checklist

Consider only when appropriate:

~~~text
□ Partial index
□ Expression index
□ INCLUDE / covering index
□ GIN
□ GiST
□ BRIN
~~~

Examples:

~~~text
Partial → active/pending subset
Expression → lower(email)
GIN → JSONB/arrays/full-text use cases
BRIN → huge physically correlated tables
~~~

Always verify benefit with the actual workload.

---

# 14. EXPLAIN Checklist

For important/slow queries:

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
~~~

Inspect:

~~~text
□ Seq Scan / Index Scan / Bitmap Scan
□ estimated rows vs actual rows
□ execution time
□ loops
□ sorts
□ join algorithms
□ rows removed by filter
□ buffer hits/reads
~~~

`EXPLAIN ANALYZE` executes the statement, so use extra care with writes.

---

# 15. Query Optimization Checklist

Follow:

~~~text
Slow endpoint
    ↓
Find exact SQL
    ↓
EXPLAIN ANALYZE
    ↓
Find expensive work/wait
    ↓
Make one targeted change
    ↓
Measure again
~~~

Verify:

~~~text
□ No unnecessary columns
□ No unnecessary joins
□ No N+1
□ Pagination is suitable
□ Index matches query
□ Statistics are healthy
□ Query is not simply waiting on a lock
~~~

---

# 16. Pagination Checklist

For small/shallow result sets:

~~~text
LIMIT + OFFSET
~~~

can be sufficient.

For deep/high-volume ordered feeds, consider keyset pagination:

~~~sql
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

Verify:

~~~text
□ Ordering is deterministic
□ Cursor contains tie-breaker
□ Matching composite index exists
~~~

---

# 17. Transaction Checklist

Use transactions when multiple DB operations form one logical unit.

Verify:

~~~text
□ BEGIN/COMMIT/ROLLBACK correctly handled
□ Every transaction query uses the same checked-out client
□ Errors always rollback
□ Client is always released
□ Transactions remain short
□ External APIs are not awaited inside long DB transactions
~~~

Node pattern:

~~~text
pool.connect()
   ↓
BEGIN
   ↓
client.query(...)
   ↓
COMMIT / ROLLBACK
   ↓
client.release()
~~~

---

# 18. Isolation Checklist

Know PostgreSQL's levels:

~~~text
READ COMMITTED ← default
REPEATABLE READ
SERIALIZABLE
~~~

`READ UNCOMMITTED` maps to PostgreSQL's READ COMMITTED behavior.

Remember:

~~~text
READ COMMITTED
→ snapshot per statement

REPEATABLE READ
→ stable transaction snapshot

SERIALIZABLE
→ strongest isolation; transactions may need retry
~~~

Do not increase isolation without understanding concurrency/performance effects.

---

# 19. Concurrency Checklist

Ask:

~~~text
What happens if two requests execute this at the same time?
~~~

Verify critical operations such as:

~~~text
□ inventory decrement
□ account/balance changes
□ booking/reservation
□ job claiming
□ payment processing
□ duplicate submission
~~~

Use as appropriate:

~~~text
atomic UPDATE
SELECT ... FOR UPDATE
unique constraints
idempotency keys
SERIALIZABLE + retry
advisory locks for application-defined resources
~~~

---

# 20. Lock Checklist

Verify:

~~~text
□ Transactions lock rows in consistent order where possible
□ Lock waits are monitored
□ Transactions are short
□ External/user interaction does not hold DB locks
□ Deadlock errors are handled
~~~

Useful tools:

~~~text
pg_locks
pg_stat_activity
pg_blocking_pids(pid)
lock_timeout
~~~

Remember: MVCC does not mean PostgreSQL has no locks.

---

# 21. MVCC Checklist

Remember:

~~~text
UPDATE
→ creates a new tuple version conceptually

DELETE
→ marks a version deleted

old versions
→ remain while snapshots may need them

VACUUM
→ later makes dead space reusable
~~~

Verify:

~~~text
□ No unnecessarily long transactions
□ Idle-in-transaction sessions are controlled
□ Autovacuum can keep up
~~~

MVCC enables strong reader/writer concurrency, but it does not solve every business race condition.

---

# 22. VACUUM Checklist

Verify:

~~~text
□ Autovacuum is enabled
□ High-write tables are monitored
□ Dead tuple growth is understood
□ Long-running transactions are investigated
□ VACUUM FULL is not used as routine maintenance
~~~

Autovacuum is essential for both cleanup and transaction ID safety.

---

# 23. WAL Checklist

Remember:

~~~text
Data change
   ↓
WAL record
   ↓
required WAL durability
   ↓
COMMIT durability
   ↓
data pages can be flushed later
~~~

WAL enables:

~~~text
crash recovery
replication
PITR
~~~

Verify:

~~~text
□ WAL/storage usage monitored
□ Archiving configured if PITR requires it
□ Replication lag monitored where applicable
~~~

---

# 24. Connection Pool Checklist

Never assume more DB connections mean more performance.

Verify:

~~~text
□ Application uses a pool
□ Pool size is intentional
□ Total pools across all instances are calculated
□ PostgreSQL max_connections leaves operational headroom
□ Pool wait/acquisition time is monitored
□ Serverless architecture uses suitable pooling strategy
~~~

Concept:

~~~text
Requests
   ↓
Connection pool
   ↓
Controlled DB concurrency
~~~

---

# 25. PgBouncer Checklist

Consider PgBouncer or a managed pooled endpoint when:

~~~text
many application instances
serverless connection bursts
connection churn
large client count
~~~

Understand pooling modes and application/driver compatibility before deploying it.

PgBouncer does not make slow SQL fast; it manages connection pressure.

---

# 26. Security Checklist

Verify:

~~~text
□ Application does not connect as superuser
□ Runtime role follows least privilege
□ Migration/admin privileges are separated where appropriate
□ DATABASE_URL is secret
□ Production secrets are not in Git
□ Database network access is restricted
□ TLS is configured appropriately
□ Authentication method is secure
□ GRANT/REVOKE permissions are reviewed
~~~

Remember:

~~~text
pg_hba.conf
→ who/how may authenticate

GRANT / REVOKE
→ what authenticated role may do
~~~

---

# 27. Application Security Checklist

Verify:

~~~text
□ Authentication
□ Authorization
□ Server-side validation
□ Parameterized SQL
□ Database constraints
□ Safe error responses
□ Sensitive data excluded from logs
~~~

Knowing a UUID does not authorize access to a record.

Example:

~~~sql
SELECT id, status
FROM orders
WHERE id = $1
  AND user_id = $2;
~~~

---

# 28. Next.js Checklist

For Next.js App Router:

~~~text
□ PostgreSQL code remains server-only
□ DATABASE_URL is not NEXT_PUBLIC_*
□ Server Components can use DAL directly
□ Route Handlers are used when an HTTP API is actually needed
□ Server Actions validate and authorize mutations
□ Dynamic params use current async App Router pattern
□ cookies()/headers() use current async APIs
□ Mutations revalidate affected UI/cache intentionally
~~~

Architecture:

~~~text
Server Component / Route Handler / Server Action
                 ↓
            Data Access Layer
                 ↓
             PostgreSQL
~~~

Client Components must not directly connect to PostgreSQL.

---

# 29. Express/Node.js Checklist

Recommended separation:

~~~text
Route
  ↓
Controller
  ↓
Service
  ↓
DB layer
  ↓
PostgreSQL
~~~

Verify:

~~~text
□ `pg` pool reused
□ input validated
□ SQL parameterized
□ transactions use one client
□ DB errors mapped safely
□ central error handling exists
□ pool shuts down gracefully
~~~

---

# 30. ORM Checklist

If using Drizzle/Prisma/another ORM:

~~~text
□ Understand generated SQL
□ Keep DB constraints
□ Check N+1 behavior
□ Inspect indexes
□ Use transactions correctly
□ Use versioned migrations
□ Use raw SQL where complex PostgreSQL features justify it
~~~

Remember:

> ORM knowledge does not replace SQL knowledge.

---

# 31. Migration Checklist

Production schema changes should be versioned.

Verify:

~~~text
□ Migration files committed to Git
□ Migration tested before production
□ Only one controlled migration process runs
□ Lock impact understood
□ Large-table operations reviewed
□ Backward compatibility considered
□ Destructive cleanup delayed when possible
~~~

Prefer expand-and-contract for risky changes.

---

# 32. Expand-and-Contract Checklist

~~~text
EXPAND
add new compatible schema
   ↓
DEPLOY
code supports transition
   ↓
BACKFILL
migrate historical data
   ↓
SWITCH
use new schema
   ↓
CONTRACT
remove obsolete schema later
~~~

This is one of the most important production migration patterns.

---

# 33. Large Migration Checklist

For large tables, ask:

~~~text
□ Will this require a strong lock?
□ How long can it run?
□ Will it rewrite the table?
□ How much WAL will it generate?
□ Can it create replication lag?
□ Can the backfill be batched?
□ Can index creation use CONCURRENTLY?
□ Is lock_timeout appropriate?
~~~

Never judge production migration safety using only a tiny local database.

---

# 34. Backup Checklist

Verify:

~~~text
□ Automated backups exist
□ Retention period is known
□ Backup jobs are monitored
□ Backups are stored safely
□ Recovery procedure is documented
□ Restore has actually been tested
~~~

Remember:

> **A backup you have never restored is an assumption, not proven recovery.**

---

# 35. PITR Checklist

If the business requires point-in-time recovery:

~~~text
□ Base backup exists
□ WAL archiving is configured
□ WAL retention/archive is healthy
□ Recovery target procedure is known
□ Recovery is tested
~~~

PITR can recover to a point before accidental destructive changes within the available recovery window.

---

# 36. RPO and RTO Checklist

Know:

~~~text
RPO
→ How much data can we afford to lose?

RTO
→ How long can we afford to be unavailable?
~~~

These requirements determine the necessary backup, replication, and HA architecture.

Do not design expensive HA before understanding the business requirements.

---

# 37. Replication Checklist

If replication is used:

~~~text
□ Primary/standby roles understood
□ Streaming replication monitored
□ Replication lag monitored
□ WAL retention prevents standby loss
□ Read replicas account for stale reads
□ Read-after-write behavior is designed
~~~

Replication improves availability/read scaling but does not replace backups.

---

# 38. High Availability Checklist

If HA is required:

~~~text
□ Standby is actually ready
□ Failover mechanism exists
□ Stable endpoint/routing exists
□ Split-brain prevention/fencing is considered
□ Application reconnects correctly
□ Failover has been tested
□ Failed primary does not rejoin unsafely
□ Redundancy is restored after failover
~~~

A replica alone is not a complete HA system.

---

# 39. Monitoring Checklist

Monitor at least the signals relevant to your system:

~~~text
□ availability
□ connections
□ pool waits
□ query latency
□ slow/frequent queries
□ lock waits
□ long transactions
□ deadlocks
□ CPU
□ memory
□ storage usage
□ disk latency/I/O
□ autovacuum
□ dead tuples/bloat signals
□ WAL generation
□ replication lag
□ backup success
~~~

Monitoring should answer:

~~~text
Is it healthy?
Is it getting worse?
Why is it slow?
What changed?
~~~

---

# 40. Logging Checklist

Configure logging deliberately.

Potentially useful information includes:

~~~text
errors
connections/disconnections when appropriate
lock waits
slow statements
checkpoints
autovacuum activity
~~~

Do not log sensitive values unnecessarily.

Do not enable extremely verbose SQL logging in production without considering performance, privacy, and storage cost.

---

# 41. pg_stat_statements Checklist

Use it to identify:

~~~text
□ highest total query time
□ highest average query time
□ most frequently executed queries
□ high-row queries
~~~

Optimization priority often comes from:

~~~text
frequency × cost
~~~

not simply the slowest individual query.

---

# 42. Alert Checklist

Alerts should be actionable.

Examples:

~~~text
□ database unavailable
□ connection usage dangerously high
□ storage approaching limit
□ replication lag excessive
□ backup failure
□ sustained query latency regression
□ unusual lock/deadlock activity
□ autovacuum/transaction ID danger
~~~

Do not alert on every harmless fluctuation.

---

# 43. Deployment Checklist

Before deployment:

~~~text
□ Tests pass
□ Migration reviewed
□ Backward compatibility checked
□ Backup/recovery status verified for risky changes
□ Secrets/configuration present
□ Connection budget checked
~~~

During deployment:

~~~text
□ Run controlled migration
□ Deploy application
□ Run smoke tests
□ Watch application + DB metrics
~~~

After deployment:

~~~text
□ Error rate normal
□ Query latency normal
□ Connections normal
□ Lock waits normal
□ Data correctness checked
~~~

---

# 44. Rollback Checklist

Application rollback is easier than database rollback.

Ask before deployment:

~~~text
□ Can previous app version run against new schema?
□ Is migration destructive?
□ Has data already changed format?
□ Can we roll forward instead?
□ Is recovery required if migration fails?
~~~

Prefer backward-compatible migrations so code can be rolled back safely while schema remains compatible.

---

# 45. Graceful Shutdown Checklist

When application instances stop:

~~~text
stop accepting new work
      ↓
finish/cancel in-flight work appropriately
      ↓
close DB pool
      ↓
exit
~~~

Node `pg`:

~~~js
await pool.end();
~~~

This matters during deployments, autoscaling, and server shutdown.

---

# 46. Performance Checklist

Before scaling infrastructure, verify:

~~~text
□ Exact bottleneck identified
□ Slow SQL inspected
□ N+1 eliminated
□ Indexes match access patterns
□ Deep OFFSET avoided where necessary
□ Transactions short
□ Locks investigated
□ Pool correctly sized
□ Autovacuum healthy
□ Statistics healthy
□ CPU/memory/storage measured
~~~

Scale only after understanding the workload.

---

# 47. Infrastructure Scaling Checklist

Possible progression:

~~~text
1. Optimize query/schema
2. Tune connection usage
3. Improve DB resources
4. Add caching where justified
5. Add read replicas when needed
6. Separate analytics workloads
7. Consider partitioning for suitable huge tables
8. Consider distributed/sharded architecture only when justified
~~~

Complexity should be earned by real requirements.

---

# 48. Docker Checklist

If PostgreSQL is containerized:

~~~text
□ Persistent volume configured
□ Backups independent of container lifecycle
□ Secrets protected
□ Resource limits understood
□ Monitoring exists
□ Upgrade procedure exists
□ Recovery procedure exists
~~~

Containerization does not automatically provide durability or HA.

---

# 49. Kubernetes Checklist

If PostgreSQL runs on Kubernetes:

~~~text
□ Stateful storage designed
□ Replication/failover handled
□ Backups handled
□ Operator/lifecycle strategy understood
□ Scheduling/failure domains considered
□ upgrades tested
□ monitoring configured
~~~

For many teams:

~~~text
Kubernetes application
      +
Managed PostgreSQL
~~~

is simpler than operating PostgreSQL inside Kubernetes.

---

# 50. E-Commerce Checklist

For ShopHub-like systems:

~~~text
□ Price authoritative on server
□ Inventory concurrency-safe
□ Order items preserve purchase-time price/name
□ Order preserves address snapshot
□ Payment status separate from fulfillment status
□ Webhook authenticity verified
□ Webhook processing idempotent
□ Checkout request idempotent where needed
□ Currency explicit
□ Refund/payment history preserved
□ External payment call not inside long DB transaction
~~~

Design failure paths, not only successful checkout.

---

# 51. Failure Scenario Checklist

Ask:

~~~text
What if DB connection fails?
What if query times out?
What if transaction deadlocks?
What if payment callback arrives twice?
What if app crashes after DB commit?
What if primary database fails?
What if a migration is partially applied?
What if someone deletes data accidentally?
What if a backup cannot be restored?
~~~

Production engineering is largely about answering these questions before incidents happen.

---

# 52. Incident Investigation Flow

~~~text
Alert / user report
      ↓
Check application errors
      ↓
Check DB availability
      ↓
Check connections / pool
      ↓
Check locks / waits
      ↓
Check slow/frequent queries
      ↓
Check CPU / memory / storage
      ↓
Check autovacuum / WAL / replication
      ↓
Identify root cause
      ↓
Mitigate safely
      ↓
Verify recovery
      ↓
Document prevention
~~~

Do not randomly restart PostgreSQL before understanding what is happening unless the incident requires emergency recovery.

---

# 53. Useful Diagnostic Views

Remember these names:

~~~text
pg_stat_activity
pg_stat_database
pg_stat_user_tables
pg_stat_user_indexes
pg_stat_replication
pg_locks
pg_stat_statements
~~~

You do not need to memorize every column.

Know what question each view helps answer.

---

# 54. Essential psql Commands

~~~text
\l              list databases
\c dbname       connect database
\dn             list schemas
\dt             list tables
\d table_name   describe table
\du             list roles
\dp table_name  privileges
\x              expanded display
\q              quit
~~~

These are `psql` meta-commands, not SQL statements.

---

# 55. Essential SQL Revision

Be comfortable with:

~~~text
SELECT
INSERT
UPDATE
DELETE
WHERE
ORDER BY
LIMIT/OFFSET
DISTINCT
GROUP BY
HAVING
JOINs
Subqueries
CTEs
Window functions
RETURNING
ON CONFLICT
Transactions
~~~

These are the foundation beneath every advanced PostgreSQL topic.

---

# 56. Window Function Revision

Remember:

~~~text
ROW_NUMBER
RANK
DENSE_RANK
LAG
LEAD
SUM(...) OVER
PARTITION BY
ORDER BY
~~~

Key distinction:

~~~text
GROUP BY
→ collapses rows

Window function
→ keeps rows and calculates across them
~~~

---

# 57. PostgreSQL-Specific Feature Revision

Remember:

~~~text
Identity columns
Sequences
UUID
RETURNING
ON CONFLICT / UPSERT
Generated columns
JSONB
Arrays
ENUM/custom types
Views
Materialized Views
Functions
Procedures
Triggers
~~~

Know when each feature solves a real problem rather than using it merely because PostgreSQL supports it.

---

# 58. Core Internals Revision

Architecture:

~~~text
Client
  ↓
PostgreSQL backend process
  ↓
Parser / Planner / Executor
  ↓
Shared buffers
  ↓
Table / Index pages
  ↓
Storage
~~~

Core internals:

~~~text
MVCC
→ concurrent row-version visibility

VACUUM
→ cleanup/reuse + XID safety

WAL
→ durability/recovery/replication
~~~

These three concepts explain much of PostgreSQL production behavior.

---

# 59. Five Most Important Production Rules

### Rule 1

> **Protect data integrity with constraints, not only application code.**

### Rule 2

> **Use transactions and concurrency controls for multi-step business operations.**

### Rule 3

> **Measure query performance before adding indexes or tuning configuration.**

### Rule 4

> **Backups are only valuable when recovery has been tested.**

### Rule 5

> **Keep the production architecture as simple as possible while still meeting reliability requirements.**

---

# 60. Top Interview Questions — Rapid Revision

## PostgreSQL vs SQL?

SQL is a language. PostgreSQL is a relational/object-relational database management system that implements SQL and adds PostgreSQL-specific capabilities.

## Primary key vs unique?

A primary key uniquely identifies each row and is NOT NULL; a table has one primary-key constraint. UNIQUE enforces uniqueness for other candidate keys and PostgreSQL normally permits multiple NULLs unless additional semantics are specified.

## INNER JOIN vs LEFT JOIN?

INNER JOIN returns matching rows from both sides. LEFT JOIN preserves every row from the left side and adds matching right-side data when available.

## WHERE vs HAVING?

WHERE filters rows before grouping; HAVING filters groups after aggregation.

## CTE vs subquery?

Both can express intermediate query logic. CTEs often improve readability/composition and can support recursion/data-modifying workflows. Neither is automatically faster.

## GROUP BY vs window functions?

GROUP BY collapses rows into groups. Window functions calculate across related rows while retaining individual rows.

## What is MVCC?

Multi-Version Concurrency Control lets transactions see appropriate row versions through snapshots, allowing ordinary readers and writers to coexist with less blocking.

## What does VACUUM do?

It makes space from obsolete tuple versions reusable, maintains visibility information, and helps protect against transaction ID wraparound.

## What is WAL?

Write-Ahead Logging records changes before corresponding data pages need to be durably written, enabling crash recovery, replication, and PITR workflows.

## What is an index?

A separate data structure that can reduce the amount of work needed to locate rows, at the cost of storage and write maintenance.

## Why might PostgreSQL ignore an index?

A sequential scan may be cheaper because the table is small, many rows are needed, statistics/selectivity favor it, or the index does not match the query.

## EXPLAIN vs EXPLAIN ANALYZE?

EXPLAIN shows the planner's chosen plan and estimates. EXPLAIN ANALYZE executes the statement and reports actual runtime information.

## What is a transaction?

A group of operations treated as one logical unit with ACID properties.

## What is the default isolation level?

READ COMMITTED.

## What is a deadlock?

Two or more transactions wait on one another in a cycle. PostgreSQL detects the deadlock and aborts one transaction.

## Why use connection pooling?

Database connections are finite and costly; pooling reuses a controlled number of sessions across many application requests.

## Why parameterized queries?

They keep untrusted values separate from SQL syntax and are the primary defense against SQL injection.

## Replication vs backup?

Replication maintains another copy for availability/read scaling, but errors/deletions can replicate too. Backups provide historical recovery.

## RPO vs RTO?

RPO is acceptable data loss; RTO is acceptable recovery time.

## Why not use PostgreSQL superuser from the app?

It violates least privilege and greatly increases the impact of application compromise or bugs.

## Why are long transactions dangerous?

They can retain locks and old snapshots, delay vacuum cleanup, increase bloat pressure, and reduce concurrency.

---

# 61. Full Backend Architecture Revision

### Express

~~~text
Browser
  ↓
Express Route
  ↓
Controller
  ↓
Service
  ↓
DB Layer / pg Pool
  ↓
PostgreSQL
~~~

### Next.js App Router

~~~text
Browser
  ↓
Server Component / Route Handler / Server Action
  ↓
Data Access Layer
  ↓
PostgreSQL
~~~

### Production

~~~text
Users
  ↓
HTTPS
  ↓
Application instances
  ↓
Connection pool / pooled endpoint
  ↓
PostgreSQL primary
  ├─ Backups / PITR
  ├─ Monitoring
  └─ Replica / standby when required
~~~

---

# 62. Full E-Commerce Revision

~~~text
USER
 ↓
CART
 ↓
CART ITEMS
 ↓
SERVER-SIDE PRICE + INVENTORY VALIDATION
 ↓
TRANSACTION
 ├─ ORDER
 ├─ ORDER ITEMS
 └─ concurrency-safe inventory operation
 ↓
PAYMENT WORKFLOW
 ↓
VERIFIED + IDEMPOTENT CALLBACK
 ↓
PAYMENT / ORDER STATE UPDATE
 ↓
FULFILLMENT
~~~

Historical order data remains stable even if catalog data changes later.

---

# 63. Before You Say "Production Ready"

Ask yourself:

~~~text
Can invalid data enter the database?
Can two simultaneous requests corrupt business state?
Can SQL injection occur?
Can the application exhaust DB connections?
Can I identify a slow query?
Can I identify blocking?
Can I restore deleted data?
Can I survive a DB/server failure according to requirements?
Can I deploy schema changes safely?
Can I detect problems before users report them?
~~~

If important answers are unknown, the system still has production work remaining.

---

# 64. Your PostgreSQL Learning Path — Completed

~~~text
SECTION 1 — Fundamentals
01 What is PostgreSQL?
02 Database, Schema & Tables
03 Basic SQL CRUD
04 Data Types

SECTION 2 — SQL Querying
05 Filtering & Conditions
06 Aggregate Functions
07 SQL Joins
08 Subqueries
09 CTEs
10 Window Functions

SECTION 3 — Database Design
11 Primary & Foreign Keys
12 Constraints
13 Normalization
14 Relationships & Modeling

SECTION 4 — PostgreSQL Features
15 PostgreSQL-Specific Features
16 JSON & JSONB
17 Arrays, ENUMs & Custom Types
18 Views & Materialized Views
19 Functions, Procedures & Triggers

SECTION 5 — Transactions & Concurrency
20 Transactions
21 Isolation Levels
22 Locks & Concurrency

SECTION 6 — Performance
23 Index Fundamentals
24 Advanced Indexing
25 EXPLAIN & EXPLAIN ANALYZE
26 Query Optimization

SECTION 7 — PostgreSQL Internals
27 PostgreSQL Architecture
28 MVCC
29 VACUUM & Autovacuum
30 Write-Ahead Logging

SECTION 8 — Backend Development
31 PostgreSQL + Node.js
32 PostgreSQL + Express API
33 PostgreSQL + Next.js
34 ORM vs Raw SQL

SECTION 9 — Security
35 PostgreSQL Security
36 SQL Injection & Secure Queries

SECTION 10 — Production
37 PostgreSQL Configuration
38 Connection Pooling & PgBouncer
39 Backup & Restore
40 Replication
41 High Availability
42 Monitoring & Logging
43 Production Deployment

SECTION 11 — Production Project
44 Production E-Commerce Database
45 Production Database Optimization
46 PostgreSQL Production Checklist
~~~

**Total: 46 lessons across 11 sections.**

---

# 65. What You Should Be Able to Explain Now

After completing this curriculum, you should be able to explain:

~~~text
How PostgreSQL stores relational data
How to write SQL queries
How to model relationships
How constraints protect data
How transactions preserve correctness
How isolation and locks affect concurrency
How MVCC works conceptually
Why VACUUM exists
Why WAL exists
How indexes improve queries
How to read execution plans
How to connect Node.js/Express/Next.js
How ORMs relate to SQL
How SQL injection is prevented
How connection pooling works
How backup/PITR works
How replication and HA differ
How to monitor PostgreSQL
How to deploy safely
How to design a production e-commerce database
How to investigate production performance
~~~

That is the foundation expected from a full-stack developer who works seriously with PostgreSQL.

---

# 66. Final Revision Strategy

Do not reread all 46 lessons every day.

Use three levels:

~~~text
LEVEL 1 — Daily quick revision
Interview summaries + diagrams

LEVEL 2 — Weak topics
Open the individual lesson and practice SQL

LEVEL 3 — Practical reinforcement
Use PostgreSQL in CareerLoop / ShopHub
~~~

Practical usage is what turns concepts into long-term knowledge.

---

# 67. Final Interview Strategy

When answering PostgreSQL interview questions:

~~~text
1. Define the concept simply.
2. Explain why it exists.
3. Give a small SQL/example.
4. Mention one important production detail.
5. Mention a common mistake when relevant.
~~~

Example:

~~~text
Question: What is an index?

Definition
→ structure that helps locate rows efficiently

Why
→ avoid unnecessary scanning

Example
→ orders(user_id, created_at DESC)

Production detail
→ indexes cost writes/storage

Mistake
→ indexing every column
~~~

This structure produces clear, practical answers.

---

# 68. Final Production Rule

~~~text
Correctness
   ↓
Security
   ↓
Recoverability
   ↓
Observability
   ↓
Performance
   ↓
Scale
~~~

Do not sacrifice correctness and recoverability merely to make an architecture look advanced.

---

## Final Key Takeaway

> **Production PostgreSQL is about much more than knowing SQL syntax. A strong developer understands data modeling, constraints, transactions, concurrency, MVCC, indexing, execution plans, secure application access, connection pooling, backups, WAL, replication, monitoring, migrations, and recovery. The most important production habits are to protect correctness in the database, keep transactions controlled, measure before optimizing, use least privilege, test recovery, and add architectural complexity only when real requirements justify it.**

---

# Curriculum Complete 🎯

You have now reached the end of the **46-lesson PostgreSQL curriculum**.

Use the repository as:

~~~text
Learning reference
Interview revision guide
SQL/PostgreSQL handbook
Production checklist
Backend development reference
~~~

---

[← Previous: Lesson 45 — Production Database Optimization](./45-production-database-optimization.md) | [Back to Roadmap](../README.md)