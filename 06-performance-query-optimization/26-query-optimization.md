# Lesson 26 — Query Optimization

## What Is Query Optimization?

Query optimization means reducing the amount of unnecessary work PostgreSQL and your application must perform to return the required result.

It is **not** simply:

~~~text
slow query
   ↓
add index
~~~

A query can be slow because of:

- too many rows being scanned
- too many columns being returned
- unnecessary JOINs
- poor indexes
- wrong composite-index order
- expensive sorting or aggregation
- deep OFFSET pagination
- N+1 queries from application code
- stale or inaccurate planner statistics
- repeated expensive reports
- poor schema/query design

A better mental model is:

~~~text
Slow request
    ↓
Find exact SQL
    ↓
Measure execution plan
    ↓
Identify unnecessary work
    ↓
Make targeted change
    ↓
Measure again
~~~

---

# Part 1 — The Main Rule

## 1. Make PostgreSQL Do Less Work

Most optimization techniques reduce one or more of these:

~~~text
rows scanned
rows returned
columns returned
sort work
join work
repeated queries
disk/page access
network transfer
application processing
~~~

Example:

~~~sql
SELECT *
FROM orders;
~~~

versus:

~~~sql
SELECT id, status, total
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
~~~

The second query expresses a much smaller requirement.

Optimization starts with asking:

**What data does the application actually need?**

---

# Part 2 — Select Only Needed Columns

## 2. Avoid SELECT * When You Do Not Need Everything

Suppose `products` contains:

~~~text
id
name
description
price
metadata
large_details
created_at
updated_at
~~~

But the product-card UI only needs:

~~~text
id
name
price
~~~

Prefer:

~~~sql
SELECT id, name, price
FROM products
WHERE category_id = $1;
~~~

instead of:

~~~sql
SELECT *
FROM products
WHERE category_id = $1;
~~~

Benefits can include:

~~~text
less data read/moved
less network transfer
smaller application objects
better chance of useful covering/index-only strategies
~~~

Do not optimize this mechanically for every tiny query, but avoid retrieving large unused data.

---

# Part 3 — Reduce Rows Early

## 3. Filter the Data You Actually Need

Bad application pattern:

~~~sql
SELECT * FROM orders;
~~~

Then JavaScript:

~~~js
const userOrders = orders.filter(
  (order) => order.user_id === userId
);
~~~

Better:

~~~sql
SELECT id, status, total
FROM orders
WHERE user_id = $1;
~~~

Why?

~~~text
Bad
DB → huge result → network → Node.js → filter

Better
DB filters → smaller result → network → Node.js
~~~

Let PostgreSQL perform relational filtering close to the data.

---

# Part 4 — Index the Access Pattern

## 4. Indexes Should Match Important Queries

Query:

~~~sql
SELECT id, status, total, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
~~~

Candidate index:

~~~sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
~~~

Reason:

~~~text
WHERE user_id
      +
ORDER BY created_at
      ↓
composite access pattern
~~~

Do not create indexes blindly. Verify them with `EXPLAIN ANALYZE`.

---

# Part 5 — Avoid Unnecessary JOINs

## 5. JOIN Only What You Need

Suppose the API needs only order data.

Unnecessary:

~~~sql
SELECT o.id, o.total
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN addresses a ON a.user_id = u.id
WHERE o.id = $1;
~~~

If no user/address data or filtering is required, those joins add work without adding value.

Better:

~~~sql
SELECT id, total
FROM orders
WHERE id = $1;
~~~

Rule:

~~~text
JOIN because the result/condition requires it
not because the relationship exists
~~~

---

# Part 6 — N+1 Query Problem

## 6. What Is N+1?

Suppose you first fetch 100 users:

~~~sql
SELECT id, name
FROM users
LIMIT 100;
~~~

Then your Node.js code runs one query per user:

~~~text
1 query → users
100 queries → orders for each user
----------------------------
101 total queries
~~~

This is the **N+1 query problem**.

Example application pattern:

~~~js
const users = await getUsers();

