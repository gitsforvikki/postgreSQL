# Lesson 45 — Production Database Optimization

## Goal of This Lesson

This lesson combines the performance concepts from the entire PostgreSQL curriculum into a practical production optimization workflow.

The most important rule is:

> **Do not optimize PostgreSQL by guessing. Measure → identify → change → measure again.**

Production optimization is not simply adding indexes or increasing RAM.

It involves:

~~~text
Application
   ↓
SQL queries
   ↓
Query plans
   ↓
Indexes / schema
   ↓
Transactions / locks
   ↓
Connections
   ↓
PostgreSQL configuration
   ↓
CPU / memory / storage
~~~

---

# 1. What Does "Database Is Slow" Mean?

A slow application can have many causes:

~~~text
slow SQL
lock waits
connection pool exhaustion
too many queries
N+1 queries
large result sets
bad indexes
poor query plans
stale statistics
long transactions
autovacuum falling behind
disk I/O saturation
CPU saturation
replication lag
network latency
external API latency
~~~

Therefore:

> **First identify where time is being spent.**

---

# 2. The Optimization Workflow

Use this mental model:

~~~text
1. Detect the symptom
        ↓
2. Measure application + database
        ↓
3. Identify expensive query/wait
        ↓
4. EXPLAIN (ANALYZE, BUFFERS)
        ↓
5. Find root cause
        ↓
6. Make ONE targeted improvement
        ↓
7. Measure again
        ↓
8. Keep or revert based on evidence
~~~

This is much safer than random tuning.

---

# 3. Start From the User-Facing Problem

Example:

~~~text
ShopHub product page takes 2.5 seconds
~~~

Break it down:

~~~text
HTTP request
  ├─ authentication
  ├─ database queries
  ├─ external calls
  ├─ rendering
  └─ network
~~~

If PostgreSQL consumes only 20 ms, database tuning will not fix a 2.5-second request.

Optimize the actual bottleneck.

---

# 4. Find Expensive Queries

`pg_stat_statements` is one of the best starting points.

Example:

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

This answers:

~~~text
Which query consumes the most total DB time?
Which query runs most frequently?
Which query has high average latency?
~~~

---

# 5. Total Cost Matters

Compare:

~~~text
Query A
5 seconds × 10 calls
= 50 seconds total

Query B
20 milliseconds × 1,000,000 calls
= 20,000 seconds total
~~~

Query B may be a much larger optimization opportunity.

Do not focus only on the single slowest query.

---

# 6. EXPLAIN ANALYZE

Once you identify a problematic query:

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, price
FROM products
WHERE category_id = $1
  AND is_active = true
ORDER BY created_at DESC
LIMIT 20;
~~~

Inspect:

~~~text
scan type
estimated rows vs actual rows
join strategy
sorts
loops
filters
rows removed
buffer activity
execution time
~~~

Remember: `EXPLAIN ANALYZE` executes the statement.

Be especially careful with data-changing statements in production.

---

# 7. Estimated vs Actual Rows

Suppose the planner estimates:

~~~text
rows = 100
~~~

but actual result is:

~~~text
actual rows = 500,000
~~~

This large mismatch can cause a poor plan.

Possible causes include:

~~~text
stale statistics
skewed data
correlated columns
insufficient statistics
complex predicates
~~~

Do not immediately assume an index is missing.

---

# 8. Keep Statistics Healthy

PostgreSQL's planner depends on statistics.

Refresh them with:

~~~sql
ANALYZE products;
~~~

Normally autovacuum/autoanalyze handles this automatically.

If estimates are consistently poor, investigate why rather than manually running `ANALYZE` forever.

---

# 9. Optimize the Query Before the Server

Bad:

~~~sql
SELECT *
FROM orders;
~~~

then filtering 100,000 rows in Node.js.

Better:

~~~sql
SELECT id, status, total_amount, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
~~~

Push appropriate filtering, sorting, grouping, and limiting into PostgreSQL.

---

# 10. Select Only Required Columns

Instead of:

~~~sql
SELECT * FROM products;
~~~

use:

~~~sql
SELECT id, name, price
FROM products;
~~~

Benefits can include:

~~~text
less data read/transferred
smaller application payload
clearer API contract
potentially better index-only opportunities
~~~

