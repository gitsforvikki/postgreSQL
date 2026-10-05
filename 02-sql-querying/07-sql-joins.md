# Lesson 7 — SQL Joins

## Core Idea

A **JOIN** combines related rows from two or more tables.

Suppose we have:

~~~text
users
+----+--------+
| id | name   |
+----+--------+
| 1  | Vikash |
| 2  | Rahul  |
| 3  | Neha   |
+----+--------+

orders
+-----+---------+--------+
| id  | user_id | amount |
+-----+---------+--------+
| 101 |    1    | 2500   |
| 102 |    1    | 1500   |
| 103 |    2    | 3000   |
+-----+---------+--------+
~~~

The relationship is:

~~~text
users.id
   │
   │ 1 : N
   ▼
orders.user_id
~~~

One user can have many orders.

---

# 1. Why Do We Need JOINs?

Normalized databases usually store different entities in separate tables.

Instead of storing:

~~~text
order
├── user_name
├── user_email
├── product
└── amount
~~~

repeatedly, we keep related data in separate tables and connect them using keys.

~~~text
users
  │
  └── id
       │
       ▼
orders
  └── user_id
~~~

A JOIN lets us retrieve related information together.

---

# 2. INNER JOIN

`INNER JOIN` returns only rows that have matching rows on both sides.

~~~sql
SELECT
    u.id,
    u.name,
    o.id AS order_id,
    o.amount
FROM users AS u
INNER JOIN orders AS o
    ON u.id = o.user_id;
~~~

Result conceptually:

~~~text
Vikash → Order 101
Vikash → Order 102
Rahul  → Order 103
~~~

Neha is not returned because she has no matching order.

Mental model:

~~~text
Table A      Table B

   A ∩ B

Only matching rows
~~~

---

# 3. JOIN is Usually INNER JOIN

Writing:

~~~sql
SELECT *
FROM users AS u
JOIN orders AS o
    ON u.id = o.user_id;
~~~

normally means the same thing as:

~~~sql
SELECT *
FROM users AS u
INNER JOIN orders AS o
    ON u.id = o.user_id;
~~~

Using `INNER JOIN` explicitly can sometimes make learning intent clearer.

---

# 4. LEFT JOIN

`LEFT JOIN` returns:

- Every row from the left table
- Matching rows from the right table
- NULL values for right-side columns when no match exists

~~~sql
SELECT
    u.id,
    u.name,
    o.id AS order_id,
    o.amount
FROM users AS u
LEFT JOIN orders AS o
    ON u.id = o.user_id;
~~~

Conceptually:

~~~text
Vikash → 101 → 2500
Vikash → 102 → 1500
Rahul  → 103 → 3000
Neha   → NULL → NULL
~~~

Mental model:

~~~text
LEFT JOIN

All rows from A
+
matching rows from B
~~~

This is very useful when you need users even if they have no orders.

---

# 5. Finding Rows Without a Match

A common pattern is:

> Find users who have never placed an order.

~~~sql
SELECT
    u.id,
    u.name
FROM users AS u
LEFT JOIN orders AS o
    ON u.id = o.user_id
WHERE o.id IS NULL;
~~~

Flow:

~~~text
all users
   ↓
LEFT JOIN orders
   ↓
users without orders have o.id = NULL
   ↓
WHERE o.id IS NULL
   ↓
users with no orders
~~~

---

# 6. RIGHT JOIN

`RIGHT JOIN` keeps every row from the right table and matching rows from the left table.

~~~sql
SELECT
    u.name,
    o.id,
    o.amount
FROM users AS u
RIGHT JOIN orders AS o
    ON u.id = o.user_id;
~~~

Mental model:

~~~text
matching rows from A
+
all rows from B
~~~

In practice, many developers prefer rewriting a RIGHT JOIN as a LEFT JOIN by reversing the table order because LEFT JOIN is often easier to read consistently.

---

# 7. FULL JOIN / FULL OUTER JOIN

PostgreSQL supports:

~~~sql
FULL JOIN
~~~

and:

~~~sql
FULL OUTER JOIN
~~~

They mean the same thing.

A full join returns:

- Matching rows
- Unmatched rows from the left
- Unmatched rows from the right

~~~sql
SELECT
    u.id AS user_id,
    u.name,
    o.id AS order_id
FROM users AS u
FULL JOIN orders AS o
    ON u.id = o.user_id;
~~~

Mental model:

~~~text
A ∪ B

all rows from both sides
with matches combined
~~~

So:

~~~text
FULL JOIN = FULL OUTER JOIN
~~~

The word `OUTER` is optional.

---

# 8. CROSS JOIN

`CROSS JOIN` creates every possible combination of rows.

Suppose:

~~~text
colors
------
Black
White

sizes
-----
S
M
L
~~~

Query:

~~~sql
SELECT
    c.name AS color,
    s.name AS size
FROM colors AS c
CROSS JOIN sizes AS s;
~~~

Result:

~~~text
Black S
Black M
Black L
White S
White M
White L
~~~

If table A has `m` rows and table B has `n` rows:

~~~text
result rows = m × n
~~~

