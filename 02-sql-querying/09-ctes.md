# Lesson 9 — CTEs (Common Table Expressions)

## Core Idea

A **CTE (Common Table Expression)** is a named temporary result that exists for a single SQL statement.

A CTE starts with the `WITH` keyword.

~~~sql
WITH paid_orders AS (
    SELECT *
    FROM orders
    WHERE status = 'paid'
)
SELECT *
FROM paid_orders;
~~~

Mental model:

~~~text
WITH
 ↓
Create named query result
 ↓
Use that name in the main query
~~~

A CTE is especially useful for making complex SQL easier to read and organize.

---

# 1. Basic CTE Syntax

~~~sql
WITH cte_name AS (
    SELECT ...
)
SELECT *
FROM cte_name;
~~~

Example:

~~~sql
WITH expensive_products AS (
    SELECT id, name, price
    FROM products
    WHERE price > 5000
)
SELECT *
FROM expensive_products;
~~~

Conceptually:

~~~text
products
   ↓
price > 5000
   ↓
expensive_products
   ↓
main SELECT
~~~

---

# 2. CTE is Temporary

A CTE is not a permanent table.

~~~text
CTE
→ exists for one SQL statement
→ not permanently stored as a table
~~~

After the statement finishes, you cannot query the CTE separately.

~~~sql
WITH active_users AS (
    SELECT *
    FROM users
    WHERE is_active = true
)
SELECT *
FROM active_users;
~~~

After this statement finishes:

~~~sql
SELECT *
FROM active_users;
~~~

will not work as though `active_users` were a permanent table.

---

# 3. Why Use a CTE?

CTEs are useful for:

- Breaking complex queries into logical steps
- Improving readability
- Reusing a named intermediate result within a statement
- Recursive queries
- Data-modifying query pipelines

Compare:

~~~sql
SELECT *
FROM (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
) AS totals
WHERE total_spent > 5000;
~~~

with:

~~~sql
WITH totals AS (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
)
SELECT *
FROM totals
WHERE total_spent > 5000;
~~~

The second version can be easier to read.

---

# 4. CTE with Aggregation

Example:

~~~sql
WITH customer_totals AS (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM orders
    WHERE status = 'paid'
    GROUP BY user_id
)
SELECT *
FROM customer_totals
WHERE total_spent >= 5000
ORDER BY total_spent DESC;
~~~

Flow:

~~~text
orders
   ↓
paid orders
   ↓
GROUP BY user
   ↓
SUM amount
   ↓
customer_totals CTE
   ↓
filter high-value customers
~~~

---

# 5. CTE with JOIN

A CTE can be joined like another query result.

~~~sql
WITH customer_totals AS (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM orders
    WHERE status = 'paid'
    GROUP BY user_id
)
SELECT
    u.id,
    u.name,
    ct.total_spent
FROM users AS u
JOIN customer_totals AS ct
    ON ct.user_id = u.id;
~~~

Mental model:

~~~text
orders
   ↓
customer_totals
       │
       │ JOIN
       ▼
     users
       ↓
customer + spending
~~~

---

# 6. Multiple CTEs

One `WITH` clause can define multiple CTEs.

~~~sql
WITH paid_orders AS (
    SELECT *
    FROM orders
    WHERE status = 'paid'
),
customer_totals AS (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM paid_orders
    GROUP BY user_id
)
SELECT *
FROM customer_totals
WHERE total_spent > 5000;
~~~

This creates a readable pipeline:

~~~text
orders
  ↓
paid_orders
  ↓
customer_totals
  ↓
final query
~~~

This is one of the strongest reasons to use CTEs in complex SQL.

---

# 7. CTE vs Subquery

A subquery:

~~~sql
SELECT *
FROM (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
) AS totals
WHERE total_spent > 5000;
~~~

A CTE:

~~~sql
WITH totals AS (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
)
SELECT *
FROM totals
WHERE total_spent > 5000;
~~~

Conceptually:

~~~text
Subquery
→ nested directly inside another query

CTE
→ gives the intermediate query a name
~~~

Neither is automatically faster.

Use the form that makes the query correct and understandable, then inspect the execution plan when performance matters.

---

# 8. CTE vs Temporary Table

These are different.

### CTE

~~~text
Lifetime
→ one SQL statement

Stored as permanent DB object?
→ no
~~~

### Temporary Table

~~~text
Lifetime
→ usually session or transaction depending configuration/use

Can be queried by multiple statements?
→ yes
~~~

Example temporary table:

~~~sql
CREATE TEMP TABLE active_users AS
SELECT *
FROM users
WHERE is_active = true;
~~~

Then multiple statements in the session can use:

~~~sql
SELECT *
FROM active_users;
~~~

A CTE is therefore not the same as a temporary table.

---

# 9. Are CTEs Always Materialized?

No.

Older PostgreSQL behavior caused many developers to think:

~~~text
CTE
→ always materialized
~~~

That is not a safe modern rule.

PostgreSQL can inline eligible non-recursive, side-effect-free CTEs into the parent query.

Therefore:

> A CTE is primarily a query-structuring tool, not automatically a performance optimization.

---

# 10. MATERIALIZED

PostgreSQL allows explicit control in appropriate cases.

~~~sql
WITH totals AS MATERIALIZED (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
)
SELECT *
FROM totals;
~~~

`MATERIALIZED` tells PostgreSQL to evaluate the CTE as a separate result rather than folding it into the parent query.

This can be useful in some workloads, but it should not be added blindly.

Measure performance.

---

# 11. NOT MATERIALIZED

You can also write:

~~~sql
WITH totals AS NOT MATERIALIZED (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
)
SELECT *
FROM totals;
~~~

This can allow the CTE to be folded into the parent query when PostgreSQL can do so.

The important interview-level point is:

~~~text
MATERIALIZED
→ request separate evaluation

NOT MATERIALIZED
→ allow/prefer folding into parent query when valid
~~~

Do not treat either option as universally faster.

---

# 12. Recursive CTE

One of the most powerful CTE features is recursion.

Recursive CTEs are useful for hierarchical data such as:

- Categories
- Employee-manager structures
- Folder trees
- Organization charts
- Comment trees

Syntax:

~~~sql
WITH RECURSIVE cte_name AS (
    -- anchor query

    UNION ALL

    -- recursive query
)
SELECT *
FROM cte_name;
~~~

---

# 13. Recursive CTE Mental Model

A recursive CTE normally has two important parts:

~~~text
1. Anchor query
   ↓
starting rows

2. Recursive query
   ↓
find next level using previous results
~~~

The recursion continues until no new rows are produced.

~~~text
Anchor
  ↓
Level 1
  ↓
Level 2
  ↓
Level 3
  ↓
...
  ↓
No more rows
  ↓
Stop
~~~

---

# 14. Category Hierarchy Example

Suppose:

~~~text
categories
+----+-------------+-----------+
| id | name        | parent_id |
+----+-------------+-----------+
| 1  | Electronics | NULL      |
| 2  | Computers   | 1         |
| 3  | Laptops     | 2         |
| 4  | Gaming      | 3         |
+----+-------------+-----------+
~~~

Hierarchy:

~~~text
Electronics
   ↓
Computers
   ↓
Laptops
   ↓
Gaming
~~~

Recursive query:

~~~sql
WITH RECURSIVE category_tree AS (
    SELECT
        id,
        name,
        parent_id,
        1 AS level
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT
        c.id,
        c.name,
        c.parent_id,
        ct.level + 1
    FROM categories AS c
    JOIN category_tree AS ct
        ON c.parent_id = ct.id
)
SELECT *
FROM category_tree;
~~~

---

# 15. Anchor Query

The anchor query provides the starting point.

~~~sql
SELECT
    id,
    name,
    parent_id,
    1 AS level
FROM categories
WHERE parent_id IS NULL;
~~~

This finds top-level categories.

Example:

~~~text
Electronics
~~~

---

# 16. Recursive Term

The recursive part finds rows related to the previous level.

~~~sql
SELECT
    c.id,
    c.name,
    c.parent_id,
    ct.level + 1
FROM categories AS c
JOIN category_tree AS ct
    ON c.parent_id = ct.id;
~~~

Flow:

~~~text
Electronics
    ↓
Computers
    ↓
Laptops
    ↓
Gaming
~~~

---

# 17. Recursive Termination

A recursive query must eventually stop producing new rows.

Otherwise, bad recursive logic can create excessive or non-terminating work.

For hierarchy queries, recursion naturally stops when there are no more children.

Mental model:

~~~text
Find children
   ↓
Any children?
   ├── yes → continue
   └── no  → stop
~~~

For graphs or data that can contain cycles, cycle handling becomes important.

---

# 18. Data-Modifying CTE

CTEs can also work with data-changing statements using `RETURNING`.

Example:

~~~sql
WITH deleted_orders AS (
    DELETE FROM orders
    WHERE status = 'cancelled'
    RETURNING id, user_id, amount
)
SELECT *
FROM deleted_orders;
~~~