for (const user of users) {
  user.orders = await getOrdersByUser(user.id);
}
~~~

This can create many database round trips.

---

## 7. Solving N+1

Depending on the required output, you might use:

- a JOIN
- batched query with `IN` / `ANY`
- aggregation
- ORM eager-loading/batching features

Example batch:

~~~sql
SELECT id, user_id, total
FROM orders
WHERE user_id = ANY($1::bigint[]);
~~~

Flow:

~~~text
100 user IDs
    ↓
one batched order query
    ↓
group results in application
~~~

Or use an appropriate JOIN when the shape makes sense.

Important:

> Avoid N+1, but do not replace it automatically with one enormous query that produces massive duplicated result sets.

Optimize the complete data-fetching pattern.

---

# Part 7 — Pagination

## 8. LIMIT Is Important

A user-facing endpoint rarely needs millions of rows at once.

~~~sql
SELECT id, title, created_at
FROM posts
ORDER BY created_at DESC
LIMIT 20;
~~~

Pagination controls result size and protects both the database and application.

---

## 9. OFFSET Pagination

Common pagination:

~~~sql
SELECT id, created_at
FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;
~~~

Problem at deep pages:

~~~text
PostgreSQL still needs to walk/process
many rows before the requested page
        ↓
discard earlier rows
        ↓
return 20
~~~

Large offsets can become expensive.

---

## 10. Keyset Pagination

Instead of saying:

~~~text
skip 100,000 rows
~~~

say:

~~~text
continue after the last row I already saw
~~~

Example:

~~~sql
SELECT id, created_at
FROM orders
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

Matching index:

~~~sql
CREATE INDEX idx_orders_created_id
ON orders (created_at DESC, id DESC);
~~~

Mental model:

~~~text
OFFSET
→ count/skip previous rows

KEYSET
→ seek from cursor position
~~~

Keyset pagination is especially useful for large feeds, activity histories, chats, and order lists.

---

# Part 8 — Avoid Functions That Defeat Your Index Strategy

## 11. Function on Indexed Column

Suppose:

~~~sql
CREATE INDEX idx_users_email
ON users (email);
~~~

But the query is:

~~~sql
SELECT *
FROM users
WHERE LOWER(email) = LOWER($1);
~~~

The important search expression is `LOWER(email)`, not plain `email`.

If this query is important, consider an expression index:

~~~sql
CREATE INDEX idx_users_lower_email
ON users (LOWER(email));
~~~

General rule:

~~~text
Query expression
must align with
index expression / operator strategy
~~~

Do not rewrite business requirements just to force an index. Design the index around legitimate access patterns.

---

# Part 9 — Sorting

## 12. Expensive Sorts

Query:

~~~sql
SELECT id, total, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
~~~

Without a suitable index, the plan may look like:

~~~text
Limit
  ↓
Sort
  ↓
Filter/Scan many orders
~~~

A matching index may allow:

~~~text
Limit
  ↓
Index Scan in required order
~~~

Candidate:

~~~sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
~~~

Again, verify with `EXPLAIN ANALYZE`.

---

# Part 10 — Aggregation

## 13. Expensive Aggregates

Suppose an admin dashboard repeatedly calculates:

~~~sql
SELECT
  DATE(created_at) AS day,
  SUM(total) AS revenue
FROM orders
WHERE status = 'paid'
GROUP BY DATE(created_at)
ORDER BY day;
~~~

On a very large table, repeating this computation for every dashboard request may become expensive.

Possible strategies depend on freshness requirements:

~~~text
better filtering/indexing
      ↓
pre-aggregation
      ↓
materialized view
      ↓
summary table / reporting pipeline
~~~

Do not create a materialized view just because aggregation exists. Measure first.

---

## 14. Materialized View for Repeated Expensive Reports

If the report is expensive and slightly stale data is acceptable:

~~~sql
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT
  DATE(created_at) AS day,
  SUM(total) AS revenue
FROM orders
WHERE status = 'paid'
GROUP BY DATE(created_at);
~~~

