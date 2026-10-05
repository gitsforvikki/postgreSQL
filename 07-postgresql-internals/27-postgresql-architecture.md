# Lesson 27 — PostgreSQL Architecture

## Why Should a Full-Stack Developer Learn PostgreSQL Architecture?

You do not need to become a PostgreSQL database administrator to build backend applications.

But understanding the architecture helps you answer questions such as:

~~~text
What happens when Node.js connects to PostgreSQL?
Why does connection pooling matter?
Where does PostgreSQL cache database pages?
What is WAL?
How can PostgreSQL handle multiple clients?
What happens internally when a query runs?
~~~

The goal of this lesson is to build a **clear mental model**, not memorize every PostgreSQL internal process.

---

# 1. Big Picture Architecture

PostgreSQL follows a **client/server architecture**.

~~~text
Client Applications
Node.js / Next.js / psql / pgAdmin
          │
          │ SQL / PostgreSQL protocol
          ▼
┌──────────────────────────────┐
│      PostgreSQL Server       │
│                              │
│  Backend Processes           │
│  Shared Memory               │
│  Background Processes        │
│                              │
└──────────────┬───────────────┘
               │
               ▼
        Database Files
        + WAL Files
~~~

Your application does not directly read PostgreSQL's database files.

It connects to the PostgreSQL server, sends SQL, and receives results.

---

# 2. Client Applications

A PostgreSQL client can be:

~~~text
Node.js using pg
Next.js server code
Express API
psql
pgAdmin
another backend service
~~~

Example Node.js connection:

~~~js
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});
~~~

The application communicates with the PostgreSQL server through a database connection.

~~~text
Application
    ↓
connection
    ↓
PostgreSQL
~~~

---

# 3. PostgreSQL Server Process Model

At a high level, PostgreSQL uses a **process-based architecture**.

There is a main server process, historically called the **postmaster**, which listens for connection requests and manages the server's child processes.

When a client connects, PostgreSQL normally creates a dedicated **backend process** for that connection.

Conceptually:

~~~text
PostgreSQL main server process
            │
      ┌─────┼─────┐
      ▼     ▼     ▼
 Backend Backend Backend
    A       B       C
    │       │       │
 Client A Client B Client C
~~~

Each connected client session is normally served by its own PostgreSQL backend process.

This is an important reason why applications should not create unlimited database connections.

---

# 4. What Is a Backend Process?

A PostgreSQL backend process handles work for one client connection.

It performs tasks such as:

~~~text
receive SQL
   ↓
parse query
   ↓
plan query
   ↓
execute query
   ↓
return rows/result
~~~

For example:

~~~sql
SELECT id, name
FROM users
WHERE email = $1;
~~~

The backend process works with PostgreSQL's shared-memory structures and storage system to execute the query.

---

# 5. Why Connection Pooling Matters

Suppose a web server receives 10,000 requests.

A bad mental model is:

~~~text
HTTP request 1 → new PostgreSQL connection
HTTP request 2 → new PostgreSQL connection
HTTP request 3 → new PostgreSQL connection
...
HTTP request 10000 → new PostgreSQL connection
~~~

PostgreSQL connections have real resource cost because each connection normally corresponds to a backend process and associated memory/resources.

Instead, applications commonly use a connection pool:

~~~text
Many HTTP requests
       ↓
Application connection pool
       ↓
limited reusable DB connections
       ↓
PostgreSQL backend processes
~~~

Node.js `pg.Pool` is an application-side example.

Later, Lesson 38 covers connection pooling and PgBouncer in production.

---

# 6. Shared Memory

PostgreSQL backend processes are separate processes, but they need shared server-wide structures.

PostgreSQL allocates **shared memory** for important shared data.

Conceptually:

~~~text
Backend A ──┐
Backend B ──┼──→ Shared Memory
Backend C ──┘
~~~

Important shared-memory areas include concepts such as:

~~~text
Shared Buffers
WAL Buffers
lock-related/shared control structures
~~~