Flow:

~~~text
DELETE cancelled orders
        ↓
RETURNING deleted rows
        ↓
deleted_orders CTE
        ↓
SELECT returned data
~~~

This is a PostgreSQL feature that can be useful for advanced data workflows.

---

# 19. CTE Does Not Mean Transaction

A common misconception is:

~~~text
CTE = transaction
~~~

That is incorrect.

A CTE organizes parts of a SQL statement.

A transaction groups database operations into an atomic unit.

~~~text
CTE
→ query structure

Transaction
→ atomicity / consistency boundary
~~~

Transactions are covered in Lesson 20.

---

# 20. Real ShopHub Example

Suppose we need the top customers by paid spending.

~~~sql
WITH paid_orders AS (
    SELECT
        user_id,
        amount
    FROM orders
    WHERE status = 'paid'
),
customer_totals AS (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM paid_orders
    GROUP BY user_id
)
SELECT
    u.id,
    u.name,
    ct.total_spent
FROM customer_totals AS ct
JOIN users AS u
    ON u.id = ct.user_id
WHERE ct.total_spent >= 10000
ORDER BY ct.total_spent DESC;
~~~

Pipeline:

~~~text
orders
   ↓
paid_orders
   ↓
customer_totals
   ↓
join users
   ↓
high-value customers
~~~

---

# 21. Common Mistakes

## Thinking a CTE is Permanent

A CTE exists only for the statement in which it is defined.

---

## Thinking CTE Means Temporary Table

A temporary table can survive across multiple statements in a session.

A CTE cannot.

---

## Assuming CTE is Automatically Faster

Do not choose a CTE simply because you believe it improves performance.

Use it primarily for clarity and query structure.

Then measure performance using:

~~~sql
EXPLAIN ANALYZE
~~~

when necessary.

---

## Assuming Every CTE is Materialized

Modern PostgreSQL can inline eligible CTEs.

Remember:

~~~text
CTE
≠ automatically materialized
~~~

---

## Creating Bad Recursive Logic

Recursive queries need a correct recursive relationship and termination behavior.

Always understand:

~~~text
anchor
+
recursive step
+
termination
~~~

---

# Interview Revision

## What is a CTE?

A CTE is a named temporary query result defined with `WITH` and available to the SQL statement that follows it.

## Why use CTEs?

Primarily for:

~~~text
readability
query decomposition
multiple logical steps
recursive queries
advanced data-modifying statements
~~~

## CTE vs Subquery?

~~~text
Subquery
→ nested query

CTE
→ named query expression defined before the main query
~~~

They can often express similar logic.

## CTE vs Temporary Table?

~~~text
CTE
→ one statement

Temporary table
→ can be used by multiple statements during its lifetime
~~~

## Are CTEs always faster?

No.

## Are PostgreSQL CTEs always materialized?

No. PostgreSQL can inline eligible CTEs.

## What is a recursive CTE?

A CTE that repeatedly uses previous results to traverse recursive or hierarchical relationships.

## Main parts of recursive CTE?

~~~text
Anchor query
+
Recursive query
+
Termination when no more rows are generated
~~~

## What are MATERIALIZED and NOT MATERIALIZED?

They provide control over whether eligible CTE processing is kept separate or can be folded into the parent query.

## Is a CTE a transaction?

No.

~~~text
CTE         → query organization
Transaction → atomic unit of database work
~~~

---

# Quick Revision

~~~text
WITH
→ define CTE

CTE
→ named temporary query result
→ available for one statement

Multiple CTEs
→ build readable query pipelines

WITH RECURSIVE
→ hierarchical / recursive queries

MATERIALIZED
→ request separate CTE evaluation

NOT MATERIALIZED
→ allow/prefer query folding when valid

CTE ≠ temporary table
CTE ≠ transaction
CTE ≠ automatically faster
~~~

### Recursive CTE

~~~text
Anchor
  ↓
Recursive Step
  ↓
Next Level
  ↓
Next Level
  ↓
No new rows
  ↓
Stop
~~~

---

## Key Takeaway

> **A CTE gives a query result a temporary name so complex SQL can be broken into understandable steps. It is especially valuable for query pipelines and recursive hierarchical queries, but it should not be assumed to improve performance automatically.**

---

[← Previous: Lesson 8 — Subqueries](./08-subqueries.md) | [Back to Roadmap](../README.md) | [Next: Lesson 10 — Window Functions →](./10-window-functions.md)
