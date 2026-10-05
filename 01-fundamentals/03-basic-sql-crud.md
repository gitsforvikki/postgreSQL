# Lesson 3 — Basic SQL CRUD

## 1. What is CRUD?

CRUD represents the four basic operations we perform on data.

| CRUD | SQL | Purpose |
|---|---|---|
| Create | `INSERT` | Add new data |
| Read | `SELECT` | Retrieve data |
| Update | `UPDATE` | Modify existing data |
| Delete | `DELETE` | Remove data |

Easy memory:

```text
C → CREATE → INSERT
R → READ   → SELECT
U → UPDATE → UPDATE
D → DELETE → DELETE
```

Assume we have:

```sql
CREATE TABLE users (
    id INTEGER,
    name TEXT,
    email TEXT,
    age INTEGER,
    is_active BOOLEAN
);
```

---

# 2. INSERT — Create Data

Use `INSERT INTO` to add rows.

```sql
INSERT INTO users (id, name, email, age, is_active)
VALUES (1, 'Vikash', 'vikash@example.com', 25, true);
```

Multiple rows can also be inserted:

```sql
INSERT INTO users (id, name, email, age, is_active)
VALUES
    (2, 'Rahul', 'rahul@example.com', 28, true),
    (3, 'Aman', 'aman@example.com', 22, false);
```

### Recommended practice

Explicitly specify the columns:

```sql
INSERT INTO users (name, email)
VALUES ('Vikash', 'vikash@example.com');
```

This is clearer and less dependent on table column order.

---

# 3. SELECT — Read Data

Retrieve all columns:

```sql
SELECT *
FROM users;
```

Retrieve only required columns:

```sql
SELECT id, name, email
FROM users;
```

In application code, selecting only the columns you need is usually better than blindly using `SELECT *`.

```text
SELECT *
→ every column

SELECT id, name
→ only required columns
```

---

# 4. WHERE — Filter Rows

Use `WHERE` to select specific rows.

```sql
SELECT id, name
FROM users
WHERE age > 24;
```

Common comparison operators:

| Operator | Meaning |
|---|---|
| `=` | Equal |
| `<>` or `!=` | Not equal |
| `>` | Greater than |
| `<` | Less than |
| `>=` | Greater than or equal |
| `<=` | Less than or equal |

Example:

```sql
SELECT *
FROM users
WHERE is_active = true;
```

Filtering is covered more deeply in Lesson 5.

---

# 5. AND, OR and NOT

### AND

Both conditions must be true.

```sql
SELECT *
FROM users
WHERE age >= 18
  AND is_active = true;
```

### OR

At least one condition must be true.

```sql
SELECT *
FROM users
WHERE age < 18
   OR age > 60;
```

### NOT

Negates a condition.

```sql
SELECT *
FROM users
WHERE NOT is_active;
```

Parentheses are important when combining conditions:

```sql
SELECT *
FROM users
WHERE is_active = true
  AND (age < 25 OR age > 40);
```

---

# 6. ORDER BY — Sort Results

Ascending:

```sql
SELECT *
FROM users
ORDER BY age ASC;
```

Descending:

```sql
SELECT *
FROM users
ORDER BY age DESC;
```

`ASC` is ascending and `DESC` is descending.

Multiple columns can be used:

```sql
SELECT *
FROM users
ORDER BY age DESC, name ASC;
```

---

# 7. LIMIT — Restrict Result Count

Return only five rows:

```sql
SELECT *
FROM users
LIMIT 5;
```

This is useful when applications should not return an unlimited number of records.

---

# 8. OFFSET — Skip Rows

```sql
SELECT *
FROM users
ORDER BY id
LIMIT 10
OFFSET 20;
```

This means:

```text
Skip first 20 rows
        ↓
Return next 10 rows
```

`LIMIT` + `OFFSET` is commonly used for basic pagination.

Later, in query optimization, we learned that large offsets can become inefficient and **keyset/cursor pagination** can be preferable for large datasets.

---

# 9. DISTINCT — Remove Duplicate Results

Suppose several users have the same age.

```sql
SELECT DISTINCT age
FROM users;
```

Instead of:

```text
22
22
25
25
28
```

you may get:

```text
22
25
28
```

`DISTINCT` removes duplicate result rows for the selected expressions.

---

# 10. Column Aliases

Aliases make output easier to read.

```sql
SELECT
    name AS user_name,
    email AS user_email
FROM users;
```

Result conceptually:

```text
user_name | user_email
----------+-------------------
Vikash    | vikash@example.com
```

Aliases are especially useful with calculations, joins, and aggregate queries.

---

# 11. UPDATE — Modify Existing Data

Update one user's email:

```sql
UPDATE users
SET email = 'new@example.com'
WHERE id = 1;
```

Update multiple columns:

```sql
UPDATE users
SET
    name = 'Vikash Kumar',
    is_active = true
WHERE id = 1;
```

## Important Warning

This is dangerous:

```sql
UPDATE users
SET is_active = false;
```

Because there is no `WHERE` condition, PostgreSQL updates **every row**.

Always check whether your update is intentionally targeting all rows.

---

# 12. DELETE — Remove Data

Delete one user:

```sql
DELETE FROM users
WHERE id = 3;
```

Again, this is dangerous:

```sql
DELETE FROM users;
```

