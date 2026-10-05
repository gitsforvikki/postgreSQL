# Lesson 18 — Views & Materialized Views

## First Understand the Problem

Imagine your ShopHub admin dashboard repeatedly needs this query:

~~~sql
SELECT
    u.id AS user_id,
    u.name,
    o.id AS order_id,
    o.total,
    o.status
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.status = 'paid';
~~~

If many parts of the application need the same complex query, repeating it everywhere makes the code harder to maintain.

PostgreSQL gives us **Views**.

Now imagine another query calculates monthly sales across millions of order rows. Running that expensive calculation every time a dashboard opens may be slow.

PostgreSQL also gives us **Materialized Views**.

~~~text
Repeated reusable query
→ VIEW

Expensive query whose result can be temporarily cached/stored
→ MATERIALIZED VIEW
~~~

---

## 1. What Is a View?

A **View** is essentially a named/saved SQL query that you can query like a table.

Example:

~~~sql
CREATE VIEW paid_orders AS
SELECT
    id,
    user_id,
    total,
    created_at
FROM orders
WHERE status = 'paid';
~~~

Now instead of repeating the full query:

~~~sql
SELECT *
FROM paid_orders;
~~~

Mental model:

~~~text
VIEW
  ↓
saved query definition
  ↓
underlying tables
  ↓
current result when queried
~~~

A normal view generally does **not** store a separate copy of the query result.

---

## 2. A View Is Not Another Copy of the Data

Suppose:

~~~text
orders table
paid order count = 100
~~~

Then you insert another paid order.

When you query the view again, it sees the underlying current data.

~~~text
orders changed
    ↓
query view
    ↓
view query runs against current tables
    ↓
new result
~~~

You do not normally need to refresh a standard view.

---

## 3. Why Use Views?

Views can help with several things.

### Reuse complex SQL

Instead of repeating:

~~~text
JOIN
 +
filters
 +
calculated fields
 +
aggregation
~~~

you can expose the logic through a named view.

### Simplify application queries

Application:

~~~sql
SELECT * FROM customer_order_summary;
~~~

instead of embedding a large query everywhere.

### Provide abstraction

Applications can depend on a useful database-facing shape rather than repeating the underlying join logic.

### Security / access design

Privileges can be designed around views in some architectures so consumers interact with a restricted representation rather than directly with all base-table data.

Important: a view is **not automatically a security boundary just because it exists**. PostgreSQL permissions and the view's security behavior still need to be configured correctly.

---

## 4. View Example with JOIN

~~~sql
CREATE VIEW customer_orders AS
SELECT
    o.id AS order_id,
    o.created_at,
    o.status,
    o.total,
    u.id AS user_id,
    u.name AS customer_name,
    u.email AS customer_email
FROM orders o
JOIN users u ON u.id = o.user_id;
~~~

Now:

~~~sql
SELECT *
FROM customer_orders
WHERE user_id = 10;
~~~

Conceptually:

~~~text
Application
    ↓
customer_orders VIEW
    ↓
orders JOIN users
    ↓
result
~~~

---

## 5. Views Can Hide Complexity

Suppose calculating order totals requires:

~~~text
orders
   +
order_items
   +
products
   +
aggregations
~~~

A view can package that SQL behind a meaningful database object.

This does not magically make the underlying query faster. It primarily improves **reuse and abstraction**.

That distinction is important.

~~~text
VIEW
→ cleaner/reusable query interface
≠
automatic performance optimization
~~~

---

## 6. Can You INSERT or UPDATE Through a View?

Some simple PostgreSQL views are automatically updatable.

For example, a straightforward view over one table may allow writes when PostgreSQL can map the operation back to the underlying table.

But complex views involving things such as joins, aggregates, grouping, or certain expressions may not be automatically updatable.

For learning purposes, think:

~~~text
Simple view
→ may be automatically updatable

Complex analytical/join view
→ usually use mainly for reading
~~~

Do not assume every view accepts INSERT/UPDATE/DELETE.

