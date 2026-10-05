# Lesson 37 — PostgreSQL Configuration

## Goal of This Lesson

PostgreSQL has many configuration parameters, but a backend developer does **not** need to memorize all of them.

For production and interviews, you should understand the important groups:

~~~text
Connections
Memory
WAL + checkpoints
Query planner
Timeouts
Logging
Autovacuum
~~~

The goal is to know **what a setting controls, why it matters, and whether changing it requires a reload or restart**.

---

# 1. Where PostgreSQL Configuration Lives

The main server configuration file is commonly:

~~~text
postgresql.conf
~~~

PostgreSQL can also load settings from other configuration sources.

To ask the running server where its main configuration file is:

~~~sql
SHOW config_file;
~~~

To inspect the data directory:

~~~sql
SHOW data_directory;
~~~

Do not assume the same filesystem path on every Linux distribution, Docker image, cloud provider, or managed PostgreSQL service.

---

# 2. Inspect a Setting with SHOW

Example:

~~~sql
SHOW max_connections;
~~~

Other examples:

~~~sql
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW effective_cache_size;
SHOW statement_timeout;
~~~

`SHOW` is one of the easiest ways to inspect the active value.

---

# 3. Inspect Many Settings

~~~sql
SHOW ALL;
~~~

This returns PostgreSQL configuration parameters visible to the current session.

For deeper inspection, PostgreSQL exposes:

~~~sql
SELECT name, setting, unit, context, source
FROM pg_settings
ORDER BY name;
~~~

`pg_settings` is very useful because it shows both the value and metadata about each setting.

---

# 4. Configuration Sources

A setting can come from different places, conceptually including:

~~~text
compiled/default value
postgresql.conf
included config files
ALTER SYSTEM
database-level setting
role-level setting
session setting
transaction-local setting
command-line startup options
~~~

When debugging a surprising value, ask:

> **Where did this active setting come from?**

`pg_settings.source` can help answer that question.

---

# 5. SET — Session-Level Configuration

Example:

~~~sql
SET statement_timeout = '5s';
~~~

This changes the setting for the current database session, subject to the parameter's allowed context.

Check it:

~~~sql
SHOW statement_timeout;
~~~

When the session ends, a normal session-level `SET` does not permanently rewrite the server configuration.

---

# 6. SET LOCAL — Transaction-Local Setting

Inside a transaction:

~~~sql
BEGIN;

SET LOCAL statement_timeout = '2s';

SELECT ...;

COMMIT;
~~~

`SET LOCAL` applies only for the current transaction.

This is useful when one operation needs a stricter setting without changing the entire application/session.

---

# 7. ALTER SYSTEM

PostgreSQL can persist server-wide configuration changes using:

~~~sql
ALTER SYSTEM SET statement_timeout = '10s';
~~~

`ALTER SYSTEM` writes settings into PostgreSQL's auto-configuration file, commonly `postgresql.auto.conf`.

Then the setting may require either:

~~~text
configuration reload
or
server restart
~~~

depending on the parameter.

Do not manually edit `postgresql.auto.conf`; let PostgreSQL manage it.

---

# 8. Reload vs Restart

Not every setting can change while PostgreSQL is running.

Three useful categories:

~~~text
Session-changeable
→ can often SET in a session

Reloadable
→ configuration reload is enough

Restart-required
→ PostgreSQL server must restart
~~~

Example reload:

~~~sql
SELECT pg_reload_conf();
~~~

Whether a parameter needs restart can be inspected through `pg_settings.context` and related metadata.

---

# 9. pg_settings.pending_restart

Useful query:

~~~sql
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart;
~~~

This helps identify settings whose changed configuration value will not fully take effect until PostgreSQL restarts.

Production lesson:

> Do not casually restart a production database merely to experiment with a setting.

---

# 10. max_connections

~~~sql
SHOW max_connections;
~~~

`max_connections` limits the number of concurrent database connections PostgreSQL allows.