Then dashboard reads can query the precomputed result.

Tradeoff:

~~~text
faster repeated reads
        ↕
refresh complexity + stale data window
~~~

---

# Part 11 — CTE vs Subquery vs JOIN

## 15. Syntax Alone Does Not Guarantee Performance

Do not memorize:

~~~text
JOIN is always faster than subquery
CTE is always slower
subquery is always bad
~~~

Those statements are incorrect as general rules.

PostgreSQL's optimizer can transform/integrate many query forms.

Choose syntax for correctness and clarity first, then inspect the actual execution plan.

~~~text
SQL syntax
   ↓
PostgreSQL planner
   ↓
actual execution plan
~~~

The plan—not your assumption about the syntax—determines the work.

---

# Part 12 — EXISTS vs IN

## 16. Choose Semantics First

Example: users who have at least one order.

~~~sql
SELECT u.id, u.name
FROM users u
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.user_id = u.id
);
~~~

`EXISTS` expresses the requirement clearly:

~~~text
Does at least one matching row exist?
~~~

`IN` can also be appropriate in many cases.

Do not choose between them based on folklore such as 'EXISTS is always faster'. PostgreSQL may transform queries, and performance depends on the plan.

Remember the semantic caveat from Lesson 8: `NOT IN` can behave unexpectedly when NULLs are present; `NOT EXISTS` is often safer for anti-join semantics.

---

# Part 13 — Statistics

## 17. Planner Statistics Matter

PostgreSQL's planner estimates:

~~~text
How many rows will match?
How selective is this condition?
Which join should I use?
Should I use an index?
~~~

using statistics.

If statistics are stale or insufficient, estimates can be wrong.

Example:

~~~text
estimated rows = 100
actual rows    = 500,000
~~~

That mismatch can lead to a poor plan.

---

## 18. ANALYZE

~~~sql
ANALYZE orders;
~~~

`ANALYZE` updates planner statistics.

PostgreSQL normally performs automatic analysis through autovacuum-related maintenance, but understanding manual `ANALYZE` is useful during diagnosis, bulk data changes, and controlled maintenance.

Do not use `ANALYZE` as a magical fix. Inspect the plan before and after.

---

# Part 14 — Parameterized Queries

## 19. Parameterization Is Primarily Security, but Also Good Query Discipline

Node.js:

~~~js
const result = await pool.query(
  `
    SELECT id, status, total
    FROM orders
    WHERE user_id = $1
    ORDER BY created_at DESC
    LIMIT $2
  `,
  [userId, limit]
);
~~~

Benefits:

~~~text
prevents SQL injection for values
keeps SQL structure separate from data
works naturally with PostgreSQL execution
~~~

Do not build values directly into SQL strings.

Security is covered more deeply in Lesson 36.

---

# Part 15 — Prepared Statements

## 20. Prepared Statements

With node-postgres, a query can be given a name:

~~~js
await client.query({
  name: 'orders-by-user',
  text: `
    SELECT id, total
    FROM orders
    WHERE user_id = $1
  `,
  values: [userId],
});
~~~

This allows PostgreSQL/node-postgres to use prepared-statement behavior for repeated executions on a connection.

But:

> Do not assume prepared statements automatically make every query faster.

Query planning behavior, parameter distributions, connection pooling, and workload all matter.

Use them where repeated query execution and measurements justify them.

---

# Part 16 — Large Result Sets

## 21. Avoid Returning Huge Results to the API

Bad endpoint:

~~~text
GET /orders
→ 2,000,000 rows
→ JSON serialization
→ network transfer
→ browser memory
~~~

Better API design:

~~~text
filters
pagination
limited fields
reasonable maximum page size
~~~

Database performance is connected to API design.

---

# Part 17 — Transactions and Performance

## 22. Keep Transactions Short

Bad:

~~~text
BEGIN
 ↓
update database
 ↓
call payment API
 ↓
wait 3 seconds
 ↓
update database
 ↓
COMMIT
~~~

