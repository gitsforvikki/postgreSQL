# Lesson 8 — Subqueries

## Core Idea

A **subquery** is a SQL query written inside another SQL query.

It allows the result of one query to be used by another query.

~~~text
Outer Query
    │
    └── Subquery
          ↓
       produces result
          ↓
Outer query uses that result
~~~

Example:

~~~sql
SELECT *
FROM products
WHERE price > (
    SELECT AVG(price)
    FROM products
);
~~~

The inner query calculates the average price, and the outer query returns products priced above that average.

---

# 1. Basic Subquery

Suppose we have:

~~~text
products
+----+----------+-------+
| id | name     | price |
+----+----------+-------+
| 1  | Mouse    | 1000  |
| 2  | Keyboard | 2500  |
| 3  | Monitor  | 12000 |
+----+----------+-------+
~~~

Query:

~~~sql
SELECT *
FROM products
WHERE price > (
    SELECT AVG(price)
    FROM products
);
~~~

Conceptually:

~~~text
SELECT AVG(price)
        ↓
average price
        ↓
outer query
        ↓
products above average
~~~

---

# 2. Scalar Subquery

A **scalar subquery** returns a single value.

Example:

~~~sql
SELECT *
FROM products
WHERE price > (
    SELECT AVG(price)
    FROM products
);
~~~

The subquery:

~~~sql
SELECT AVG(price)
FROM products;
~~~

returns one value.

That value can therefore be compared using:

~~~text
>
<
=
>=
<=
~~~

---

# 3. Error When a Scalar Comparison Gets Multiple Rows

This can be problematic:

~~~sql
SELECT *
FROM users
WHERE id = (
    SELECT user_id
    FROM orders
);
~~~

If the subquery returns several `user_id` values, `=` expects one value but receives multiple rows.

For multiple values, operators such as `IN` or constructs such as `EXISTS` may be appropriate depending on the requirement.

---

# 4. Multi-Row Subquery with IN

Suppose we want:

> Return users who have placed an order.

~~~sql
SELECT *
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
);
~~~

Flow:

~~~text
orders
   ↓
SELECT user_id
   ↓
[1, 2, 5, ...]
   ↓
users.id IN (...)
   ↓
users who placed orders
~~~

---

# 5. NOT IN

To find users whose IDs are not returned by another query:

~~~sql
SELECT *
FROM users
WHERE id NOT IN (
    SELECT user_id
    FROM orders
);
~~~

However, `NOT IN` requires special care when the subquery can return NULL.

If the list contains NULL, SQL's three-valued logic can make the result surprising.

For anti-match queries, `NOT EXISTS` is often safer when NULLs are possible.

---

# 6. EXISTS

`EXISTS` checks whether a subquery returns at least one row.

Example:

~~~sql
SELECT *
FROM users AS u
WHERE EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.user_id = u.id
);
~~~

Meaning:

> Return a user if at least one matching order exists.

Mental model:

~~~text
For each user
    ↓
Does a matching order exist?
    ↓
YES → return user
NO  → do not return user
~~~

---

# 7. Why SELECT 1 with EXISTS?

You will often see:

~~~sql
SELECT 1
FROM orders
WHERE ...
~~~

inside `EXISTS`.

The actual selected value is not important.

`EXISTS` only cares whether at least one matching row exists.

So:

~~~sql
EXISTS (
    SELECT 1
    FROM orders
    WHERE ...
)
~~~

communicates the intention clearly.

---

# 8. NOT EXISTS

To find users with no orders:

~~~sql
SELECT *
FROM users AS u
WHERE NOT EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.user_id = u.id
);
~~~

Conceptually:

~~~text
For each user
    ↓
Does matching order exist?
    ↓
NO
    ↓
return user
~~~

This is a common **anti-join** pattern.

---

# 9. IN vs EXISTS

Both can sometimes express similar requirements.

### IN

~~~sql
SELECT *
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
);
~~~

Think:

~~~text
Is this value inside the returned set?
~~~

### EXISTS

~~~sql
SELECT *
FROM users AS u
WHERE EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.user_id = u.id
);
~~~

Think:

~~~text
Does at least one matching row exist?
~~~

Do not memorize that one is always faster.

PostgreSQL's optimizer can transform queries and choose execution strategies based on the actual query, schema, indexes, and statistics.

Choose the form that correctly and clearly expresses the requirement, then measure performance when necessary.

---

# 10. Correlated Subquery

A **correlated subquery** refers to a value from the outer query.

Example:

~~~sql
SELECT
    u.id,
    u.name