Beginner mistake:

~~~text
More connections
=
More performance
~~~

This is false.

Each connection consumes server resources, and excessive concurrency can reduce performance.

Better architecture:

~~~text
Many application requests
        ↓
Connection pool
        ↓
Controlled DB connections
        ↓
PostgreSQL
~~~

Connection pooling is covered deeply in Lesson 38.

---

# 11. Why max_connections Affects Memory

PostgreSQL has both shared memory and per-session/per-operation memory behavior.

If you configure:

~~~text
very high max_connections
       +
large per-operation memory limits
~~~

the theoretical memory demand can become dangerous.

Do not tune memory parameters independently of connection concurrency.

---

# 12. shared_buffers

~~~sql
SHOW shared_buffers;
~~~

`shared_buffers` controls PostgreSQL's main shared buffer cache.

Conceptually:

~~~text
Query
 ↓
PostgreSQL shared buffers
 ↓ cache miss
Operating system / storage
~~~

Frequently accessed table/index pages can remain in shared buffers.

Important: PostgreSQL also benefits from the operating system's filesystem cache, so `shared_buffers` is not the only cache involved.

---

# 13. Do Not Use a Universal shared_buffers Number

You may hear rules such as:

~~~text
Set shared_buffers to exactly 25% of RAM
~~~

Treat such numbers as starting heuristics, not universal laws.

The appropriate value depends on:

~~~text
server memory
workload
operating system
other PostgreSQL settings
other processes
managed-provider architecture
~~~

Measure rather than blindly copying tuning recipes.

---

# 14. work_mem

~~~sql
SHOW work_mem;
~~~

`work_mem` is a memory limit used by certain query operations such as sorts and hash operations before they may spill to temporary disk files.

Conceptually:

~~~text
ORDER BY / hash join / aggregation
             ↓
        work memory
             ↓ if insufficient
       temporary disk work
~~~

Important:

> `work_mem` is **not simply one allocation per database connection**.

A single query can have multiple memory-consuming plan nodes, and parallel execution can multiply memory usage.

Therefore setting it extremely high globally can be dangerous.

---

# 15. work_mem Example

Suppose a plan performs:

~~~text
Sort A
Hash Join
Sort B
~~~

Several operations may each be eligible to consume work memory.

Now multiply this by:

~~~text
many concurrent queries
×
multiple application connections
~~~

This is why production memory tuning requires workload awareness.

---

# 16. maintenance_work_mem

~~~sql
SHOW maintenance_work_mem;
~~~

This memory setting is used for maintenance operations such as:

~~~text
VACUUM
CREATE INDEX
some ALTER TABLE operations
~~~

It can often be larger than ordinary `work_mem` because maintenance jobs are less frequent than every-query operations.

Autovacuum has related memory behavior/settings, so do not assume every maintenance worker always uses the global value in exactly the same way.

---

# 17. effective_cache_size

~~~sql
SHOW effective_cache_size;
~~~

`effective_cache_size` is **not a memory allocation**.

It is a planner estimate of how much memory is likely available for caching data, including PostgreSQL and operating-system caching effects.

~~~text
effective_cache_size
      ↓
planner assumption
      ↓
helps estimate whether index access is likely to benefit from cache
~~~

Changing it does not reserve that amount of RAM.

---

# 18. shared_buffers vs effective_cache_size

Interview distinction:

~~~text
shared_buffers
→ actual PostgreSQL shared buffer allocation

effective_cache_size
→ planner estimate, not allocated memory
~~~

This is frequently misunderstood.

---

# 19. WAL Configuration

From Lesson 30:

~~~text
Data change
   ↓
WAL record
   ↓
WAL durability
   ↓
data pages can be flushed later
~~~

Important WAL-related settings include concepts such as:

~~~text
wal_level
wal_buffers
synchronous_commit
checkpoint settings
max_wal_size
min_wal_size
~~~

You do not need to memorize every default value.