For full-stack interviews, the two most important names here are **shared buffers** and **WAL buffers**.

---

# 7. Shared Buffers

`shared_buffers` is PostgreSQL's main shared cache for database pages.

When PostgreSQL needs table or index data, it works with database pages.

High-level flow:

~~~text
Query needs database page
          ↓
Is page already in shared buffers?
       /        \
     yes         no
      │           │
      │       read page
      │       from storage
      │           │
      └──────┬────┘
             ▼
      use page in memory
~~~

Keeping frequently used pages in memory avoids repeatedly requesting the same pages from storage.

Important: PostgreSQL also benefits from the operating system's filesystem cache. Do not think of shared buffers as the only caching layer.

---

# 8. What Is a Page?

PostgreSQL stores tables and indexes in fixed-size blocks/pages.

The default database block size in standard PostgreSQL builds is commonly **8 KB**.

Conceptually:

~~~text
Table file
┌────────┐
│ Page 1 │
├────────┤
│ Page 2 │
├────────┤
│ Page 3 │
└────────┘
~~~

PostgreSQL generally moves and manages table/index data in pages rather than thinking only in terms of individual application rows.

This helps explain why EXPLAIN with `BUFFERS` talks about page/buffer activity.

---

# 9. WAL Buffers

WAL means **Write-Ahead Log**.

PostgreSQL uses WAL records to describe changes that must be recoverable.

`wal_buffers` provides shared memory used for WAL data before it is written to WAL storage.

Very simplified flow:

~~~text
Transaction changes data
        ↓
WAL record generated
        ↓
WAL buffer
        ↓
WAL written to durable storage
        ↓
data pages can be written later
~~~

The key idea is in the name:

> **Write-Ahead** means the relevant WAL information must be safely recorded before changed data pages are allowed to reach durable storage in a way that would violate recovery guarantees.

WAL is covered deeply in Lesson 30.

---

# 10. Database Storage

PostgreSQL ultimately persists data on storage.

High-level view:

~~~text
PostgreSQL Data Directory
        │
        ├── table/index data files
        ├── WAL
        ├── configuration/state
        └── other internal files
~~~

You normally should **not manually edit PostgreSQL's internal database files**.

Applications interact through SQL and PostgreSQL's supported interfaces/tools.

---

# 11. Background Processes

PostgreSQL runs background processes that perform important database work.

You do not need to memorize every process, but know these concepts:

~~~text
checkpointer
background writer
WAL writer
autovacuum launcher/workers
archiver (when configured)
~~~

Some PostgreSQL versions/configurations also have additional auxiliary processes.

Focus on responsibilities, not a fixed process list.

---

# 12. Checkpointer

PostgreSQL periodically performs **checkpoints**.

A checkpoint establishes a recovery point and coordinates writing dirty pages toward storage.

Very simplified:

~~~text
Changed pages in memory
        ↓
checkpoint activity
        ↓
pages written toward durable storage
        ↓
recovery has a known checkpoint reference
~~~

Checkpoints interact closely with WAL and crash recovery.

You do not need checkpoint tuning yet; just understand the architectural role.

---

# 13. Background Writer

The background writer helps write dirty shared-buffer pages gradually.

Conceptually:

~~~text
shared buffers
  dirty pages
      ↓
background writer
      ↓
storage
~~~

A **dirty page** means a memory page whose contents have changed relative to the durable copy.

The goal is to smooth some page-writing work rather than leaving all writes to backend processes/checkpoints.

---

# 14. WAL Writer

The WAL writer helps flush WAL data from WAL buffers to WAL files.

~~~text
WAL buffers
     ↓
WAL writer / WAL flushing
     ↓
WAL files
~~~

WAL durability is fundamental to PostgreSQL recovery behavior.

---

# 15. Autovacuum

Because PostgreSQL uses MVCC, updates and deletes can leave old row versions that eventually need cleanup.

Autovacuum performs maintenance including:

~~~text
cleaning reusable dead-row space
updating planner statistics through auto-analyze
preventing transaction ID wraparound problems
~~~

Architecture view:

~~~text
PostgreSQL server
      ↓
autovacuum launcher
      ↓
autovacuum workers
      ↓
maintain tables
~~~

MVCC and VACUUM are covered in Lessons 28 and 29.

---

# 16. Query Lifecycle — Big Picture

Now connect everything.

Node.js sends:

~~~sql
SELECT id, name
FROM users
WHERE email = $1;
~~~

High-level lifecycle:

~~~text
Node.js
   ↓
PostgreSQL connection
   ↓
Backend process
   ↓
Parser
   ↓
Analyzer / Rewriter
   ↓
Planner / Optimizer
   ↓
Executor
   ↓
Shared buffers / indexes / table pages
   ↓
Storage if required
   ↓
Rows returned to Node.js
~~~

This is one of the most useful architecture diagrams to remember.

---

# 17. Parser

The parser examines SQL syntax and builds an internal representation.

Example:

~~~sql
SELECT id
FROM users
WHERE email = $1;
~~~

If SQL syntax is invalid, the request fails before meaningful execution.

Conceptually:

~~~text
SQL text
   ↓
Parser
   ↓
parsed representation
~~~

---

# 18. Analysis and Rewrite

PostgreSQL resolves referenced database objects and applies relevant rewrite rules.

At a high level, PostgreSQL needs to understand things such as:

~~~text
Which table is users?
Which column is email?
What are the data types?
Are views/rules involved?
~~~

You do not need internal parse-tree details for normal interviews.

---

# 19. Planner / Optimizer

The planner decides **how** PostgreSQL should execute the query.

It considers possibilities such as:

~~~text
Seq Scan?
Index Scan?
Bitmap Scan?
Nested Loop?
Hash Join?
Merge Join?
Sort?
Which join order?
~~~

It uses:

~~~text
table statistics
available indexes
estimated row counts
cost parameters
query structure
~~~

This is exactly what you inspected in Lesson 25 with `EXPLAIN`.

---

# 20. Executor

After PostgreSQL chooses a plan, the executor performs it.

Example plan:

~~~text
Index Scan using idx_users_email
~~~

The executor:

~~~text
uses index
   ↓
locates candidate tuple/page
   ↓
checks visibility/conditions
   ↓
produces result
~~~

`EXPLAIN ANALYZE` executes the plan and reports actual behavior.

---

# 21. Read Query Flow

Example:

~~~sql
SELECT *
FROM products
WHERE id = 10;
~~~

Simplified read flow:

~~~text
Client
  ↓
Backend process
  ↓
Planner chooses access path
  ↓
Executor needs page
  ↓
Shared buffers
  ↓
page present? ── yes → use it
      │
      no
      ↓
request/read from storage
      ↓
place/use page in shared buffers
      ↓
return visible row
~~~

PostgreSQL's MVCC rules determine which row version the transaction is allowed to see.

---

# 22. Write Query Flow

Example:

~~~sql
UPDATE products
SET stock = stock - 1
WHERE id = 10;
~~~

Highly simplified flow:

~~~text
Client UPDATE
     ↓
Backend process
     ↓
find target row
     ↓
create/update row version under MVCC
     ↓
generate WAL records
     ↓
change page in shared buffers
     ↓
COMMIT
     ↓
required WAL made durable
     ↓
changed data page can be flushed later
~~~

Important:

> PostgreSQL does not need to synchronously write every modified table page to its final data file before acknowledging every commit. WAL is central to making this safe.

This is one of the reasons WAL matters so much.

---

# 23. What Happens During COMMIT?

At a simplified conceptual level:

~~~text
transaction modifies data
       ↓
WAL records generated
       ↓
COMMIT requested
       ↓
required WAL durability established
       ↓
COMMIT succeeds
~~~

The modified table/index pages may be written later.

On crash recovery, WAL can be used to restore the database to a consistent state.

