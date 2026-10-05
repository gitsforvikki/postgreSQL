# Lesson 25 — EXPLAIN & EXPLAIN ANALYZE

## Why Do We Need EXPLAIN?

Suppose an API endpoint is slow.

You may suspect:

~~~text
Missing index?
Bad JOIN?
Too many rows?
Expensive sort?
Wrong row estimates?
~~~

But guessing is not a good optimization strategy.

PostgreSQL has a **query planner** that decides how to execute every SQL query.

`EXPLAIN` lets you inspect that plan.

~~~text
SQL query
   ↓
PostgreSQL planner
   ↓
Execution plan
   ↓
Seq Scan?
Index Scan?
Join?
Sort?
Aggregate?
~~~

The goal of this lesson is to learn how to **read that plan without being afraid of it**.

---

# Part 1 — EXPLAIN

## 1. What Does EXPLAIN Do?

`EXPLAIN` shows the execution plan PostgreSQL intends to use.

Example:

~~~sql
EXPLAIN
SELECT *
FROM users
WHERE email = 'vikash@example.com';
~~~

A possible result might look like:

~~~text
Index Scan using idx_users_email on users
  (cost=0.42..8.44 rows=1 width=80)
  Index Cond: (email = 'vikash@example.com')
~~~

This tells you PostgreSQL plans to use the email index.

Important:

`EXPLAIN` normally **does not execute a SELECT query**. It shows the planner's estimated plan.

---

## 2. EXPLAIN Is an Estimate

Think:

~~~text
EXPLAIN
→ What PostgreSQL THINKS will happen
~~~

The planner estimates:

- which operations to use
- how many rows will be produced
- how expensive operations will be
- which join strategy to use
- whether to use indexes

Those estimates are based heavily on table statistics.

---

# Part 2 — EXPLAIN ANALYZE

## 3. What Does EXPLAIN ANALYZE Do?

`EXPLAIN ANALYZE` actually executes the query and reports what happened.

~~~sql
EXPLAIN ANALYZE
SELECT *
FROM users
WHERE email = 'vikash@example.com';
~~~

Possible output:

~~~text
Index Scan using idx_users_email on users
  (cost=0.42..8.44 rows=1 width=80)
  (actual time=0.025..0.027 rows=1 loops=1)
  Index Cond: (email = 'vikash@example.com')

Planning Time: 0.120 ms
Execution Time: 0.060 ms
~~~

Now you can compare:

~~~text
Planner estimate
      vs
Actual execution
~~~

That comparison is one of the most useful PostgreSQL performance skills.

---

## 4. EXPLAIN vs EXPLAIN ANALYZE

| Command | Executes SELECT? | Main purpose |
|---|---:|---|
| `EXPLAIN` | No | Show estimated plan |
| `EXPLAIN ANALYZE` | Yes | Execute and show estimated + actual behavior |

Easy memory:

~~~text
EXPLAIN
→ plan

EXPLAIN ANALYZE
→ plan + execute + measure
~~~

---

## 5. Be Careful with Data-Modifying Statements

`EXPLAIN ANALYZE` executes the statement.

So this:

~~~sql
EXPLAIN ANALYZE
DELETE FROM orders
WHERE status = 'cancelled';
~~~

really performs the DELETE unless you protect the test appropriately.

A useful testing pattern is:

~~~sql
BEGIN;

EXPLAIN ANALYZE
DELETE FROM orders
WHERE status = 'cancelled';

ROLLBACK;
~~~

The statement executes for measurement, but the transaction is rolled back afterward.

> Be especially careful with INSERT, UPDATE, DELETE, and other statements that change data.

---

# Part 3 — Reading a Query Plan

## 6. Plans Are Trees

A PostgreSQL plan is a tree of operations.

Example:

~~~text
Sort
 └── Hash Join
      ├── Seq Scan on orders
      └── Hash
           └── Seq Scan on users
~~~

Usually read from the **deepest/most-indented child nodes upward**.

