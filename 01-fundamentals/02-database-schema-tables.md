# Lesson 2 — Database, Schema, Tables, Rows & Columns

## 1. PostgreSQL Structure

A PostgreSQL server can manage multiple databases. Inside each database, data and other database objects are organized into schemas.

```text
PostgreSQL Server
│
├── Database
│   │
│   ├── Schema
│   │   │
│   │   ├── Tables
│   │   ├── Views
│   │   ├── Sequences
│   │   ├── Functions
│   │   └── Types
│   │
│   └── Other Schemas
│
└── Other Databases
```

The basic hierarchy to remember is:

```text
PostgreSQL Server
        ↓
     Database
        ↓
      Schema
        ↓
      Table
        ↓
 Rows + Columns
```

---

## 2. What is a Database?

A **database** is a logical collection of related data and database objects.

For example, an e-commerce application might have a database called:

```text
shophub
```

That database can contain:

- users
- products
- orders
- payments
- categories

Create a database:

```sql
CREATE DATABASE shophub;
```

In `psql`, connect to it with:

```text
\c shophub
```

List databases:

```text
\l
```

---

## 3. What is a Schema?

A **schema** is a namespace inside a database used to organize database objects.

A schema can contain:

```text
Schema
├── Tables
├── Views
├── Sequences
├── Functions
└── Types
```

PostgreSQL commonly uses a schema named:

```text
public
```

So when you create:

```sql
CREATE TABLE users (
    id INTEGER,
    name TEXT
);
```

it will normally be created as:

```text
public.users
```

unless another schema/search path is being used.

You can explicitly reference it:

```sql
SELECT *
FROM public.users;
```

### Why are schemas useful?

Schemas help organize objects and avoid naming conflicts.

For example:

```text
shop.users
admin.users
analytics.users
```

These can be different tables because they belong to different schemas.

---

## 4. Database vs Schema

This distinction is important.

```text
PostgreSQL Server
│
├── Database: shophub
│   ├── Schema: public
│   ├── Schema: analytics
│   └── Schema: admin
│
└── Database: careerloop
    └── Schema: public
```

### Database

A higher-level logical container managed by PostgreSQL.

### Schema

A namespace **inside a database** used to organize tables and other objects.

Interview memory:

```text
Server → Database → Schema → Table
```

---

## 5. What is a Table?

A **table** stores structured data using columns and rows.

Example:

```sql
CREATE TABLE users (
    id INTEGER,
    name TEXT,
    email TEXT
);
```

Conceptually:

| id | name | email |
|---:|---|---|
| 1 | Vikash | vikash@example.com |
| 2 | Rahul | rahul@example.com |

Here:

```text
users
  │
  ├── id
  ├── name
  └── email
```

are columns.

Each user's data forms a row.

---

## 6. What is a Column?

A **column** represents an attribute or field of the entity stored in the table.

For a user:

```text
id
name
email
age
created_at
```

Example:

```sql
CREATE TABLE users (
    id INTEGER,
    name TEXT,
    email TEXT,
    age INTEGER
);
```

Each column has a data type.

```text
id     → INTEGER
name   → TEXT
email  → TEXT
age    → INTEGER
```

Data types are covered deeply in Lesson 4.

---

## 7. What is a Row?

A **row** represents one record in a table.

Example:

| id | name | email |
|---:|---|---|
| 1 | Vikash | vikash@example.com |

This complete line represents one user record.

Another row:

| id | name | email |
|---:|---|---|
| 2 | Rahul | rahul@example.com |

So:

```text
Column → attribute/field
Row    → individual record
Table  → collection of related rows
```

---

## 8. Creating a Table

Example:

```sql
CREATE TABLE users (
    id INTEGER,
    name TEXT,
    email TEXT
);
```

The structure is:

```text
CREATE TABLE table_name (
    column_name data_type,
    column_name data_type
);
```

---

## 9. Inspecting Tables with psql

List tables:

```text
\dt
```

Describe a table:

```text
\d users
```

This is useful for inspecting:

- columns
- data types
- constraints
- indexes
- relationships

Remember that these are **psql meta-commands**, not SQL statements.

---

## 10. Inserting Rows

After creating the table:

```sql
INSERT INTO users (id, name, email)
VALUES (1, 'Vikash', 'vikash@example.com');
```

Another row:

```sql
INSERT INTO users (id, name, email)
VALUES (2, 'Rahul', 'rahul@example.com');
```

Now:

```sql
SELECT *
FROM users;
```

returns the stored rows.

CRUD operations are covered in detail in Lesson 3.

---

# 11. Altering a Table

Real applications evolve, so table structures also change.

PostgreSQL provides:

```sql
ALTER TABLE
```

for changing an existing table.

---

## Add a Column