Understand their purpose.

---

# 20. wal_level

`wal_level` controls how much information PostgreSQL writes to WAL for features such as crash recovery, replication, and logical decoding.

Different replication architectures may require different levels.

Do not change it casually on a production system because replication/recovery requirements depend on it.

---

# 21. wal_buffers

`wal_buffers` controls shared memory used for WAL data that has not yet been written out to WAL storage.

Conceptually:

~~~text
Backend processes
      ↓
WAL buffers
      ↓
WAL files
~~~

PostgreSQL's automatic/default behavior is often appropriate unless measurement indicates a specific need.

---

# 22. synchronous_commit

~~~sql
SHOW synchronous_commit;
~~~

This setting affects when PostgreSQL reports a transaction commit as successful relative to WAL durability/replication requirements.

With the normal durable behavior, a transaction waits for the required WAL flush before success is reported.

Relaxing `synchronous_commit` can reduce commit latency in some workloads but can allow a small window of recently acknowledged transactions to be lost after certain crashes.

Important:

> Changing durability settings is a business/reliability decision, not merely a performance trick.

---

# 23. fsync

`fsync` is fundamental to PostgreSQL durability.

When enabled, PostgreSQL attempts to ensure required updates reach durable storage according to its WAL/recovery guarantees.

Disabling it can improve benchmark speed but risks unrecoverable corruption/data loss after crashes.

Production rule:

> Do not disable `fsync` for a real durable production database just to gain speed.

---

# 24. Checkpoints

A checkpoint establishes a recovery point and causes dirty buffers to be written as part of PostgreSQL's checkpoint process.

Relevant configuration includes concepts such as:

~~~text
checkpoint_timeout
checkpoint_completion_target
max_wal_size
~~~

Too-frequent checkpoints can increase write pressure.

Too-infrequent/large checkpoint behavior can increase recovery/WAL/storage considerations.

PostgreSQL tuning balances these tradeoffs.

---

# 25. checkpoint_timeout

This places a time-based upper bound between automatic checkpoints, subject to other checkpoint triggers.

Do not think:

~~~text
checkpoint every N minutes only
~~~

because WAL volume and administrative actions can also cause checkpoints.

---

# 26. checkpoint_completion_target

This influences how checkpoint write work is spread across the checkpoint interval.

Conceptually:

~~~text
Bad pattern
→ huge burst of writes

Smoother checkpoint
→ spread writes over time
~~~

Modern defaults are designed to avoid unnecessarily aggressive bursts, but workload measurement still matters.

---

# 27. max_wal_size

`max_wal_size` is a soft limit influencing when checkpoints are triggered because of WAL volume.

It is not simply:

~~~text
maximum possible WAL directory size
~~~

Replication slots, archiving failures, and other conditions can cause WAL retention beyond what a beginner might expect from this setting.

---

# 28. Planner Cost Settings

PostgreSQL's planner compares estimated costs.

Settings include:

~~~text
random_page_cost
seq_page_cost
cpu_tuple_cost
cpu_index_tuple_cost
~~~

These influence planner estimates.

Do not change them because one query used a sequential scan.

First inspect:

~~~text
query
statistics
selectivity
indexes
EXPLAIN ANALYZE
actual workload/storage
~~~

Planner cost tuning is advanced and should be evidence-based.

---

# 29. effective_io_concurrency

This setting can help PostgreSQL model/use concurrent I/O behavior for supported operations and storage environments.

The useful value depends heavily on storage characteristics.

Do not copy a value from an SSD tuning blog without understanding your infrastructure.

---

# 30. Statistics and default_statistics_target

PostgreSQL's planner depends on table statistics.

~~~sql
SHOW default_statistics_target;
~~~

A higher statistics target can improve estimates for difficult data distributions but increases ANALYZE work and statistics size.

Usually:

~~~text
bad row estimate
   ↓
inspect statistics/data distribution
   ↓
consider targeted statistics improvements
~~~