Conceptually:

~~~text
scan data
   ↓
join data
   ↓
sort result
   ↓
return result
~~~

---

## 7. Important Plan Nodes

You should recognize these common nodes:

~~~text
Seq Scan
Index Scan
Index Only Scan
Bitmap Index Scan
Bitmap Heap Scan
Nested Loop
Hash Join
Merge Join
Sort
Hash
Aggregate
~~~

You do not need to memorize every PostgreSQL plan node. Focus on understanding what work each important node performs.

---

# Part 4 — Sequential Scan

## 8. Seq Scan

Example plan:

~~~text
Seq Scan on products
  (cost=0.00..18500.00 rows=500000 width=64)
~~~

Meaning:

~~~text
PostgreSQL reads table pages sequentially
        ↓
checks rows
        ↓
returns matches
~~~

A sequential scan is **not automatically a problem**.

It may be correct when:

- the table is small
- a large percentage of rows is required
- no useful index exists
- an index would cost more than scanning

Never optimize solely because you see `Seq Scan`.

---

# Part 5 — Index Scan

## 9. Index Scan

Example:

~~~text
Index Scan using idx_users_email on users
  (cost=0.42..8.44 rows=1 width=80)
~~~

Meaning:

~~~text
search index
   ↓
find matching index entries
   ↓
visit required heap/table rows
   ↓
return result
~~~

This is often good for highly selective lookups.

---

# Part 6 — Index Only Scan

## 10. Index Only Scan

Example:

~~~text
Index Only Scan using idx_orders_user_created on orders
~~~

PostgreSQL can potentially obtain the required indexed values without ordinary heap access for every tuple.

~~~text
index
  ↓
required data available
  ↓
return result
~~~

But MVCC visibility still matters, so heap fetches may still occur when PostgreSQL cannot confirm visibility from the visibility map.

This connects directly to Lesson 24's covering indexes.

---

# Part 7 — Bitmap Scans

## 11. Bitmap Index Scan + Bitmap Heap Scan

You may see:

~~~text
Bitmap Heap Scan on products
  └── Bitmap Index Scan on idx_products_category
~~~

High-level flow:

~~~text
index finds many matching row locations
            ↓
build bitmap of locations
            ↓
heap pages fetched efficiently
            ↓
matching rows returned
~~~

This can be useful when the query matches more rows than a simple index scan would efficiently handle, but not enough rows to make a sequential scan best.

Think of it as a middle ground in many workloads.

---

# Part 8 — Understanding cost

## 12. What Does cost Mean?

You may see:

~~~text
cost=0.42..8.44
~~~

This does **not** mean:

~~~text
0.42 ms to 8.44 ms
~~~

PostgreSQL cost is measured in internal planner cost units.

The two numbers represent:

~~~text
cost=startup_cost..total_cost
~~~

### Startup cost

Estimated work required before the node can begin producing rows.

### Total cost

Estimated total work if the node runs to completion.

Easy memory:

~~~text
cost=10..500
     ↑    ↑
 startup total
~~~

Do not compare planner cost directly to milliseconds.

---

# Part 9 — rows and width

## 13. Estimated rows

Example:

~~~text
rows=100
~~~

This means the planner estimates the node will output about 100 rows.

With EXPLAIN ANALYZE you may also see:

~~~text
actual rows=10000
~~~

Now there is a large mismatch:

~~~text
estimated = 100
actual    = 10,000
~~~

This is important because bad row estimates can lead PostgreSQL to choose an inefficient plan.

---

## 14. Why Row Estimates Can Be Wrong

Possible reasons include:

- stale statistics
- unusual/skewed data distribution
- correlated columns
- expressions that are hard to estimate
- planner statistics not detailed enough for the data

`ANALYZE` refreshes planner statistics:

~~~sql
ANALYZE orders;
~~~

But do not blindly run commands and assume the problem is solved. First understand the mismatch.

---

## 15. width

Example:

~~~text
width=120
~~~