---

# 11. Index the Query Pattern

Suppose the query is:

~~~sql
SELECT id, status, total_amount, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

A useful index may be:

~~~sql
CREATE INDEX orders_user_created_idx
ON orders(user_id, created_at DESC, id DESC);
~~~

The index follows the real access pattern:

~~~text
equality filter
   ↓
ordering / cursor
~~~

---

# 12. Composite Index Order

Index:

~~~text
(user_id, created_at, id)
~~~

is not equivalent to:

~~~text
(created_at, id, user_id)
~~~

Column order affects which query patterns the index can efficiently support.

A useful starting heuristic is:

~~~text
equality conditions first
then range/order columns
~~~

But always verify with the actual query plan.

---

# 13. Partial Indexes

If most product queries use only active products:

~~~sql
CREATE INDEX products_active_category_created_idx
ON products(category_id, created_at DESC, id DESC)
WHERE is_active = true;
~~~

Benefits:

~~~text
smaller index
less maintenance than indexing irrelevant rows
better match for a frequent predicate
~~~

The query predicate must be compatible with the partial-index predicate.

---

# 14. Expression Indexes

Query:

~~~sql
SELECT id
FROM users
WHERE lower(email) = lower($1);
~~~

Index:

~~~sql
CREATE UNIQUE INDEX users_email_lower_uidx
ON users(lower(email));
~~~

Without a matching expression index, applying a function to an indexed column can prevent use of the ordinary index for that expression.

---

# 15. Covering Indexes

If a query frequently needs a few extra columns:

~~~sql
CREATE INDEX orders_user_created_cover_idx
ON orders(user_id, created_at DESC)
INCLUDE (status, total_amount);
~~~

`INCLUDE` columns are payload rather than search-key columns.

This can make index-only scans possible when PostgreSQL's visibility rules permit them.

Do not create huge covering indexes indiscriminately.

---

# 16. Too Many Indexes Hurt Writes

Every INSERT/UPDATE/DELETE may need index maintenance.

Conceptually:

~~~text
INSERT row
  ├─ table write
  ├─ index A update
  ├─ index B update
  ├─ index C update
  └─ ...
~~~

More indexes mean:

~~~text
more storage
more WAL
more write work
more vacuum/index maintenance
~~~

Indexes are a tradeoff, not free acceleration.

---

# 17. Find Potentially Unused Indexes

Example:

~~~sql
SELECT
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
~~~

But:

> **Low or zero `idx_scan` does not automatically mean an index should be dropped.**

Consider statistics reset time, rare critical queries, constraints, maintenance, and representative workload periods.

---

# 18. Avoid Duplicate Indexes

Suppose you already have:

~~~text
PRIMARY KEY (id)
~~~

PostgreSQL already created the required unique index.

Adding:

~~~sql
CREATE INDEX products_id_idx ON products(id);
~~~

is normally redundant.

Likewise inspect overlapping composite indexes before adding another one.

---

# 19. Foreign-Key Indexes

PostgreSQL does not automatically index the referencing side of a foreign key.

For:

~~~text
orders.id
   ↓
order_items.order_id
~~~

this is often useful:

~~~sql
CREATE INDEX order_items_order_id_idx
ON order_items(order_id);
~~~

It can improve joins and parent-row update/delete checks depending on workload.

---

# 20. N+1 Query Problem

Bad application flow:

~~~text
SELECT 100 orders
      ↓
for each order
      ↓
SELECT order_items WHERE order_id = ?
~~~

Result:

~~~text
1 + 100 queries
~~~

Solutions include:

~~~text
JOIN
batch query
= ANY($1::uuid[])
ORM eager/batched loading
aggregation
~~~

Do not replace N+1 with one enormous duplication-heavy query without considering response shape.

---

# 21. Batch Queries

Instead of 100 separate queries:

~~~sql
SELECT *
FROM order_items
WHERE order_id = ANY($1::uuid[]);
~~~

Then group rows in application code if needed.

This reduces network/database round trips.

---

# 22. OFFSET Pagination

Query:

~~~sql
SELECT id, name
FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 200000;
~~~

PostgreSQL may still need to process/skips many preceding rows.

Deep OFFSET pagination becomes increasingly expensive.

---

# 23. Keyset Pagination

