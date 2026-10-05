# Lesson 38 — Connection Pooling & PgBouncer

## Goal of This Lesson

Opening a PostgreSQL connection has a cost, and PostgreSQL cannot safely accept unlimited application connections.

Connection pooling solves this by **reusing a controlled number of database connections**.

Core architecture:

~~~text
Many application requests
        ↓
Connection Pool
        ↓
Small controlled set of PostgreSQL connections
        ↓
PostgreSQL
~~~

For production systems, you should understand two levels of pooling:

~~~text
Application pool
→ for example node-postgres pg.Pool

External pooler
→ for example PgBouncer
~~~

---

# 1. Why Database Connections Are Expensive

From Lesson 27, PostgreSQL uses a process-oriented architecture.

A normal client connection is associated with a PostgreSQL backend process.

Conceptually:

~~~text
Application connection 1 → PostgreSQL backend 1
Application connection 2 → PostgreSQL backend 2
Application connection 3 → PostgreSQL backend 3
...
~~~

Each connection consumes resources such as:

~~~text
memory
backend process resources
session state
server scheduling overhead
~~~

Creating and destroying a database connection for every HTTP request is therefore inefficient.

---

# 2. Bad Architecture — New Connection Per Request

Imagine an Express API:

~~~text
HTTP Request 1
   ↓
Open PostgreSQL connection
   ↓
Query
   ↓
Close connection

HTTP Request 2
   ↓
Open PostgreSQL connection
   ↓
Query
   ↓
Close connection
~~~

Repeated connection setup adds unnecessary overhead.

Better:

~~~text
Application starts
      ↓
Create connection pool
      ↓
Requests borrow/reuse connections
~~~

---

# 3. What Is a Connection Pool?

A connection pool maintains reusable database connections.

~~~text
Request A ─┐
Request B ─┼→ Pool → PostgreSQL connections
Request C ─┘
~~~

If a connection is available:

~~~text
request
 ↓
borrow connection
 ↓
run query
 ↓
return connection to pool
~~~

If all pool connections are busy, additional work usually waits until a connection becomes available or a configured timeout/error occurs.

---

# 4. node-postgres pg.Pool

For Node.js applications using `pg`:

~~~js
import { Pool } from 'pg';

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
});
~~~

`max` is the maximum number of clients/connections that this pool will hold at once.

Do not blindly set it to a large number.

---

# 5. pool.query()

For a normal independent query:

~~~js
const result = await pool.query(
  'SELECT id, name FROM users WHERE id = $1',
  [userId]
 );
~~~

`pool.query()` conveniently:

~~~text
acquires a client
   ↓
executes query
   ↓
returns client to pool
~~~

It is excellent for single independent queries.

---

# 6. pool.connect()

When you need to control one specific connection:

~~~js
const client = await pool.connect();

try {
  const result = await client.query(
    'SELECT id, name FROM users WHERE id = $1',
    [userId]
  );
} finally {
  client.release();
}
~~~

Always release checked-out clients.

Otherwise the pool can eventually become exhausted.

---

# 7. Transactions Need One Checked-Out Client

From Lesson 20:

~~~js
const client = await pool.connect();

