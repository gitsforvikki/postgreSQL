# Lesson 23 — Index Fundamentals

## First Understand the Problem

Imagine your ShopHub products table contains millions of rows.

If you run:

~~~sql
SELECT * FROM products WHERE id = 500000;
~~~

without a useful index, PostgreSQL may need to inspect a large portion of the table.

~~~text
Without useful index
→ scan table rows

With useful index
→ use index to locate candidate rows
→ fetch required data
~~~

Think of a book: without an index you search page by page; with an index you find the topic location and jump near the correct page.

---

## 1. What Is an Index?

An **index** is a separate database structure that helps PostgreSQL locate rows efficiently for suitable queries.

~~~sql
CREATE INDEX idx_products_name
ON products (name);
~~~

A query such as:

~~~sql
SELECT id, name, price
FROM products
WHERE name = 'MacBook Pro';
~~~

may use that index.

> **Important:** Creating an index does not force PostgreSQL to use it. The query planner chooses the plan it estimates to be cheapest.

---

## 2. Table vs Index

~~~text
TABLE
→ stores the actual row data

INDEX
→ stores organized lookup information
→ helps PostgreSQL locate rows
~~~

Indexes are separate structures, so they require additional storage and maintenance.

---

## 3. Why Not Index Every Column?

Indexes can improve reads, but they are not free. They require storage and extra work during INSERT, UPDATE, and DELETE.

~~~text
More indexes
→ potentially faster suitable reads
→ more disk usage
→ more write maintenance
~~~

Good indexing means choosing indexes that support **real query patterns**.

---

## 4. B-tree — PostgreSQL's Default Index

If you write:

~~~sql
CREATE INDEX idx_products_price
ON products (price);
~~~

PostgreSQL normally creates a **B-tree** index.

B-tree is the most common general-purpose index and is useful for operations such as:

~~~text
=
<
<=
>
>=
BETWEEN
ORDER BY
~~~

depending on the query and planner cost.

---

## 5. Equality Lookup

~~~sql
CREATE INDEX idx_users_email
ON users (email);
~~~

~~~sql
SELECT id, name, email
FROM users
WHERE email = 'vikash@example.com';
~~~

An email lookup is often highly selective: millions of users may exist, but only one row matches a particular unique email.

---

## 6. Range Queries

B-tree indexes are also useful for ranges.

~~~sql
CREATE INDEX idx_products_price
ON products (price);
~~~

~~~sql
SELECT id, name, price
FROM products
WHERE price BETWEEN 50000 AND 80000;
~~~

PostgreSQL can navigate to the relevant portion of the index rather than necessarily scanning the entire table.

---

## 7. Indexes and ORDER BY

~~~sql
CREATE INDEX idx_orders_created_at
ON orders (created_at);
~~~

~~~sql
SELECT id, user_id, created_at
FROM orders
ORDER BY created_at DESC
LIMIT 20;
~~~

A suitable B-tree index may let PostgreSQL retrieve rows in the needed order without a separate expensive sort.

Common examples are latest orders, newest posts, recent transactions, and latest messages.

---

# How PostgreSQL Uses an Index

## 8. Sequential Scan

PostgreSQL may choose a **Seq Scan** when no useful index exists or when scanning the table is estimated to be cheaper.

~~~text
start table
   ↓
read rows/pages
   ↓
check condition
   ↓
return matches
~~~

A sequential scan is **not automatically bad**. For small tables or queries returning much of the table, it can be the best plan.

---

## 9. Index Scan

~~~text
query condition
      ↓
index
      ↓
matching index entries
      ↓
required table rows
      ↓
result
~~~

An index scan is useful when the index can narrow the amount of work enough to justify using it.

---

## 10. Why PostgreSQL May Ignore Your Index

PostgreSQL chooses plans based on estimated cost. It may ignore an index because:

- the table is small
- the query returns a large percentage of rows
- statistics favor another plan
- the query does not match the index effectively
- the index columns are in an unsuitable order
- the query applies an expression that the ordinary index does not support directly

For example, a normal index on email is different from an expression index designed for LOWER(email). Expression indexes come in Lesson 24.

---

# Selectivity

## 11. What Is Selectivity?

Selectivity describes how strongly a condition narrows the result.

Suppose there are 1,000,000 users.

~~~text
WHERE email = one specific email
→ perhaps 1 row
→ high selectivity

WHERE is_active = true
→ perhaps 950,000 rows
→ low selectivity
~~~

Easy memory:

~~~text
High selectivity
→ few matching rows
→ index often more useful

Low selectivity
→ many matching rows
→ sequential scan may be cheaper
~~~

---