Better for large ordered feeds:

~~~sql
SELECT id, name, created_at
FROM products
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

Matching index:

~~~sql
CREATE INDEX products_created_id_idx
ON products(created_at DESC, id DESC);
~~~

Benefits:

~~~text
stable performance at deep pages
good index alignment
less skipped work
~~~

---

# 24. Deterministic Ordering

Do not paginate only by a non-unique timestamp:

~~~sql
ORDER BY created_at DESC
~~~

Multiple rows can share the same timestamp.

Use a tie-breaker:

~~~sql
ORDER BY created_at DESC, id DESC
~~~

and include both values in the cursor.

---

# 25. JOIN Optimization

For a slow join, inspect:

~~~text
join condition
row counts
indexes on join keys
filter selectivity
join algorithm
estimated vs actual rows
~~~

Do not assume:

~~~text
Nested Loop = bad
Hash Join = good
~~~

Each can be optimal depending on the data and query.

---

# 26. Nested Loop

Often good when:

~~~text
outer result is small
inner lookup is indexed
~~~

Conceptually:

~~~text
small outer rows
   ↓
indexed lookup per row
~~~

A nested loop becomes expensive when repeated inner work is unexpectedly large.

---

# 27. Hash Join

Often useful for larger equality joins.

Conceptually:

~~~text
build hash table from one input
      ↓
probe using rows from other input
~~~

It requires memory and can spill depending on workload/settings.

---

# 28. Merge Join

Useful when both inputs can be processed in compatible sorted order.

Do not manually force join strategies as your first optimization technique.

Fix data access, indexes, statistics, and query design first.

---

# 29. Sort Optimization

If a query repeatedly performs an expensive sort:

~~~text
Filter
   ↓
Sort 1,000,000 rows
   ↓
LIMIT 20
~~~

a matching index may allow PostgreSQL to retrieve rows already in useful order.

Example:

~~~sql
CREATE INDEX orders_user_created_idx
ON orders(user_id, created_at DESC);
~~~

Always verify the plan.

---

# 30. work_mem

`work_mem` can be used by sort/hash operations.

Too low:

~~~text
sort/hash may spill to disk
~~~

Too high globally:

~~~text
many sessions
×
multiple operations per query
×
large work_mem
=
memory pressure
~~~

Do not calculate memory as:

~~~text
work_mem × connections only
~~~

because a query can have multiple memory-consuming plan nodes.

---

# 31. Temporary File Spills

Large sorts/hashes may use temporary files.

Conceptually:

~~~text
Sort / Hash
   ↓
memory threshold exceeded
   ↓
temporary disk I/O
   ↓
query slows
~~~

Possible fixes:

~~~text
better query
better index
less data
appropriate per-session work_mem
more suitable architecture
~~~

Do not globally increase `work_mem` without concurrency analysis.

---

# 32. Connection Optimization

Too many PostgreSQL connections can hurt performance.

Architecture:

~~~text
Thousands of HTTP requests
      ↓
application pool / PgBouncer
      ↓
controlled number of DB sessions
~~~

Monitor:

~~~text
active connections
idle connections
pool wait time
max_connections
connection churn
~~~

---

# 33. Pool Size Is Not Throughput

Common mistake:

~~~text
10 connections slow
→ set pool to 500
~~~

This can make the database worse through:

~~~text
CPU contention
memory pressure
context switching
more concurrent I/O
~~~

Use enough concurrency to keep the database productive without overwhelming it.

---

# 34. Pool Queueing Is Useful

A connection pool intentionally limits database concurrency.

~~~text
100 incoming requests
      ↓
10 DB connections
      ↓
some requests briefly wait
~~~

This can be healthier than allowing all 100 to execute expensive database work simultaneously.

Queueing is not automatically a failure; excessive queueing is a capacity/latency signal.

---

# 35. Transaction Optimization

Keep transactions short.

Bad:

~~~text
BEGIN
 ↓
update inventory
 ↓
call payment API
 ↓
wait 5 seconds
 ↓
COMMIT
~~~

During that time, locks/snapshots/resources may remain active.

External network calls should normally not occur inside long DB transactions.

---

# 36. Idle in Transaction

Monitor:

~~~sql
SELECT
    pid,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
~~~

Long idle transactions can:

~~~text
retain locks
delay vacuum cleanup
retain old snapshots
increase bloat risk
~~~

Fix the application behavior, not only the symptom.

---

# 37. Lock Contention

A query may be slow because it is waiting, not computing.

Check:

~~~text
wait_event_type
wait_event
pg_blocking_pids(pid)
pg_locks
~~~

Before changing an index, determine whether the query is blocked.

---

# 38. Reduce Lock Duration

Strategies:

~~~text
short transactions
consistent lock ordering
targeted row updates
avoid user/external waits inside transaction
commit promptly
~~~

Concurrency optimization can improve latency without changing query plans.

---

# 39. Atomic Updates

Instead of:

~~~text
SELECT stock
      ↓
application checks
      ↓
UPDATE stock later
~~~

use when suitable:

~~~sql
UPDATE products
SET stock = stock - $1
WHERE id = $2
  AND stock >= $1
RETURNING stock;
~~~

This is both concurrency-safe for the invariant and efficient because it avoids an unnecessary read/decision round trip.

---

# 40. Bulk Operations

Bad:

~~~text
10,000 rows
→ 10,000 independent INSERT round trips
~~~

Better approaches may include:

~~~text
multi-row INSERT
batched operations
COPY for large imports
~~~

Example:

~~~sql
INSERT INTO tags(name)
VALUES
  ($1),
  ($2),
  ($3);
~~~

Choose batch size based on payload and workload.

---

# 41. COPY

For large PostgreSQL data imports, `COPY` is designed for efficient bulk transfer.

Conceptually:

~~~text
CSV / stream
    ↓
COPY
    ↓
PostgreSQL table
~~~

For millions of rows, this can be far more appropriate than individual INSERT requests.

---

# 42. Avoid Chatty Application Design

Bad:

~~~text
Request
 ├─ query 1
 ├─ query 2
 ├─ query 3
 ├─ query 4
 ├─ query 5
 └─ query 6
~~~

Ask whether some operations can be:

~~~text
combined
batched
performed in one SQL statement
performed concurrently when independent and safe
cached appropriately
~~~

Round trips have cost even when each query is individually fast.

---

# 43. ORM Optimization

ORMs do not remove database performance concerns.

Inspect:

~~~text
generated SQL
number of queries
selected columns
JOIN behavior
N+1 patterns
transaction boundaries
indexes
~~~

Use raw SQL for complex or performance-critical queries when it genuinely improves clarity/control.

Do not abandon an ORM merely because one query is slow.

---

# 44. CTEs Are Not Automatically Slow or Fast

Modern PostgreSQL can inline eligible CTEs in many situations.

Therefore:

~~~text
CTE
≠
always materialized
≠
automatically slower
~~~

Use `EXPLAIN ANALYZE` rather than performance folklore.

`MATERIALIZED` and `NOT MATERIALIZED` can influence behavior when appropriate.

---

# 45. EXISTS vs IN

Do not memorize:

~~~text
EXISTS is always faster than IN
~~~

That is false.

Choose correct semantics first and inspect the plan.

Remember the `NOT IN` + NULL behavior discussed earlier; `NOT EXISTS` is often safer for anti-join semantics when NULLs are possible.

---

# 46. JSONB Optimization

JSONB is useful, but repeatedly querying deep flexible fields may need appropriate GIN/expression indexes.

Example:

~~~sql
CREATE INDEX products_metadata_gin_idx
ON products USING GIN(metadata);
~~~

But if a JSONB property becomes a core business field used constantly for joins, constraints, or sorting, consider promoting it to a normal column.

Schema design is itself a performance tool.

---

# 47. Array Optimization

GIN indexes can help supported array operations such as containment.

But if array elements represent real relational entities:

~~~text
product.tags = [1, 5, 9]
~~~

a normalized junction table may be more appropriate for relationships and constraints.

Do not use arrays only to avoid relational modeling.

---

# 48. Materialized Views

For expensive repeated analytics:

~~~text
many joins
large aggregation
same report repeatedly
~~~

a materialized view may precompute results.

~~~text
Transactional tables
      ↓
expensive aggregation
      ↓
Materialized View
      ↓
fast repeated reads
~~~

Tradeoff: data can be stale until refreshed.

---