It targets **all rows**.

The table itself remains.

```text
DELETE FROM users;
       ↓
rows removed
       ↓
table remains
```

---

# 13. RETURNING — Very Useful in PostgreSQL

PostgreSQL can return affected rows directly after data-changing statements.

### INSERT

```sql
INSERT INTO users (id, name, email)
VALUES (4, 'Neha', 'neha@example.com')
RETURNING *;
```

### UPDATE

```sql
UPDATE users
SET is_active = true
WHERE id = 4
RETURNING id, name, is_active;
```

### DELETE

```sql
DELETE FROM users
WHERE id = 4
RETURNING id, name;
```

This is very useful in backend applications because the database can return the changed row immediately.

Example flow:

```text
POST /users
    ↓
INSERT INTO users ...
RETURNING *
    ↓
PostgreSQL
    ↓
new user
    ↓
API response
```

We study `RETURNING` again as a PostgreSQL-specific feature in Lesson 15.

---

# 14. CRUD and REST APIs

CRUD maps naturally to common REST API operations.

| HTTP | CRUD | SQL |
|---|---|---|
| POST | Create | INSERT |
| GET | Read | SELECT |
| PATCH / PUT | Update | UPDATE |
| DELETE | Delete | DELETE |

Example:

```text
POST /users
     ↓
INSERT

GET /users
     ↓
SELECT

PATCH /users/10
     ↓
UPDATE

DELETE /users/10
     ↓
DELETE
```

This becomes especially important when PostgreSQL is connected to Express or Next.js.

---

# 15. Conceptual SQL Query Order

A query may be written like:

```sql
SELECT name
FROM users
WHERE age >= 18
ORDER BY name
LIMIT 10;
```

But a useful conceptual processing order is:

```text
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
```

For this lesson, remember:

```text
FROM
 ↓
WHERE
 ↓
SELECT
 ↓
ORDER BY
 ↓
LIMIT
```

Later lessons add `GROUP BY` and `HAVING`.

---

# 16. Complete Example

```sql
CREATE TABLE products (
    id INTEGER,
    name TEXT,
    price NUMERIC,
    stock INTEGER
);
```

### Create

```sql
INSERT INTO products (id, name, price, stock)
VALUES (1, 'Keyboard', 2500, 20);
```

### Read

```sql
SELECT id, name, price
FROM products
WHERE stock > 0
ORDER BY price DESC;
```

### Update

```sql
UPDATE products
SET price = 2300
WHERE id = 1
RETURNING *;
```

### Delete

```sql
DELETE FROM products
WHERE id = 1
RETURNING id;
```

---

# 17. Common Beginner Mistakes

### Forgetting WHERE in UPDATE

```sql
UPDATE users
SET is_active = false;
```

This updates every row.

### Forgetting WHERE in DELETE

```sql
DELETE FROM users;
```

This targets every row.

### Selecting unnecessary columns

```sql
SELECT *
FROM users;
```

may retrieve more data than the application needs.

Prefer specific columns when appropriate.

### Assuming LIMIT guarantees which rows are returned

For predictable pagination/results, combine it with a meaningful `ORDER BY`.

### Confusing DELETE and DROP

```text
DELETE → removes rows
DROP   → removes the database object itself
```

---

# Interview Revision

## What does CRUD mean?

```text
Create → INSERT
Read   → SELECT
Update → UPDATE
Delete → DELETE
```

## What is WHERE used for?

`WHERE` filters rows based on a condition.

## What does ORDER BY do?

It sorts query results using ascending (`ASC`) or descending (`DESC`) order.

## What is LIMIT?

`LIMIT` restricts how many rows a query returns.

## What is OFFSET?

`OFFSET` skips a specified number of rows before returning results.

## What does DISTINCT do?

It removes duplicate rows from the selected result.

## What is RETURNING in PostgreSQL?

`RETURNING` returns rows affected by an `INSERT`, `UPDATE`, or `DELETE` statement.

## What happens if UPDATE has no WHERE clause?

Every row targeted by the statement is updated.

## What happens if DELETE has no WHERE clause?

Every row targeted by the statement is deleted, while the table remains.

---

# Quick Revision

```text
INSERT
→ create rows

SELECT
→ read rows

UPDATE
→ modify rows

DELETE
→ remove rows

WHERE
→ filter rows

ORDER BY
→ sort results

LIMIT
→ restrict result count

OFFSET
→ skip rows

DISTINCT
→ remove duplicate result rows

AS
→ alias

RETURNING
→ return changed rows
```

### CRUD + API

```text
POST   → INSERT
GET    → SELECT
PATCH  → UPDATE
DELETE → DELETE
```

### Safety Rule

```text
UPDATE / DELETE
      ↓
Check WHERE carefully
      ↓
Avoid accidental changes to every row
```

---

## Key Takeaway

> **CRUD is the foundation of database interaction: INSERT creates data, SELECT reads it, UPDATE modifies it, and DELETE removes it. Filtering, sorting, pagination, and RETURNING make these operations practical for real applications.**

---

[← Previous: Lesson 2 — Database, Schema, Tables, Rows & Columns](./02-database-schema-tables.md) | [Back to Roadmap](../README.md) | [Next: Lesson 4 — PostgreSQL Data Types →](./04-data-types.md)