---

## 7. WITH CHECK OPTION

Suppose a view exposes only active products:

~~~sql
CREATE VIEW active_products AS
SELECT id, name, price, is_active
FROM products
WHERE is_active = true
WITH CHECK OPTION;
~~~

If the view is updatable, `WITH CHECK OPTION` helps ensure writes made through the view continue to satisfy its condition.

Mental model:

~~~text
active_products view
        ↓
write through view
        ↓
does resulting row still satisfy is_active = true?
        ↓
yes → allowed
no  → rejected
~~~

This matters when using an updatable view as a controlled interface.

---

## 8. What Is a Materialized View?

A **Materialized View** stores the result of a query physically.

Example:

~~~sql
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT
    date_trunc('month', created_at) AS month,
    SUM(total) AS revenue,
    COUNT(*) AS total_orders
FROM orders
WHERE status = 'paid'
GROUP BY date_trunc('month', created_at);
~~~

Now PostgreSQL stores the calculated result.

~~~text
orders
   ↓
expensive aggregation
   ↓
MATERIALIZED VIEW
   ↓
stored result
~~~

When queried, PostgreSQL can read that stored result instead of recalculating the entire underlying query each time.

---

## 9. View vs Materialized View — Most Important Difference

~~~text
VIEW
→ stores query definition
→ normally reads current underlying data
→ query work happens when view is queried

MATERIALIZED VIEW
→ stores query result physically
→ reads can be much cheaper for expensive calculations
→ result can become stale
→ needs refresh
~~~

Easy memory:

~~~text
VIEW = saved query

MATERIALIZED VIEW = saved result of query
~~~

---

## 10. The Stale Data Problem

Suppose the materialized view says:

~~~text
October revenue = ₹500,000
~~~

Then new paid orders are inserted.

The base `orders` table changes, but the materialized view does not automatically become current just because the table changed.

~~~text
orders updated
     ↓
materialized view still contains old result
     ↓
STALE DATA
~~~

To update it, refresh the materialized view.

---

## 11. REFRESH MATERIALIZED VIEW

~~~sql
REFRESH MATERIALIZED VIEW monthly_sales;
~~~

Flow:

~~~text
Base tables changed
      ↓
REFRESH
      ↓
query recalculated
      ↓
stored result replaced
      ↓
materialized view current again
~~~

The application or an operational job decides when refreshing should happen.

Examples:

~~~text
every 5 minutes
hourly
nightly
after a data pipeline
on demand
~~~

The correct refresh schedule depends on how fresh the data must be.

---

## 12. Normal Refresh and Read Availability

A regular materialized-view refresh can block concurrent reads of that materialized view while the refresh is occurring.

For reporting systems where users should continue reading the old result during refresh, PostgreSQL provides another option.

---

## 13. REFRESH MATERIALIZED VIEW CONCURRENTLY

~~~sql
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales;
~~~

High-level idea:

~~~text
Old materialized result
      ↓
readers can continue using it
      ↓
PostgreSQL refreshes
      ↓
new result becomes available
~~~

This reduces disruption for readers, but it has requirements.

One important requirement is that the materialized view has a suitable **UNIQUE index** satisfying PostgreSQL's requirements for concurrent refresh.

Example:

~~~sql
CREATE UNIQUE INDEX idx_monthly_sales_month
ON monthly_sales (month);
~~~

Then concurrent refresh can be used when the materialized view and index meet the required conditions.

Interview memory:

~~~text
REFRESH MATERIALIZED VIEW
→ refresh stored result

REFRESH ... CONCURRENTLY
→ allow concurrent reads during refresh
→ requires suitable UNIQUE index
~~~

---

## 14. Materialized Views Can Have Indexes

Because a materialized view stores rows physically, you can create indexes on it.

Example:

~~~sql
CREATE INDEX idx_monthly_sales_revenue
ON monthly_sales (revenue);
~~~

This can make querying the stored result efficient for relevant access patterns.