# 49. Cache After Query Optimization

Bad strategy:

~~~text
slow broken query
      ↓
put Redis in front
~~~

Better:

~~~text
measure
 ↓
optimize SQL/index/schema
 ↓
then cache if repeated reads justify it
~~~

Caching adds:

~~~text
invalidation
staleness
memory cost
operational complexity
~~~

Use it deliberately.

---

# 50. What Should Be Cached?

Good candidates may include:

~~~text
frequently read data
expensive repeated computation
data tolerant of short staleness
~~~

Poor candidates may include:

~~~text
highly volatile inventory without careful design
authorization decisions with unsafe stale state
one-time queries
~~~

Database optimization and caching solve different problems.

---

# 51. Autovacuum Performance

Heavy UPDATE/DELETE workloads create obsolete tuple versions.

If autovacuum cannot keep up:

~~~text
dead tuples grow
   ↓
table/index bloat pressure
   ↓
more pages read
   ↓
slower queries / more I/O
~~~

Monitor per-table behavior and tune high-write tables when evidence supports it.

---

# 52. Long Transactions Hurt Vacuum

An old transaction snapshot may prevent PostgreSQL from removing/reusing tuple versions that could otherwise be cleaned.

Therefore:

~~~text
long transaction
      ↓
old versions retained
      ↓
more table growth / cleanup delay
~~~

Performance optimization includes transaction hygiene.

---

# 53. VACUUM FULL Is Not Routine Optimization

`VACUUM FULL` rewrites a table and requires an aggressive lock.

It can return space to the operating system, but it is not a normal first response to bloat.

Use ordinary autovacuum/VACUUM as the normal maintenance mechanism and investigate why abnormal bloat occurred.

---

# 54. Checkpoint and WAL Pressure

Heavy writes generate WAL.

Conceptually:

~~~text
high write workload
    ↓
high WAL generation
    ↓
checkpoint/storage pressure
    ↓
latency spikes if infrastructure/configuration cannot keep up
~~~

Monitor WAL rate, checkpoint behavior, and storage latency together.

Do not tune checkpoint parameters in isolation.

---

# 55. Storage Performance

Database performance can be storage-bound.

Watch:

~~~text
read latency
write latency
IOPS
throughput
queue depth/saturation
temporary-file I/O
~~~

A perfect index cannot fix a severely overloaded storage layer.

---

# 56. CPU Saturation

If CPU is high, identify the workload responsible.

Possible causes:

~~~text
expensive joins
large aggregations
too much concurrency
functions/expression work
poor query plans
background maintenance
~~~

Scaling CPU can help capacity, but inefficient queries may simply consume the new CPU too.

---

# 57. Memory Pressure

Memory is shared across:

~~~text
PostgreSQL shared memory
per-query operations
connections
OS filesystem cache
other processes
~~~

Symptoms of poor memory configuration can include:

~~~text
swap pressure
OOM events
excessive temp files
cache churn
~~~

Tune with workload/concurrency in mind.

---

# 58. Read Replicas

If the primary is genuinely read-bound after query/index optimization:

~~~text
Primary
  ├─ writes
  └─ critical reads

Replica
  └─ reporting / suitable read traffic
~~~

But replicas introduce:

~~~text
replication lag
read-after-write inconsistency
routing complexity
extra cost
~~~

Do not add replicas to hide a missing index.

---

# 59. Separate Analytics Workloads

If large reports hurt OLTP traffic:

~~~text
Primary transactional DB
      ↓ replication/export
Read replica / warehouse / analytics system
      ↓
heavy reports
~~~

This protects checkout/API latency from analytical scans.

---

# 60. Partitioning

Partitioning divides a logical table into physical partitions.

Example concept:

~~~text
events
 ├─ 2026_01
 ├─ 2026_02
 ├─ 2026_03
 └─ ...
~~~

It can help manage very large tables and certain access/maintenance patterns.

But:

> **Partitioning is not a magic speed button.**

For ordinary application tables, correct indexes and queries often matter more.

---

# 61. When Partitioning May Help

Examples:

~~~text
very large time-series/event tables
data lifecycle by date
dropping old data by partition
queries consistently target partition keys
maintenance on subsets
~~~

Do not partition a small table simply because it sounds scalable.

---