## 12. Low-Cardinality Columns

Columns such as booleans or statuses can have only a small number of values.

~~~sql
CREATE INDEX idx_users_is_active
ON users (is_active);
~~~

If almost every user is active, this query:

~~~sql
SELECT * FROM users WHERE is_active = true;
~~~

may return most of the table, so the index may provide little benefit.

> Do not create an index merely because a column appears in a WHERE clause. Consider data distribution and the real workload.

---

# Constraints and Indexes

## 13. Primary Key Automatically Gets an Index

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email TEXT
);
~~~

PostgreSQL creates the unique index needed to enforce the primary key. You normally should not create another equivalent index on id.

---

## 14. UNIQUE Also Creates an Index

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email TEXT UNIQUE
);
~~~

PostgreSQL creates a unique index to enforce the UNIQUE constraint.

Always inspect existing indexes before adding another one.

---

## 15. Foreign Keys Are Different

~~~sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id)
);
~~~

PostgreSQL does **not** automatically create an index on the referencing foreign-key column orders.user_id.

But this is a common query:

~~~sql
SELECT *
FROM orders
WHERE user_id = 10;
~~~

So this may be useful:

~~~sql
CREATE INDEX idx_orders_user_id
ON orders (user_id);
~~~

This is an important PostgreSQL interview point.

---

# Composite Indexes

## 16. What Is a Composite Index?

A composite index contains multiple columns.

~~~sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at);
~~~

This can be useful for:

~~~sql
SELECT id, total, created_at
FROM orders
WHERE user_id = 10
ORDER BY created_at DESC;
~~~

The index begins with user_id and then orders entries by created_at within that leading structure.

---

## 17. Column Order Matters

An index on:

~~~text
(user_id, created_at)
~~~

is not equivalent to:

~~~text
(created_at, user_id)
~~~

Conceptually:

~~~text
user 1
  date A
  date B

user 2
  date A
  date C
~~~

The first index naturally supports queries that start by narrowing on user_id.

---

## 18. Leftmost Principle

For a B-tree composite index:

~~~text
(A, B)
~~~

natural query patterns include:

~~~text
A
A + B
~~~

A query using only B generally cannot use the index as effectively as one whose leading column is B.

Easy memory:

~~~text
Index (A, B)

A      → natural match
A + B  → natural match
B only → usually much less useful
~~~

Advanced composite-index behavior is covered in Lesson 24.

---

# ShopHub Example

## 19. Recent Orders for a User

Common query:

~~~sql
SELECT id, status, total, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
~~~

A strong candidate is:

~~~sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
~~~

Why?

~~~text
WHERE user_id = ?
→ first index column

ORDER BY created_at DESC
→ next index column
~~~

This is indexing based on the **complete query pattern**.

---

## 20. Think in Query Patterns, Not Individual Columns

Beginner thinking:

~~~text
query uses user_id → index user_id
query uses created_at → index created_at
~~~

Better thinking:

~~~text
What exact important query does the application run?

WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20
        ↓
design an index around that access pattern
~~~

Indexes support workloads, not merely columns.

---

# Index-Only Scan

## 21. What Is an Index-Only Scan?

Sometimes PostgreSQL can obtain the required values from an index without ordinary heap fetches for every returned tuple.

~~~text
Index Scan
index → locate row → heap/table → data

Possible Index-Only Scan
index → required values available → result
~~~

PostgreSQL MVCC visibility rules still matter, so simply having all needed columns in an index does not guarantee an index-only scan.

Covering indexes with INCLUDE are covered in Lesson 24.

---

# Statistics and ANALYZE

## 22. How Does PostgreSQL Decide?

The query planner uses table statistics to estimate things such as:

- row counts
- value distribution
- distinct values
- NULL frequency
- common values

~~~sql
ANALYZE products;
~~~

ANALYZE updates planner statistics. PostgreSQL normally maintains statistics automatically through its maintenance processes, but understanding ANALYZE is important.

---

## 23. Bad Estimates Can Produce Bad Plans

Suppose PostgreSQL estimates:

~~~text
10 matching rows
~~~

but reality is:

~~~text
500,000 matching rows
~~~

The planner may choose an inefficient plan because its estimate was wrong.

In Lesson 25 you will learn to compare:

~~~text
estimated rows
vs
actual rows
~~~

using EXPLAIN ANALYZE.

---

# Main PostgreSQL Index Types

## 24. Index Types Overview

### B-tree

General-purpose index for equality, ranges, and ordering.

### Hash

Primarily useful for equality comparisons.

### GIN

Commonly useful for searchable multi-valued structures such as JSONB, arrays, and full-text search, depending on operators/operator classes.

