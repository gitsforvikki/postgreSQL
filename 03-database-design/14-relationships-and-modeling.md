# Lesson 14 — Relationships & Database Modeling

## First Understand the Big Picture

So far you learned:

~~~text
Primary Key
→ identifies a row

Foreign Key
→ connects tables

Constraints
→ protect data

Normalization
→ decides where facts belong
~~~

Now we combine them to answer a bigger question:

> **How do we design a real database from application requirements?**

That is the purpose of **database modeling**.

Suppose ShopHub has:

~~~text
Users
Products
Orders
Categories
~~~

These things do not exist independently.

~~~text
User places Order
Order contains Products
Product belongs to Category
~~~

A relational database models these connections as **relationships**.

---

# 1. What Is an Entity?

An **entity** is a real business concept that we want to store independently.

Examples:

~~~text
User
Product
Order
Category
Payment
~~~

An entity commonly becomes a table.

~~~text
Entity       Table

User      → users
Product   → products
Order     → orders
Category  → categories
~~~

Not every noun automatically deserves a table. Ask whether it has its own identity, attributes, lifecycle, or relationships.

---

# 2. What Is an Attribute?

An **attribute** describes an entity.

Example:

~~~text
User
 ├── id
 ├── name
 ├── email
 └── created_at
~~~

In a relational table, attributes commonly become columns.

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
~~~

Simple mental model:

~~~text
Entity    → table
Attribute → column
Record    → row
~~~

---

# 3. What Is a Relationship?

A relationship describes how entities are connected.

Examples:

~~~text
User places Order
Category contains Products
Order contains Products
Employee reports to Employee
~~~

The three relationship types you must understand are:

~~~text
1 : 1
1 : N
M : N
~~~

Read them as:

~~~text
One-to-One
One-to-Many
Many-to-Many
~~~

---

# 4. Cardinality — Very Important Concept

**Cardinality** tells us how many rows of one entity can be related to rows of another entity.

Ask two questions:

1. For one A, how many B records can exist?
2. For one B, how many A records can exist?

Example:

~~~text
One user can place many orders.
One order belongs to one user.
~~~

Therefore:

~~~text
User 1 : N Order
~~~

This questioning method makes relationship design much easier.

---

# 5. One-to-One Relationship — 1:1

Example requirement:

> Each user has at most one profile, and each profile belongs to one user.

Diagram:

~~~text
users             profiles
  1  ─────────────  1
~~~

A simple schema:

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email TEXT UNIQUE NOT NULL
);

CREATE TABLE profiles (
    id BIGINT PRIMARY KEY,
    user_id BIGINT UNIQUE NOT NULL
        REFERENCES users(id),
    bio TEXT
);
~~~

Why is `UNIQUE` important?

Without it:

~~~text
user_id = 10
user_id = 10
user_id = 10
~~~

could appear in multiple profile rows.

Then the relationship would effectively allow:

~~~text
User 1 : N Profiles
~~~

`UNIQUE(user_id)` limits a user to at most one profile.

### 1:1 Mental Model

~~~text
users.id
   │
   │ referenced by UNIQUE foreign key
   ▼
profiles.user_id
~~~

### When Is 1:1 Useful?

Examples include:

- Optional profile details
- Separating rarely used large data
- Security/privacy separation
- Extension tables

Sometimes the columns could simply belong in the same table, so do not create 1:1 tables without a reason.

---

# 6. One-to-Many Relationship — 1:N

This is probably the most common relationship.

Example:

> One user can place many orders, but each order belongs to one user.

Diagram:

~~~text
User
  1
  │
  │
  N
Orders
~~~

Where should the foreign key go?

On the **many side**.

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL
        REFERENCES users(id),
    total NUMERIC(12, 2) NOT NULL
);
~~~

Data:

~~~text
users

id | name
---+-------
1  | Vikash

orders

id  | user_id | total
----+---------+------
101 | 1       | 5000
102 | 1       | 2500
103 | 1       | 1000
~~~

One user ID appears in many order rows.

### Easy Rule

~~~text
1 : N

Put the foreign key on the N side.
~~~

Examples:

~~~text
Category 1:N Products
→ products.category_id

User 1:N Orders
→ orders.user_id

Post 1:N Comments
→ comments.post_id
~~~

---

# 7. Many-to-Many Relationship — M:N

Example:

> One order can contain many products, and one product can appear in many orders.

From the order side:

~~~text
Order 101
 ├── Laptop
 ├── Mouse
 └── Keyboard
~~~

From the product side:

~~~text
Laptop
 ├── Order 101
 ├── Order 205
 └── Order 400
~~~

So:

~~~text
Orders M : N Products
~~~

How do we store this?

Do **not** put a comma-separated product list in `orders`.

Instead create a third table.

---

# 8. Junction Table — The Solution to M:N

A many-to-many relationship is normally implemented using a **junction table** (also called associative/bridge table).

~~~text
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

The original many-to-many relationship becomes:

~~~text
Orders 1:N OrderItems N:1 Products
~~~

Schema:

~~~sql
CREATE TABLE order_items (
    order_id BIGINT NOT NULL
        REFERENCES orders(id),
    product_id BIGINT NOT NULL
        REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, product_id)
);
~~~

Example data:

~~~text
order_id | product_id | quantity
---------+------------+---------
101      | 10         | 1
101      | 20         | 2
102      | 10         | 1
~~~

This means:

~~~text
Order 101 contains Product 10
Order 101 contains Product 20
Order 102 also contains Product 10
~~~

---

# 9. Junction Tables Can Store Relationship Data

A junction table is not only for foreign keys.

Sometimes the **relationship itself has attributes**.

For an order-product relationship:

~~~text
quantity
price_at_order
discount_at_order
~~~

These values do not describe only the order or only the product.

They describe:

> **this product inside this particular order**.

Example:

~~~sql
CREATE TABLE order_items (
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_at_order NUMERIC(12, 2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
~~~

This connects directly to normalization from Lesson 13.

~~~text
quantity
→ belongs to Order + Product relationship

price_at_order
→ belongs to Order + Product relationship
~~~

---

# 10. Optional vs Mandatory Relationships

Relationships also have **optionality**.

Ask:

> Must this relationship exist?

Example 1: every order must belong to a user.

~~~sql
user_id BIGINT NOT NULL REFERENCES users(id)
~~~

This is mandatory.

Example 2: an employee may not have a manager.

~~~sql
manager_id BIGINT REFERENCES employees(id)
~~~

This allows NULL.

~~~text
NOT NULL foreign key
→ relationship required

nullable foreign key
→ relationship optional
~~~

This is a very useful modeling rule.

---

# 11. Self-Referencing Relationship

An entity can relate to itself.

Example category hierarchy:

~~~text
Electronics
   ↓
Computers
   ↓
Laptops
~~~

Schema:

~~~sql
CREATE TABLE categories (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    parent_id BIGINT REFERENCES categories(id)
);
~~~

Relationship:

~~~text
Category
   │
   └──── parent of another Category
~~~

Another common example:

~~~text
Employee
   ↓
Manager is also an Employee
~~~

This is called a **self-referencing foreign key**.

---

# 12. ER Diagram — Simple Meaning

An **ER diagram (Entity-Relationship diagram)** visually represents:

~~~text
Entities
Attributes
Relationships
Cardinality
~~~

Simple ShopHub ER diagram:

~~~text
+---------+       +---------+       +-------------+       +----------+
|  users  | 1   N | orders  | 1   N | order_items | N   1 | products |
+---------+-------+---------+-------+-------------+-------+----------+
| id PK   |       | id PK   |       | order_id FK |       | id PK    |
| name    |       | user_id |       | product_id  |       | name     |
+---------+       +---------+       | quantity    |       | price    |
                                      +-------------+       +----------+
                                                                  | N
                                                                  |
                                                                  | 1
                                                           +------------+
                                                           | categories |
                                                           +------------+
                                                           | id PK      |
                                                           +------------+
~~~

Read it as:

~~~text
User 1:N Orders
Order 1:N OrderItems
Product 1:N OrderItems
Category 1:N Products

Therefore:
Orders M:N Products through OrderItems
~~~

---

# 13. How to Convert Requirements into a Database Design

This is one of the most useful practical skills.

Suppose the requirement says:

> Users can place orders. An order can contain multiple products. Every product belongs to a category.

Do not immediately start writing SQL.

Follow this process.

## Step 1 — Find Entities

Look for important independent business concepts:

~~~text
User
Order
Product
Category
~~~

## Step 2 — Find Attributes

~~~text
User
→ id, name, email

Order
→ id, created_at, status

Product
→ id, name, price

Category
→ id, name
~~~

## Step 3 — Find Relationships

From the requirement:

~~~text
User places Order
Order contains Product
Product belongs to Category
~~~

## Step 4 — Determine Cardinality

Ask both directions.

### User ↔ Order

~~~text
One user → many orders
One order → one user

Result: 1:N
~~~

### Order ↔ Product

~~~text
One order → many products
One product → many orders

Result: M:N
~~~

### Category ↔ Product

~~~text
One category → many products
One product → one category

Result: 1:N
~~~

## Step 5 — Place Foreign Keys

~~~text
User 1:N Order
→ orders.user_id

Category 1:N Product
→ products.category_id

Order M:N Product
→ create order_items
~~~

## Step 6 — Add Constraints

Examples:

~~~text
email
→ UNIQUE + NOT NULL

price
→ CHECK price >= 0

quantity
→ CHECK quantity > 0

required relationship
→ FK + NOT NULL
~~~

## Step 7 — Normalize

Ask:

~~~text
Does each fact belong to this entity?
Is unnecessary information repeated?
Is there a partial dependency?
Is there a transitive dependency?
~~~

## Step 8 — Think About Access Patterns

After the logical model is correct, consider queries and indexes.

Examples:

~~~text
Get orders for user
→ index orders.user_id may help

Get products in category
→ index products.category_id may help
~~~

Do not let premature optimization destroy a clear data model.

---

# 14. Complete ShopHub Example

~~~sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);

CREATE TABLE categories (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT UNIQUE NOT NULL
);

CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category_id BIGINT NOT NULL REFERENCES categories(id),
    name TEXT NOT NULL,
    price NUMERIC(12, 2) NOT NULL CHECK (price >= 0)
);

CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_at_order NUMERIC(12, 2) NOT NULL CHECK (price_at_order >= 0),
    PRIMARY KEY (order_id, product_id)
);
~~~

Relationship map:

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
  ▲
  │ N:1
  │
categories
~~~

---

# 15. Surrogate Key vs Natural Key — High Level

A **natural key** comes from business data.

Example:

~~~text
email
SKU
country code
~~~

A **surrogate key** is an artificial identifier created mainly for database identity.

Example:

~~~text
id BIGINT
UUID
~~~

Typical application design:

~~~sql
id BIGINT PRIMARY KEY,
email TEXT UNIQUE NOT NULL
~~~

Here:

~~~text
id
→ surrogate primary key

email
→ business value protected by UNIQUE
~~~

This keeps row identity separate from a business value that may change.

---

# 16. Relationship Design Mistakes

## Mistake 1 — Storing IDs as Comma-Separated Text

Bad:

~~~text
orders.product_ids = '10,20,30'
~~~

Better:

~~~text
order_items
order_id | product_id
~~~

because Order ↔ Product is a real many-to-many relationship.

## Mistake 2 — Putting the Foreign Key on the Wrong Side

For:

~~~text
User 1:N Orders
~~~

normally use:

~~~text
orders.user_id
~~~

not a list of order IDs inside users.

## Mistake 3 — Forgetting UNIQUE in a True 1:1 Relationship

A foreign key by itself can allow multiple child rows.

For one profile per user:

~~~sql
user_id BIGINT UNIQUE NOT NULL REFERENCES users(id)
~~~

## Mistake 4 — Creating a Direct M:N Foreign Key

A single foreign-key column cannot directly represent many values on both sides.

Use a junction table.

## Mistake 5 — Ignoring Optionality

Ask whether the foreign key should allow NULL.

## Mistake 6 — Creating Tables for Everything

Not every field is an entity.

Example: a simple user's first name is normally an attribute, not a separate `names` table.

## Mistake 7 — Mixing Current and Historical Facts

Example:

~~~text
products.price
→ current catalog price

order_items.price_at_order
→ historical purchase price
~~~

Both can legitimately exist because they represent different facts.

---

# Interview Revision

## What is an entity?

A business object or concept that has data we need to store independently, commonly represented as a table.

## What is an attribute?

A property that describes an entity, commonly represented as a column.

## What is cardinality?

Cardinality describes how many instances of one entity can relate to instances of another entity.

## What are the main relationship types?

~~~text
1:1 → One-to-One
1:N → One-to-Many
M:N → Many-to-Many
~~~

## Where does the foreign key go in 1:N?

Normally on the **many side**.

Example:

~~~text
User 1:N Orders
→ orders.user_id
~~~

## How do you implement 1:1?

A common approach is a foreign key with a UNIQUE constraint on the dependent side.

## How do you implement M:N?

Create a junction table containing foreign keys to both entities.

## What is a junction table?

A table that represents the relationship between two entities in a many-to-many relationship and can also store attributes of that relationship.

## What is a self-referencing relationship?

A row references another row in the same table through a foreign key.

## What is an ER diagram?

A visual model showing entities, attributes, relationships, and cardinality.

## What is optionality?

It describes whether a relationship must exist. A nullable foreign key often represents an optional relationship; `NOT NULL` can make it mandatory.

## Natural key vs surrogate key?

A natural key comes from business data. A surrogate key is an artificial database identifier such as an identity ID or UUID.

---

# Quick Revision

~~~text
ENTITY
→ table

ATTRIBUTE
→ column

RELATIONSHIP
→ connection between entities

CARDINALITY
→ how many rows can relate
~~~

### Relationship Rules

~~~text
1 : 1
→ FK + UNIQUE is a common implementation

1 : N
→ foreign key on N side

M : N
→ junction table
~~~

### Example

~~~text
User 1:N Orders

users.id
   ▲
   │
orders.user_id
~~~

### Many-to-Many

~~~text
Orders M:N Products

becomes

Orders 1:N OrderItems N:1 Products
~~~

### Database Design Flow

~~~text
Requirements
    ↓
Entities
    ↓
Attributes
    ↓
Relationships
    ↓
Cardinality
    ↓
Primary + Foreign Keys
    ↓
Constraints
    ↓
Normalization
    ↓
Queries / Indexes
~~~

---

## Key Takeaway
> **Database modeling is the process of turning business requirements into entities, attributes, and relationships. Determine cardinality first, then use primary keys, foreign keys, junction tables, constraints, and normalization to represent those relationships correctly.**

---

[← Previous: Lesson 13 — Normalization](./13-normalization.md) | [Back to Roadmap](../README.md) | [Next: Lesson 15 — PostgreSQL-Specific Features →](../04-postgresql-features/15-postgresql-specific-features.md)