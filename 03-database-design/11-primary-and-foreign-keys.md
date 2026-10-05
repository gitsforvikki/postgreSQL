# Lesson 11 — Primary Keys & Foreign Keys

## Core Idea

**Primary Keys** and **Foreign Keys** are fundamental to relational database design.

~~~text
Primary Key
→ uniquely identifies a row

Foreign Key
→ creates and protects a relationship between tables
~~~

Example relationship:

~~~text
users
  │
  │ users.id
  │
  ▼
orders.user_id
~~~

One user can have many orders.

---

## 1. Primary Key

A **Primary Key (PK)** uniquely identifies each row in a table.

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name TEXT
);
~~~

A primary key is:

~~~text
UNIQUE
+
NOT NULL
~~~

It gives every row a reliable identity.

---

## 2. Why Primary Keys Matter

Names, titles, and other business values may repeat.

~~~text
id | name
---+------
1  | Rahul
2  | Rahul
3  | Rahul
~~~

The ID lets us reliably identify each individual row.

Primary keys are used for:

- Identifying records
- Updating specific records
- Deleting specific records
- Creating relationships
- Referencing rows from other tables

---

## 3. Generated Numeric Primary Keys

Modern PostgreSQL can generate numeric identifiers with identity columns:

~~~sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);
~~~

Then:

~~~sql
INSERT INTO users (name)
VALUES ('Vikash');
~~~

PostgreSQL generates the ID automatically.

Identity columns are covered more deeply in Lesson 15.

---

## 4. Primary Keys and Indexes

A PostgreSQL primary-key constraint creates the supporting unique index needed to enforce uniqueness.

~~~text
PRIMARY KEY
     ↓
unique constraint behavior
     +
supporting unique index
~~~

This also makes lookups such as this efficient in normal cases:

~~~sql
SELECT *
FROM users
WHERE id = 100;
~~~

Indexes are covered deeply in Lesson 23.

---

# 5. Foreign Key

A **Foreign Key (FK)** references a key in another table and helps enforce a valid relationship.

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
   │ foreign-key reference
   │
orders.user_id
~~~

An order's user ID must reference an existing user, unless the foreign-key value is NULL and NULL is allowed.

---

# 6. Referential Integrity

Foreign keys enforce **referential integrity**.

Suppose users contain IDs:

~~~text
1
2
3
~~~

Trying to create:

~~~sql
INSERT INTO orders (id, user_id, amount)
VALUES (101, 999, 2500);
~~~

fails if user 999 does not exist.

~~~text
Order
  ↓
user_id = 999
  ↓
No matching parent user
  ↓
Rejected
~~~

The database therefore protects the relationship even if application code contains a bug.

---

# 7. Foreign-Key Syntax

Inline:

~~~sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT REFERENCES users(id)
);
~~~

Named constraint:

~~~sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,

    CONSTRAINT orders_user_fk
        FOREIGN KEY (user_id)
        REFERENCES users(id)
);
~~~

Naming important constraints can make schema inspection and database errors easier to understand.

---

# 8. Can Foreign Keys Repeat?

Yes.

~~~text
order_id | user_id
---------+--------
101      | 1
102      | 1
103      | 1
~~~

This is valid because one user can have many orders.

~~~text
Primary Key
→ unique

Foreign Key
→ can repeat by default
~~~

---

# 9. Can a Foreign Key Be NULL?

Yes, unless it is also declared `NOT NULL`.

Optional relationship:

~~~sql
user_id BIGINT REFERENCES users(id)
~~~

Mandatory relationship:

~~~sql
user_id BIGINT NOT NULL REFERENCES users(id)
~~~

If every order must belong to a user, the second design expresses that rule more strongly.

---

# 10. ON DELETE Behavior

When a parent row is referenced by child rows, we need to decide what should happen if the parent is deleted.

Important referential actions include:

~~~text
CASCADE
SET NULL
RESTRICT
NO ACTION
~~~

The correct choice depends on the business rule.

---

## ON DELETE CASCADE

~~~sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL
        REFERENCES users(id)
        ON DELETE CASCADE
);
~~~

Conceptually:

~~~text
Delete parent
    ↓