### GiST

Useful for specialized search structures such as geometric, range, and nearest-neighbor use cases depending on the data type/operator class.

### BRIN

Useful for very large tables when indexed values correlate strongly with physical row order, such as large append-heavy timestamped tables.

Lesson 24 covers advanced indexing in more detail.

---

# Measuring Before Indexing

## 25. Do Not Guess

Bad workflow:

~~~text
API slow
   ↓
create random indexes
   ↓
hope
~~~

Better workflow:

~~~text
API slow
   ↓
identify slow SQL
   ↓
EXPLAIN / EXPLAIN ANALYZE
   ↓
understand plan
   ↓
targeted index/query change
   ↓
measure again
~~~

Indexes should be evidence-driven.

---

## 26. Check Existing Indexes

In psql you can inspect a table with:

~~~text
\d users
\d orders
~~~

Before adding an index, verify that an equivalent or overlapping index does not already exist.

---

# Common Mistakes

## 27. Indexing Every Column

More indexes do not automatically mean better performance. Every index has maintenance cost.

## 28. Creating Duplicate Indexes

PRIMARY KEY and UNIQUE may already provide the required index.

## 29. Assuming Foreign Keys Are Automatically Indexed

PostgreSQL does not automatically index the referencing foreign-key column.

## 30. Ignoring Composite Index Order

(user_id, created_at) and (created_at, user_id) support different access patterns.

## 31. Thinking Sequential Scan Is Always Bad

For small tables or queries returning much of the table, a sequential scan can be optimal.

## 32. Assuming PostgreSQL Must Use Your Index

The planner chooses based on estimated cost.

## 33. Ignoring Write Cost

Indexes can make INSERT, UPDATE, and DELETE more expensive.

## 34. Indexing Without Measuring the Query

Design indexes around WHERE, JOIN, ORDER BY, ranges, selectivity, and real application workload.

---

# Interview Revision

## What is an index?

A separate database structure that helps PostgreSQL locate rows efficiently for suitable queries.

## What is PostgreSQL's default index type?

**B-tree.**

## What is B-tree commonly useful for?

Equality, ranges, and ordering.

## Does PostgreSQL always use an available index?

No. The planner chooses the plan it estimates to be cheapest.

## Is a sequential scan always bad?

No. It can be optimal for small tables or queries returning a large percentage of rows.

## What is selectivity?

How strongly a condition narrows the result set.

## Does PRIMARY KEY create an index?

Yes. PostgreSQL creates the unique index needed to enforce it.

## Does UNIQUE create an index?

Yes.

## Does a foreign key automatically create an index on the referencing column?

**No.**

## What is a composite index?

An index containing multiple columns.

## Why does composite index order matter?

Because B-tree ordering starts with the leading column, affecting which query patterns can use it efficiently.

## What is the leftmost principle?

For an index such as (A, B), queries using A or A+B naturally match the index structure better than queries using only B.

## What is an index-only scan?

A plan where PostgreSQL can obtain required values from the index without ordinary heap fetches for every returned tuple, subject to visibility requirements.

## What does ANALYZE do?

It collects/updates statistics used by the PostgreSQL query planner.

---

# Quick Revision

~~~text
INDEX
→ separate lookup structure
→ can improve suitable reads
→ costs storage
→ adds write maintenance
~~~

### Default Index

~~~text
B-tree
→ equality
→ ranges
→ ordering
~~~

### Selectivity

~~~text
email = one value
→ few rows
→ high selectivity
→ index often useful

is_active = true
→ perhaps most rows
→ low selectivity
→ index may be less useful
~~~

### Composite Index

~~~text
Index: (user_id, created_at)

Good pattern:
WHERE user_id = ?
ORDER BY created_at
~~~

### Constraints

~~~text
PRIMARY KEY
→ unique index automatically

UNIQUE
→ unique index automatically

FOREIGN KEY referencing column
→ index NOT automatically created
~~~

### Production Rule

~~~text
Don't ask:
Which columns can I index?

Ask:
Which important queries are slow,
and what access pattern do they use?
~~~

---

## Key Takeaway

> **An index is a separate lookup structure that can make suitable PostgreSQL queries much faster, but every index costs storage and write performance. Design indexes around real query patterns and selectivity, understand composite column order, and use EXPLAIN/EXPLAIN ANALYZE to verify whether the index actually helps.**

---

[← Previous: Lesson 22 — Locks & Concurrency](../05-transactions-concurrency/22-locks-and-concurrency.md) | [Back to Roadmap](../README.md) | [Next: Lesson 24 — Advanced Indexing →](./24-advanced-indexing.md)