Use CROSS JOIN intentionally because result size can grow quickly.

---

# 9. JOIN Summary

~~~text
INNER JOIN
→ only matches

LEFT JOIN
→ all left + matching right

RIGHT JOIN
→ matching left + all right

FULL JOIN
→ all rows from both sides

CROSS JOIN
→ every possible combination
~~~

Visual memory:

~~~text
INNER → A ∩ B
LEFT  → all A + matches from B
RIGHT → all B + matches from A
FULL  → A ∪ B
CROSS → A × B
~~~

---

# 10. What Does ON Do?

The `ON` clause defines how rows from the tables relate.

~~~sql
ON u.id = o.user_id
~~~

Conceptually:

~~~text
users.id
   =
orders.user_id
~~~

PostgreSQL uses this condition to determine which rows match.

---

# 11. ON vs WHERE

This distinction becomes especially important with outer joins.

### ON

Defines which rows match during the JOIN.

### WHERE

Filters the result after the join logically occurs.

Example:

~~~sql
SELECT *
FROM users AS u
LEFT JOIN orders AS o
    ON u.id = o.user_id
WHERE u.is_active = true;
~~~

The `ON` condition defines the relationship.

The `WHERE` condition filters the resulting users.

---

# 12. Important LEFT JOIN Trap

Suppose we want:

> Return every user, but only join their paid orders.

This may look reasonable:

~~~sql
SELECT
    u.id,
    u.name,
    o.id AS order_id
FROM users AS u
LEFT JOIN orders AS o
    ON u.id = o.user_id
WHERE o.status = 'paid';
~~~

But users with no matching order have:

~~~text
o.status = NULL
~~~

and:

~~~text
NULL = 'paid'
→ UNKNOWN
~~~

So the `WHERE` clause removes them.

The query can therefore behave like an INNER JOIN for this condition.

---

# 13. Preserve LEFT Rows by Filtering in ON

If the requirement is:

> Keep every user and join only paid orders.

Use:

~~~sql
SELECT
    u.id,
    u.name,
    o.id AS order_id
FROM users AS u
LEFT JOIN orders AS o
    ON u.id = o.user_id
   AND o.status = 'paid';
~~~

Now:

~~~text
LEFT JOIN
   ↓
keep all users
   ↓
only paid orders qualify as right-side matches
~~~

This distinction is extremely important in real SQL.

---

# 14. COUNT with LEFT JOIN

Suppose we want every user and their number of orders.

~~~sql
SELECT
    u.id,
    u.name,
    COUNT(o.id) AS total_orders
FROM users AS u
LEFT JOIN orders AS o
    ON u.id = o.user_id
GROUP BY u.id, u.name;
~~~

For a user with no orders:

~~~text
u.id   → 3
u.name → Neha
o.id   → NULL
~~~

Therefore:

~~~text
COUNT(o.id)
→ 0
~~~

This is usually what we want.

---

# 15. COUNT(*) vs COUNT(o.id) in LEFT JOIN

With a LEFT JOIN:

~~~sql
COUNT(*)
~~~

counts result rows.

A left-side row without a match still produces one output row.

So a user with no orders can appear to have:

~~~text
COUNT(*) = 1
~~~

But:

~~~sql
COUNT(o.id)
~~~

counts only non-NULL order IDs.

Therefore:

~~~text
COUNT(o.id) = 0
~~~

for a user without orders.

Interview memory:

~~~text
LEFT JOIN + counting matches
→ usually count a non-NULL right-side key
→ COUNT(o.id)
~~~

---

# 16. Multiple JOINs

Real applications often join more than two tables.

Suppose:

~~~text
users
  |
  | 1:N
  v
orders
  |
  | 1:N
  v
order_items
  |
  | N:1
  v
products
~~~

Query:

~~~sql
SELECT
    u.name AS customer,
    o.id AS order_id,
    p.name AS product,
    oi.quantity
FROM users AS u
JOIN orders AS o
    ON o.user_id = u.id
JOIN order_items AS oi
    ON oi.order_id = o.id
JOIN products AS p
    ON p.id = oi.product_id;
~~~

Flow:

~~~text
users
  ↓
orders
  ↓
order_items
  ↓
products
~~~

This is a normal pattern in relational applications.

---

# 17. Table Aliases

Without aliases:

~~~sql
SELECT users.name, orders.amount
FROM users
JOIN orders
    ON users.id = orders.user_id;
~~~

With aliases:

~~~sql
SELECT
    u.name,
    o.amount
FROM users AS u
JOIN orders AS o
    ON u.id = o.user_id;
~~~

Aliases improve readability, especially when several tables are involved.

Common convention:

~~~text
users       → u
orders      → o
order_items → oi
products    → p
~~~

Use aliases that remain understandable.

---

# 18. Ambiguous Column Names

If both tables contain:

~~~text
id
~~~

this may be ambiguous:

~~~sql
SELECT id
FROM users
JOIN orders
    ON users.id = orders.user_id;
~~~

Prefer:

~~~sql
SELECT
    users.id,
    orders.id
