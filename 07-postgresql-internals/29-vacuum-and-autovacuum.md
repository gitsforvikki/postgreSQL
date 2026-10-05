# Lesson 29 — VACUUM & Autovacuum

## Why Does PostgreSQL Need VACUUM?

In Lesson 28, you learned that PostgreSQL uses **MVCC**.

When a row is updated, PostgreSQL generally creates a new row version instead of simply overwriting the old tuple in place.

~~~text
Original tuple
stock = 5
    ↓ UPDATE
New tuple
stock = 4

Old tuple remains temporarily
~~~

When no active transaction can need the old tuple anymore, it becomes a **dead tuple**.

PostgreSQL needs a maintenance mechanism to deal with these obsolete row versions.

That mechanism is **VACUUM**.

~~~text
MVCC
 ↓
UPDATE / DELETE
 ↓
obsolete tuple versions
 ↓
VACUUM
 ↓
space can be reused
~~~

---

# 1. What Does VACUUM Do?

At a high level, regular VACUUM:

- identifies tuple versions that are no longer needed
- makes their space reusable
- maintains visibility information
- helps prevent transaction-ID wraparound problems
- performs other table/index maintenance work

Important:

> Regular VACUUM normally does **not** shrink the table file back to the operating system.

It mainly makes space available for PostgreSQL to reuse.

---

# 2. Dead Tuples

Suppose:

~~~sql
UPDATE products
SET price = 50000
WHERE id = 10;
~~~

Conceptually:

~~~text
Before
Tuple A → price 55000

After UPDATE
Tuple A → old version
Tuple B → price 50000
~~~

When Tuple A is no longer visible to any relevant transaction, it becomes dead.

Similarly:

~~~sql
DELETE FROM products
WHERE id = 20;
~~~

does not necessarily erase the physical tuple immediately.

Eventually VACUUM can make obsolete tuple space reusable.

---

# 3. Why Not Remove the Old Tuple Immediately?

Because another transaction may still need it.

Example:

~~~text
Transaction A
starts with an older snapshot
       ↓
can still see old tuple

Transaction B
updates row and commits
       ↓
creates newer tuple
~~~

If PostgreSQL physically destroyed the old version immediately, Transaction A could lose the consistent snapshot it is entitled to see.

So PostgreSQL waits until the old version is no longer needed.

---

# 4. Regular VACUUM Reuses Space

Imagine a table page:

~~~text
Before VACUUM

┌─────────────┐
│ live tuple  │
│ dead tuple  │
│ live tuple  │
│ dead tuple  │
└─────────────┘
~~~

After regular VACUUM, conceptually:

~~~text
┌─────────────┐
│ live tuple  │
│ reusable    │
│ live tuple  │
│ reusable    │
└─────────────┘
~~~

Future INSERTs or UPDATE-created tuple versions may reuse that space.

However, the physical table file usually remains allocated.

---

# 5. VACUUM Does Not Usually Return Space to the OS

This is one of the most important interview distinctions.

Suppose a table file has grown to:

~~~text
20 GB
~~~

You delete a large amount of data and run:

~~~sql
VACUUM orders;
~~~

PostgreSQL may now have lots of reusable internal space, but the file may still remain close to its previous physical size.

Think:

~~~text
Regular VACUUM
→ reclaim space for PostgreSQL reuse
→ normally does not compact entire table
→ normally does not return most freed space to OS
~~~

---

# 6. VACUUM FULL

If you need to physically compact a table and return unused space to the operating system, PostgreSQL provides:

~~~sql
VACUUM FULL orders;
~~~

`VACUUM FULL` rewrites the table into a compact form.

Conceptually:

~~~text
Bloated table
┌─────────────────────┐
│ live | free | live  │
│ free | live | free  │
└─────────────────────┘
          ↓
     VACUUM FULL
          ↓
Compact table
┌───────────────┐
│ live | live   │
│ live          │
└───────────────┘
~~~

But there is a major cost:

> `VACUUM FULL` requires an **ACCESS EXCLUSIVE** lock on the table while it works.

That means normal application access to that table is blocked during the operation.