Normal views do not store their own result rows, so you normally optimize the underlying tables/query instead.

---

## 15. When Should You Use a Normal View?

Good cases:

~~~text
Reusable joins
Reusable filtering
Consistent query abstraction
Simplifying application SQL
Controlled representation of data
~~~

Example:

~~~text
Customer's current order details
~~~

where you want current data whenever the page loads.

---

## 16. When Should You Use a Materialized View?

Good cases:

~~~text
Expensive aggregation
Analytics dashboards
Reporting
Large repeated joins
Summary data
Read-heavy calculations
Data that can tolerate some staleness
~~~

Example:

~~~text
Monthly revenue dashboard
Top-selling products report
Daily sales summary
~~~

Instead of recalculating millions of rows on every request:

~~~text
calculate periodically
       ↓
store result
       ↓
dashboard reads small result
~~~

---

## 17. Materialized View Is a Tradeoff

You gain faster repeated reads for expensive calculations, but pay for:

~~~text
storage
refresh work
refresh scheduling
potential stale data
~~~

So the real decision is:

> Is it acceptable for this data to be slightly old in exchange for cheaper/faster reads?

If the answer is no, a materialized view may not be appropriate for that requirement.

---

## 18. View vs CTE

You learned CTEs in Lesson 9.

Both can make SQL easier to understand, but their lifetime differs.

~~~text
CTE
→ named result/query component inside one SQL statement
→ exists only for that statement

VIEW
→ persistent database object
→ reusable across many statements
~~~

Example CTE:

~~~sql
WITH paid_orders AS (
    SELECT *
    FROM orders
    WHERE status = 'paid'
)
SELECT *
FROM paid_orders;
~~~

Example view:

~~~sql
CREATE VIEW paid_orders AS
SELECT *
FROM orders
WHERE status = 'paid';
~~~

Use a CTE for structuring one query. Use a view when a query abstraction should be reusable as a database object.

---

## 19. View vs Temporary Table

A temporary table is different again.

~~~text
VIEW
→ persistent query definition
→ normally no stored result

TEMP TABLE
→ temporary physical table/data
→ normally exists for the database session/transaction depending on configuration

MATERIALIZED VIEW
→ persistent database object
→ physically stored query result
~~~

These solve different problems.

---

## 20. ShopHub Example — Current Order View

Suppose ShopHub frequently needs current order information with customer details.

~~~sql
CREATE VIEW order_details AS
SELECT
    o.id AS order_id,
    o.status,
    o.total,
    o.created_at,
    u.id AS user_id,
    u.name AS customer_name,
    u.email AS customer_email
FROM orders o
JOIN users u ON u.id = o.user_id;
~~~

Application:

~~~sql
SELECT *
FROM order_details
WHERE order_id = 101;
~~~

Why a normal view?

~~~text
Order page
→ should reflect current data
→ query is reusable
→ normal VIEW fits
~~~

---

## 21. ShopHub Example — Monthly Revenue

Suppose an admin dashboard repeatedly calculates years of revenue.

~~~sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    date_trunc('month', created_at) AS month,
    COUNT(*) AS orders_count,
    SUM(total) AS revenue
FROM orders
WHERE status = 'paid'
GROUP BY date_trunc('month', created_at);
~~~

Dashboard query:

~~~sql
SELECT *
FROM monthly_revenue
ORDER BY month DESC;
~~~

Instead of repeatedly scanning and aggregating the full orders dataset, the dashboard can read the precomputed result.

Refresh based on freshness requirements.

~~~text
Orders
  ↓
expensive aggregation during refresh
  ↓
monthly_revenue materialized view
  ↓
fast dashboard reads
~~~

---

## 22. Security and Views

Views can be part of a database security design, but simply creating a view does not secure the underlying system.

You still need to understand:

~~~text
GRANT
REVOKE
roles
ownership
schema privileges
view security behavior
~~~

Example architecture:

~~~text
Application role
      ↓
permission to query selected view
      ↓
restricted representation
~~~

PostgreSQL security is covered deeply in Lesson 35.

