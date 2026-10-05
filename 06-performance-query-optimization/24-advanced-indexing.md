# Lesson 24 — Advanced Indexing

## Why Do We Need Advanced Indexing?

Lesson 23 gave you the foundation: an index helps PostgreSQL locate rows efficiently, but indexes should be designed around real query patterns.

Now imagine these production queries:

~~~sql
-- Recent orders for one user
SELECT id, total, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;

-- Only pending orders
SELECT *
FROM orders
WHERE status = 'pending';

-- Case-insensitive email lookup
SELECT *
FROM users
WHERE LOWER(email) = LOWER($1);

-- Search inside JSONB
SELECT *
FROM products
WHERE metadata @> '{"brand":"Apple"}';
~~~

A simple one-column B-tree index is not always the best solution.

Advanced indexing means choosing an index whose **structure matches the real access pattern**.

~~~text
Query pattern
     ↓
Choose suitable index
     ↓
Reduce unnecessary work
     ↓
Measure with EXPLAIN ANALYZE
~~~

---

# Part 1 — Composite Indexes

## 1. What Is a Composite Index?

A composite index contains more than one column.

~~~sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
~~~

It can support a query such as:

~~~sql
SELECT id, total, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
~~~

Why does this combination make sense?

~~~text
WHERE user_id = ?
        ↓
first index column

ORDER BY created_at DESC
        ↓
next index column
~~~

The index is designed around the complete query rather than treating each column independently.

---

## 2. Column Order Is Critical

These indexes are not equivalent:

~~~sql
CREATE INDEX idx_a
ON orders (user_id, created_at);

CREATE INDEX idx_b
ON orders (created_at, user_id);
~~~

Think of the first index as:

~~~text
user 1
 ├── Jan
 ├── Feb
 └── Mar

user 2
 ├── Jan
 └── Apr
~~~

The data is organized first by user_id, then by created_at within that leading ordering.

So the order should be chosen from the queries you actually need to optimize.

---

## 3. Leftmost-Prefix Principle

For an index:

~~~text
(A, B, C)
~~~

the leading columns are especially important.

Natural access patterns include:

~~~text
A
A + B
A + B + C
~~~

A query using only C usually cannot exploit this B-tree ordering nearly as effectively.

Example:

~~~sql
CREATE INDEX idx_orders_user_status_created
ON orders (user_id, status, created_at DESC);
~~~

Good candidate:

~~~sql
SELECT *
FROM orders
WHERE user_id = $1
  AND status = 'paid'
ORDER BY created_at DESC;
~~~

But if your major workload is:

~~~sql
SELECT *
FROM orders
WHERE status = 'paid';
~~~

the leading user_id may make that index a poor fit for this query.

---

## 4. Equality, Range, and Column Order

A useful design heuristic for many B-tree indexes is:

~~~text
equality filters first
      ↓
then range / ordering columns
~~~

Example query:

~~~sql
SELECT *
FROM orders
WHERE user_id = $1
  AND created_at >= $2
ORDER BY created_at DESC;
~~~

Candidate:

~~~sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
~~~

Here user_id is an equality condition and created_at is the range/order dimension.

This is a heuristic, not a universal formula. Always verify the real plan.

---

# Part 2 — Partial Indexes

## 5. What Is a Partial Index?

A partial index stores entries only for rows matching a condition.

Example:

~~~sql
CREATE INDEX idx_orders_pending
ON orders (created_at DESC)
WHERE status = 'pending';
~~~

Instead of indexing every order:

~~~text
paid
cancelled
shipped
pending
refunded
~~~

the index contains only:

~~~text
pending orders
~~~

This can make the index smaller and cheaper to maintain when the indexed subset is relatively small and frequently queried.

---

## 6. Partial Index Example

Suppose ShopHub has:

~~~text
10,000,000 total orders
100,000 pending orders
~~~

The admin dashboard frequently runs:

~~~sql
SELECT id, user_id, created_at
FROM orders
WHERE status = 'pending'
ORDER BY created_at DESC
LIMIT 100;
~~~

A partial index can target exactly that workload:

~~~sql
CREATE INDEX idx_pending_orders_created
ON orders (created_at DESC)
WHERE status = 'pending';
~~~

Mental model:

~~~text
Full index
→ entries for all 10,000,000 rows

Partial index
→ entries only for relevant subset
~~~

---

## 7. Important Partial-Index Rule

The query condition must be compatible with the index predicate for PostgreSQL to use the partial index.

If the index is:

~~~sql
CREATE INDEX idx_pending_orders
ON orders (created_at)
WHERE status = 'pending';
~~~

then a query specifically targeting pending orders can match it naturally.

A query asking for all statuses cannot simply treat that partial index as a complete index for the table.

---

# Part 3 — Expression Indexes

## 8. The Function-on-Column Problem

Suppose you have:

~~~sql
CREATE INDEX idx_users_email
ON users (email);
~~~

But the application queries:

~~~sql
SELECT *
FROM users
WHERE LOWER(email) = LOWER($1);
~~~

The query is searching on the expression:

~~~text
LOWER(email)
~~~

rather than simply the original email value.

A normal index on email is not the same structure as an index on LOWER(email).

---

## 9. Expression Index

PostgreSQL allows an index on an expression:

~~~sql
CREATE INDEX idx_users_lower_email
ON users (LOWER(email));
~~~

Now this access pattern matches the indexed expression:

~~~sql
SELECT *
FROM users
WHERE LOWER(email) = LOWER($1);
~~~

Mental model:

~~~text
Query uses LOWER(email)
          ↓
Index stores LOWER(email)
          ↓
structures match
~~~

---

## 10. Unique Expression Index

You can also enforce uniqueness on an expression.

Example: case-insensitive email uniqueness:

~~~sql
CREATE UNIQUE INDEX uq_users_lower_email
ON users (LOWER(email));
~~~

Then values such as:

~~~text
Vikash@example.com
vikash@example.com
~~~

conflict according to the indexed lower-case representation.

This is a powerful combination of performance and data integrity.

---

# Part 4 — Covering Indexes and INCLUDE

## 11. The Heap Lookup

With a normal index scan, PostgreSQL may:

~~~text
search index
    ↓
find matching entry
    ↓
visit table heap
    ↓
read additional columns
~~~

If a query runs extremely often, reducing heap access may help.

---

## 12. INCLUDE Columns

PostgreSQL lets you store additional non-key columns in an index with INCLUDE.

Example:

~~~sql
CREATE INDEX idx_orders_user_created_cover
ON orders (user_id, created_at DESC)
INCLUDE (status, total);
~~~

Query:

~~~sql
SELECT created_at, status, total
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
~~~

The searchable/order key is:

~~~text
user_id, created_at
~~~

Additional payload columns are:

~~~text
status, total
~~~

Those INCLUDE columns can help make an index-only scan possible when PostgreSQL's visibility requirements are also satisfied.

---

## 13. Key Columns vs INCLUDE Columns

Index:

~~~sql
ON orders (user_id, created_at DESC)
INCLUDE (status, total)
~~~

Think:

~~~text
KEY COLUMNS
user_id
created_at
→ control search/order structure

INCLUDED COLUMNS
status
total
→ stored as payload
→ available to satisfy query output
~~~

INCLUDE columns do not play the same role as key columns in B-tree search ordering.

---

## 14. Why Not INCLUDE Everything?

A huge covering index has costs:

~~~text
larger index
    ↓
more disk
    ↓
more cache pressure
    ↓
more write maintenance
~~~

Use INCLUDE for important, frequently executed queries where measurements justify it.

---

# Part 5 — Index-Only Scans

## 15. What Is an Index-Only Scan?

An index-only scan means PostgreSQL can return required indexed values without ordinary heap access for every tuple.

~~~text
Index Scan
index → heap → result

Index-Only Scan
index → result
when visibility can be confirmed efficiently
~~~

However, PostgreSQL uses MVCC, so it still needs to know whether tuples are visible to the current transaction.

The visibility map helps PostgreSQL determine when heap visits can be avoided.

So:

> Having every selected column in the index does **not guarantee** PostgreSQL will always perform an index-only scan.

---

# Part 6 — GIN Indexes

## 16. What Is GIN?

GIN stands for **Generalized Inverted Index**.

It is especially useful when one row can contain multiple searchable values.

Common PostgreSQL use cases include:

~~~text
JSONB
arrays
full-text search
~~~

Think of GIN as:

~~~text
searchable value
      ↓
which rows contain it?
~~~

---

## 17. JSONB + GIN

Suppose:

~~~sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    metadata JSONB NOT NULL
);
~~~

Data:

~~~json
{
  "brand": "Apple",
  "color": "black",
  "storage": "256GB"
}
~~~

Query:

~~~sql
SELECT *
FROM products
WHERE metadata @> '{"brand":"Apple"}';
~~~

A GIN index can support JSONB containment queries:

~~~sql
CREATE INDEX idx_products_metadata_gin
ON products USING GIN (metadata);
~~~

This is a common real PostgreSQL use case.

---

## 18. Arrays + GIN

Suppose:

~~~sql
tags TEXT[]
~~~

Query:

~~~sql
SELECT *
FROM posts
WHERE tags @> ARRAY['postgresql'];
~~~

Candidate:

~~~sql
CREATE INDEX idx_posts_tags_gin
ON posts USING GIN (tags);
~~~

GIN is useful because an array contains multiple searchable elements.

---

## 19. GIN Tradeoff

GIN can make suitable searches fast, but it can be larger and more expensive to maintain during writes than a simple B-tree index.

Again:

~~~text
faster targeted reads
        ↕
index storage + write cost
~~~

---

# Part 7 — GiST Indexes

## 20. What Is GiST?

GiST means **Generalized Search Tree**.

It is a framework used by PostgreSQL for several specialized search strategies.

Depending on data type/operator class, it can support use cases involving:

~~~text
geometric data
range types
nearest-neighbor searches
other specialized structures
~~~

You do not need to memorize GiST internals for normal full-stack interviews.

Remember:

~~~text
B-tree
→ common scalar comparisons

GIN
→ values containing multiple searchable components

GiST
→ specialized search structures
~~~

---

# Part 8 — BRIN Indexes

## 21. What Is BRIN?

BRIN means **Block Range Index**.

BRIN is useful for very large tables where values correlate with physical row order.

Example:

~~~text
event_logs

older rows
older timestamps
   ↓
newer rows
newer timestamps
~~~

If rows are mostly appended in timestamp order, a BRIN index on created_at may be extremely compact.

~~~sql
CREATE INDEX idx_event_logs_created_brin
ON event_logs USING BRIN (created_at);
~~~

---

## 22. B-tree vs BRIN Mental Model

### B-tree

~~~text
more detailed lookup structure
larger than BRIN
good for precise/range lookup
~~~

### BRIN

~~~text
summarizes ranges of table blocks
very small
best when values correlate with physical order
~~~

BRIN is not simply a smaller replacement for B-tree. It solves a different kind of problem.

---

# Part 9 — Hash Index

## 23. Hash Index

PostgreSQL also supports hash indexes.

~~~sql
CREATE INDEX idx_sessions_token_hash
ON sessions USING HASH (token);
~~~

Hash indexes are focused on equality comparisons.

For most ordinary application indexing, B-tree remains the default starting point because it supports a broader set of operations.

Know that Hash exists; do not reach for it automatically just because a query uses equality.

---

# Part 10 — Indexes and JOINs

## 24. Indexing Join Columns

Suppose:

~~~sql
SELECT o.id, o.total, u.name
FROM orders o
JOIN users u
  ON u.id = o.user_id
WHERE o.user_id = $1;
~~~

users.id is a primary key and therefore indexed.

But orders.user_id is a foreign-key column and PostgreSQL does not automatically index it.

A useful index may be:

~~~sql
CREATE INDEX idx_orders_user_id
ON orders (user_id);
~~~

For common parent-child lookups, indexing the referencing foreign-key column is often important.

---

# Part 11 — Indexes and Pagination

## 25. Keyset Pagination

Suppose the application needs recent orders page by page.

Offset pagination:

~~~sql
SELECT id, created_at
FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 100000;
~~~

At deep offsets, PostgreSQL still has to process/skips many earlier rows.

Keyset pagination can use the previous page's final key:

~~~sql
SELECT id, created_at
FROM orders
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

Matching index:

~~~sql
CREATE INDEX idx_orders_cursor
ON orders (created_at DESC, id DESC);
~~~

Conceptually:

~~~text
OFFSET
→ count/skip many rows

KEYSET
→ continue from known indexed position
~~~

This is especially valuable for large feeds and histories.

---

# Part 12 — Duplicate and Overlapping Indexes

## 26. Too Many Similar Indexes

Suppose you have:

~~~text
INDEX (user_id)
INDEX (user_id, created_at)
INDEX (user_id, created_at, status)
~~~

Some indexes may overlap in what they can support.

That does **not** mean the smaller index is always useless, because index size and workload matter. But you should evaluate whether each index provides enough value to justify its cost.

Do not keep indexes merely because they were created in the past.

---

## 27. Index Maintenance Cost

Every INSERT may require:

~~~text
write table row
      +
update index A
      +
update index B
      +
update index C
~~~