So `VACUUM FULL` is not routine maintenance for a busy production table.

---

# 7. VACUUM vs VACUUM FULL

| Feature | VACUUM | VACUUM FULL |
|---|---|---|
| Makes dead-tuple space reusable | Yes | Yes |
| Usually shrinks physical table file substantially | No | Yes |
| Rewrites table | No | Yes |
| Allows normal concurrent table use | Generally yes | No, takes ACCESS EXCLUSIVE lock |
| Routine maintenance | Yes | Usually no |

Easy memory:

~~~text
VACUUM
→ clean/reuse

VACUUM FULL
→ rewrite/compact
~~~

---

# 8. What Is ANALYZE?

`ANALYZE` is related to maintenance but solves a different problem.

~~~sql
ANALYZE orders;
~~~

It collects statistics about table data for PostgreSQL's query planner.

Statistics help estimate:

~~~text
row counts
distinct values
common values
NULL frequency
data distribution
~~~

The planner uses these estimates to choose:

~~~text
Seq Scan vs Index Scan
join strategies
join order
aggregation strategy
other execution decisions
~~~

---

# 9. VACUUM ANALYZE

You can run both maintenance operations together:

~~~sql
VACUUM ANALYZE orders;
~~~

Conceptually:

~~~text
VACUUM
→ maintain obsolete tuple space / visibility

ANALYZE
→ update planner statistics
~~~

Do not confuse them.

---

# 10. Why Autovacuum Exists

Manually running VACUUM for every table would be impractical.

PostgreSQL therefore includes **autovacuum**.

Autovacuum automatically monitors tables and performs maintenance when thresholds are reached.

High-level architecture:

~~~text
PostgreSQL
    ↓
Autovacuum launcher
    ↓
Autovacuum workers
    ↓
Tables requiring maintenance
~~~

Autovacuum is essential production infrastructure, not an optional cleanup feature you normally disable.

---

# 11. What Autovacuum Does

Autovacuum workers can perform automatic:

~~~text
VACUUM
and
ANALYZE
~~~

This helps PostgreSQL:

- reclaim reusable space from obsolete tuples
- maintain visibility information
- maintain planner statistics
- prevent transaction-ID wraparound

Without effective autovacuum, a busy PostgreSQL database can degrade significantly.

---

# 12. Autovacuum Threshold — Basic Idea

PostgreSQL does not necessarily vacuum a table after every single UPDATE.

Instead, automatic maintenance is triggered based on configurable thresholds and scale factors.

Conceptually:

~~~text
dead tuples / table changes increase
        ↓
threshold reached
        ↓
autovacuum worker processes table
~~~

For vacuuming, the threshold conceptually depends on a base threshold plus a fraction of table size, with additional version-specific/configuration details.

You do not need to memorize default numbers for a full-stack interview.

Remember:

> Autovacuum decisions are threshold-based and can be configured globally or per table.

---

# 13. Autovacuum and Large Tables

Imagine:

~~~text
small table = 10,000 rows
large table = 500,000,000 rows
~~~

A percentage-based threshold can mean very different numbers of changed/dead tuples.

That is why high-write large tables sometimes require autovacuum tuning.

Examples:

~~~text
orders
events
messages
audit_logs
~~~

But do not tune autovacuum blindly.

Measure table activity and maintenance behavior first.

---

# 14. Autovacuum Should Usually Stay Enabled

A common production mistake is:

~~~text
autovacuum causes load
      ↓
disable autovacuum
~~~

This can create much worse problems later:

~~~text
dead tuples accumulate
      ↓
table/index bloat pressure
      ↓
query performance degrades
      ↓
transaction-ID wraparound danger
~~~

Better approach:

~~~text
observe
→ understand workload
→ tune autovacuum if necessary
~~~

not:

~~~text
disable it
~~~

---

# 15. Transaction-ID Wraparound

This is one of the most important reasons VACUUM is essential.

PostgreSQL transaction IDs are finite and eventually wrap around.

MVCC uses transaction IDs to reason about tuple visibility.

If old transaction metadata were never maintained, wraparound could make transaction-age comparisons unsafe.