```sql
ALTER TABLE users
ADD COLUMN age INTEGER;
```

Now:

```text
users
├── id
├── name
├── email
└── age
```

---

## Rename a Column

```sql
ALTER TABLE users
RENAME COLUMN name TO full_name;
```

---

## Rename a Table

```sql
ALTER TABLE users
RENAME TO customers;
```

---

## Drop a Column

```sql
ALTER TABLE users
DROP COLUMN age;
```

Be careful: dropping a column removes the data stored in that column.

---

# 12. DELETE vs TRUNCATE vs DROP

This is a common interview topic.

## DELETE

Removes rows from a table.

```sql
DELETE FROM users
WHERE id = 1;
```

The table still exists.

```text
Table structure → remains
Selected data   → removed
```

Without a `WHERE` clause:

```sql
DELETE FROM users;
```

all rows are targeted, but the table remains.

---

## TRUNCATE

Removes all rows from a table efficiently.

```sql
TRUNCATE TABLE users;
```

```text
Table structure → remains
All rows        → removed
```

Use it when you intentionally want to empty the whole table.

---

## DROP

Removes the table itself.

```sql
DROP TABLE users;
```

```text
Table structure → removed
Table data      → removed
Table itself    → removed
```

---

## Easy Comparison

| Command | Removes Rows | Keeps Table | Can Filter Rows with WHERE |
|---|---|---|---|
| `DELETE` | Yes | Yes | Yes |
| `TRUNCATE` | All rows | Yes | No |
| `DROP` | Removes table with its data | No | No |

Memory trick:

```text
DELETE   → delete rows
TRUNCATE → empty table
DROP     → remove table
```

---

# 13. Example Structure for ShopHub

Imagine the ShopHub database.

```text
PostgreSQL Server
        │
        ▼
Database: shophub
        │
        ▼
Schema: public
        │
        ├── users
        ├── products
        ├── orders
        └── order_items
```

Inside `users`:

```text
users
│
├── id
├── name
├── email
└── created_at
```

And each user becomes a row:

```text
users
+----+--------+--------------------+
| id | name   | email              |
+----+--------+--------------------+
| 1  | Vikash | vikash@example.com |
| 2  | Rahul  | rahul@example.com  |
+----+--------+--------------------+
```

This gives the complete mental model:

```text
Server
  ↓
Database
  ↓
Schema
  ↓
Table
  ↓
Columns define structure
  ↓
Rows contain records
```

---

# 14. Common Beginner Mistakes

### Confusing database and table

A database contains tables; it is not itself a table.

### Confusing schema and database

A schema exists inside a database and acts as a namespace for database objects.

### Thinking a column is a record

A column represents an attribute. A row represents a record.

### Using DROP when you only want to remove data

`DROP TABLE` removes the table itself.

### Forgetting that ALTER TABLE changes structure

`ALTER TABLE` is used when the table definition needs to change.

---

# Interview Revision

## What is a database?

A database is a logical collection of related data and database objects managed by a DBMS.

## What is a schema in PostgreSQL?

A schema is a namespace inside a database used to organize objects such as tables, views, functions, sequences, and types.

## What is the default/common schema?

PostgreSQL commonly uses the `public` schema by default.

## What is a table?

A table stores structured data in rows and columns.

## What is a row?

A row represents an individual record.

## What is a column?

A column represents an attribute/field and has a defined data type.

## What is ALTER TABLE?

`ALTER TABLE` modifies the structure or definition of an existing table.

## DELETE vs TRUNCATE vs DROP?

```text
DELETE
→ removes rows
→ supports WHERE
→ keeps table

TRUNCATE
→ removes all rows
→ no row filtering
→ keeps table

DROP
→ removes the table itself
```

---

# Quick Revision

```text
PostgreSQL Server
      ↓
   Database
      ↓
    Schema
      ↓
    Table
   ↙     ↘
Columns   Rows
   ↓       ↓
Fields   Records
```

Important commands:

```sql
CREATE DATABASE shophub;

CREATE TABLE users (
    id INTEGER,
    name TEXT,
    email TEXT
);

ALTER TABLE users
ADD COLUMN age INTEGER;

ALTER TABLE users
RENAME COLUMN name TO full_name;

ALTER TABLE users
DROP COLUMN age;

TRUNCATE TABLE users;

DROP TABLE users;
```

Useful psql commands:

```text
\l
\c shophub
\dt
\d users
```

---

## Key Takeaway

> **PostgreSQL organizes data through a hierarchy: server → database → schema → table. Columns define what data a table stores, while rows contain the actual records.**

---

[← Previous: Lesson 1 — What is PostgreSQL?](./01-what-is-postgresql.md) | [Back to Roadmap](../README.md) | [Next: Lesson 3 — Basic SQL CRUD →](./03-basic-sql-crud.md)
