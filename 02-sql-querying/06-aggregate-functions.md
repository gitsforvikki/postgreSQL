# Lesson 6 — Aggregate Functions

## Core Idea

Aggregate functions calculate a summary value from multiple rows.

The five most important functions are:

- `COUNT()` — count rows or values
- `SUM()` — calculate a total
- `AVG()` — calculate an average
- `MIN()` — find the smallest value
- `MAX()` — find the largest value

Assume an `orders` table:

| id | user_id | status | amount |
|---:|---:|---|---:|
| 1 | 101 | paid | 1000 |
| 2 | 101 | paid | 2500 |
| 3 | 102 | pending | 1500 |
| 4 | 103 | paid | 3000 |
| 5 | 103 | cancelled | 500 |

---

## 1. COUNT()

Count all rows:

~~~sql
SELECT COUNT(*) AS total_orders
FROM orders;
~~~

### COUNT(*) vs COUNT(column)

~~~sql
SELECT COUNT(*), COUNT(amount)
FROM orders;
~~~

The difference is important:

~~~text
COUNT(*)
→ counts rows

COUNT(column)
→ counts only non-NULL values in that column
~~~

If four rows exist but one has `amount = NULL`:

~~~text
COUNT(*)      → 4
COUNT(amount) → 3
~~~

---

## 2. SUM()

Calculate a total:

~~~sql
SELECT SUM(amount) AS total_revenue
FROM orders
WHERE status = 'paid';
~~~

Typical uses include total revenue, total quantity, total stock, and other numeric totals.

---

## 3. AVG()

Calculate the average of non-NULL input values:

~~~sql
SELECT AVG(amount) AS average_order_value
FROM orders;
~~~

Conceptually:

~~~text
Total of non-NULL values
          ÷
Number of non-NULL values
          ↓
        Average
~~~

---

## 4. MIN() and MAX()

Smallest value:

~~~sql
SELECT MIN(amount) AS smallest_order
FROM orders;
~~~

Largest value:

~~~sql
SELECT MAX(amount) AS largest_order
FROM orders;
~~~

They can also work with dates:

~~~sql
SELECT
    MIN(created_at) AS first_order,
    MAX(created_at) AS latest_order
FROM orders;
~~~

---

## 5. Multiple Aggregates

Several aggregate calculations can be performed together:

~~~sql
SELECT
    COUNT(*) AS total_orders,
    SUM(amount) AS total_amount,
    AVG(amount) AS average_amount,
    MIN(amount) AS minimum_amount,
    MAX(amount) AS maximum_amount
FROM orders;
~~~

Without grouping, this produces a summary for the entire input.

---

# GROUP BY

`GROUP BY` divides rows into groups so an aggregate can be calculated for each group.

For example:

> How many orders does each user have?

~~~sql
SELECT
    user_id,
    COUNT(*) AS total_orders
FROM orders
GROUP BY user_id;
~~~

Conceptually:

~~~text
orders
   ↓
GROUP BY user_id
   ↓
user 101 → its rows
user 102 → its rows
user 103 → its rows
   ↓
COUNT each group
~~~

Possible result:

| user_id | total_orders |
|---:|---:|
| 101 | 2 |
| 102 | 1 |
| 103 | 2 |

---

## GROUP BY with SUM()

Total amount for each user:

~~~sql
SELECT
    user_id,
    SUM(amount) AS total_amount
FROM orders
GROUP BY user_id;
~~~

Conceptually:

~~~text
user 101 → 1000 + 2500 → 3500
user 102 → 1500
user 103 → 3000 + 500 → 3500
~~~

---

## GROUP BY Multiple Columns

~~~sql
SELECT
    user_id,
    status,
    COUNT(*) AS total
FROM orders
GROUP BY user_id, status;
~~~

Groups are created for each unique combination:

~~~text
(user_id, status)
~~~

For example:

~~~text
(101, paid)
(102, pending)
(103, paid)
(103, cancelled)
~~~

---

## Important GROUP BY Rule

This is generally invalid:

~~~sql
SELECT
    user_id,
    status,
    COUNT(*)