Long transactions can:

- hold locks longer
- keep database connections occupied
- increase contention
- delay cleanup of old row versions

Prefer short database transactions around the database work that truly needs atomicity.

External APIs cannot be made part of a PostgreSQL transaction simply by keeping the DB transaction open.

---

# Part 18 — Batch Operations

## 23. Avoid Thousands of Tiny Round Trips

Bad pattern:

~~~text
for 10,000 records
    ↓
10,000 separate database calls
~~~

Depending on the operation, consider:

~~~text
multi-row INSERT
batched queries
COPY for bulk loading
set-based UPDATE/DELETE
~~~

Example multi-row insert:

~~~sql
INSERT INTO tags (name)
VALUES
  ($1),
  ($2),
  ($3);
~~~

SQL databases are designed to work efficiently with sets of rows.

---

# Part 19 — ORM Performance

## 24. ORM Does Not Remove Query Optimization

Whether you use:

~~~text
Prisma
Drizzle
Sequelize
TypeORM
raw pg
~~~

PostgreSQL still executes SQL.

ORM-generated code can still cause:

- N+1 queries
- unnecessary columns
- unnecessary relations
- poor pagination
- missing indexes
- long transactions

Therefore:

~~~text
ORM
   ↓
generated SQL
   ↓
PostgreSQL planner
   ↓
execution plan
~~~

When an ORM endpoint is slow, inspect the SQL it generates.

---

# Part 20 — Caching

## 25. Caching Is Not the First Fix for a Bad Query

Bad approach:

~~~text
query takes 5 seconds
   ↓
put cache in front
   ↓
ignore database problem
~~~

Better:

~~~text
understand query
   ↓
optimize unnecessary work
   ↓
then cache if workload benefits
~~~

Caching is useful for data that is expensive to compute/read repeatedly and can tolerate the chosen freshness strategy.

But caching introduces:

~~~text
invalidation
staleness
extra infrastructure
consistency decisions
~~~

Do not use cache as a substitute for understanding PostgreSQL.

---

# Part 21 — ShopHub Optimization Example

## 26. Product Listing Endpoint

Suppose ShopHub has:

~~~text
GET /products?category=5&page=...
~~~

Initial query:

~~~sql
SELECT *
FROM products
WHERE category_id = $1
ORDER BY created_at DESC
LIMIT 20 OFFSET $2;
~~~

At small scale this may be fine.

As data grows, inspect:

~~~text
Are all columns needed?
Is category_id selective?
Is there a matching composite index?
Are deep offsets expensive?
Is the sort avoidable?
~~~

Possible query:

~~~sql
SELECT id, name, price, created_at
FROM products
WHERE category_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

Candidate index:

~~~sql
CREATE INDEX idx_products_category_cursor
ON products (category_id, created_at DESC, id DESC);
~~~

Now the access pattern becomes:

~~~text
category equality
       ↓
cursor position
       ↓
already ordered
       ↓
take next 20
~~~

Then confirm with `EXPLAIN ANALYZE`.

---

# Part 22 — Optimization Checklist

## 27. When a Query Is Slow, Ask These Questions

~~~text
1. What exact SQL is slow?
2. How long does it actually take?
3. What does EXPLAIN ANALYZE show?
4. How many rows are scanned vs returned?
5. Are estimates close to actual rows?
6. Are many rows removed by filters?
7. Is a node repeated many times?
8. Is there an expensive sort?
9. Are joins necessary and efficient?
10. Does an index match the access pattern?
11. Are we selecting unnecessary columns?
12. Is pagination appropriate?
13. Is the application creating N+1 queries?
14. Are statistics current?
15. Are transactions held open too long?
16. Did performance improve after the change?
~~~

This checklist is more valuable than memorizing isolated tricks.

---

# Common Mistakes

## 28. Optimizing Without Measuring

Always establish a baseline and compare after changes.

## 29. Adding Indexes to Every WHERE Column

Indexes should match important query patterns and data distribution.

## 30. Believing JOIN Is Always Faster Than a Subquery