`width` is the planner's estimate of the average row size in bytes for that plan node.

Why can width matter?

~~~text
more columns / wider rows
        ↓
more data moved through operations
        ↓
potentially more memory / I/O work
~~~

This is another reason not to write `SELECT *` when the application only needs a few columns.

---

# Part 10 — actual time

## 16. actual time

EXPLAIN ANALYZE may show:

~~~text
actual time=0.025..0.027
~~~

These are measured times in milliseconds for that node's execution timing.

Think:

~~~text
actual time=start..end
~~~

Do not simply add every node's displayed time together. Plan nodes are nested, and their timing relationships require careful interpretation.

Use the overall `Execution Time` as an important end-to-end database execution measurement.

---

# Part 11 — loops

## 17. What Does loops Mean?

Example:

~~~text
actual time=0.010..0.020 rows=1 loops=1000
~~~

The node ran 1000 times.

This is extremely important with nested loops.

A node that looks cheap once may become expensive when repeated thousands of times.

~~~text
small cost per loop
        ×
many loops
        =
large total work
~~~

Whenever you inspect an expensive plan, look at both:

~~~text
rows
loops
~~~

---

# Part 12 — Filter vs Index Cond

## 18. Index Cond

Example:

~~~text
Index Cond: (user_id = 10)
~~~

This means PostgreSQL is using the index condition to locate relevant index entries.

---

## 19. Filter

Example:

~~~text
Filter: (status = 'paid')
~~~

A filter is applied to rows produced by that plan node.

You may also see:

~~~text
Rows Removed by Filter: 500000
~~~

If PostgreSQL reads a huge number of rows and then discards most of them, that can reveal an optimization opportunity.

Mental model:

~~~text
Index Cond
→ helps locate candidate rows through index access

Filter
→ checks/removes rows at that node
~~~

---

# Part 13 — Sort

## 20. Sort Node

Example:

~~~text
Sort
  Sort Key: created_at DESC
  └── Seq Scan on orders
~~~

PostgreSQL reads rows and then sorts them.

If the query is:

~~~sql
SELECT id, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
~~~

a matching composite index such as:

~~~sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
~~~

may allow PostgreSQL to avoid a separate sort, depending on the full plan and costs.

---

# Part 14 — Join Nodes

## 21. Nested Loop

High-level idea:

~~~text
for each row from outer input
        ↓
look for matching rows in inner input
~~~

Conceptually:

~~~text
Outer row 1 → search inner
Outer row 2 → search inner
Outer row 3 → search inner
...
~~~

Nested loops can be excellent when the outer result is small and the inner lookup is efficiently indexed.

They can be expensive when large inputs cause an inner operation to repeat many times.

Check `loops`.

---

## 22. Hash Join

High-level flow:

~~~text
one input
   ↓
build hash table
   ↓
scan other input
   ↓
probe hash table for matches
~~~

Hash joins are commonly useful for equality joins, especially when processing substantial row sets.

Example join condition:

~~~sql
ON orders.user_id = users.id
~~~

Do not memorize `Hash Join = good` or `Nested Loop = bad`. The correct strategy depends on data size, estimates, indexes, memory, and query shape.

---

## 23. Merge Join

High-level idea:

~~~text
sorted input A
      +
sorted input B
      ↓
walk through both in order
      ↓
match values
~~~

Merge joins can be effective when both inputs are already sorted or can be sorted efficiently for the join keys.

Again, the goal is not to force a join type. Understand why PostgreSQL selected it.

---

# Part 15 — BUFFERS

## 24. EXPLAIN (ANALYZE, BUFFERS)

A very useful diagnostic form is:

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total
FROM orders
WHERE user_id = 10;
~~~

`BUFFERS` adds information about PostgreSQL buffer/page activity.

You may see information such as:

~~~text
shared hit
shared read
~~~

High-level meaning:

~~~text
shared hit
→ required page was already available in PostgreSQL shared buffers