FROM users AS u
WHERE EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.user_id = u.id
      AND o.amount > 5000
);
~~~

The inner query refers to:

~~~sql
u.id
~~~

from the outer query.

Conceptually:

~~~text
Outer user
    ↓
Inner query uses that user's id
    ↓
check matching orders
~~~

That relationship makes the subquery correlated.

---

# 11. Subquery in WHERE

This is the most common location.

Example:

~~~sql
SELECT *
FROM products
WHERE price > (
    SELECT AVG(price)
    FROM products
);
~~~

Another example:

~~~sql
SELECT *
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
    WHERE status = 'paid'
);
~~~

---

# 12. Subquery in FROM

A subquery can act as a derived table.

Example:

~~~sql
SELECT
    customer_totals.user_id,
    customer_totals.total_spent
FROM (
    SELECT
        user_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
) AS customer_totals
WHERE customer_totals.total_spent > 5000;
~~~

The inner query creates a temporary result:

~~~text
user_id | total_spent
~~~

The outer query then filters that result.

---

# 13. Alias for a FROM Subquery

When a subquery is used in `FROM`, give the derived result a clear alias.

Example:

~~~sql
FROM (
    SELECT ...
) AS customer_totals
~~~

Then its columns can be referenced as:

~~~sql
customer_totals.user_id
~~~

Clear aliases make complex queries much easier to understand.

---

# 14. Subquery in SELECT

A scalar subquery can also appear in the select list.

Example:

~~~sql
SELECT
    u.id,
    u.name,
    (
        SELECT COUNT(*)
        FROM orders AS o
        WHERE o.user_id = u.id
    ) AS total_orders
FROM users AS u;
~~~

Conceptually:

~~~text
Each user
   ↓
run logically related count
   ↓
total_orders
~~~

This can be useful, but always consider readability and performance when using correlated scalar subqueries over large datasets.

A JOIN with aggregation may sometimes be clearer or more efficient.

---

# 15. Subquery in HAVING

Subqueries can also participate in aggregate filtering.

Example:

~~~sql
SELECT
    user_id,
    SUM(amount) AS total_spent
FROM orders
GROUP BY user_id
HAVING SUM(amount) > (
    SELECT AVG(amount)
    FROM orders
);
~~~

The exact business meaning should always be checked carefully, because this compares each user's total spending with the average individual order amount.

The important lesson is that a scalar subquery can provide a value used by `HAVING`.

---

# 16. Subquery vs JOIN

Suppose we want users who have orders.

Using a subquery:

~~~sql
SELECT *
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
);
~~~

Using a JOIN:

~~~sql
SELECT DISTINCT u.*
FROM users AS u
JOIN orders AS o
    ON o.user_id = u.id;
~~~

Both may solve the requirement.

Conceptually:

~~~text
Subquery
→ useful when one query needs the result of another
→ EXISTS is expressive for existence checks

JOIN
→ useful when data from related tables must be combined
~~~

Do not use the rule:

~~~text
JOIN is always faster than subquery
~~~

That is not generally true.

PostgreSQL's optimizer may transform logically equivalent queries into similar execution plans.

Use `EXPLAIN ANALYZE` when performance actually matters.

---

# 17. NOT IN and NULL — Important Trap

Suppose the subquery returns:

~~~text
1
2
NULL
~~~

Now:

~~~sql
WHERE id NOT IN (1, 2, NULL)
~~~

can produce unexpected results because comparisons involving NULL can become `UNKNOWN`.

For example:

~~~text
3 <> 1
AND
3 <> 2
AND
3 <> NULL

TRUE
AND
TRUE
AND
UNKNOWN

→ UNKNOWN
~~~

A `WHERE` condition only returns rows where the final result is TRUE.

This is why `NOT EXISTS` is often preferred for anti-match logic when NULLs may be present.

---

# 18. Safer Anti-Match with NOT EXISTS

Instead of:

~~~sql
SELECT *
FROM users
WHERE id NOT IN (
    SELECT user_id
    FROM orders
);
~~~

use:

~~~sql
SELECT *
FROM users AS u
WHERE NOT EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.user_id = u.id
);
~~~

Mental model:

~~~text
Need rows with no related match?
          ↓
      NOT EXISTS
~~~

This expresses the intent clearly and avoids the classic `NOT IN` + NULL issue.

---

# 19. Subquery Returning NULL vs No Rows

These are different situations.

A scalar subquery may return a row whose value is NULL:

~~~text
one row
value = NULL
~~~

or it may return no row.

SQL expressions can behave differently depending on the surrounding operator and query.

Do not automatically treat:

~~~text
NULL result
~~~

and:

~~~text
no matching row
~~~

as the same concept.

---

# 20. Real ShopHub Example — Products Above Average Price

~~~sql
SELECT
    id,
    name,
    price
FROM products
WHERE price > (
    SELECT AVG(price)
    FROM products
)
ORDER BY price DESC;
~~~

Flow:

~~~text
products
    ↓
calculate average price
    ↓
compare each product
    ↓
return products above average
~~~

---

# 21. Real ShopHub Example — Customers with Paid Orders

~~~sql
SELECT
    u.id,
    u.name
FROM users AS u
WHERE EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.user_id = u.id
      AND o.status = 'paid'
);
~~~

This directly expresses:

> Return users for whom at least one paid order exists.

---

# 22. Real ShopHub Example — Customers Without Orders

~~~sql
SELECT
    u.id,
    u.name
FROM users AS u
WHERE NOT EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.user_id = u.id
);
~~~

This is a common production requirement for analytics, onboarding, or customer segmentation.

---

# 23. Common Mistakes

## Using = When the Subquery Returns Multiple Rows

Potentially wrong:

~~~sql
WHERE id = (
    SELECT user_id
    FROM orders
)
~~~

If several rows are returned, a scalar comparison cannot use them as one value.

Depending on intent, use `IN`, `EXISTS`, a JOIN, or redesign the query.

---

## Ignoring NULL with NOT IN

~~~sql
NOT IN (subquery)
~~~

can behave unexpectedly if the subquery can return NULL.

Consider `NOT EXISTS` for anti-match queries.

---

## Forgetting an Alias for a Derived Table

Prefer:

~~~sql
FROM (
    SELECT ...
) AS totals
~~~

so the result has a clear name.

---

## Assuming Subqueries Are Always Slow

They are not automatically slow.

PostgreSQL can optimize and transform many queries.

Performance depends on factors such as:

- Query structure
- Indexes
- Number of rows
- Statistics
- Selectivity
- Execution plan

Measure with `EXPLAIN ANALYZE` when needed.

---

## Overusing Correlated Subqueries

A correlated subquery can be perfectly valid, but repeated work can become expensive in some plans and datasets.

When a query is slow, compare alternatives such as:

- JOIN
- aggregation
- CTE
- better indexing

and inspect the execution plan.

---

# Interview Revision

## What is a subquery?

A subquery is a query nested inside another SQL statement whose result is used by the outer statement.

## What is a scalar subquery?

A subquery that returns a single value.

## What is a correlated subquery?

A subquery that references columns from the outer query.

## IN vs EXISTS?

~~~text
IN
→ checks whether a value belongs to a returned set

EXISTS
→ checks whether at least one matching row exists
~~~

## What is NOT EXISTS?

It returns true when the subquery produces no matching rows.

It is commonly used for anti-match queries such as finding users with no orders.

## Why can NOT IN be dangerous with NULL?

Because SQL uses three-valued logic. A NULL inside the comparison set can make the predicate evaluate to UNKNOWN instead of TRUE.

## Can a subquery appear only in WHERE?

No. Subqueries can appear in places such as:

~~~text
WHERE
FROM
SELECT
HAVING
~~~

when their result shape is valid for that context.

## Are JOINs always faster than subqueries?

No. PostgreSQL may optimize equivalent forms similarly. Choose the clearest correct query and measure with `EXPLAIN ANALYZE` when performance matters.

---

# Quick Revision

~~~text
Subquery
→ query inside another query

Scalar subquery
→ one value

IN
→ value exists in returned set

EXISTS
→ at least one matching row exists

NOT EXISTS
→ no matching row exists

Correlated subquery
→ inner query references outer query

FROM subquery
→ derived table

NOT IN + NULL
→ potential three-valued-logic trap
~~~

### Mental Model

~~~text
Outer Query
    ↓
needs information
    ↓
Subquery
    ↓
returns value / rows / existence result
    ↓
Outer Query continues
~~~

### Anti-Match Rule

~~~text
Need rows with no related record?
          ↓
      NOT EXISTS
          ↓
clear and NULL-safe pattern
~~~

---

## Key Takeaway

> **A subquery lets one query use the result of another. Understand scalar subqueries, IN, EXISTS, NOT EXISTS, and correlated subqueries—and remember that NOT IN can become problematic when NULL values are involved.**

---

[← Previous: Lesson 7 — SQL Joins](./07-sql-joins.md) | [Back to Roadmap](../README.md) | [Next: Lesson 9 — CTEs →](./09-ctes.md)