Inspect the execution plan instead of relying on syntax myths.

## 31. Fetching Everything and Filtering in JavaScript

Push relational filtering into SQL when appropriate.

## 32. Using Deep OFFSET Forever

Consider keyset pagination for large ordered datasets.

## 33. Ignoring N+1

Database round trips can dominate endpoint latency.

## 34. Returning Millions of Rows

Use filtering, pagination, and bounded API responses.

## 35. Holding Transactions Open During External API Calls

Keep database transactions short.

## 36. Assuming an ORM Optimizes Everything

Inspect generated SQL and execution plans.

## 37. Using Cache to Hide a Bad Query

Fix unnecessary database work first; then decide whether caching adds value.

## 38. Changing Five Things at Once

If you change query shape, indexes, configuration, and caching simultaneously, you may not know which change helped.

Prefer:

~~~text
measure
→ one targeted change
→ measure
~~~

---

# Interview Revision

## What is query optimization?

Reducing unnecessary database/application work while preserving correct results and acceptable consistency.

## What is the first step when a query is slow?

Identify the exact query and measure it, usually with `EXPLAIN ANALYZE` in an appropriate environment.

## Why avoid SELECT *?

It can retrieve unnecessary data, increase I/O/network work, and prevent narrower query/index strategies.

## What is N+1?

One initial query followed by one additional query for each returned item, causing excessive database round trips.

## How can N+1 be solved?

JOINs, batched queries, aggregation, or appropriate ORM loading/batching depending on the required data shape.

## OFFSET vs keyset pagination?

OFFSET skips earlier rows and can become expensive at deep pages. Keyset pagination continues from a known ordered key/cursor.

## Is JOIN always faster than a subquery?

No. PostgreSQL may transform query forms; inspect the execution plan.

## Is EXISTS always faster than IN?

No. Choose correct semantics first and measure the plan.

## Why do planner statistics matter?

They help PostgreSQL estimate row counts/selectivity and choose scans, joins, indexes, and other strategies.

## What does ANALYZE do?

It updates planner statistics.

## Does an ORM remove the need for SQL optimization?

No. The ORM ultimately causes SQL to execute against PostgreSQL.

## Should caching be the first fix for a slow SQL query?

Usually no. Understand and optimize the query first, then use caching where its freshness/complexity tradeoff is worthwhile.

## Why keep transactions short?

To reduce lock duration, connection occupancy, contention, and other concurrency/maintenance costs.

---

# Quick Revision

~~~text
QUERY OPTIMIZATION
        ↓
Do less unnecessary work
~~~

### Main Areas

~~~text
Rows
→ filter early / return only needed rows

Columns
→ avoid unnecessary SELECT *

Indexes
→ match real access patterns

Joins
→ remove unnecessary joins

N+1
→ batch / join / aggregate

Pagination
→ LIMIT + consider keyset

Sorts
→ matching index when justified

Aggregates
→ precompute only when measured need exists

Statistics
→ planner needs good estimates

Transactions
→ keep short
~~~

### Correct Workflow

~~~text
Slow endpoint
    ↓
Exact SQL
    ↓
EXPLAIN ANALYZE
    ↓
Find expensive work
    ↓
Targeted change
    ↓
EXPLAIN ANALYZE again
    ↓
Compare
~~~

### Golden Rule

~~~text
Never optimize from assumptions.

Measure → change → measure.
~~~

---

## Key Takeaway

> **PostgreSQL query optimization is the process of reducing unnecessary work. Fetch only what you need, design indexes around real access patterns, avoid N+1 and deep pagination problems, keep transactions short, inspect planner statistics, and use EXPLAIN ANALYZE to measure every important optimization.**

---

[← Previous: Lesson 25 — EXPLAIN & EXPLAIN ANALYZE](./25-explain-and-explain-analyze.md) | [Back to Roadmap](../README.md) | [Next: Lesson 27 — PostgreSQL Architecture →](../07-postgresql-internals/27-postgresql-architecture.md)