FROM users
JOIN orders
    ON users.id = orders.user_id;
~~~

or aliases:

~~~sql
SELECT
    u.id AS user_id,
    o.id AS order_id
FROM users AS u
JOIN orders AS o
    ON u.id = o.user_id;
~~~

---

# 19. JOINs and Foreign Keys

A foreign key often defines the relationship used in a join.

Example:

~~~sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    amount NUMERIC(12, 2)
);
~~~

Relationship:

~~~text
users.id
   ▲
   │ foreign key
   │
orders.user_id
~~~

Then:

~~~sql
JOIN orders AS o
    ON u.id = o.user_id
~~~

The foreign key protects referential integrity, while the JOIN retrieves related data.

Important:

> A JOIN does not require a foreign-key constraint syntactically, but foreign keys are valuable for enforcing valid relationships.

---

# 20. Real ShopHub Example

Suppose ShopHub needs an order-history page.

Tables:

~~~text
users
orders
order_items
products
~~~

Query:

~~~sql
SELECT
    o.id AS order_id,
    o.created_at,
    p.name AS product_name,
    oi.quantity,
    oi.unit_price
FROM orders AS o
JOIN order_items AS oi
    ON oi.order_id = o.id
JOIN products AS p
    ON p.id = oi.product_id
WHERE o.user_id = 10
ORDER BY o.created_at DESC;
~~~

Flow:

~~~text
User 10
   ↓
their orders
   ↓
order items
   ↓
product details
   ↓
order history
~~~

---

# 21. Common Mistakes

## Missing JOIN condition

Accidentally joining without a proper condition can create a huge number of row combinations.

Always verify:

~~~sql
ON table_a.key = table_b.foreign_key
~~~

when that is the intended relationship.

---

## Using the wrong columns in ON

Wrong relationships can produce incorrect results even if the SQL executes successfully.

Understand the data model first.

---

## Filtering the right table in WHERE after LEFT JOIN

Potential problem:

~~~sql
LEFT JOIN orders AS o
    ON ...
WHERE o.status = 'paid';
~~~

This removes NULL right-side rows.

If you need to preserve all left rows, the right-side match condition often belongs in:

~~~sql
ON ... AND o.status = 'paid'
~~~

---

## Using COUNT(*) to count right-side matches

With LEFT JOIN, this can produce misleading counts.

Prefer:

~~~sql
COUNT(o.id)
~~~

when counting matched orders.

---

## Assuming JOIN means INNER JOIN in every explanation

In SQL syntax, bare `JOIN` normally means INNER JOIN, but when discussing design always specify which join semantics you actually need.

---

# Interview Revision

## What is a JOIN?

A JOIN combines related rows from two or more tables based on a join condition.

## INNER JOIN?

Returns rows where matching rows exist on both sides.

## LEFT JOIN?

Returns every row from the left table plus matching rows from the right table. Unmatched right-side columns become NULL.

## RIGHT JOIN?

Returns every row from the right table plus matching rows from the left.

## FULL JOIN?

Returns matching rows plus unmatched rows from both sides.

`FULL JOIN` and `FULL OUTER JOIN` are equivalent.

## CROSS JOIN?

Returns every possible combination of rows from the joined tables.

## ON vs WHERE?

~~~text
ON
→ defines join matching conditions

WHERE
→ filters the resulting rows
~~~

This distinction is especially important with outer joins.

## How do you find users with no orders?

~~~sql
SELECT u.*
FROM users AS u
LEFT JOIN orders AS o
    ON o.user_id = u.id
WHERE o.id IS NULL;
~~~

## Why can WHERE turn a LEFT JOIN into inner-like behavior?

If `WHERE` requires a right-side column to satisfy a condition, unmatched rows contain NULL on that side and are filtered out.

## COUNT(*) vs COUNT(o.id) after LEFT JOIN?

~~~text
COUNT(*)
→ counts output rows

COUNT(o.id)
→ counts actual non-NULL matched order IDs
~~~

---

# Quick Revision

~~~text
INNER JOIN
→ matches only

LEFT JOIN
→ all left + matches

RIGHT JOIN
→ all right + matches

FULL JOIN
→ all rows from both sides

CROSS JOIN
→ every combination
~~~

### Visual Memory

~~~text
INNER → A ∩ B
LEFT  → all A + matching B
RIGHT → all B + matching A
FULL  → A ∪ B
CROSS → A × B
~~~

### Important LEFT JOIN Rule

~~~text
Need every left row?
        ↓
Be careful filtering right-side columns in WHERE
        ↓
Consider whether the condition belongs in ON
~~~

### Counting Matches

~~~text
LEFT JOIN orders

COUNT(*)
→ result rows

COUNT(orders.id)
→ matched orders
~~~

---

## Key Takeaway

> **JOINs combine related tables. Choose the join type based on whether unmatched rows must be preserved, and pay special attention to ON vs WHERE when using outer joins.**

---

[← Previous: Lesson 6 — Aggregate Functions](./06-aggregate-functions.md) | [Back to Roadmap](../README.md) | [Next: Lesson 8 — Subqueries →](./08-subqueries.md)