FROM orders
GROUP BY user_id;
~~~

Here `status` is neither grouped nor aggregated.

A correct version is:

~~~sql
SELECT
    user_id,
    status,
    COUNT(*)
FROM orders
GROUP BY user_id, status;
~~~

Interview memory:

> When using GROUP BY, selected expressions generally need to be grouped or aggregated.

---

# WHERE with Aggregates

Suppose we only want paid orders before calculating revenue:

~~~sql
SELECT SUM(amount)
FROM orders
WHERE status = 'paid';
~~~

Flow:

~~~text
orders
   ↓
WHERE status = 'paid'
   ↓
paid rows
   ↓
SUM(amount)
~~~

`WHERE` filters individual rows **before grouping and aggregation**.

---

# HAVING

`HAVING` filters groups after aggregation.

Example:

> Show users whose total order amount is greater than 3000.

~~~sql
SELECT
    user_id,
    SUM(amount) AS total_amount
FROM orders
GROUP BY user_id
HAVING SUM(amount) > 3000;
~~~

Flow:

~~~text
orders
   ↓
GROUP BY user_id
   ↓
SUM each group
   ↓
HAVING SUM(amount) > 3000
   ↓
matching groups
~~~

---

# WHERE vs HAVING

This is one of the most important interview distinctions.

~~~text
WHERE
→ filters individual rows
→ happens before grouping

HAVING
→ filters groups
→ happens after grouping
~~~

Example:

~~~sql
SELECT
    user_id,
    SUM(amount) AS total_paid
FROM orders
WHERE status = 'paid'
GROUP BY user_id
HAVING SUM(amount) > 2000;
~~~

Conceptually:

~~~text
orders
   ↓
WHERE status = 'paid'
   ↓
paid rows
   ↓
GROUP BY user_id
   ↓
SUM each group
   ↓
HAVING SUM(amount) > 2000
   ↓
final groups
~~~

This is incorrect:

~~~sql
SELECT user_id, SUM(amount)
FROM orders
WHERE SUM(amount) > 3000
GROUP BY user_id;
~~~

The aggregate result does not yet exist at the `WHERE` stage.

Use `HAVING` instead.

---

# Logical Query Processing Order

A useful conceptual order is:

~~~text
FROM
 ↓
WHERE
 ↓
GROUP BY
 ↓
HAVING
 ↓
SELECT
 ↓
ORDER BY
 ↓
LIMIT
~~~

Example:

~~~sql
SELECT
    user_id,
    SUM(amount) AS total
FROM orders
WHERE status = 'paid'
GROUP BY user_id
HAVING SUM(amount) > 1000
ORDER BY total DESC
LIMIT 5;
~~~

Mental flow:

~~~text
FROM orders
     ↓
filter rows
     ↓
create groups
     ↓
filter groups
     ↓
produce selected output
     ↓
sort
     ↓
limit
~~~

This is a conceptual model. PostgreSQL's optimizer can choose an efficient physical execution plan while preserving the query's meaning.

---

# DISTINCT vs GROUP BY

These can sometimes produce similar-looking results.

### DISTINCT

~~~sql
SELECT DISTINCT status
FROM orders;
~~~

removes duplicate result values.

### GROUP BY

~~~sql
SELECT status
FROM orders
GROUP BY status;
~~~

also produces one row per group here.

But conceptually:

~~~text
DISTINCT
→ remove duplicate result rows

GROUP BY
→ create groups, commonly for aggregate calculations
~~~

Example:

~~~sql
SELECT
    status,
    COUNT(*) AS total
FROM orders
GROUP BY status;
~~~

---

# NULL and Aggregate Functions

Most common aggregate functions ignore NULL input values.

Suppose:

~~~text
amount
------
100
200
NULL
300
~~~

Then:

~~~text
COUNT(*)      → 4
COUNT(amount) → 3
SUM(amount)   → 600
AVG(amount)   → 200
~~~

Understanding this is important when creating reports.

---

# Aggregates with JOINs

Suppose:

~~~text
users
  |
  | 1:N
  v
orders
~~~