Important interview point:

> A view can help expose a restricted representation, but permissions must still be configured correctly.

---

## 23. Do Views Improve Performance?

A normal view does **not automatically make a query faster**.

PostgreSQL plans the query using the view definition and underlying relations.

If the underlying query is expensive, wrapping it in a normal view does not magically remove that work.

~~~text
Complex query
    ↓
put into normal view
    ↓
still fundamentally complex work
~~~

A materialized view is different because it stores the result.

~~~text
Expensive query
    ↓
precompute/store result
    ↓
cheap repeated reads
~~~

---

## 24. Common Mistakes

## Mistake 1 — Thinking a View Stores Data

A normal view primarily stores the query definition, not a separate result copy.

## Mistake 2 — Thinking Materialized Views Are Always Current

They can become stale and require refresh.

## Mistake 3 — Thinking Views Automatically Improve Performance

A normal view mainly provides reuse and abstraction.

## Mistake 4 — Using Materialized Views for Data That Must Be Real-Time

If every read must reflect the latest committed base-table data, stale snapshots may be unacceptable.

## Mistake 5 — Forgetting Refresh Strategy

Creating a materialized view without deciding when/how to refresh it can produce misleading reports.

## Mistake 6 — Assuming Every View Is Updatable

Simple views may be automatically updatable; complex views often are not.

## Mistake 7 — Treating a View as Automatic Security

Security still requires correct roles and privileges.

---

## Interview Revision

## What is a View?

A persistent database object representing a saved query that can be queried similarly to a table.

## Does a normal View store its query result?

Generally no. It stores the query definition and reads from the underlying data when queried.

## Does a View show current data?

Normally yes, because its query runs against the current underlying tables.

## What is a Materialized View?

A database object that physically stores the result of a query.

## Main difference?

~~~text
VIEW
→ saved query definition
→ current underlying data

MATERIALIZED VIEW
→ stored query result
→ may be stale
~~~

## How do you update a Materialized View?

~~~sql
REFRESH MATERIALIZED VIEW view_name;
~~~

## What is concurrent refresh?

`REFRESH MATERIALIZED VIEW CONCURRENTLY` refreshes while allowing concurrent reads, subject to PostgreSQL requirements including a suitable unique index.

## Can a Materialized View have indexes?

Yes.

## Can a normal View have its own ordinary index?

A normal view does not store its own rows, so indexes are normally created on the underlying tables rather than on the view itself.

## View vs CTE?

A CTE exists within one statement. A view is a reusable persistent database object.

## View vs Materialized View for an analytics dashboard?

If the calculation is expensive and slightly stale data is acceptable, a materialized view can be a strong option.

## Does a View automatically improve performance?

No.

---

## Quick Revision

~~~text
VIEW
→ saved query
→ current data
→ no manual refresh
→ reusable abstraction

MATERIALIZED VIEW
→ saved query result
→ physically stored
→ can become stale
→ requires refresh
→ can have indexes
~~~

### Refresh Flow

~~~text
Base tables change
      ↓
Materialized view becomes stale
      ↓
REFRESH
      ↓
Stored result updated
~~~

### Practical Choice

~~~text
Current reusable application data
→ VIEW

Expensive repeated analytics
 +
stale data acceptable
→ MATERIALIZED VIEW
~~~

### Related Concepts

~~~text
CTE
→ one statement

VIEW
→ persistent saved query

TEMP TABLE
→ temporary stored table

MATERIALIZED VIEW
→ persistent stored query result
~~~

---

## Key Takeaway
> **A normal View gives a reusable query over current underlying data, while a Materialized View stores a precomputed result for faster repeated reads at the cost of refresh work and potentially stale data.**

---

[← Previous: Lesson 17 — Arrays, ENUMs & Custom Types](./17-arrays-enums-custom-types.md) | [Back to Roadmap](../README.md) | [Next: Lesson 19 — Functions, Procedures & Triggers →](./19-functions-procedures-triggers.md)