Every UPDATE of indexed values can also require index maintenance.

So an over-indexed table may have excellent reads but unnecessarily expensive writes.

---

# Part 13 — Choosing an Index

## 28. Practical Decision Process

When a query is slow:

~~~text
1. Find the exact slow query
        ↓
2. Run EXPLAIN ANALYZE
        ↓
3. Check filtering / joining / sorting
        ↓
4. Check row estimates and selectivity
        ↓
5. Check existing indexes
        ↓
6. Design the smallest useful index
        ↓
7. Measure again
~~~

Do not begin with:

~~~text
Let's create five indexes and see what happens.
~~~

Performance work should be measured.

---

# Common Mistakes

## 29. Wrong Composite Column Order

Design column order from the real query pattern, not alphabetical order or guesswork.

## 30. Indexing Low-Selectivity Data Blindly

A normal full index on a status/boolean column may provide little benefit. A partial index may sometimes fit better.

## 31. Applying a Function but Indexing the Raw Column

If the important query uses LOWER(email), an expression index on LOWER(email) may be appropriate.

## 32. Adding INCLUDE Columns Everywhere

Covering indexes can become large. Add payload columns only when a measured query benefits.

## 33. Assuming Index-Only Scan Is Guaranteed

MVCC visibility and PostgreSQL's planner still determine actual heap access and plan choice.

## 34. Using GIN for Ordinary Numeric Equality

Choose the index type that matches the operator and data structure. B-tree is normally the starting point for ordinary scalar equality/range queries.

## 35. Using BRIN on Randomly Distributed Values

BRIN works best when values correlate with physical row ordering.

## 36. Keeping Duplicate or Unused Indexes

Unused indexes still consume storage and write resources.

## 37. Creating an Index Without Measuring

Always verify performance with query plans and realistic data.

---

# Interview Revision

## What is a composite index?

An index containing multiple key columns, such as (user_id, created_at).

## Why does column order matter?

Because B-tree ordering starts with the leading key columns, so different orders support different access patterns.

## What is a partial index?

An index containing only rows that satisfy a predicate.

## When is a partial index useful?

When an important query repeatedly targets a relatively small subset of a table.

## What is an expression index?

An index built on the result of an expression, such as LOWER(email).

## What is INCLUDE?

It adds non-key payload columns to an index, potentially helping a query be served by an index-only scan.

## Do INCLUDE columns control B-tree search order?

No. They are payload columns rather than search key columns.

## What is a GIN index commonly used for?

JSONB, arrays, full-text search, and other multi-valued/searchable structures depending on operators and operator classes.

## What is GiST?

A generalized index framework used for specialized searches such as ranges, geometric data, and nearest-neighbor operations depending on the data type.

## What is BRIN?

A compact block-range index useful for huge tables when indexed values correlate strongly with physical row order.

## Why can too many indexes be harmful?

They consume storage and increase INSERT, UPDATE, and DELETE maintenance cost.

## What is the best way to know whether an index helps?

Measure the query using EXPLAIN/EXPLAIN ANALYZE before and after the change.

---

# Quick Revision

~~~text
Composite Index
→ multiple search/order columns
→ column order matters

Partial Index
→ indexes only selected rows

Expression Index
→ indexes computed expression

INCLUDE
→ stores extra payload columns

GIN
→ JSONB / arrays / full-text style searches

GiST
→ specialized search structures

BRIN
→ huge physically correlated tables

Hash
→ equality-focused index
~~~

### ShopHub Examples

~~~text
Recent orders for user
→ (user_id, created_at DESC)

Only pending orders
→ partial index WHERE status = 'pending'

Case-insensitive email
→ index LOWER(email)

JSONB product metadata
→ GIN

Huge time-ordered event table
→ consider BRIN
~~~

### Most Important Rule

~~~text
Query pattern
     ↓
Choose index structure
     ↓
EXPLAIN ANALYZE
     ↓
Measure
~~~

---

## Key Takeaway

> **Advanced indexing is not about creating more indexes. It is about designing the smallest appropriate index for a real query pattern. Composite, partial, expression, covering, GIN, GiST, and BRIN indexes solve different problems, so choose based on the workload and verify the result with EXPLAIN ANALYZE.**

---

[← Previous: Lesson 23 — Index Fundamentals](./23-index-fundamentals.md) | [Back to Roadmap](../README.md) | [Next: Lesson 25 — EXPLAIN & EXPLAIN ANALYZE →](./25-explain-and-explain-analyze.md)