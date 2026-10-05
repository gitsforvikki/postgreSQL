# Lesson 42 — PostgreSQL Monitoring & Logging

## Goal of This Lesson

A production database must be observable.

You should be able to answer questions such as:

~~~text
Is PostgreSQL healthy?
Why is the API slow?
Which queries consume the most time?
Are connections exhausted?
Are transactions stuck?
Are queries waiting for locks?
Is a replica lagging?
Is autovacuum keeping up?
Is storage growing dangerously?
~~~

Monitoring tells you **what is happening**.

Logging gives you a historical record of **events and statements worth investigating**.

---

# 1. Observability Mental Model

Think in four layers:

~~~text
Application metrics
      ↓
PostgreSQL activity/statistics
      ↓
PostgreSQL logs
      ↓
Host / cloud infrastructure metrics
~~~

A slow request can originate from any of these layers.

Do not assume every slow API request means PostgreSQL itself is slow.

---

# 2. What Should We Monitor?

Core categories:

~~~text
Availability
Connections
Query performance
Transactions
Locks
CPU / memory / I/O
Cache behavior
Table/index activity
Vacuum / analyze
WAL / checkpoints
Replication
Disk/storage
Errors
Backups
~~~

Do not collect metrics merely because they exist.

Collect metrics that help detect or diagnose real failures.

---

# 3. pg_stat_activity

`pg_stat_activity` is one of the most important PostgreSQL monitoring views.

Example:

~~~sql
SELECT
  pid,
  usename,
  datname,
  application_name,
  client_addr,
  state,
  query_start,
  wait_event_type,
  wait_event,
  query
FROM pg_stat_activity;
~~~

It shows information about PostgreSQL server processes/sessions.

---

# 4. Important pg_stat_activity States

Common states include:

~~~text
active
idle
idle in transaction
idle in transaction (aborted)
~~~

Interpretation:

~~~text
active
→ currently executing a query

idle
→ waiting for next client command

idle in transaction
→ transaction remains open but client is currently doing nothing
~~~

`idle in transaction` deserves special attention.

---

# 5. Why Idle in Transaction Is Dangerous

Example:

~~~text
BEGIN
 ↓
SELECT / UPDATE
 ↓
application forgets COMMIT or ROLLBACK
 ↓
connection sits idle
~~~

Possible effects:

~~~text
locks remain held
old MVCC snapshot remains relevant
VACUUM cleanup can be delayed
connections remain occupied
table bloat can worsen
~~~

From Lesson 37, `idle_in_transaction_session_timeout` can help limit this problem.

---

# 6. Find Long-Running Queries

Example:

~~~sql
SELECT
  pid,
  now() - query_start AS duration,
  state,
  wait_event_type,
  wait_event,
  query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
~~~

This can reveal long-running active statements.

But:

> Long-running does not automatically mean bad.

A legitimate report may run longer than an API lookup.

Always interpret duration in workload context.

---

# 7. Query Waiting vs Query Computing

A query can be slow because it is:

~~~text
using CPU
reading storage
sorting/hashing
waiting for a lock
waiting for client/network
waiting for another resource
~~~

`wait_event_type` and `wait_event` help identify what a backend is waiting on.

Do not optimize SQL blindly before checking whether it is actually blocked.

---

# 8. Lock Monitoring

PostgreSQL exposes locks through:

~~~sql
SELECT * FROM pg_locks;
~~~

`pg_locks` is most useful when joined with activity information.

High-level question:

~~~text
Which session is waiting?
      ↓
Which lock does it need?
      ↓
Which session is blocking it?
~~~

---

# 9. Find Blocking Sessions — Conceptual Query

PostgreSQL provides `pg_blocking_pids(pid)` to identify blockers.

Example:

~~~sql
SELECT
  pid,
  pg_blocking_pids(pid) AS blocking_pids,
  query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
~~~

This quickly identifies sessions currently blocked by another backend.

Then inspect both blocked and blocking transactions before deciding to terminate anything.

---