To count each user's orders:

~~~sql
SELECT
    u.id,
    u.name,
    COUNT(o.id) AS total_orders
FROM users AS u
LEFT JOIN orders AS o
    ON o.user_id = u.id
GROUP BY u.id, u.name;
~~~

With a `LEFT JOIN`, `COUNT(o.id)` is useful because a user without an order has:

~~~text
o.id → NULL
~~~

Therefore:

~~~text
COUNT(o.id) → 0
~~~

while `COUNT(*)` counts the left-join output row itself.

We explore this more deeply in Lesson 7.

---

# Real E-Commerce Examples

### Total orders

~~~sql
SELECT COUNT(*) AS total_orders
FROM orders;
~~~

### Paid revenue

~~~sql
SELECT SUM(amount) AS revenue
FROM orders
WHERE status = 'paid';
~~~

### Average paid order

~~~sql
SELECT AVG(amount) AS average_order
FROM orders
WHERE status = 'paid';
~~~

### Revenue by user

~~~sql
SELECT
    user_id,
    SUM(amount) AS total_spent
FROM orders
WHERE status = 'paid'
GROUP BY user_id;
~~~

### High-value customers

~~~sql
SELECT
    user_id,
    SUM(amount) AS total_spent
FROM orders
WHERE status = 'paid'
GROUP BY user_id
HAVING SUM(amount) >= 5000
ORDER BY total_spent DESC;
~~~

---

# Common Mistakes

### Confusing COUNT(*) with COUNT(column)

~~~text
COUNT(*)      → rows
COUNT(column) → non-NULL values
~~~

### Selecting an ungrouped column

Do not select unrelated columns alongside aggregates without grouping or otherwise handling them appropriately.

### Using WHERE for aggregate filtering

Wrong idea:

~~~sql
WHERE COUNT(*) > 5
~~~

For grouped aggregate filtering, use:

~~~sql
HAVING COUNT(*) > 5
~~~

### Forgetting NULL behavior

Most common aggregates ignore NULL input values.

### Assuming GROUP BY sorts results

`GROUP BY` creates groups. It does not guarantee output ordering.

Use:

~~~sql
ORDER BY
~~~

when order matters.

---

# Interview Revision

## What is an aggregate function?

An aggregate function calculates a summary result from multiple input rows or from each group of rows.

## Most important aggregate functions?

~~~text
COUNT()
SUM()
AVG()
MIN()
MAX()
~~~

## COUNT(*) vs COUNT(column)?

~~~text
COUNT(*)
→ counts rows

COUNT(column)
→ counts non-NULL values in that column
~~~

## What does GROUP BY do?

It divides rows into groups based on one or more expressions so aggregate calculations can be performed for each group.

## WHERE vs HAVING?

~~~text
WHERE
→ filters rows before grouping

HAVING
→ filters groups after grouping
~~~

## Does GROUP BY guarantee sorting?

No. Use `ORDER BY` when ordering matters.

## How do aggregates handle NULL?

Most common aggregates ignore NULL input values. `COUNT(*)` is different because it counts rows.

---

# Quick Revision

~~~text
COUNT() → count
SUM()   → total
AVG()   → average
MIN()   → smallest
MAX()   → largest

GROUP BY → create groups
WHERE    → filter rows
HAVING   → filter groups
ORDER BY → sort result
~~~

### Query Order

~~~text
FROM
 ↓
WHERE
 ↓
GROUP BY
 ↓
HAVING
 ↓
SELECT
 ↓
ORDER BY
 ↓
LIMIT
~~~

### Important Rule

~~~text
COUNT(*)
→ all rows

COUNT(column)
→ non-NULL values
~~~

---

## Key Takeaway

> **Aggregate functions summarize data. GROUP BY calculates summaries for individual groups, WHERE filters rows before grouping, and HAVING filters the aggregated groups afterward.**

---

[← Previous: Lesson 5 — Filtering and Conditions](./05-filtering-and-conditions.md) | [Back to Roadmap](../README.md) | [Next: Lesson 7 — SQL Joins →](./07-sql-joins.md)
