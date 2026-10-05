# Lesson 10 — Window Functions

## Core Idea

A **window function** performs a calculation across a set of related rows **without collapsing those rows into one result row**.

This is the biggest difference from `GROUP BY`.

~~~text
GROUP BY
→ combines rows into summary rows

Window Function
→ keeps individual rows
→ adds calculations across related rows
~~~

Example:

~~~sql
SELECT
    id,
    user_id,
    amount,
    SUM(amount) OVER () AS total_sales
FROM orders;
~~~

Every order remains visible while each row can also see the overall total.

---

# 1. GROUP BY vs Window Functions

Suppose:

~~~text
orders
+----+---------+--------+
| id | user_id | amount |
+----+---------+--------+
| 1  | 101     | 1000   |
| 2  | 101     | 2500   |
| 3  | 102     | 1500   |
+----+---------+--------+
~~~

Using `GROUP BY`:

~~~sql
SELECT
    user_id,
    SUM(amount) AS total
FROM orders
GROUP BY user_id;
~~~

Result:

~~~text
101 → 3500
102 → 1500
~~~

Individual orders are collapsed.

Using a window function:

~~~sql
SELECT
    id,
    user_id,
    amount,
    SUM(amount) OVER (
        PARTITION BY user_id
    ) AS user_total
FROM orders;
~~~

Result conceptually:

~~~text
1 | 101 | 1000 | 3500
2 | 101 | 2500 | 3500
3 | 102 | 1500 | 1500
~~~

The original rows remain.

---

# 2. OVER()

The `OVER()` clause turns an aggregate or supported function into a window calculation.

~~~sql
SELECT
    id,
    amount,
    SUM(amount) OVER () AS total_amount
FROM orders;
~~~

~~~text
SUM(amount)
→ normal aggregate

SUM(amount) OVER ()
→ window aggregate
~~~

---

# 3. PARTITION BY

`PARTITION BY` divides rows into logical groups for the window calculation.

~~~sql
SELECT
    id,
    user_id,
    amount,
    SUM(amount) OVER (
        PARTITION BY user_id
    ) AS user_total
FROM orders;
~~~

Mental model:

~~~text
All rows
  ↓
PARTITION BY user_id
  ↓
User 101 window
User 102 window
User 103 window
  ↓
calculate inside each window
~~~

Unlike `GROUP BY`, the rows are not collapsed.

---

# 4. ORDER BY Inside OVER()

Window functions can define an order inside the window.

~~~sql
SELECT
    id,
    user_id,
    amount,
    ROW_NUMBER() OVER (
        PARTITION BY user_id
        ORDER BY amount DESC
    ) AS position
FROM orders;
~~~

This means:

~~~text
Partition rows by user
        ↓
Order each user's rows by amount DESC
        ↓
Assign row numbers
~~~

---

# 5. ROW_NUMBER()

`ROW_NUMBER()` assigns a unique sequential number to each row in its window order.

~~~sql
SELECT
    id,
    user_id,
    amount,
    ROW_NUMBER() OVER (
        PARTITION BY user_id
        ORDER BY amount DESC
    ) AS row_num
FROM orders;
~~~

Example:

~~~text
user 101

amount | row_num
-------+--------
5000   | 1
3000   | 2
1000   | 3
~~~

If ordering values tie, PostgreSQL still assigns different row numbers. Add a deterministic tie-breaker when stable ordering matters.

Example:

~~~sql
ORDER BY amount DESC, id ASC
~~~

---

# 6. RANK()

`RANK()` assigns the same rank to tied values and leaves gaps afterward.

Suppose scores are:

~~~text
100
100
90
80
~~~

Using:

~~~sql
RANK() OVER (
    ORDER BY score DESC
)
~~~

produces:

~~~text
score | rank
------+-----
100   | 1
100   | 1
90    | 3
80    | 4
~~~

Notice rank `2` is skipped.

---

# 7. DENSE_RANK()

`DENSE_RANK()` also gives tied values the same rank, but does not leave gaps.

~~~text
score | dense_rank
------+-----------
100   | 1
100   | 1
90    | 2
80    | 3
~~~

---

# 8. ROW_NUMBER vs RANK vs DENSE_RANK

This is an important interview question.

Suppose:

~~~text
100
100
90
80
~~~

Results:

| Score | ROW_NUMBER | RANK | DENSE_RANK |
|---:|---:|---:|---:|
| 100 | 1 | 1 | 1 |
| 100 | 2 | 1 | 1 |
| 90 | 3 | 3 | 2 |
| 80 | 4 | 4 | 3 |

Memory:

~~~text
ROW_NUMBER
→ every row gets a unique sequence number

RANK
→ ties share rank
→ gaps appear

DENSE_RANK
→ ties share rank
→ no gaps
~~~

---

# 9. Top-N Per Group

This is one of the most important real-world window-function patterns.

Requirement:

> Find the top 3 highest-value orders for each user.

First rank the rows:

~~~sql
WITH ranked_orders AS (
    SELECT
        id,
        user_id,
        amount,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY amount DESC, id ASC
        ) AS rn
    FROM orders
)
SELECT *
FROM ranked_orders
WHERE rn <= 3;
~~~

Flow:

~~~text
orders
  ↓
partition by user
  ↓
order each user's orders
  ↓
ROW_NUMBER
  ↓
CTE
  ↓
rn <= 3
  ↓
top 3 per user
~~~

This pattern is highly useful in interviews and production SQL.

---

# 10. Running Total

A running total accumulates values as rows progress.

~~~sql
SELECT
    id,
    created_at,
    amount,
    SUM(amount) OVER (
        ORDER BY created_at, id
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS running_total
FROM orders;
~~~

Example:

~~~text
amount | running_total
-------+--------------
1000   | 1000
500    | 1500
2000   | 3500
750    | 4250
~~~

Mental model:

~~~text
Row 1 → first value
Row 2 → row 1 + row 2
Row 3 → rows 1 + 2 + 3
...
~~~

---

# 11. Window Frames

For ordered window aggregates, the **window frame** controls which rows around the current row participate in the calculation.

Explicit running-total frame:

~~~sql
ROWS BETWEEN UNBOUNDED PRECEDING
         AND CURRENT ROW
~~~

Meaning:

~~~text
start at first row in partition
        ↓
include rows through current row
~~~

Writing the frame explicitly is useful when you need precise behavior, especially when ordering values can tie.

---

# 12. Running Total Per User

Combine `PARTITION BY` and `ORDER BY`:

~~~sql
SELECT
    id,
    user_id,
    amount,
    SUM(amount) OVER (
        PARTITION BY user_id
        ORDER BY created_at, id
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS user_running_total
FROM orders;
~~~

Now every user gets an independent running total.

~~~text
User 101
→ running total starts from 0 conceptually

User 102
→ separate running total
~~~

---

# 13. Running Average

The same pattern works with `AVG()`.

~~~sql
SELECT
    id,
    amount,
    AVG(amount) OVER (
        ORDER BY created_at, id
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS running_average
FROM orders;
~~~

Window aggregates commonly include:

~~~text
SUM()
AVG()
COUNT()
MIN()
MAX()
~~~

---

# 14. LAG()

`LAG()` accesses a value from a previous row in the window order.

~~~sql
SELECT
    id,
    created_at,
    amount,
    LAG(amount) OVER (
        ORDER BY created_at, id
    ) AS previous_amount
FROM orders;
~~~

Conceptually:

~~~text
Current row
    ↑
previous row
    ↑
LAG()
~~~

Example:

~~~text
amount | previous_amount
-------+----------------
1000   | NULL
1500   | 1000
2000   | 1500
~~~

---

# 15. LEAD()

`LEAD()` accesses a value from a following row.

~~~sql
SELECT
    id,
    amount,
    LEAD(amount) OVER (
        ORDER BY created_at, id
    ) AS next_amount
FROM orders;
~~~

Example:

~~~text
amount | next_amount
-------+------------
1000   | 1500
1500   | 2000
2000   | NULL
~~~

Memory:

~~~text
LAG
→ previous

LEAD
→ next
~~~

---

# 16. Compare Current Row with Previous Row

A useful analytics pattern:

~~~sql
WITH order_changes AS (
    SELECT
        id,
        amount,
        LAG(amount) OVER (
            ORDER BY created_at, id
        ) AS previous_amount
    FROM orders
)
SELECT
    id,
    amount,
    previous_amount,
    amount - previous_amount AS difference
FROM order_changes;
~~~

This can answer:

> How much did the order value change compared with the previous order?

---

# 17. LAG Per User

Use `PARTITION BY` when comparison should restart for every user.

~~~sql
SELECT
    id,
    user_id,
    amount,
    LAG(amount) OVER (
        PARTITION BY user_id
        ORDER BY created_at, id
    ) AS previous_user_order
FROM orders;
~~~

Mental model:

~~~text
User 101
→ compare only with User 101's previous order

User 102
→ compare only with User 102's previous order
~~~

---

# 18. Window Function Ordering vs Final ORDER BY

These are related but different.

~~~sql
SELECT
    id,
    amount,
    ROW_NUMBER() OVER (
        ORDER BY amount DESC
    ) AS rn
FROM orders
ORDER BY id;
~~~

Inside `OVER()`:

~~~text
ORDER BY amount DESC
→ determines row-number calculation
~~~

Final:

~~~text
ORDER BY id
→ determines presentation order of result
~~~

Do not confuse the two.

---

# 19. Multiple Window Functions

You can calculate several window values in one query.

~~~sql
SELECT
    id,
    user_id,
    amount,

    SUM(amount) OVER (
        PARTITION BY user_id
    ) AS user_total,

    AVG(amount) OVER (
        PARTITION BY user_id
    ) AS user_average,

    ROW_NUMBER() OVER (
        PARTITION BY user_id
        ORDER BY amount DESC, id
    ) AS order_rank
FROM orders;
~~~

Each original order remains visible.

---

# 20. Real ShopHub Example — Customer Spending

Suppose the admin dashboard needs:

- Every order
- Customer's total spending
- Customer's average order
- Order rank for that customer

~~~sql
SELECT
    o.id,
    o.user_id,
    o.amount,

    SUM(o.amount) OVER (
        PARTITION BY o.user_id
    ) AS total_spent,

    AVG(o.amount) OVER (
        PARTITION BY o.user_id
    ) AS average_order,

    RANK() OVER (
        PARTITION BY o.user_id
        ORDER BY o.amount DESC
    ) AS amount_rank

FROM orders AS o
WHERE o.status = 'paid';
~~~

One query can provide both row-level and analytical information.

---

# 21. Real ShopHub Example — Top Product in Each Category

~~~sql
WITH ranked_products AS (
    SELECT
        id,
        category_id,
        name,
        price,
        ROW_NUMBER() OVER (
            PARTITION BY category_id
            ORDER BY price DESC, id
        ) AS rn
    FROM products
)
SELECT *
FROM ranked_products
WHERE rn = 1;
~~~

This returns one highest-priced product per category according to the specified ordering.

---

# 22. Common Mistakes

## Confusing GROUP BY with PARTITION BY

~~~text
GROUP BY
→ collapses rows

PARTITION BY
→ groups rows for window calculations
→ rows remain visible
~~~

---

## Forgetting ORDER BY for Ranking Logic

For meaningful ranking:

~~~sql
ROW_NUMBER() OVER (
    ORDER BY amount DESC
)
~~~

The ordering defines what first, second, third, etc. mean.

---

## Ignoring Ties

If values can tie, understand whether you need:

~~~text
ROW_NUMBER
RANK
DENSE_RANK
~~~

They produce different results.

---

## Non-Deterministic Tie Ordering

If:

~~~sql
ORDER BY amount DESC
~~~

contains tied amounts, `ROW_NUMBER()` still assigns unique numbers, but their order among tied rows is not guaranteed by that ordering alone.

Add a tie-breaker when needed:

~~~sql
ORDER BY amount DESC, id ASC
~~~

---

## Confusing Window ORDER BY with Result ORDER BY

Ordering inside:

~~~sql
OVER (...)
~~~

controls the window calculation.

A final:

~~~sql
ORDER BY
~~~

controls how output rows are displayed.

---

## Assuming Window Functions Reduce Row Count

Normally they do not.

Their key advantage is:

~~~text
calculate across rows
+
preserve row-level detail
~~~

---

# Interview Revision

## What is a window function?

A window function calculates a value across related rows while preserving individual rows in the result.

## GROUP BY vs window function?

~~~text
GROUP BY
→ collapses rows into groups

Window function
→ calculates across a window
→ preserves rows
~~~

## What does OVER() do?

It defines the set/window of rows used by a window function.

## What does PARTITION BY do?

It divides rows into independent groups for the window calculation without collapsing them.

## ROW_NUMBER vs RANK vs DENSE_RANK?

~~~text
ROW_NUMBER
→ unique sequential number

RANK
→ ties share rank
→ gaps after ties

DENSE_RANK
→ ties share rank
→ no gaps
~~~

## What does LAG do?

Returns a value from a previous row according to the window ordering.

## What does LEAD do?

Returns a value from a following row.

## How do you calculate a running total?

Use an ordered window aggregate such as:

~~~sql
SUM(amount) OVER (
    ORDER BY created_at, id
    ROWS BETWEEN UNBOUNDED PRECEDING
             AND CURRENT ROW
)
~~~

## How do you find top N rows per group?

Use a ranking function with `PARTITION BY`, usually inside a CTE/subquery, and filter the generated rank.

---

# Quick Revision

~~~text
OVER()
→ define window

PARTITION BY
→ divide rows into independent windows

ORDER BY inside OVER
→ define calculation order

ROW_NUMBER()
→ unique sequence

RANK()
→ ties + gaps

DENSE_RANK()
→ ties + no gaps

LAG()
→ previous row

LEAD()
→ next row

SUM() OVER(...)
→ running/partition totals

AVG() OVER(...)
→ window averages
~~~

### Most Important Difference

~~~text
GROUP BY

rows
 ↓
groups
 ↓
fewer summary rows


WINDOW FUNCTION

rows
 ↓
window calculation
 ↓
same row-level detail + calculated values
~~~

### Top-N Per Group

~~~text
PARTITION BY group
       ↓
ORDER BY target metric
       ↓
ROW_NUMBER / RANK
       ↓
filter rank <= N
~~~

---

## Key Takeaway

> **Window functions let you perform rankings, running totals, previous/next-row comparisons, and per-group analytics while keeping the original rows visible.**

---

[← Previous: Lesson 9 — CTEs](./09-ctes.md) | [Back to Roadmap](../README.md) | [Next: Lesson 11 — Primary Keys & Foreign Keys →](../03-database-design/11-primary-and-foreign-keys.md)