Automatically delete referencing children
~~~

This is powerful and should only be used when child rows should truly disappear with the parent.

---

## ON DELETE SET NULL

~~~sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT
        REFERENCES users(id)
        ON DELETE SET NULL
);
~~~

When the parent is deleted:

~~~text
child.user_id
     ↓
    NULL
~~~

The child row remains.

The foreign-key column must allow NULL for this design to work as intended.

---

## ON DELETE RESTRICT

Conceptually:

~~~text
Parent has child references
        ↓
Try to delete parent
        ↓
Rejected
~~~

Use this when the parent should not be deleted while dependent records exist.

---

## NO ACTION

`NO ACTION` is the default referential action when another action is not specified.

At a high level, it prevents the database from ending with an invalid foreign-key relationship.

There are timing and deferrability nuances between `NO ACTION` and `RESTRICT`, but the beginner-level mental model is that both protect referenced parent rows from invalid deletion.

---

# 11. Choosing a Delete Action

Ask:

> What should happen to the child record if its parent disappears?

Examples:

~~~text
User → Profile
possibly CASCADE

Category → Product
possibly RESTRICT

Manager → Employee
possibly SET NULL

Customer → Historical Order
often preserve order history
~~~

Do not use `CASCADE` everywhere automatically.

---

# 12. One-to-Many Relationship

Primary and foreign keys commonly create one-to-many relationships.

~~~text
User
  1
  │
  │
  N
Orders
~~~

Example:

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    amount NUMERIC(12, 2)
);
~~~

One user can be referenced by many orders.

---

# 13. Composite Primary Key

A primary key can contain multiple columns.

~~~sql
CREATE TABLE order_items (
    order_id BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    quantity INTEGER NOT NULL,

    PRIMARY KEY (order_id, product_id)
);
~~~

The combination must be unique:

~~~text
order_id | product_id
---------+-----------
10       | 5   ✓
10       | 6   ✓
11       | 5   ✓
10       | 5   ✗ duplicate pair
~~~

This is called a **composite primary key**.

---

# 14. Multiple Foreign Keys

A table can contain several foreign keys.

~~~sql
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY,

    order_id BIGINT NOT NULL
        REFERENCES orders(id),

    product_id BIGINT NOT NULL
        REFERENCES products(id),

    quantity INTEGER NOT NULL
);
~~~

Relationship:

~~~text
orders
   ▲
   │ order_id
order_items
   │ product_id
   ▼
products
~~~

---

# 15. Self-Referencing Foreign Key

A table can reference itself.

Category hierarchy:

~~~sql
CREATE TABLE categories (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    parent_id BIGINT REFERENCES categories(id)
);
~~~

Conceptually:

~~~text
Electronics
   ↓
Computers
   ↓
Laptops
~~~

Each child category can reference another category row.

---

# 16. Employee-Manager Example

~~~sql
CREATE TABLE employees (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    manager_id BIGINT REFERENCES employees(id)
);
~~~

~~~text
CEO
 ↓
Manager
 ↓
Developer
~~~

The top-level employee can have:

~~~text
manager_id = NULL
~~~

---

# 17. UUID Primary Keys

Primary keys do not have to be integers.

~~~sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL
);
~~~

A related table can use:

~~~sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id)
);
~~~

UUIDs can be useful for distributed/public identifiers, although they have different storage and indexing characteristics from integer identifiers.

---

# 18. Foreign Keys Are Not Automatically Indexed

This is an important PostgreSQL interview and performance point.

Suppose:

~~~sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT REFERENCES users(id)
);
~~~

The referenced primary key has its supporting index.

But PostgreSQL does **not automatically create an index on**:

~~~text
orders.user_id
~~~

just because it is a foreign key.

For common lookups:

~~~sql
SELECT *
FROM orders
WHERE user_id = 10;
~~~

an index may be useful:

~~~sql
CREATE INDEX idx_orders_user_id
ON orders(user_id);
~~~

Whether the index is worthwhile depends on the workload.

---

# 19. Why Index Foreign-Key Columns?

Foreign-key columns are commonly used in:

~~~text
JOINs
WHERE filters
parent-child lookups
referential checks during parent changes
~~~