try {
  await client.query('BEGIN');

  await client.query(
    'INSERT INTO orders (user_id) VALUES ($1)',
    [userId]
  );

  await client.query(
    'UPDATE products SET stock = stock - 1 WHERE id = $1',
    [productId]
  );

  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
~~~

Golden rule:

~~~text
ONE TRANSACTION
      =
ONE DATABASE CONNECTION
~~~

Do not use separate `pool.query()` calls for statements that must belong to one transaction.

---

# 8. Why pool.query() Is Wrong Across a Transaction

Bad:

~~~js
await pool.query('BEGIN');
await pool.query('INSERT ...');
await pool.query('UPDATE ...');
await pool.query('COMMIT');
~~~

The pool may choose different connections for those calls.

Conceptually:

~~~text
BEGIN  → connection A
INSERT → connection B
UPDATE → connection C
COMMIT → connection A
~~~

That is not one valid application transaction.

Use `pool.connect()` and keep the same client until COMMIT/ROLLBACK.

---

# 9. Connection Pool Sizing

Suppose:

~~~text
PostgreSQL max_connections = 100
~~~

You should **not** automatically configure:

~~~text
application pool max = 100
~~~

because PostgreSQL may also need connections for:

~~~text
other application instances
administrators
migrations
monitoring
background jobs
provider-reserved connections
~~~

Pool sizing is a system-wide capacity problem.

---

# 10. Multiple Application Instances

Suppose production has:

~~~text
5 application instances
pool max = 20 per instance
~~~

Potential connections:

~~~text
5 × 20 = 100
~~~

If PostgreSQL allows only around that number, you have left little or no capacity for anything else.

This is one of the most common production connection mistakes.

---

# 11. Pool Sizing Formula — Mental Model

A useful planning approximation is:

~~~text
Total possible app DB connections
≈
number of app instances × pool max per instance
~~~

Then account for:

~~~text
+ workers
+ migrations
+ monitoring
+ admin access
+ safety reserve
~~~

This is a planning model, not a universal formula for the perfect pool size.

---

# 12. Bigger Pool Does Not Mean Faster

Imagine PostgreSQL can efficiently execute a limited amount of concurrent work.

Sending hundreds of active queries simultaneously can cause:

~~~text
CPU contention
memory pressure
I/O contention
lock contention
context switching
longer latency
~~~

A pool also acts as a concurrency control boundary.

~~~text
1000 incoming requests
       ↓
pool of controlled size
       ↓
limited concurrent DB work
~~~

Waiting briefly in the application can be healthier than overwhelming PostgreSQL.

---

# 13. Pool Queueing

When every pool connection is busy:

~~~text
Request
   ↓
waits in pool queue
   ↓
connection becomes available
   ↓
query executes
~~~

This creates backpressure.

But unlimited waiting is also dangerous.

Configure sensible acquisition/connection timeouts according to your driver/platform.

---

# 14. connectionTimeoutMillis

node-postgres can configure connection acquisition/creation timeout behavior, for example:

~~~js
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
  connectionTimeoutMillis: 5000,
});
~~~

Do not let requests wait indefinitely when the database/pool is unavailable.

Application HTTP timeouts and database timeouts should be designed coherently.

---

# 15. idleTimeoutMillis

Example:

~~~js
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30000,
});
~~~

This controls how long an idle pool client may remain before the pool removes it, subject to driver behavior/configuration.

Do not confuse:

~~~text
idle pool connection
with
idle transaction
~~~

An idle transaction is much more dangerous because it can retain locks/snapshots.

---

# 16. Pool Error Handling

A useful node-postgres pattern:

~~~js
pool.on('error', (error) => {
  console.error('Unexpected PostgreSQL pool error', error);
});
~~~

Idle clients can experience backend/network errors.

Production applications should observe and handle unexpected pool errors rather than silently ignoring them.

---

# 17. Graceful Shutdown

When a long-running Node process shuts down gracefully:

~~~js
await pool.end();
~~~

This closes the pool and its clients cleanly.

Typical flow:

~~~text
SIGTERM
  ↓
stop accepting new work
  ↓
finish/terminate active work appropriately
  ↓
pool.end()
  ↓
process exits
~~~

Exact shutdown architecture depends on your framework/runtime.

---

# 18. What Is PgBouncer?

PgBouncer is a lightweight external connection pooler for PostgreSQL.

Architecture:

~~~text
Applications
     ↓
PgBouncer
     ↓
PostgreSQL
~~~

Instead of every application connection mapping permanently to its own PostgreSQL backend connection, PgBouncer can multiplex many client connections over a smaller controlled set of server connections depending on pooling mode.

---

# 19. Why Use PgBouncer?

PgBouncer can help when:

~~~text
many app instances exist
many short-lived clients exist
connection churn is high
PostgreSQL connection limits are tight
serverless workloads create bursts
centralized connection control is useful
~~~

It reduces pressure caused by excessive direct PostgreSQL connections.

---

# 20. Application Pool vs PgBouncer

Application pool:

~~~text
Node.js process
   ↓
pg.Pool
   ↓
PostgreSQL
~~~

External pooler:

~~~text
Many app processes
      ↓
PgBouncer
      ↓
PostgreSQL
~~~

Combined architecture:

~~~text
App instance A → local pool ─┐
App instance B → local pool ─┼→ PgBouncer → PostgreSQL
App instance C → local pool ─┘
~~~