shared read
→ PostgreSQL had to read a page into shared buffers
~~~

This helps distinguish CPU/query-processing work from significant data-access/I/O behavior.

---

## 25. Why BUFFERS Matters

Two queries can have similar SQL but very different data-access behavior.

~~~text
Query A
→ most pages already cached

Query B
→ many pages must be read
~~~

`BUFFERS` gives you another layer of evidence when diagnosing performance.

At your level, focus on recognizing excessive page work rather than memorizing every buffer counter.

---

# Part 16 — Planning Time and Execution Time

## 26. Planning Time

Example:

~~~text
Planning Time: 0.300 ms
~~~

This is time PostgreSQL spent producing the execution plan.

## 27. Execution Time

Example:

~~~text
Execution Time: 45.200 ms
~~~

This is the measured execution time reported by EXPLAIN ANALYZE for the query execution.

Mental model:

~~~text
SQL
 ↓
Planning Time
 ↓
chosen plan
 ↓
Execution Time
 ↓
result
~~~

---

# Part 17 — Practical ShopHub Example

## 28. Slow Order Query

Suppose this endpoint is slow:

~~~sql
SELECT id, status, total, created_at
FROM orders
WHERE user_id = 100
ORDER BY created_at DESC
LIMIT 20;
~~~

You run:

~~~sql
EXPLAIN ANALYZE
SELECT id, status, total, created_at
FROM orders
WHERE user_id = 100
ORDER BY created_at DESC
LIMIT 20;
~~~

Imagine you see:

~~~text
Limit
  └── Sort
       Sort Key: created_at DESC
       └── Seq Scan on orders
            Filter: (user_id = 100)
            Rows Removed by Filter: 999000
~~~

Interpretation:

~~~text
scan huge table
      ↓
discard almost everything
      ↓
sort remaining rows
      ↓
return only 20
~~~

This suggests a possible access-pattern problem.

A candidate index might be:

~~~sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
~~~

Then measure again.

Possible improved shape:

~~~text
Limit
  └── Index Scan using idx_orders_user_created
       Index Cond: (user_id = 100)
~~~

Now PostgreSQL may locate that user's newest orders directly without scanning and sorting a huge unrelated set.

---

# Part 18 — Estimated vs Actual Rows

## 29. One of the Most Important Checks

Suppose:

~~~text
estimated rows = 10
actual rows    = 100000
~~~

That is a major estimation error.

Why does it matter?

~~~text
planner expects tiny result
        ↓
chooses strategy suitable for tiny result
        ↓
actual result is huge
        ↓
strategy may perform badly
~~~

Whenever you read EXPLAIN ANALYZE, compare estimated and actual row counts at important nodes.

---

# Part 19 — Optimization Workflow

## 30. A Repeatable Workflow

When an API or SQL query is slow:

~~~text
1. Identify the exact SQL query
        ↓
2. Run EXPLAIN ANALYZE
        ↓
3. Read plan from child nodes upward
        ↓
4. Look at scan types
        ↓
5. Compare estimated vs actual rows
        ↓
6. Check rows removed by filters
        ↓
7. Check loops
        ↓
8. Check sorts / joins
        ↓
9. Add BUFFERS when useful
        ↓
10. Make ONE targeted change
        ↓
11. Run EXPLAIN ANALYZE again
        ↓
12. Compare before vs after
~~~

This is much better than random optimization.

---

# Part 20 — EXPLAIN Options

## 31. Useful Form

Common diagnostic command:

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
~~~

PostgreSQL supports additional EXPLAIN options, but you do not need all of them immediately.

For now, master:

~~~text
EXPLAIN
EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS)
~~~

---

# Common Mistakes

## 32. Thinking cost Means Milliseconds

`cost` is a planner estimate in internal cost units, not elapsed milliseconds.

## 33. Thinking Every Seq Scan Is Bad

A sequential scan can be the correct plan.

## 34. Looking Only at the Top Plan Node

Plans are trees. Expensive work may happen deep inside child nodes.