# 62. Denormalization

Normalization improves integrity and reduces unnecessary duplication.

Denormalization can improve specific read paths by intentionally storing derived/repeated data.

Examples from ShopHub:

~~~text
order_items.product_name
order_items.unit_price
orders.total_amount
~~~

These are justified primarily by historical/business correctness and convenient reads.

Measure before introducing performance-only duplication.

---

# 63. Precomputation

For expensive dashboard values:

~~~text
calculate every request
vs
precompute periodically / incrementally
~~~

Examples:

~~~text
daily revenue
top-selling products
monthly customer metrics
~~~

Materialized views, summary tables, or analytics systems may help.

Tradeoff: freshness.

---

# 64. Query Timeouts

Prevent runaway queries from consuming resources indefinitely.

Relevant controls include:

~~~text
statement_timeout
lock_timeout
idle_in_transaction_session_timeout
application request timeout
pool acquisition timeout
~~~

Timeouts should be designed across layers rather than independently.

---

# 65. Cancel Work That No Longer Matters

If an HTTP client disconnects or request deadline expires, continuing a very expensive DB query may waste resources.

Where your application stack supports safe cancellation, propagate cancellation/deadline behavior appropriately.

This is an advanced optimization but important at scale.

---

# 66. Prepared Statements

Prepared statements can reduce repeated parse/plan overhead in appropriate workloads.

But:

~~~text
prepared statement
≠
automatically faster query
~~~

Plan selection and parameter distribution can matter.

Use driver/framework support appropriately and measure.

---

# 67. Avoid Premature Configuration Tuning

Do not start performance work by changing:

~~~text
shared_buffers
work_mem
effective_cache_size
checkpoint settings
random_page_cost
planner enable_* settings
~~~

without evidence.

First inspect:

~~~text
query
plan
indexes
statistics
locks
connections
hardware utilization
~~~

Configuration tuning comes after understanding the workload.

---

# 68. Do Not Disable Planner Strategies to "Fix" One Query

Settings that disable sequential scans or join strategies are diagnostic tools in some cases, not normal production fixes.

Fix the underlying reason the planner chooses a poor plan.

Examples:

~~~text
missing index
bad statistics
incorrect estimates
query design
data distribution
~~~

---

# 69. Test With Production-Like Data

A query against:

~~~text
100 development rows
~~~

may behave completely differently against:

~~~text
50 million production rows
~~~

Performance testing should approximate:

~~~text
data volume
distribution
indexes
concurrency
query patterns
~~~

Use synthetic/sanitized data where appropriate.

---

# 70. Benchmark Correctly

Bad benchmark:

~~~text
run query once
→ 80 ms
→ declare success
~~~

Consider:

~~~text
cold vs warm cache
multiple runs
concurrency
real parameter distributions
database load
p95/p99 latency
total throughput
~~~

Measure enough to make the comparison meaningful.

---

# 71. Optimize One Variable at a Time

If you simultaneously:

~~~text
add index
increase work_mem
change pool size
rewrite query
upgrade server
~~~

and performance improves, you do not know why.

Prefer controlled changes where practical:

~~~text
baseline
 ↓
one change
 ↓
measure
 ↓
next change
~~~

---

# 72. Compare Before and After

Record:

~~~text
execution time
buffer reads/hits
rows processed
CPU
I/O
query calls
total DB time
lock waits
application p95/p99
~~~

Optimization without before/after measurement is speculation.

---

# 73. Performance Regression Monitoring

After deployment:

~~~text
new feature
   ↓
new query pattern
   ↓
query frequency increases
   ↓
database load increases
~~~

Monitor releases for regressions.

A query can be individually fast but harmful when called thousands of times per request or second.

---

# 74. ShopHub Example — Slow Product Listing

Problem:

~~~text
GET /products?category=laptops
p95 = 1.8 seconds
~~~

Investigation:

~~~text
pg_stat_statements
      ↓
product listing is expensive
      ↓
EXPLAIN ANALYZE
      ↓
large scan + sort
      ↓
query filters category + active
and orders by created_at/id
~~~

Possible targeted index:

~~~sql
CREATE INDEX products_active_category_created_idx
ON products(category_id, created_at DESC, id DESC)
WHERE is_active = true;
~~~