PostgreSQL therefore uses VACUUM to **freeze** sufficiently old tuple transaction information.

High-level:

~~~text
transactions keep increasing
        ↓
old tuple XIDs age
        ↓
VACUUM freezes old tuples
        ↓
visibility remains safe across XID wraparound
~~~

---

# 16. What Is Freezing?

At a simplified level, freezing marks sufficiently old tuple transaction information so PostgreSQL no longer needs to treat it like an ordinary aging transaction ID for future visibility decisions.

Think:

~~~text
Very old tuple
    ↓
VACUUM freeze processing
    ↓
safe from future XID-age ambiguity
~~~

You do not need internal frozen-XID implementation details for a normal full-stack interview.

Key point:

> VACUUM is required not only for space reuse, but also for transaction-ID safety.

---

# 17. Anti-Wraparound Autovacuum

PostgreSQL can launch aggressive/mandatory autovacuum activity to prevent transaction-ID wraparound even when ordinary autovacuum behavior would otherwise not run.

This is why disabling autovacuum does not mean PostgreSQL can safely ignore wraparound forever.

Wraparound protection is fundamental to database correctness.

---

# 18. Long-Running Transactions vs VACUUM

Suppose:

~~~text
Transaction A
BEGIN
   ↓
stays open for hours
~~~

Meanwhile:

~~~text
thousands of UPDATEs / DELETEs
~~~

Old tuple versions may still be potentially visible to Transaction A's old snapshot.

Therefore VACUUM cannot remove/reclaim everything it otherwise could.

~~~text
Old open transaction
        ↓
old snapshot remains relevant
        ↓
obsolete versions cannot all be reclaimed
        ↓
dead tuples accumulate
~~~

This is why long-running or idle-in-transaction sessions are dangerous.

---

# 19. Idle in Transaction

A particularly problematic session state is:

~~~text
idle in transaction
~~~

Example backend bug:

~~~text
BEGIN
  ↓
run query
  ↓
application forgets COMMIT/ROLLBACK
  ↓
connection remains open
~~~

This can:

- hold locks
- retain an old transaction snapshot
- interfere with vacuum cleanup
- consume a pooled connection

Backend transaction code must always end with COMMIT or ROLLBACK.

---

# 20. Monitoring Sessions

PostgreSQL exposes activity information through:

~~~sql
SELECT
  pid,
  usename,
  state,
  query_start,
  xact_start,
  query
FROM pg_stat_activity;
~~~

This can help identify:

~~~text
long-running queries
long-running transactions
idle-in-transaction sessions
~~~

In production, you should inspect carefully rather than terminating sessions blindly.

---

# 21. Monitoring Dead Tuples

PostgreSQL statistics views expose approximate dead-tuple information.

Example:

~~~sql
SELECT
  relname,
  n_live_tup,
  n_dead_tup,
  last_vacuum,
  last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
~~~

This helps answer:

~~~text
Which tables have many dead tuples?
When did vacuum last run?
When did autovacuum last run?
~~~

Statistics are estimates/observations, so interpret them as monitoring signals rather than exact forensic counts in every situation.

---

# 22. Table Bloat

Repeated UPDATEs and DELETEs can leave reusable or inefficiently arranged space.

Over time, tables and indexes can become larger than the amount of live logical data might suggest.

This is commonly discussed as **bloat**.

Conceptually:

~~~text
logical live data
████████

physical relation
████████░░░░░░░░
        ↑
unused/reusable/dead-space effects
~~~

Bloat can increase:

~~~text
disk usage
pages scanned
cache pressure
I/O
backup size
~~~

Regular autovacuum helps control problems, but it does not physically compact every relation back to minimum size.

---

# 23. Why VACUUM May Not Fix Physical Bloat

Regular VACUUM:

~~~text
dead space
   ↓
mark reusable
~~~

But the table file generally remains allocated.

To physically compact heavily bloated storage, options may include:

~~~text
VACUUM FULL
table rewrite operations
specialized maintenance approaches/tools
~~~

These can have operational costs, so production decisions require planning.

---