# 10. Never Kill Sessions Blindly

PostgreSQL provides administrative functions such as:

~~~sql
SELECT pg_cancel_backend(pid);
SELECT pg_terminate_backend(pid);
~~~

Difference at a high level:

~~~text
pg_cancel_backend
→ requests cancellation of the current query

pg_terminate_backend
→ terminates the database session
~~~

These require appropriate privileges.

Production rule:

> Understand the session, transaction, and business impact before cancelling or terminating it.

---

# 11. Connection Monitoring

Count connections by state:

~~~sql
SELECT
  state,
  COUNT(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY COUNT(*) DESC;
~~~

Useful questions:

~~~text
How many connections exist?
How many are active?
How many are idle?
How many are idle in transaction?
Are we approaching max_connections?
~~~

---

# 12. Connections by Application

If clients set `application_name`, you can group activity:

~~~sql
SELECT
  application_name,
  COUNT(*)
FROM pg_stat_activity
GROUP BY application_name
ORDER BY COUNT(*) DESC;
~~~

This helps distinguish:

~~~text
web API
background worker
migration job
admin tool
reporting service
~~~

Use meaningful application names when your driver/deployment supports them.

---

# 13. pg_stat_database

`pg_stat_database` provides database-level statistics.

Example:

~~~sql
SELECT
  datname,
  numbackends,
  xact_commit,
  xact_rollback,
  blks_read,
  blks_hit,
  tup_returned,
  tup_fetched,
  tup_inserted,
  tup_updated,
  tup_deleted
FROM pg_stat_database;
~~~

This gives a broad view of database activity.

---

# 14. Commit vs Rollback Rate

Useful signals:

~~~text
xact_commit
xact_rollback
~~~

A sudden increase in rollbacks may indicate:

~~~text
application errors
constraint failures
deadlocks
serialization failures
timeouts
failed business transactions
~~~

Investigate changes relative to normal baseline.

---

# 15. Cache Hit Statistics

`blks_hit` means requested blocks were found in PostgreSQL's shared buffer cache.

`blks_read` means PostgreSQL had to request blocks not already present in shared buffers.

A rough shared-buffer hit ratio can be calculated conceptually as:

~~~text
blks_hit
--------------------
blks_hit + blks_read
~~~

But do not treat one ratio as a universal health score.

The operating-system cache also matters, and workloads such as large sequential scans can naturally have different behavior.

---

# 16. pg_stat_user_tables

This view provides statistics for user tables.

Example:

~~~sql
SELECT
  relname,
  seq_scan,
  idx_scan,
  n_live_tup,
  n_dead_tup,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
~~~

It helps inspect:

~~~text
sequential scans
index scans
estimated live/dead tuples
vacuum activity
analyze activity
~~~

---

# 17. Sequential Scans Are Not Automatically Bad

A common mistake:

~~~text
seq_scan > 0
→ problem
~~~

False.

Sequential scans are often correct for:

~~~text
small tables
queries reading a large percentage of a table
analytics
low-selectivity filters
~~~

Use `EXPLAIN ANALYZE` for specific slow queries rather than declaring every sequential scan a problem.

---

# 18. Dead Tuple Monitoring

`n_dead_tup` provides an estimate of dead tuples.

From Lessons 28–29:

~~~text
UPDATE / DELETE
      ↓
old tuple versions
      ↓
VACUUM eventually makes reusable space
~~~

If dead tuples continually grow and autovacuum does not keep up, investigate:

~~~text
autovacuum settings
long-running transactions
high write rate
table-specific thresholds
vacuum progress
~~~

---

# 19. pg_stat_user_indexes

Example:

~~~sql
SELECT
  relname AS table_name,
  indexrelname AS index_name,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
~~~

This helps inspect index usage.

But:

> An index with zero scans is not automatically safe to drop.

It may support:

~~~text
rare critical queries
monthly jobs
constraints
recently deployed features
failover/recovery workloads
~~~

Observe over a representative period before removing indexes.

---

# 20. pg_stat_statements

`pg_stat_statements` is one of PostgreSQL's most valuable performance extensions.

It tracks execution statistics for normalized SQL statements.

Conceptually:

~~~text
Thousands of query executions
      ↓
pg_stat_statements
      ↓
aggregated statistics by statement
~~~

It helps answer:

~~~text
Which query consumes the most total time?
Which query runs most often?
Which query has high average latency?
Which queries return many rows?
~~~

---

# 21. Enable pg_stat_statements — High Level

Typical setup involves loading the extension through PostgreSQL configuration and then creating the extension in the database.

Example database command:

~~~sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
~~~

Depending on PostgreSQL setup, `shared_preload_libraries` may need to include `pg_stat_statements`, which requires a server restart.

Managed providers may expose a provider-specific way to enable it.

---

# 22. Find Expensive Queries

Example pattern:

~~~sql
SELECT
  calls,
  total_exec_time,
  mean_exec_time,
  rows,
  query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
~~~

This finds queries consuming the most cumulative execution time.

Important distinction:

~~~text
One query taking 5 seconds once
vs
One query taking 20 ms × 1,000,000 calls
~~~

The second query may consume much more total database capacity.

---

# 23. Total Time vs Mean Time

Monitor both:

~~~text
total_exec_time
→ cumulative cost to system

mean_exec_time
→ average latency per execution

calls
→ frequency
~~~

Optimization priority often depends on all three.

Example:

~~~text
Query A: 2s average × 5 calls
Query B: 30ms average × 500,000 calls
~~~

Query B may deserve attention first because of total resource usage.

---

# 24. Query Normalization

`pg_stat_statements` groups structurally similar statements rather than storing every literal value as a completely separate entry.

Conceptually:

~~~sql
SELECT * FROM users WHERE id = 10;
SELECT * FROM users WHERE id = 20;
~~~

can be tracked as the same normalized query pattern.

This makes workload-level analysis practical.

---

# 25. Statistics Reset

PostgreSQL statistics can be reset manually or as part of operational events.

When reading counters, always know:

~~~text
Since when have these statistics accumulated?
~~~

Do not compare raw cumulative counters between servers/time periods without understanding the observation window.

Monitoring systems usually convert counters into rates over time.

---

# 26. EXPLAIN ANALYZE Still Matters

`pg_stat_statements` identifies important queries.

`EXPLAIN (ANALYZE, BUFFERS)` explains why a specific query behaves as it does.

Workflow:

~~~text
Monitoring detects high DB latency
      ↓
pg_stat_statements finds expensive query
      ↓
EXPLAIN ANALYZE + BUFFERS
      ↓
identify scan/join/sort/estimate issue
      ↓
optimize
      ↓
monitor result
~~~

This connects Lessons 25, 26, and 42.

---

# 27. Checkpoint Monitoring

Checkpoint behavior can create write pressure.

Modern PostgreSQL exposes checkpoint/background-writer statistics through statistics views whose exact organization can vary by PostgreSQL version.

Monitor concepts such as:

~~~text
checkpoint frequency
checkpoint duration
buffers written
WAL volume
storage write latency
~~~

If checkpoints occur excessively because WAL limits are reached, configuration/workload may need investigation.

---

# 28. WAL Monitoring

Useful questions:

~~~text
How quickly is WAL being generated?
Is WAL storage growing?
Are replication slots retaining WAL?
Is archiving succeeding?
~~~

High WAL volume can come from:

~~~text
heavy INSERT/UPDATE/DELETE workload
bulk imports
index maintenance
vacuum-related page changes
schema/data maintenance
~~~

Interpret WAL rate in context.

---

# 29. Replication Monitoring

From Lesson 40, primary-side view:

~~~sql
SELECT
  application_name,
  state,
  sync_state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn
FROM pg_stat_replication;
~~~

Monitor:

~~~text
replica connected?
WAL being sent?
WAL received/flushed?
WAL replayed?
lag increasing?
~~~

---

# 30. Replication Slots

Inspect:

~~~sql
SELECT
  slot_name,
  slot_type,
  active,
  restart_lsn
FROM pg_replication_slots;
~~~

An inactive slot can retain WAL.

Alert before retained WAL fills storage.

---

# 31. Autovacuum Monitoring

Questions:

~~~text
When was table last autovacuumed?
Are dead tuples increasing?
Are workers keeping up?
Are long transactions preventing cleanup?
~~~

Use statistics views plus logs/monitoring.

For currently running vacuum operations, PostgreSQL provides progress views such as `pg_stat_progress_vacuum`.

---

# 32. Vacuum Progress

Example:

~~~sql
SELECT * FROM pg_stat_progress_vacuum;
~~~

This can show information about currently running VACUUM operations.

Useful when a large table vacuum appears to be taking significant time.

Do not terminate maintenance merely because it is long-running without understanding why it is needed.

---

# 33. Index Creation Progress

PostgreSQL provides progress reporting for operations such as `CREATE INDEX` through views such as:

~~~sql
SELECT * FROM pg_stat_progress_create_index;
~~~

This is useful during large production migrations/index builds.

Progress views help distinguish:

~~~text
operation is progressing
vs
operation appears stuck
~~~

---

# 34. Database Size

Check database size:

~~~sql
SELECT pg_size_pretty(pg_database_size(current_database()));
~~~

Table size:

~~~sql
SELECT pg_size_pretty(pg_total_relation_size('orders'));
~~~

`pg_total_relation_size` includes associated storage such as indexes and TOAST data for the relation.

Storage growth should be monitored as a trend, not only checked after disk is nearly full.

---

# 35. Largest Tables

Example:

~~~sql
SELECT
  relname,
  pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
~~~

This helps identify where storage is concentrated.

Large size is not automatically bloat—it may simply be legitimate data/index growth.

---

# 36. Bloat — High Level

Because PostgreSQL uses MVCC, updates/deletes create old tuple versions.

VACUUM makes reusable space available inside tables, but normal VACUUM generally does not shrink the physical table file back to the operating system.

Over time, inefficiently reusable space can contribute to **bloat**.

Symptoms may include:

~~~text
table/index much larger than expected
increased I/O
slower scans
more cache pressure
~~~

Accurate bloat analysis requires more than looking at `n_dead_tup` alone.

---

# 37. Logging Configuration

Important PostgreSQL logging settings include concepts such as:

~~~text
log_destination
logging_collector
log_line_prefix
log_min_messages
log_min_error_statement
log_min_duration_statement
log_statement
log_connections
log_disconnections
log_lock_waits
log_checkpoints
~~~

Exact availability/defaults can vary by PostgreSQL version/platform.

Use `SHOW` or `pg_settings` to inspect the running server.

---

# 38. log_min_duration_statement

This logs completed statements whose duration is at or above the configured threshold.

Example concept:

~~~text
log_min_duration_statement = 500ms
~~~

Then statements taking at least the threshold can be logged.

This is useful for slow-query investigation.

Be careful with sensitive SQL values in logs.

---

# 39. log_statement

`log_statement` can log SQL statements according to configured categories.

Possible tradeoff:

~~~text
more visibility
      vs
more log volume
more I/O
sensitive-data exposure
~~~

Logging every statement on a busy production system can be expensive and noisy.

Prefer targeted observability based on requirements.

---

# 40. log_lock_waits

When enabled, PostgreSQL can log sessions waiting longer than `deadlock_timeout` for a lock.

Useful flow:

~~~text
API latency spike
    ↓
log shows lock wait
    ↓
inspect blocker
    ↓
find long transaction / conflicting update
~~~

This prevents wasting time optimizing an index when the real issue is lock contention.

---

# 41. Deadlock Logging

PostgreSQL detects deadlocks and aborts one transaction.

Logs can provide details about involved processes/statements.

Application monitoring should also track deadlock-related errors.

Fix patterns such as:

~~~text
inconsistent row lock ordering
transactions held too long
unnecessarily broad locking
~~~

rather than merely retrying forever.

---

# 42. log_line_prefix

`log_line_prefix` controls metadata added to PostgreSQL log lines.

Useful metadata can include concepts such as:

~~~text
timestamp
process ID
user
database
application name
session identifier
~~~

A good prefix makes logs easier to correlate across sessions.

Exact formatting tokens should be checked in your PostgreSQL version's documentation.

---

# 43. Structured Logs

Depending on PostgreSQL version/configuration, logs can be emitted in formats suitable for structured processing, such as CSV or JSON logging support.

Structured logs are useful for centralized systems:

~~~text
PostgreSQL
   ↓
structured logs
   ↓
log collector/platform
   ↓
search + dashboards + alerts
~~~

Choose the format supported by your operational tooling.

---

# 44. Centralized Logging

Production architecture:

~~~text
PostgreSQL instance(s)
      ↓
log shipping / managed collector
      ↓
central log platform
      ↓
search / dashboard / alerts
~~~

Do not rely only on SSHing into one server and reading a local log file during an incident.

Managed database platforms often centralize logs/metrics for you.

---

# 45. Sensitive Data in Logs

SQL logs can expose:

~~~text
email addresses
search values
tokens
personal data
business data
~~~

Never intentionally log:

~~~text
passwords
database credentials
payment secrets
access tokens
~~~

Use appropriate retention, redaction, and access control.

Observability must not become a security vulnerability.

---

# 46. Slow Query Monitoring Strategy

A practical production strategy:

~~~text
1. Detect endpoint/database latency
2. Identify expensive query pattern
3. Check waiting vs executing
4. Inspect pg_stat_statements
5. Run EXPLAIN ANALYZE safely
6. Inspect indexes/statistics
7. Fix one bottleneck
8. Compare metrics after deployment
~~~

Monitoring closes the optimization feedback loop.

---

# 47. CPU Monitoring

High CPU may indicate:

~~~text
expensive queries
too much concurrency
large joins/sorts
inefficient functions
missing indexes
parallel workloads
background maintenance
~~~

Do not solve high CPU by immediately adding hardware.

First identify which workload consumes it.

---

# 48. Memory Monitoring

Memory pressure can involve:

~~~text
shared_buffers
many connections
work_mem across many operations
maintenance work
operating-system cache
other processes
~~~

Remember from Lesson 37:

> `work_mem` can be used by multiple plan operations across concurrent queries.

A seemingly moderate setting can create high aggregate memory demand.

---

# 49. Disk I/O Monitoring

Watch:

~~~text
read latency
write latency
IOPS
throughput
queueing/saturation
temporary-file activity
checkpoint write behavior
~~~

A query may have good CPU usage but still be slow because storage is saturated.

Use database metrics together with host/cloud storage metrics.

---

# 50. Temporary Files

Large sorts/hashes may spill to temporary disk when memory is insufficient.

PostgreSQL statistics/logging can help observe temporary-file usage.

Conceptually:

~~~text
Sort / Hash
   ↓
work_mem insufficient
   ↓
temporary file
   ↓
disk I/O
~~~

Do not automatically solve this by globally increasing `work_mem`; inspect query and concurrency first.

---

# 51. Monitoring Checkpoints and WAL Together

Heavy writes can create:

~~~text
more WAL
   ↓
checkpoint pressure
   ↓
more storage writes
~~~

If latency spikes correlate with checkpoint/storage activity, investigate:

~~~text
WAL generation rate
checkpoint configuration
storage performance
write workload
~~~

Use evidence rather than guessing.

---

# 52. Monitoring Backups

A production dashboard should not ignore backups.

Track:

~~~text
last successful backup
backup duration
backup size
PITR/WAL archive health
restore-test status
retention
~~~

A database can be perfectly healthy today and still be operationally unsafe if backups have silently failed for a week.

---

# 53. Monitoring HA

From Lesson 41, monitor:

~~~text
current primary
healthy standby count
replication lag
failover events
timeline/role changes
proxy endpoint health
backup health
~~~

Alert when redundancy is lost even if the primary is still serving traffic.

---

# 54. Baselines

Metrics are more useful when you know what is normal.

Example:

~~~text
Normal connections: 20–35
Normal API query p95: 40ms
Normal WAL: 2 GB/hour
Normal replica lag: near zero
~~~

Then a change becomes visible:

~~~text
Connections: 90
Query p95: 900ms
WAL: 15 GB/hour
Replica lag: 4 minutes
~~~

Baseline first; alert on meaningful deviation/capacity thresholds.

---

# 55. Percentiles

Average latency can hide bad user experiences.

Example:

~~~text
Average = 50 ms
p95 = 700 ms
p99 = 2 seconds
~~~

Monitor latency distributions/percentiles for important APIs and database operations where your monitoring stack supports them.

One very slow tail can matter even when the average looks healthy.

---

# 56. Alerting Philosophy

Good alert:

~~~text
Replica lag > business-safe threshold for sustained period
~~~

Poor alert:

~~~text
one query took 101 ms once
~~~

Alerts should be:

~~~text
actionable
severity-aware
sustained enough to avoid noise
linked to runbooks
based on user/business impact where possible
~~~

Too many meaningless alerts cause alert fatigue.

---

# 57. Example Production Dashboard

A useful dashboard could show:

~~~text
Availability
  ├─ DB reachable
  └─ current primary / healthy replicas

Connections
  ├─ active
  ├─ idle
  ├─ idle in transaction
  └─ waiting pool clients

Queries
  ├─ throughput
  ├─ p95/p99 latency
  ├─ top total-time queries
  └─ errors / cancellations

Storage
  ├─ DB size
  ├─ disk free
  ├─ read/write latency
  └─ temp files

Maintenance
  ├─ dead tuples
  ├─ autovacuum activity
  └─ analyze activity

Replication
  ├─ lag
  ├─ slot retention
  └─ standby health

Recovery
  ├─ last backup
  └─ last restore test
~~~

---

# 58. ShopHub Incident Example

Problem:

~~~text
Checkout API suddenly takes 8 seconds
~~~

Investigation:

~~~text
API monitoring
   ↓
database call is slow
   ↓
pg_stat_activity
   ↓
checkout UPDATE waiting on lock
   ↓
pg_blocking_pids
   ↓
find long admin transaction
   ↓
fix transaction workflow
~~~

Notice: the solution was **not** adding an index.

Monitoring tells you what kind of problem you actually have.

---

# 59. ShopHub Query Example

Problem:

~~~text
Product listing gradually slows as data grows
~~~

Flow:

~~~text
pg_stat_statements
   ↓
listing query has high total/mean time
   ↓
EXPLAIN ANALYZE
   ↓
large scan + sort
   ↓
add appropriate composite index
   ↓
measure again
~~~

This is the full production optimization loop.

---

# 60. CareerLoop Monitoring Strategy

For a smaller managed application, do not build a huge observability platform immediately.

Start with provider/app metrics for:

~~~text
database availability
connection usage
query latency
slow queries
storage
errors
backup/PITR status
~~~

Then add deeper PostgreSQL statistics as the workload grows.

Use the managed provider's built-in monitoring before operating unnecessary infrastructure yourself.

---

# 61. Useful Production Investigation Flow

~~~text
User reports slow API
      ↓
Check application latency/error metrics
      ↓
Check DB connections and active queries
      ↓
Check wait events / blockers
      ↓
Check pg_stat_statements
      ↓
Check EXPLAIN ANALYZE for suspect query
      ↓
Check CPU / memory / disk / WAL / replication
      ↓
Apply targeted fix
      ↓
Verify metrics returned to baseline
~~~

This is much stronger than guessing.

---

# Common Mistakes

## 62. Monitoring Only CPU

Database failures can come from locks, storage, connections, replication, or long transactions even when CPU looks fine.

## 63. Treating Every Sequential Scan as Bad

Sequential scans are often the correct plan.

## 64. Looking Only at Average Query Time

Use frequency, total time, and tail latency too.

## 65. Killing Long Queries Without Investigation

The query may be legitimate or may be blocked by another transaction.

## 66. Ignoring Idle-in-Transaction Sessions

They can retain locks/snapshots and interfere with VACUUM.

## 67. Dropping Every Index with Low idx_scan

Observe over a representative workload and understand constraint/rare-query needs.

## 68. Logging Every SQL Statement Indefinitely

This can create overhead, huge log volume, and sensitive-data risk.

## 69. Monitoring Replication but Not Replication Slots

An inactive slot can fill storage with retained WAL.

## 70. Monitoring Backups but Never Testing Restore

Backup success does not prove recovery success.

## 71. Creating Alerts for Everything

Alert on actionable failures and meaningful thresholds to avoid alert fatigue.

---

# Interview Revision

## What is pg_stat_activity?

A PostgreSQL system view showing information about server processes/sessions, including state, query, timestamps, and wait events.

## Why is idle in transaction dangerous?

The session has an open transaction while doing no current work, potentially retaining locks, old snapshots, connections, and preventing cleanup.

## How do you find blocking sessions?

Inspect `pg_stat_activity`, `pg_locks`, and helpers such as `pg_blocking_pids(pid)`.

## What is pg_stat_statements?

An extension that aggregates execution statistics for normalized SQL statements, helping identify frequently executed and expensive queries.

## Total execution time vs mean execution time?

Total time shows cumulative system cost; mean time shows average cost per execution. Query frequency must also be considered.

## What is pg_stat_user_tables useful for?

Monitoring table-level scan activity, tuple estimates, dead tuples, and vacuum/analyze history.

## Is a sequential scan always bad?

No. It can be optimal for small tables or queries reading a large portion of a table.

## How do you investigate a slow query?

Use workload monitoring/`pg_stat_statements` to identify it, check waits, then use `EXPLAIN (ANALYZE, BUFFERS)` and inspect indexes/statistics/query design.

## What should be monitored for replication?

Replica connectivity, send/write/flush/replay progress, lag, slot retention, disk usage, and standby health.

## Why monitor backups?

To detect failed/stale backups and ensure the system still meets recovery objectives. Restore testing is also required.

## What is the purpose of PostgreSQL logging?

To provide operational history for errors, slow statements, lock waits, connections, checkpoints, and other events needed for diagnosis/auditing according to configuration.

---

# Quick Revision

~~~text
POSTGRESQL OBSERVABILITY

Application metrics
      ↓
pg_stat_activity
pg_stat_database
pg_stat_user_tables
pg_stat_user_indexes
pg_stat_statements
pg_locks
      ↓
PostgreSQL logs
      ↓
CPU / memory / disk / network
~~~

### Slow Query Flow

~~~text
Detect latency
   ↓
Find expensive query
   ↓
Check waits
   ↓
EXPLAIN ANALYZE + BUFFERS
   ↓
Fix query/index/schema
   ↓
Measure again
~~~

### Important Signals

~~~text
Connections
Long transactions
Lock waits
Query latency
Query frequency
Dead tuples
Autovacuum
WAL
Replication lag
Disk space
Backups
~~~

### Golden Rule

~~~text
Do not guess why PostgreSQL is slow.
Observe → identify → measure → fix → verify.
~~~

---

## Key Takeaway

> **Production PostgreSQL monitoring should cover availability, connections, query workload, transactions, locks, maintenance, WAL, replication, storage, and backups. Use `pg_stat_activity` to understand current sessions and waits, `pg_stat_statements` to identify costly workload patterns, `EXPLAIN ANALYZE` to diagnose specific SQL, and logs/infrastructure metrics to provide the missing context. Establish baselines, alert on actionable conditions, and always verify that a fix improves measured behavior.**

---

[← Previous: Lesson 41 — High Availability](./41-high-availability.md) | [Back to Roadmap](../README.md) | [Next: Lesson 43 — Production Deployment →](./43-production-deployment.md)