Then run `EXPLAIN ANALYZE` again and compare.

---

# 75. ShopHub Example — Slow Checkout

Symptom:

~~~text
checkout takes 6 seconds
~~~

Investigation shows:

~~~text
inventory UPDATE itself = 5 ms
but
query waits 5.8 seconds on row lock
~~~

Root cause:

~~~text
another transaction updates inventory
then calls external payment API
before COMMIT
~~~

Fix:

~~~text
redesign transaction boundary
keep DB transaction short
do not wait on external API while holding locks
~~~

No index was needed.

---

# 76. ShopHub Example — Too Many Queries

Product page performs:

~~~text
1 product query
1 category query
8 image queries
10 review/user queries
5 recommendation queries
~~~

Each query may be only 3–5 ms, but network/database round trips accumulate.

Optimization:

~~~text
batch
JOIN where appropriate
fetch relations intentionally
remove unnecessary calls
cache stable data where justified
~~~

Application architecture is part of database performance.

---

# 77. CareerLoop Example — Application Feed

Suppose CareerLoop grows to millions of applications.

Query:

~~~sql
SELECT id, company_name, role, status, applied_at
FROM job_applications
WHERE user_id = $1
  AND status = $2
ORDER BY applied_at DESC, id DESC
LIMIT 30;
~~~

Potential index:

~~~sql
CREATE INDEX job_applications_user_status_applied_idx
ON job_applications(user_id, status, applied_at DESC, id DESC);
~~~

This follows the access pattern rather than indexing fields independently without reason.

---

# 78. CareerLoop Search Optimization

If users search company/role names with arbitrary substring matching:

~~~sql
WHERE company_name ILIKE '%openai%'
~~~

ordinary B-tree indexing may not help this pattern much.

At larger scale, PostgreSQL's `pg_trgm` extension with suitable GIN/GiST trigram indexes may be considered.

Do not add specialized indexes before search volume/data size justify them.

---

# 79. Production Optimization Checklist

~~~text
QUERY
✓ exact slow SQL identified
✓ unnecessary columns removed
✓ filtering done in DB
✓ N+1 checked
✓ pagination appropriate

PLAN
✓ EXPLAIN ANALYZE inspected
✓ estimates vs actual checked
✓ waits distinguished from execution
✓ buffers/I/O inspected

INDEX
✓ real access pattern indexed
✓ composite order intentional
✓ partial/expression index considered
✓ duplicate/unused indexes reviewed

TRANSACTION
✓ transaction short
✓ lock contention checked
✓ external calls outside DB transaction
✓ concurrency-safe update logic

CONNECTION
✓ pool size controlled
✓ connection budget understood
✓ pool waits monitored

MAINTENANCE
✓ autovacuum healthy
✓ statistics current
✓ long transactions controlled
✓ bloat investigated with evidence

INFRASTRUCTURE
✓ CPU checked
✓ memory checked
✓ storage latency checked
✓ WAL/checkpoint behavior checked

VALIDATION
✓ before metrics recorded
✓ after metrics recorded
✓ production regression monitoring enabled
~~~

---

# 80. Common Optimization Mistakes

## Adding Indexes Without EXPLAIN

You may add write/storage cost without helping the query.

## Assuming Every Sequential Scan Is Bad

Sequential scan may be optimal for small tables or large-result queries.

## Increasing max_connections to Fix Pool Exhaustion

This can simply move overload into PostgreSQL.

## Increasing work_mem Globally

Concurrency can multiply memory usage dramatically.

## Using Cache to Hide Bad SQL

Fix the underlying query first.

## Using OFFSET for Extremely Deep Pagination

Use keyset pagination where the UX/query allows it.

## Keeping Transactions Open During External Calls

This increases lock and MVCC pressure.

## Running VACUUM FULL as Routine Maintenance

It is disruptive and does not fix the underlying cause of recurring bloat.

## Optimizing Development Data Only

Plans change with production-scale data and distribution.

## Changing Five Things at Once

You lose the ability to identify which change helped or hurt.

---

# 81. Interview Questions

## How do you optimize a slow PostgreSQL query?

First identify the exact expensive query using application metrics and tools such as `pg_stat_statements`. Then inspect `EXPLAIN (ANALYZE, BUFFERS)`, compare estimates with actual rows, check scans/joins/sorts/waits, make a targeted query/index/schema/statistics change, and measure again.