When combining layers, size them carefully. Large pools at every layer can defeat the purpose.

---

# 21. PgBouncer Pooling Modes

Three important modes are:

~~~text
Session pooling
Transaction pooling
Statement pooling
~~~

For interviews, focus mainly on **session** and **transaction** pooling.

---

# 22. Session Pooling

In session pooling, a PostgreSQL server connection is assigned to a client for the lifetime of that client session.

~~~text
Client session A
      ↓
PgBouncer
      ↓
Server connection 1
      ↓
held for whole client session
~~~

Advantages:

~~~text
closest to direct PostgreSQL session behavior
session-level features work naturally
~~~

Tradeoff:

~~~text
less aggressive connection multiplexing
~~~

---

# 23. Transaction Pooling

In transaction pooling, PgBouncer assigns a PostgreSQL server connection only for the duration of a transaction.

~~~text
Client
  ↓
BEGIN
  ↓
PgBouncer assigns server connection
  ↓
transaction executes
  ↓
COMMIT / ROLLBACK
  ↓
server connection returns to pool
~~~

The next transaction from the same client may run on a different PostgreSQL server connection.

This allows much stronger multiplexing.

---

# 24. Transaction Pooling Mental Model

~~~text
Client A transaction 1 → PostgreSQL connection 1
Client B transaction 1 → PostgreSQL connection 2

after commit:

Client C transaction 1 → PostgreSQL connection 1
Client A transaction 2 → PostgreSQL connection 2
~~~

Therefore:

> Do not assume persistent server-session state across transactions when using transaction pooling.

---

# 25. Session State and Transaction Pooling

Some PostgreSQL behavior is session-scoped.

Examples can include:

~~~text
session-level SET state
temporary tables
LISTEN state
session advisory locks
some prepared-statement assumptions/tool behaviors
~~~

With transaction pooling, the same client may not receive the same server connection for the next transaction.

So application/library compatibility must be checked.

---

# 26. Prepared Statements and PgBouncer

This topic requires nuance.

Older advice often says:

~~~text
Prepared statements do not work with transaction pooling.
~~~

That statement is too broad for modern PgBouncer.

Modern PgBouncer versions can support protocol-level named prepared statements in transaction pooling when configured appropriately, notably with `max_prepared_statements` greater than zero.

However, not every session-state pattern is safe, and SQL-level `PREPARE`/`DEALLOCATE` plus driver/framework behavior can have different compatibility considerations.

Production rule:

> Check your exact PgBouncer version, pooling mode, driver, ORM, and prepared-statement behavior instead of relying on an old blanket rule.

---

# 27. Statement Pooling

Statement pooling returns a server connection to the pool after each statement.

This is the most restrictive mode.

Multi-statement transactions are not compatible with normal statement pooling semantics.

For typical transactional web applications, understand it conceptually but focus more on session/transaction pooling.

---

# 28. PgBouncer Does Not Replace Transactions

PgBouncer manages connections.

PostgreSQL manages transaction semantics.

~~~text
PgBouncer
→ connection multiplexing

PostgreSQL
→ BEGIN / COMMIT / ROLLBACK / isolation / locks
~~~

Do not confuse connection pooling with transaction management.

---

# 29. PgBouncer Does Not Make Slow SQL Fast

If a query performs:

~~~text
full scan of huge table
bad join
missing index
expensive sort
~~~

adding PgBouncer does not fix the query plan.

Use:

~~~text
EXPLAIN ANALYZE
indexes
query optimization
schema improvements
~~~

PgBouncer solves a different problem: **connection management**.

---

# 30. Direct vs Pooled Database URL

Some managed PostgreSQL providers expose both:

~~~text
direct database endpoint
pooled database endpoint
~~~

Conceptually:

~~~text
DATABASE_URL
      ↓
provider pooler
      ↓
PostgreSQL
~~~

and possibly:

~~~text
DIRECT_DATABASE_URL
      ↓
PostgreSQL directly
~~~

Exact environment-variable names differ by provider/framework.

Do not invent them; use the provider's documented connection strings.

---

# 31. Why Migrations May Need a Direct Connection

Some schema migration tools or operations require session behavior or connection semantics that differ from transaction-pooled application traffic.

A common architecture is:

~~~text
Runtime application
      ↓
pooled connection endpoint

Migration job
      ↓
direct / migration-compatible endpoint
~~~

Whether this is required depends on the provider, pooler mode, and migration tool.

Always follow the tool/provider documentation.

---

# 32. Serverless Connection Problem

Traditional server:

~~~text
1 Node.js process
      ↓
1 stable pool
      ↓
PostgreSQL
~~~

Serverless architecture can look more like:

~~~text
many short-lived function instances
        ↓
each may create DB connections
        ↓
connection burst
        ↓
PostgreSQL exhaustion
~~~

This is why serverless-friendly drivers, provider poolers, proxies, or pooled endpoints are important.

---

# 33. Next.js Deployment Consideration

A local Next.js application may have one development process.

Production can involve:

~~~text
multiple server instances
serverless functions
autoscaling
edge/server runtimes
provider-specific execution model
~~~

Do not calculate database connection usage from your laptop architecture alone.

Understand the deployment platform.

---

# 34. Global Pool Reuse in Long-Lived Node Runtimes

In a conventional long-running Node.js server, create a shared pool rather than constructing a new pool inside every request handler.

Good:

~~~js
// db.js
export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});
~~~

Then import/reuse it.

Bad:

~~~js
app.get('/users', async (req, res) => {
  const pool = new Pool(...);
  // creates another pool repeatedly
});
~~~

A pool should itself be reused.

---

# 35. Development Hot Reload

Framework development servers may reload modules repeatedly.

If your setup creates a fresh pool on every reload, local development can accumulate unnecessary connections.

Some projects use a development-only global singleton pattern to reuse a pool/client across hot reloads.

However:

> The correct implementation depends on the driver and framework runtime.

Do not blindly copy singleton snippets designed for a different database client.

---

# 36. Serverless Drivers

Some managed PostgreSQL platforms provide drivers designed for serverless environments and HTTP/WebSocket-based connectivity.

These can reduce traditional long-lived TCP connection assumptions.

But the same architecture question remains:

~~~text
How many database sessions/connections/transactions can the deployment create under load?
~~~

Use the connection strategy recommended by your provider and driver.

---

# 37. Pool Metrics

With node-postgres, useful pool state includes concepts such as:

~~~text
pool.totalCount
pool.idleCount
pool.waitingCount
~~~

These help answer:

~~~text
How many clients exist?
How many are idle?
Are requests waiting for a client?
~~~

A growing waiting queue is an important signal.

---

# 38. Database-Side Connection Monitoring

PostgreSQL exposes session activity through:

~~~sql
SELECT
  pid,
  usename,
  datname,
  state,
  query_start
FROM pg_stat_activity;
~~~

This helps inspect:

~~~text
active connections
idle connections
idle-in-transaction sessions
long-running queries
application users
~~~

Be mindful of permissions and sensitive query text when using monitoring views.

---

# 39. Connection Exhaustion

Typical symptom:

~~~text
remaining connection slots are reserved...
too many clients...
connection acquisition timeout
~~~

Possible causes:

~~~text
pool too large
too many app instances
connection leak
new pool per request
long-running transactions
slow queries holding clients
serverless burst
missing external pooler
~~~

Do not immediately solve it by only increasing `max_connections`.

---

# 40. Connection Leak

Bad:

~~~js
const client = await pool.connect();
await client.query('SELECT ...');
// forgot client.release()
~~~

Repeated requests consume more checked-out clients until none are available.

Correct:

~~~js
const client = await pool.connect();

try {
  await client.query('SELECT ...');
} finally {
  client.release();
}
~~~

`finally` is important because it runs even when the query throws.

---

# 41. Slow Query Can Exhaust the Pool

Suppose:

~~~text
pool max = 10
10 queries each take 30 seconds
~~~

Then:

~~~text
all 10 connections busy
      ↓
new requests wait
      ↓
HTTP latency grows
      ↓
timeouts
~~~

Pool exhaustion may therefore be a **query-performance symptom**, not only a pool-sizing problem.

Always inspect slow queries and lock waits.

---

# 42. Long Transactions and Pooling

A transaction keeps its connection checked out until it ends.

Bad:

~~~text
BEGIN
 ↓
database query
 ↓
call external payment API for 15 seconds
 ↓