rather than globally maximizing the setting.

---

# 31. statement_timeout

~~~sql
SHOW statement_timeout;
~~~

This limits how long a statement may run before PostgreSQL cancels it.

Example:

~~~sql
SET statement_timeout = '5s';
~~~

Useful for preventing unexpectedly long queries from consuming resources indefinitely.

However, choose timeouts based on workload:

~~~text
API query
→ may need strict timeout

analytics/report
→ may legitimately run longer
~~~

---

# 32. lock_timeout

~~~sql
SHOW lock_timeout;
~~~

This limits how long a statement waits to acquire a lock.

From Lesson 22:

~~~text
Transaction A holds lock
        ↓
Transaction B waits
        ↓
lock_timeout reached
        ↓
B gets error
~~~

This can prevent requests from waiting indefinitely behind lock contention.

---

# 33. idle_in_transaction_session_timeout

This is extremely useful to recognize.

It can terminate sessions that remain idle while a transaction is still open for too long.

Why dangerous?

~~~text
BEGIN
 ↓
query
 ↓
application forgets COMMIT/ROLLBACK
 ↓
transaction sits idle
 ↓
old snapshot / locks / cleanup problems
~~~

Long idle transactions can interfere with VACUUM and hold locks/resources.

---

# 34. idle_session_timeout

This can terminate sessions that remain idle outside a transaction for too long.

Be careful when using it with connection pools, because the pool and database must agree on connection lifecycle behavior.

Do not enable aggressive values without understanding your application's pooling strategy.

---

# 35. Logging

PostgreSQL can log important operational information.

Relevant settings include concepts such as:

~~~text
log_destination
logging_collector
log_min_duration_statement
log_statement
log_connections
log_disconnections
log_lock_waits
log_line_prefix
~~~

Logging helps answer:

~~~text
Which queries are slow?
Are clients failing to connect?
Are lock waits occurring?
Which database/user/session produced an event?
~~~

---

# 36. log_min_duration_statement

This logs statements whose execution duration exceeds a threshold.

Conceptual example:

~~~text
log_min_duration_statement = 500ms
~~~

Then queries exceeding the configured duration can be logged.

This can be more practical than logging every SQL statement in a busy production system.

---

# 37. log_statement

`log_statement` can log SQL statements according to configured categories.

Be careful:

~~~text
More logging
→ more visibility
but also
→ more I/O
→ larger logs
→ possible sensitive-data exposure
~~~

Do not blindly enable maximum SQL logging in production.

---

# 38. log_lock_waits

This helps identify sessions waiting unusually long for locks.

Combined with `deadlock_timeout`, PostgreSQL can log lock waits useful for concurrency diagnosis.

From an application perspective:

~~~text
slow request
   ↓
not always slow SQL computation
   ↓
could be lock waiting
~~~

---

# 39. Autovacuum Configuration

From Lesson 29, autovacuum is essential for PostgreSQL health.

Important settings include concepts such as:

~~~text
autovacuum
autovacuum_max_workers
autovacuum_naptime
autovacuum_vacuum_threshold
autovacuum_vacuum_scale_factor
autovacuum_analyze_threshold
autovacuum_analyze_scale_factor
~~~

Do not disable autovacuum globally as a normal performance fix.

---

# 40. Why Large Tables Need Thoughtful Autovacuum Tuning

Scale-factor thresholds depend partly on table size.

For a very large, frequently updated table:

~~~text
large table
×
scale factor
=
many changed/dead tuples before trigger
~~~

High-write production tables sometimes benefit from per-table autovacuum tuning.

Example concept:

~~~sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.05
 );
~~~

Do this based on observed workload/bloat, not blindly.

---

# 41. Per-Database Settings

You can set some parameters for a database:

~~~sql
ALTER DATABASE shophub
SET statement_timeout = '10s';
~~~

New sessions connecting to that database can inherit the configured setting, subject to PostgreSQL precedence rules.