# 24. Visibility Map

PostgreSQL maintains a **visibility map** that tracks heap pages with useful visibility properties such as pages whose tuples are all visible to all current/future transactions under the relevant rules.

VACUUM helps maintain this information.

Why does it matter?

Because of **Index Only Scan**.

~~~text
Index contains requested columns
        ↓
Need to know tuple visibility
        ↓
Visibility map says page is all-visible
        ↓
heap page may not need to be visited
~~~

So VACUUM can indirectly improve query performance by enabling more effective index-only scans.

---

# 25. Free Space Map

PostgreSQL also maintains information about available free space in heap/index pages.

High-level:

~~~text
VACUUM finds reusable space
        ↓
free-space information maintained
        ↓
future INSERT/UPDATE can locate pages with room
~~~

You do not need internal free-space-map implementation details for normal interviews.

---

# 26. VACUUM and Indexes

VACUUM also performs index-related cleanup associated with dead tuples, depending on the operation and conditions.

Conceptually:

~~~text
dead heap tuples
      ↓
stale index references become unnecessary
      ↓
VACUUM maintenance
~~~

Indexes and tables both participate in MVCC maintenance costs.

This is another reason heavy UPDATE/DELETE workloads need healthy autovacuum.

---

# 27. VACUUM VERBOSE

For diagnostic output:

~~~sql
VACUUM (VERBOSE) orders;
~~~

PostgreSQL reports information about the vacuum operation.

This can help when learning or diagnosing maintenance behavior.

Do not confuse verbose output with a performance improvement—it only provides additional reporting.

---

# 28. VACUUM ANALYZE in Practice

After a large data load or major data distribution change, you may sometimes run:

~~~sql
VACUUM (ANALYZE) orders;
~~~

or simply:

~~~sql
ANALYZE orders;
~~~

depending on what maintenance is actually required.

Remember:

~~~text
Need obsolete tuple maintenance?
→ VACUUM

Need planner statistics?
→ ANALYZE

Need both?
→ VACUUM ANALYZE
~~~

---

# 29. Autovacuum and ANALYZE

Autovacuum infrastructure also performs automatic analyze activity when table-change thresholds are reached.

Why?

Because changing data distributions can make old planner statistics inaccurate.

~~~text
many INSERT/UPDATE/DELETE operations
        ↓
data distribution changes
        ↓
auto-analyze
        ↓
planner receives fresher statistics
~~~

This connects autovacuum directly to query optimization.

---

# 30. ShopHub Example

Suppose ShopHub has a large `orders` table.

Every day:

~~~text
100,000 new orders
50,000 status updates
5,000 cancelled/deleted test records
~~~

Updates create newer tuple versions.

~~~text
orders
  ↓
many UPDATEs
  ↓
dead tuple versions
  ↓
autovacuum
  ↓
space reusable + visibility maintenance
~~~

Meanwhile auto-analyze keeps planner statistics reasonably current so queries such as:

~~~sql
SELECT id, total, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
~~~

can be planned using useful statistics.

---

# 31. Backend Developer Responsibilities

As a full-stack/backend developer, you usually do **not** manually vacuum after every API request.

Your responsibilities are more like:

~~~text
Keep transactions short
Always COMMIT or ROLLBACK
Avoid idle-in-transaction connections
Monitor unusual DB growth
Understand high-update tables
Do not disable autovacuum casually
Use EXPLAIN/monitoring when performance degrades
~~~

Database maintenance and application design are connected.

---

# Common Mistakes

## 32. Thinking VACUUM Deletes Application Rows

VACUUM cleans obsolete internal row versions; it is not equivalent to SQL DELETE.

## 33. Thinking VACUUM Shrinks the Table File

Regular VACUUM normally makes space reusable internally rather than returning most space to the OS.

## 34. Running VACUUM FULL Routinely

`VACUUM FULL` rewrites the table and takes an ACCESS EXCLUSIVE lock. Use it only when its tradeoffs are justified.

## 35. Disabling Autovacuum Because It Uses Resources

Autovacuum is essential for MVCC cleanup, statistics, and wraparound prevention. Tune rather than casually disable.