database query
 ↓
COMMIT
~~~

During the external API call:

~~~text
DB connection remains occupied
transaction remains open
locks/snapshot may remain
~~~

Keep database transactions short.

---

# 43. Pooling and External APIs

For ShopHub payment flow, avoid:

~~~text
BEGIN
 ↓
reserve DB resources
 ↓
wait on Cashfree/Razorpay network call
 ↓
COMMIT
~~~

Instead design clear boundaries, for example:

~~~text
short DB transaction
      ↓
external payment workflow
      ↓
verified callback/webhook
      ↓
short DB transaction to finalize state
~~~

The exact payment architecture depends on business consistency requirements, but do not hold DB connections/locks unnecessarily while waiting on external networks.

---

# 44. ShopHub Pooling Architecture

Possible production design:

~~~text
Users
  ↓
Next.js / Node instances
  ↓
small application pools / provider connection strategy
  ↓
PgBouncer or managed pooled endpoint
  ↓
PostgreSQL
~~~

For each layer ask:

~~~text
How many instances?
How large is each pool?
What is the DB connection limit?
How many admin/migration connections need reserve?
What happens during autoscaling?
~~~

---

# 45. CareerLoop / Neon-Style Architecture

For a serverless-oriented managed PostgreSQL setup, a common mental model is:

~~~text
Next.js deployment
      ↓
provider-supported serverless/pooled connection path
      ↓
PostgreSQL
~~~

Because providers differ, use the provider's recommended driver/pooled URL for your deployment model rather than forcing a traditional large local pool everywhere.

Your current Drizzle/managed-PostgreSQL stack still relies on the same principles: **control concurrency and avoid connection explosions**.

---

# 46. Pooling and ORMs

ORMs still need database connectivity underneath.

~~~text
Prisma / Drizzle / other ORM
        ↓
driver / connection layer
        ↓
pooler or PostgreSQL
~~~

An ORM does not eliminate:

~~~text
connection limits
pool sizing
transaction connection requirements
serverless connection bursts
~~~

Read the ORM's current connection-management documentation for your runtime.

---

# 47. Pooling and Prepared Queries — Interview View

Keep the answer nuanced:

~~~text
Session pooling
→ server session stays associated with client session
→ session-state compatibility is easiest

Transaction pooling
→ server connection may change after each transaction
→ session-state features require compatibility review
~~~

For prepared statements, modern PgBouncer has support for protocol-level named prepared statements in transaction mode when configured appropriately, but compatibility still depends on the exact driver/tool behavior.

Do not repeat outdated absolute claims.

---

# 48. PgBouncer Is Not a Database Proxy for Everything

PgBouncer focuses on lightweight PostgreSQL connection pooling.

It does not replace:

~~~text
PostgreSQL replication
backup
high availability
query optimization
authorization
database monitoring
~~~

Keep architectural responsibilities separate.

---

# 49. Practical Pool Sizing Workflow

Use this process:

~~~text
1. Know PostgreSQL/provider connection limit
        ↓
2. Reserve connections for admin/migrations/monitoring
        ↓
3. Know maximum application instance count
        ↓
4. Choose conservative per-instance pool size
        ↓
5. Load test
        ↓
6. Observe DB CPU/I/O/locks + pool waiting
        ↓
7. Adjust from evidence
~~~

There is no universal `pool max = 20` rule.

---

# 50. What to Monitor

Important signals:

~~~text
active DB connections
idle connections
idle-in-transaction sessions
pool waiting count
connection acquisition latency
query latency
database CPU
database memory
lock waits
transaction duration
connection errors
~~~

Pooling should be tuned together with database/query performance.

---

# Common Mistakes

## 51. Opening a New DB Connection for Every HTTP Request

Reuse a pool or provider-supported pooled/serverless connection strategy.

## 52. Creating a New Pool Inside Every Request

The pool itself should normally be shared for the lifetime appropriate to the runtime.

## 53. Setting Pool Max Equal to PostgreSQL max_connections

Account for multiple app instances and non-application connections.

## 54. Forgetting client.release()

Use `finally` after `pool.connect()`.

## 55. Using pool.query() Across a Transaction

Use one checked-out client for the entire transaction.

## 56. Assuming Bigger Pools Always Improve Throughput