Exact behavior can depend on durability settings such as `synchronous_commit`, but the default mental model above is appropriate for learning.

---

# 24. Multiple Databases and Schemas

One PostgreSQL server/cluster can contain multiple databases.

~~~text
PostgreSQL cluster/server
        │
        ├── database_a
        │     ├── schema public
        │     └── other schemas
        │
        └── database_b
              └── schema public
~~~

A client connection connects to one database at a time.

Inside that database, schemas organize objects such as tables, views, functions, and types.

---

# 25. PostgreSQL Cluster — Important Terminology

In PostgreSQL terminology, a **database cluster** is a collection of databases managed by one PostgreSQL server instance and stored under one data directory.

This does **not necessarily mean** a distributed multi-machine cluster.

That terminology often confuses beginners.

~~~text
PostgreSQL database cluster
≈ one PostgreSQL server instance/data directory
  containing multiple databases
~~~

---

# 26. Connections vs Transactions

A connection and a transaction are different concepts.

~~~text
Connection
→ session between client and PostgreSQL

Transaction
→ BEGIN ... COMMIT/ROLLBACK unit of database work
~~~

One connection can execute many transactions over its lifetime.

Connection pool:

~~~text
Connection 1
  transaction A
  transaction B
  transaction C
~~~

This is why pooled connections can be reused across many application requests.

---

# 27. Backend Process vs Application Process

Do not confuse:

~~~text
Node.js process
→ runs your JavaScript application

PostgreSQL backend process
→ handles one database client connection/session
~~~

Architecture:

~~~text
Node.js server
   │
   ├── pooled DB connection 1 ──→ PostgreSQL backend 1
   ├── pooled DB connection 2 ──→ PostgreSQL backend 2
   └── pooled DB connection 3 ──→ PostgreSQL backend 3
~~~

This is a useful interview explanation for why database connection count matters.

---

# 28. PostgreSQL and the Operating System

PostgreSQL does not operate independently of the OS.

The operating system provides:

~~~text
process scheduling
filesystem
memory management
filesystem/page cache
disk I/O
networking
~~~

So a simplified storage/cache path is:

~~~text
PostgreSQL shared buffers
          ↓
Operating-system cache/filesystem
          ↓
storage device
~~~

This is why database performance is affected by both PostgreSQL configuration and the underlying machine/storage.

---

# 29. Architecture and High Concurrency

Suppose ShopHub receives many requests.

~~~text
Thousands of HTTP requests
          ↓
Node.js instances
          ↓
connection pools
          ↓
controlled number of PostgreSQL connections
          ↓
backend processes
          ↓
shared memory + storage
~~~

If every application request creates a new DB connection, PostgreSQL can spend unnecessary resources managing connection/process overhead.

This is why production architectures usually control connection counts.

---

# 30. ShopHub Architecture Example

~~~text
Browser
   ↓ HTTPS
Next.js / Node.js backend
   ↓
pg / ORM connection pool
   ↓
PostgreSQL Server
   │
   ├── Backend Processes
   │
   ├── Shared Memory
   │     ├── Shared Buffers
   │     └── WAL Buffers
   │
   ├── Background Processes
   │     ├── Checkpointer
   │     ├── Background Writer
   │     ├── WAL Writer
   │     └── Autovacuum
   │
   └── Storage
         ├── Table / Index Data
         └── WAL
~~~

This is the core architecture you should remember.

---

# Common Mistakes

## 31. Thinking PostgreSQL Is Just a File

PostgreSQL is a database server with processes, shared memory, concurrency control, caching, WAL, and storage management.

## 32. Creating One DB Connection Per HTTP Request

Use connection pooling and control connection counts.

## 33. Thinking Shared Buffers Store Entire Tables Permanently

Shared buffers cache database pages; contents change according to workload and memory pressure.

## 34. Thinking COMMIT Means Every Table Page Was Immediately Written to Its Final Data File