This can be useful when workloads differ between databases.

---

# 42. Per-Role Settings

Example:

~~~sql
ALTER ROLE reporting_user
SET statement_timeout = '60s';
~~~

Now reporting sessions can have a different default than your web application role.

Architecture:

~~~text
runtime_app
→ 5s timeout

reporting_user
→ 60s timeout
~~~

This can be cleaner than one global value for every workload.

---

# 43. Role + Database Settings

PostgreSQL can also scope settings to a role in a specific database.

Conceptually:

~~~text
role
   +
database
   ↓
specific default configuration
~~~

This is useful when one role behaves differently across environments/databases.

---

# 44. Configuration Precedence — High Level

Different configuration sources can override others.

Do not memorize the full precedence hierarchy for interviews.

Remember:

~~~text
server defaults/config
      ↓
database/role defaults
      ↓
session SET
      ↓
transaction-local SET LOCAL
~~~

with exact precedence depending on the setting/source.

When confused, inspect `pg_settings` and the active session.

---

# 45. Configuration in Managed PostgreSQL

On managed platforms you may not have direct filesystem access to `postgresql.conf`.

Instead, configuration may be exposed through:

~~~text
provider dashboard
parameter groups
SQL commands
provider API
restricted setting subset
~~~

Always follow the provider's supported mechanism.

Do not assume a cloud database behaves exactly like your local Ubuntu installation.

---

# 46. Configuration in Docker

In Docker, configuration can be supplied through mechanisms such as:

~~~text
mounted config files
container command options
environment-driven image behavior
SQL ALTER SYSTEM / ALTER ROLE / ALTER DATABASE
~~~

Example architecture:

~~~text
docker-compose.yml
      ↓
PostgreSQL container
      ↓
postgresql.conf / command settings
      ↓
running server
~~~

Persist database data separately from disposable container filesystem layers.

---

# 47. Do Not Tune Before Measuring

Bad workflow:

~~~text
Read tuning blog
   ↓
change 20 parameters
   ↓
hope performance improves
~~~

Better:

~~~text
Observe problem
   ↓
Measure workload
   ↓
EXPLAIN ANALYZE / metrics / logs
   ↓
identify bottleneck
   ↓
change one justified setting/design
   ↓
measure again
~~~

Many performance problems are caused by:

~~~text
bad query
missing/wrong index
N+1 requests
too many connections
lock contention
poor schema design
~~~

not a magical missing configuration value.

---

# 48. ShopHub Production Example

Suppose ShopHub has:

~~~text
3 application instances
each pool max = 20
~~~

Potential application connections:

~~~text
3 × 20 = 60
~~~

Now compare that with:

~~~text
PostgreSQL max_connections
other admin connections
migration jobs
monitoring connections
provider reserved connections
~~~

This is why application pool sizing and PostgreSQL configuration must be designed together.

---

# 49. CareerLoop Example

For CareerLoop using a managed PostgreSQL provider:

~~~text
Next.js deployment
      ↓
provider/serverless connection strategy
      ↓
managed PostgreSQL
~~~

Do not start by manually changing dozens of server parameters.

Start with:

~~~text
correct connection strategy
good schema
indexes
bounded queries
transactions
provider metrics
~~~

Then tune configuration only when the workload demonstrates a need.

---

# Common Mistakes

## 50. Increasing max_connections to Fix Connection Problems

Often the real solution is better pooling/concurrency control.

## 51. Setting work_mem Extremely High

It can be consumed by multiple operations across concurrent queries, creating dangerous aggregate memory demand.

## 52. Treating effective_cache_size as Allocated RAM

It is a planner estimate, not a memory reservation.

## 53. Disabling fsync for Production Speed

This sacrifices fundamental durability guarantees and can risk corruption/data loss.

## 54. Disabling Autovacuum

Autovacuum is essential for tuple cleanup, statistics, and transaction-ID safety.

## 55. Logging Everything Forever