Excess concurrency can overload PostgreSQL and increase latency.

## 57. Increasing max_connections Instead of Fixing Leaks/Slow Queries

Find the actual reason connections remain occupied.

## 58. Holding a Transaction During an External API Call

Keep transactions and checked-out connection time short.

## 59. Assuming PgBouncer Fixes Slow Queries

It solves connection management, not bad query plans.

## 60. Ignoring Pooling Mode Compatibility

Transaction pooling changes server-session affinity; review session-state and prepared-statement behavior.

## 61. Copying a Local Pool Strategy into Serverless Production

Understand how the deployment platform scales instances and connections.

---

# Interview Revision

## What is connection pooling?

Reusing a controlled set of database connections instead of opening a new PostgreSQL connection for every operation/request.

## Why does PostgreSQL need connection pooling?

Connections consume server resources and PostgreSQL uses a backend process per normal connection, so excessive connections can reduce stability and performance.

## What is pg.Pool?

node-postgres's application-level connection pool for reusing PostgreSQL clients.

## pool.query() vs pool.connect()?

`pool.query()` is convenient for independent queries. `pool.connect()` checks out a specific client and is required when several statements must use the same connection, especially a transaction.

## Why must a transaction use one client?

Transaction state belongs to a PostgreSQL connection/session. Different pool clients would not be part of the same transaction.

## What is PgBouncer?

A lightweight external PostgreSQL connection pooler that can multiplex client connections over a controlled set of PostgreSQL server connections.

## Session pooling vs transaction pooling?

Session pooling keeps a server connection for the whole client session. Transaction pooling assigns a server connection only for a transaction and can reuse it for another client after COMMIT/ROLLBACK.

## Why is transaction pooling more scalable?

It allows more client sessions to share fewer PostgreSQL server connections because connections are returned after each transaction.

## What is the tradeoff of transaction pooling?

The client cannot assume the same PostgreSQL server session across transactions, so session-scoped features require compatibility review.

## Do prepared statements work with PgBouncer transaction pooling?

Modern PgBouncer can support protocol-level named prepared statements in transaction mode when configured appropriately, but compatibility depends on PgBouncer configuration/version and client behavior. Avoid blanket yes/no answers.

## Does PgBouncer improve a slow SQL query?

No. It manages connections. Query optimization still requires indexes, better SQL, statistics, EXPLAIN, and schema/workload improvements.

## Why can serverless cause connection problems?

Autoscaling can create many short-lived application instances, each potentially opening database connections and causing connection bursts.

## How should pool size be chosen?

From the database connection limit, number of application instances, reserved operational connections, workload, and load-test/monitoring evidence.

---

# Quick Revision

~~~text
WITHOUT POOLING
Request → New DB connection → Query → Close
Request → New DB connection → Query → Close
Request → New DB connection → Query → Close
~~~

~~~text
WITH APPLICATION POOL
Requests
   ↓
pg.Pool
   ↓
reused DB connections
   ↓
PostgreSQL
~~~

~~~text
WITH PGBOUNCER
Many app instances
       ↓
small local pools / clients
       ↓
PgBouncer
       ↓
controlled PostgreSQL connections
       ↓
PostgreSQL
~~~

### Transactions

~~~text
pool.connect()
    ↓
client
    ↓
BEGIN
query
query
COMMIT / ROLLBACK
    ↓
client.release()
~~~

### Pool Sizing

~~~text
instances × per-instance pool
        +
admin/migrations/monitoring reserve
        ≤
safe DB connection capacity
~~~

### Golden Rule

~~~text
Connections are a limited resource.
Reuse them, bound them, monitor them.
~~~

---

## Key Takeaway

> **Connection pooling protects PostgreSQL from excessive connection creation and concurrency. Reuse application pools, use one checked-out client for each transaction, size pools across all application instances rather than in isolation, and consider PgBouncer or managed pooled endpoints when many clients or serverless workloads would otherwise create too many PostgreSQL connections. PgBouncer manages connections—it does not replace correct transactions, query optimization, or database capacity planning.**

---

[← Previous: Lesson 37 — PostgreSQL Configuration](./37-postgresql-configuration.md) | [Back to Roadmap](../README.md) | [Next: Lesson 39 — Backup & Restore →](./39-backup-and-restore.md)