Example:

~~~sql
SELECT
    u.name,
    o.id,
    o.amount
FROM users AS u
JOIN orders AS o
    ON o.user_id = u.id
WHERE u.id = 10;
~~~

On a large orders table, an appropriate index on `orders.user_id` can be valuable.

---

# 20. Real ShopHub Relationship

A simplified e-commerce model:

~~~text
users
  │
  │ 1:N
  ▼
orders
  │
  │ 1:N
  ▼
order_items
  ▲
  │ N:1
  │
products
~~~

Example schema:

~~~sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id)
);

CREATE TABLE order_items (
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL,

    PRIMARY KEY (order_id, product_id)
);
~~~

---

# 21. Primary Key vs UNIQUE

Both enforce uniqueness, but their roles differ.

~~~text
PRIMARY KEY
→ main row identifier
→ NOT NULL
→ one primary-key constraint per table

UNIQUE
→ enforces uniqueness for another candidate value/set
→ multiple UNIQUE constraints can exist
~~~

Example:

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email TEXT UNIQUE
);
~~~

Here:

~~~text
id
→ primary identity

email
→ separately unique
~~~

Constraints are covered deeply in Lesson 12.

---

# 22. Common Mistakes

### Using an unstable business value as a primary key

Names and other mutable/non-unique values are often poor primary keys.

### Assuming foreign keys are automatically indexed

~~~text
Foreign-key constraint
≠ automatic index on referencing column
~~~

### Using CASCADE everywhere

Cascading deletion can remove large amounts of related data. It should reflect an intentional domain rule.

### Allowing NULL for a mandatory relationship

If every order must belong to a user, use:

~~~sql
user_id BIGINT NOT NULL REFERENCES users(id)
~~~

### Relying only on application validation

Foreign-key constraints protect relationships at the database level regardless of which application/script performs the write.

---

# Interview Revision

## What is a primary key?

A primary key uniquely identifies each row and is unique and non-NULL.

## What is a foreign key?

A foreign key references a key in another table and enforces referential integrity.

## Can a table have multiple primary keys?

A table has one primary-key constraint, but that constraint can contain multiple columns.

## What is a composite primary key?

A primary key consisting of two or more columns.

~~~sql
PRIMARY KEY (order_id, product_id)
~~~

## Can foreign-key values repeat?

Yes. This is normal for one-to-many relationships.

## Can a foreign key be NULL?

Yes, unless the column is declared `NOT NULL`.

## What is referential integrity?

It ensures references between related tables remain valid according to their foreign-key constraints.

## What does ON DELETE CASCADE do?

It automatically deletes referencing child rows when the parent row is deleted.

## What does ON DELETE SET NULL do?

It keeps the child row but changes its foreign-key value to NULL.

## Does PostgreSQL automatically index a foreign-key column?

No.

## Does a primary key create an index?

Yes. PostgreSQL creates a supporting unique index.

---

# Quick Revision

~~~text
PRIMARY KEY
→ uniquely identifies row
→ UNIQUE + NOT NULL
→ supporting unique index

FOREIGN KEY
→ references another key
→ enforces referential integrity

Foreign-key values
→ can repeat
→ can be NULL unless NOT NULL

ON DELETE CASCADE
→ delete children

ON DELETE SET NULL
→ keep children, clear reference

ON DELETE RESTRICT / NO ACTION
→ prevent invalid parent deletion

Composite PK
→ multiple columns form identity

Self-reference
→ table references itself

Important PostgreSQL rule:
Foreign Key ≠ automatic index
~~~

### Relationship Mental Model

~~~text
Parent Table
     │
     │ Primary Key
     ▼
Referenced Value
     ▲
     │ Foreign Key
     │
Child Table
~~~

---

## Key Takeaway

> **Primary keys give rows a reliable identity, while foreign keys connect tables and enforce valid relationships. Together they form the foundation of relational data integrity.**

---

[← Previous: Lesson 10 — Window Functions](../02-sql-querying/10-window-functions.md) | [Back to Roadmap](../README.md) | [Next: Lesson 12 — Constraints →](./12-constraints.md)