## Is an index always faster?

No. Indexes add write/storage cost, and sequential scans can be faster when many rows are needed or the table is small.

## What is the N+1 problem?

Fetching parent rows and then issuing a separate child query for each parent, causing excessive database round trips. Solve with joins, batching, aggregation, or appropriate ORM loading.

## OFFSET vs keyset pagination?

OFFSET is simple but becomes expensive at deep pages because preceding rows must still be processed/skipped. Keyset pagination uses the last ordered key as a cursor and can remain efficient with a matching index.

## Why can too many DB connections reduce performance?

Each PostgreSQL backend consumes resources, and excessive concurrent work creates CPU, memory, scheduling, and I/O contention.

## Why can a query be slow even with a good index?

It may be blocked on locks, return too many rows, suffer from poor estimates, storage latency, CPU pressure, connection queueing, or other system bottlenecks.

## What does `work_mem` control?

It is a per-operation memory budget for certain sorts/hashes and related operations. Multiple operations across concurrent queries can multiply memory usage.

## Why are long transactions harmful?

They can hold locks, retain old MVCC snapshots, delay vacuum cleanup, increase bloat pressure, and reduce concurrency.

## How do you optimize write-heavy tables?

Keep necessary indexes only, use short/batched transactions, monitor autovacuum/WAL/storage, avoid unnecessary updates, and tune based on measured workload.

## When should you use a read replica?

When suitable read workload remains a bottleneck after query/index optimization and the application can tolerate replication lag/consistency implications.

## Is partitioning always a performance improvement?

No. It is mainly useful for very large datasets with suitable partition-key access and lifecycle/maintenance requirements.

---

# 82. Interview-Ready Optimization Answer

If asked, **"Our production PostgreSQL database is slow. What would you do?"**, answer in this order:

~~~text
1. Measure application and DB latency.
2. Find expensive/frequent queries with pg_stat_statements.
3. Check whether queries are executing or waiting on locks/connections.
4. Use EXPLAIN (ANALYZE, BUFFERS).
5. Check estimates, scans, joins, sorts, and rows processed.
6. Improve query shape/indexes/statistics/schema as needed.
7. Check N+1, pagination, transaction boundaries, and pool sizing.
8. Check autovacuum, CPU, memory, disk, WAL, and replication.
9. Make one targeted change.
10. Compare before/after metrics and monitor for regression.
~~~

This shows much stronger production knowledge than saying:

> "I would add an index."

---

# 83. Complete Optimization Mental Model

~~~text
                 SLOW APPLICATION
                        ↓
              Where is time spent?
                        ↓
        ┌───────────────┼───────────────┐
        ↓               ↓               ↓
   Application       Database       External
        │               │
        │               ↓
        │         Query or Wait?
        │          ┌────┴────┐
        │          ↓         ↓
        │        Query      Lock/Pool
        │          ↓         ↓
        │       EXPLAIN    concurrency
        │          ↓
        │   ┌──────┼──────────┐
        │   ↓      ↓          ↓
        │ Query   Index    Statistics
        │   │      │          │
        └───┴──────┴──────────┘
                 ↓
           targeted fix
                 ↓
            measure again
~~~

---

# 84. Five Golden Rules

~~~text
1. Measure before optimizing.

2. Optimize the query/access pattern before blindly tuning PostgreSQL.

3. Index for real queries, not individual columns by habit.

4. Keep transactions and connection concurrency controlled.

5. Verify every optimization with before/after evidence.
~~~

---

## Key Takeaway

> **Production PostgreSQL optimization is an evidence-driven process. Start from the user-facing symptom, identify expensive queries or waits, inspect real execution plans and workload statistics, then optimize the smallest correct layer—query shape, indexes, schema, pagination, transactions, connection pooling, maintenance, or infrastructure. PostgreSQL performance problems are not all indexing problems, and every optimization should be validated with measurable before-and-after results.**

---

[← Previous: Lesson 44 — Production E-Commerce Database Design](./44-production-ecommerce-database.md) | [Back to Roadmap](../README.md) | [Next: Lesson 46 — PostgreSQL Production Checklist →](./46-postgresql-production-checklist.md)