Excessive logging creates I/O/storage overhead and may expose sensitive values.

## 56. Changing Planner Costs Without EXPLAIN Evidence

Fix query/index/statistics problems first.

## 57. Forgetting Restart Requirements

Some settings cannot take effect with a reload alone.

## 58. Copying Configuration from Another Server

Different RAM, storage, concurrency, PostgreSQL versions, and workloads require different tuning.

---

# Interview Revision

## What is postgresql.conf?

The main PostgreSQL server configuration file containing parameters for connections, memory, WAL, logging, planner behavior, autovacuum, and other server features.

## How do you inspect a setting?

Use `SHOW setting_name` or query `pg_settings`.

## What is ALTER SYSTEM?

A SQL command for persisting server-wide configuration changes into PostgreSQL's auto-configuration file.

## Reload vs restart?

Some settings can change per session, some take effect after configuration reload, and some require a server restart.

## What is max_connections?

The maximum allowed concurrent PostgreSQL connections. Increasing it is not automatically a performance improvement.

## What is shared_buffers?

PostgreSQL's main shared memory cache for database pages.

## What is work_mem?

A limit used by individual query operations such as sorts/hashes before spilling to temporary disk; multiple operations and concurrent queries can each consume it.

## What is maintenance_work_mem?

Memory available for maintenance operations such as VACUUM and CREATE INDEX.

## What is effective_cache_size?

A planner estimate of available cache, not allocated memory.

## What is statement_timeout?

A limit on how long a statement may run before PostgreSQL cancels it.

## What is lock_timeout?

A limit on how long a statement waits to acquire a lock.

## Why is idle_in_transaction_session_timeout useful?

It can terminate sessions that sit idle inside open transactions, reducing long-held locks/snapshots and vacuum interference.

## Should autovacuum be disabled for performance?

Normally no. It is essential to PostgreSQL maintenance and transaction-ID safety.

## How should PostgreSQL be tuned?

Measure the workload first, identify the bottleneck, make targeted changes, and measure again.

---

# Quick Revision

~~~text
POSTGRESQL CONFIGURATION
        │
        ├── Connections
        │    └── max_connections
        │
        ├── Memory
        │    ├── shared_buffers
        │    ├── work_mem
        │    ├── maintenance_work_mem
        │    └── effective_cache_size
        │
        ├── WAL / Checkpoints
        │    ├── wal_level
        │    ├── synchronous_commit
        │    ├── checkpoint_timeout
        │    └── max_wal_size
        │
        ├── Timeouts
        │    ├── statement_timeout
        │    ├── lock_timeout
        │    └── idle_in_transaction_session_timeout
        │
        ├── Logging
        │    ├── log_min_duration_statement
        │    └── log_lock_waits
        │
        └── Maintenance
             └── autovacuum
~~~

### Memory

~~~text
shared_buffers
→ actual PostgreSQL shared cache

work_mem
→ per query operation limit

effective_cache_size
→ planner estimate only
~~~

### Safe Tuning Flow

~~~text
Measure
 ↓
Find bottleneck
 ↓
Change one justified thing
 ↓
Measure again
~~~

### Golden Rule

~~~text
Do not tune PostgreSQL by superstition.
Tune from workload evidence.
~~~

---

## Key Takeaway

> **PostgreSQL configuration controls connection capacity, memory, WAL/checkpoint behavior, planner assumptions, timeouts, logging, and maintenance. Learn the purpose of the important settings rather than memorizing defaults. In production, tune from measurements, coordinate database settings with application connection pools and workload, and never sacrifice durability or autovacuum health for an unmeasured performance guess.**

---

[← Previous: Lesson 36 — SQL Injection & Secure Queries](../09-security/36-sql-injection-and-secure-queries.md) | [Back to Roadmap](../README.md) | [Next: Lesson 38 — Connection Pooling & PgBouncer →](./38-connection-pooling-and-pgbouncer.md)