WAL allows PostgreSQL to establish durability before all changed data pages are flushed to their final locations.

## 35. Thinking WAL Is the Same as the Table Data

WAL is a log of changes used for durability/recovery and replication-related mechanisms; table/index data is stored separately.

## 36. Confusing PostgreSQL Cluster with Kubernetes Cluster

In PostgreSQL terminology, a database cluster commonly means one PostgreSQL server/data directory containing multiple databases.

## 37. Memorizing Every Internal Process

For full-stack interviews, understand the major responsibilities and query flow rather than memorizing implementation details that can vary by PostgreSQL version/configuration.

---

# Interview Revision

## What architecture does PostgreSQL use?

PostgreSQL uses a client/server, process-based architecture. A main server process accepts connections, and client sessions are normally handled by dedicated backend processes.

## What is a PostgreSQL backend process?

A server process that handles SQL work for a client connection/session.

## Why is connection pooling important?

PostgreSQL connections consume server resources and normally map to backend processes, so pooling reuses a controlled number of connections across many application requests.

## What are shared buffers?

PostgreSQL's main shared-memory cache for table and index pages.

## Does PostgreSQL rely only on shared buffers for caching?

No. It also benefits from the operating system's filesystem/page cache.

## What are WAL buffers?

Shared memory used for WAL records before they are flushed to WAL storage.

## What is WAL?

Write-Ahead Logging records changes needed for durability and crash recovery; relevant WAL is persisted before changed data pages can be safely considered durable.

## What does the planner do?

It chooses an execution strategy using query structure, statistics, indexes, estimated costs, and other information.

## What does the executor do?

It executes the chosen query plan and produces the result.

## What does autovacuum do?

It performs automatic maintenance such as reclaiming reusable space from dead tuples, updating statistics through auto-analyze, and preventing transaction-ID wraparound issues.

## What is a PostgreSQL database cluster?

A collection of databases managed by one PostgreSQL server instance/data directory; it does not necessarily mean multiple machines.

## Connection vs transaction?

A connection is a client/server session. A transaction is an atomic unit of database work performed over a connection.

---

# Quick Revision

~~~text
CLIENT
Node.js / Next.js / psql
       ↓
CONNECTION
       ↓
POSTGRESQL BACKEND PROCESS
       ↓
Parser
       ↓
Analyzer / Rewriter
       ↓
Planner
       ↓
Executor
       ↓
Shared Buffers / WAL / Storage
       ↓
RESULT
~~~

### Main Architecture

~~~text
PostgreSQL Server
│
├── Backend Processes
│   → handle client sessions
│
├── Shared Memory
│   ├── Shared Buffers
│   └── WAL Buffers
│
├── Background Processes
│   ├── Checkpointer
│   ├── Background Writer
│   ├── WAL Writer
│   └── Autovacuum
│
└── Storage
    ├── Table / Index Data
    └── WAL
~~~

### Read

~~~text
Query
 ↓
Planner chooses path
 ↓
Executor
 ↓
Shared buffers
 ↓
storage if needed
 ↓
result
~~~

### Write

~~~text
UPDATE
 ↓
modify row version/page
 ↓
generate WAL
 ↓
COMMIT
 ↓
WAL durability
 ↓
data page may flush later
~~~

### Most Important Interview Connection

~~~text
Many web requests
      ↓
Connection Pool
      ↓
Controlled DB connections
      ↓
PostgreSQL backend processes
~~~

---

## Key Takeaway

> **PostgreSQL is a client/server, process-based database system. Client connections are normally handled by backend processes that execute queries using shared memory such as shared buffers and WAL buffers, while background processes and persistent storage support caching, durability, recovery, and maintenance. Understanding this architecture explains why connection pooling, WAL, VACUUM, and query planning matter in production.**

---

[← Previous: Lesson 26 — Query Optimization](../06-performance-query-optimization/26-query-optimization.md) | [Back to Roadmap](../README.md) | [Next: Lesson 28 — MVCC →](./28-mvcc.md)