## 36. Confusing VACUUM with ANALYZE

VACUUM handles obsolete tuple/visibility maintenance; ANALYZE gathers planner statistics.

## 37. Leaving Transactions Open

Old snapshots can prevent VACUUM from reclaiming obsolete tuple versions.

## 38. Ignoring Dead Tuples on High-Write Tables

Tables with frequent UPDATE/DELETE activity need effective maintenance.

## 39. Assuming Dead Tuples Mean PostgreSQL Is Broken

Dead tuples are a normal result of MVCC. The problem is when maintenance cannot keep up.

---

# Interview Revision

## What is VACUUM?

PostgreSQL maintenance that processes obsolete tuple versions, makes their space reusable, maintains visibility information, and helps protect against transaction-ID wraparound.

## Why does PostgreSQL need VACUUM?

Because MVCC leaves old tuple versions after UPDATE and DELETE instead of immediately removing them.

## Does regular VACUUM shrink the table file?

Normally no. It mainly makes space reusable inside the relation.

## What is VACUUM FULL?

A table-rewrite operation that compacts the table and can return unused space to the OS, but requires an ACCESS EXCLUSIVE lock.

## VACUUM vs ANALYZE?

VACUUM maintains obsolete tuple space/visibility and transaction-age safety; ANALYZE collects statistics for the query planner.

## What is autovacuum?

PostgreSQL's automatic maintenance system that launches workers to vacuum and analyze tables when needed.

## Should autovacuum be disabled?

Normally no. It is essential production maintenance; tune it when necessary instead of casually disabling it.

## What is a dead tuple?

An obsolete row version that is no longer needed by active transactions and whose space can eventually be reclaimed/reused.

## What is transaction-ID wraparound?

PostgreSQL transaction IDs are finite; VACUUM freezes sufficiently old tuple metadata so visibility remains safe as transaction IDs age and eventually wrap.

## Why are long-running transactions bad for VACUUM?

Old snapshots can keep obsolete row versions potentially visible, preventing VACUUM from reclaiming them promptly.

## What is the visibility map?

A PostgreSQL structure tracking page visibility properties; it helps VACUUM and can allow index-only scans to avoid heap visits when pages are all-visible.

## What is table bloat?

Excess physical relation size/inefficiency caused by accumulated dead/reusable space and workload patterns relative to live data.

---

# Quick Revision

~~~text
MVCC
 ↓
UPDATE / DELETE
 ↓
old tuple versions
 ↓
dead tuples
 ↓
VACUUM
 ↓
reusable space
~~~

### VACUUM

~~~text
Regular VACUUM
→ reuses internal space
→ maintains visibility
→ protects against XID wraparound
→ normally does NOT shrink file substantially
~~~

### VACUUM FULL

~~~text
VACUUM FULL
→ rewrite table
→ compact table
→ return unused space to OS
→ ACCESS EXCLUSIVE lock
~~~

### ANALYZE

~~~text
ANALYZE
→ collect statistics
→ help query planner
~~~

### Autovacuum

~~~text
Autovacuum launcher
       ↓
workers
       ↓
VACUUM + ANALYZE as needed
~~~

### Long Transaction Problem

~~~text
old transaction snapshot
        ↓
old tuples remain potentially visible
        ↓
VACUUM cannot reclaim them yet
        ↓
dead tuples / bloat pressure
~~~

### Most Important Rule

~~~text
Do not disable autovacuum casually.
Keep transactions short.
Always COMMIT or ROLLBACK.
~~~

---

## Key Takeaway

> **VACUUM is essential because PostgreSQL's MVCC design leaves obsolete tuple versions after UPDATE and DELETE. Regular VACUUM makes that space reusable, maintains visibility information, and protects against transaction-ID wraparound; autovacuum performs this maintenance automatically, while ANALYZE maintains the statistics used by the query planner.**

---

[← Previous: Lesson 28 — MVCC](./28-mvcc.md) | [Back to Roadmap](../README.md) | [Next: Lesson 30 — Write-Ahead Logging (WAL) →](./30-write-ahead-logging.md)