## 35. Ignoring loops

A cheap inner node repeated thousands of times can dominate the query.

## 36. Ignoring Estimated vs Actual Rows

Large estimation errors can explain poor plan choices.

## 37. Creating an Index Before Looking at the Plan

Measure first. Then make a targeted change.

## 38. Running EXPLAIN ANALYZE on DELETE/UPDATE Carelessly

It executes the statement. Use a safe environment or transaction + ROLLBACK when appropriate.

## 39. Assuming Index Scan Is Always Faster

If most rows are required, a sequential scan may be cheaper.

## 40. Optimizing Without Testing Again

Always compare the new plan and execution behavior after your change.

---

# Interview Revision

## What does EXPLAIN do?

It shows PostgreSQL's estimated execution plan without normally executing a SELECT query.

## What does EXPLAIN ANALYZE do?

It executes the query and reports both planner estimates and actual execution statistics.

## Is EXPLAIN ANALYZE safe on UPDATE or DELETE?

It really executes the statement, so use it carefully. In a controlled test you can wrap the operation in a transaction and roll it back.

## What does cost=0.42..8.44 mean?

The first value is estimated startup cost and the second is estimated total cost, in PostgreSQL planner cost units—not milliseconds.

## What does rows mean?

The planner's estimated number of rows produced by the node.

## What does actual rows mean?

The number of rows actually produced during EXPLAIN ANALYZE execution.

## Why compare estimated and actual rows?

Large mismatches can cause the planner to choose an inefficient strategy.

## What does loops mean?

How many times that plan node was executed.

## Index Cond vs Filter?

`Index Cond` participates in locating entries through index access; `Filter` evaluates rows at that node and may discard them.

## Is Seq Scan always bad?

No.

## What is an Index Only Scan?

A scan that can obtain required indexed values without ordinary heap access for every tuple, subject to MVCC visibility requirements.

## What are Bitmap Index Scan and Bitmap Heap Scan?

They commonly work together to collect matching row locations from indexes and fetch relevant heap pages efficiently.

## What does BUFFERS add?

Information about buffer/page activity, helping diagnose data-access and I/O behavior.

## What are Planning Time and Execution Time?

Planning Time measures plan generation; Execution Time measures the query execution reported by EXPLAIN ANALYZE.

---

# Quick Revision

~~~text
EXPLAIN
→ estimated plan

EXPLAIN ANALYZE
→ execute + estimated plan + actual measurements

EXPLAIN (ANALYZE, BUFFERS)
→ execution measurements + buffer activity
~~~

### Important Plan Nodes

~~~text
Seq Scan
Index Scan
Index Only Scan
Bitmap Scan
Nested Loop
Hash Join
Merge Join
Sort
Aggregate
~~~

### Important Numbers

~~~text
cost
→ planner estimate, NOT milliseconds

rows
→ estimated output rows

actual rows
→ real output rows

loops
→ number of executions

width
→ estimated average row width
~~~

### What to Look For

~~~text
Huge Seq Scan?
Large Rows Removed by Filter?
Estimated rows very different from actual?
Inner node running many loops?
Expensive sort?
Wrong/missing index for access pattern?
Lots of buffer reads?
~~~

### Performance Workflow

~~~text
Slow query
   ↓
EXPLAIN ANALYZE
   ↓
Understand actual work
   ↓
Targeted optimization
   ↓
EXPLAIN ANALYZE again
   ↓
Compare
~~~

---

## Key Takeaway

> **EXPLAIN shows what PostgreSQL plans to do; EXPLAIN ANALYZE executes the query and shows what actually happened. Learn to read scans, estimated vs actual rows, loops, filters, joins, sorts, and buffers—and optimize from evidence rather than guesses.**

---

[← Previous: Lesson 24 — Advanced Indexing](./24-advanced-indexing.md) | [Back to Roadmap](../README.md) | [Next: Lesson 26 — Query Optimization →](./26-query-optimization.md)