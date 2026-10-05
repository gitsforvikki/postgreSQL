# Lesson 17 — Arrays, ENUMs & Custom Types

## First Understand Why These Features Exist

PostgreSQL gives us more data-modeling choices than only numbers, text, and JSONB.

Three useful features are:

~~~text
ARRAY
→ store multiple values of the same type

ENUM
→ restrict a value to a fixed named set

CUSTOM TYPES / DOMAINS
→ define reusable database-specific types/rules
~~~

The important skill is not only knowing the syntax. You should know **when to use each one and when not to use it**.

---

## 1. PostgreSQL Arrays

An ARRAY lets one column contain multiple values of the same PostgreSQL type.

Example:

~~~sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    tags TEXT[]
);
~~~

Insert:

~~~sql
INSERT INTO products (name, tags)
VALUES (
    'Gaming Laptop',
    ARRAY['gaming', 'laptop', 'electronics']
);
~~~

Conceptually:

~~~text
product
  │
  └── tags
       ├── gaming
       ├── laptop
       └── electronics
~~~

All elements belong to the declared array type.

---

## 2. Array Syntax

You can declare an array with:

~~~sql
tags TEXT[]
~~~

Other examples:

~~~sql
scores INTEGER[]
prices NUMERIC[]
ids UUID[]
~~~

Create array values using:

~~~sql
ARRAY['red', 'blue', 'black']
~~~

or PostgreSQL array literal syntax when appropriate.

---

## 3. Accessing Array Elements

PostgreSQL arrays normally use **1-based indexing**.

Suppose:

~~~text
tags = ['gaming', 'laptop', 'electronics']
~~~

Then:

~~~sql
SELECT tags[1]
FROM products;
~~~

returns the first element.

This differs from JavaScript:

~~~text
JavaScript array
→ first index = 0

PostgreSQL array
→ first index normally = 1
~~~

This is a useful interview/detail point.

---

## 4. Searching Arrays with ANY

Suppose:

~~~text
tags = ['gaming', 'laptop', 'electronics']
~~~

Find products containing `gaming`:

~~~sql
SELECT *
FROM products
WHERE 'gaming' = ANY(tags);
~~~

Read it as:

> Is `gaming` equal to any element inside `tags`?

---

## 5. Array Containment with @>

PostgreSQL arrays support containment operators too.

~~~sql
SELECT *
FROM products
WHERE tags @> ARRAY['gaming'];
~~~

Meaning:

~~~text
Does tags contain 'gaming'?
~~~

Multiple required values:

~~~sql
SELECT *
FROM products
WHERE tags @> ARRAY['gaming', 'electronics'];
~~~

The row matches when the array contains the requested values.

---

## 6. Adding Values to an Array

One approach is `array_append()`.

~~~sql
UPDATE products
SET tags = array_append(tags, 'featured')
WHERE id = 1;
~~~

Conceptually:

~~~text
before
['gaming', 'laptop']

append featured
       ↓

after
['gaming', 'laptop', 'featured']
~~~

You can also concatenate arrays with PostgreSQL array operators.

---

## 7. Removing an Array Value

~~~sql
UPDATE products
SET tags = array_remove(tags, 'featured')
WHERE id = 1;
~~~

This removes matching occurrences of that value from the array.

---

## 8. When Arrays Are Useful

Arrays can be useful for a **small, simple collection of homogeneous values that naturally belongs to one row**.

Examples might include:

~~~text
simple tags
small sets of flags/labels
some stored measurements or simple value collections
~~~

But you must ask an important question:

> Are these just values, or are they actually separate entities/relationships?

---

## 9. When NOT to Use an Array

Suppose an order contains products.

Bad design:

~~~sql
product_ids BIGINT[]
~~~

Why?

Because products are real entities with relationships.

You may need:

~~~text
quantity
price_at_order
discount
foreign-key integrity
product-level querying
~~~

Use a junction table instead:

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

### Important Rule

~~~text
Simple collection of values
→ ARRAY may be useful

Real entities / relationships
→ separate table + PK/FK
~~~

---

## 10. Array Indexing

PostgreSQL can use GIN indexes for useful array search patterns.

~~~sql
CREATE INDEX idx_products_tags
ON products
USING GIN (tags);
~~~

This may help supported queries such as containment searches on large datasets.

Remember from Lesson 16:

~~~text
GIN
→ useful when one stored value contains multiple searchable elements
~~~

Do not add indexes blindly. Measure your actual queries.

---

## 11. What Is an ENUM?

ENUM stands for **enumerated type**.

It defines a fixed named set of allowed values.

Example order statuses:

~~~text
pending
paid
shipped
cancelled
~~~

Create the type:

~~~sql
CREATE TYPE order_status AS ENUM (
    'pending',
    'paid',
    'shipped',
    'cancelled'
);
~~~

Use it:

~~~sql
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status order_status NOT NULL DEFAULT 'pending'
);
~~~

Now PostgreSQL understands `order_status` as its own type.

---

## 12. Why ENUM Can Be Useful

Without a rule, a text column could accidentally receive:

~~~text
paid
Paid
PAID
payment_done
abc
~~~

An ENUM limits values to the defined set.

~~~text
Application sends status
        ↓
PostgreSQL ENUM
        ↓
Allowed?
  ├── yes → store
  └── no  → reject
~~~

This gives strong database-level validation.

---

## 13. ENUM vs CHECK Constraint

You already learned another way to restrict values:

~~~sql
status TEXT NOT NULL
CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled'))
~~~

So when should you use ENUM?

Both approaches are valid, but they have different tradeoffs.

~~~text
ENUM
→ dedicated PostgreSQL type
→ reusable as that type
→ strong semantic meaning
→ good when values are stable

TEXT + CHECK
→ ordinary text column
→ constraint controls allowed values
→ often easier to evolve with normal constraint migrations
~~~

Do not memorize that one is always better.

Ask how stable the business values are and how you expect the schema to evolve.

---

## 14. When ENUM Is a Good Fit

ENUM can be a good choice when values are:

~~~text
small
well-defined
stable
meaningful as one domain/type
~~~

Example:

~~~text
order status
account state
small stable workflow states
~~~

However, if business users frequently add/remove/reorder configurable values, a lookup table may be more appropriate.

---

## 15. ENUM vs Lookup Table

Suppose product categories are:

~~~text
Electronics
Clothing
Books
Furniture
...
~~~

Should category be an ENUM?

Usually not if categories are business data that can be created, renamed, disabled, or have additional attributes.

A table is more flexible:

~~~sql
CREATE TABLE categories (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true
);
~~~

Then:

~~~sql
products.category_id
REFERENCES categories(id)
~~~

Decision:

~~~text
Small stable predefined set
→ ENUM may fit

Dynamic business-managed set
→ lookup/reference table often fits better
~~~

---

## 16. Changing ENUM Values — High Level

PostgreSQL lets you evolve ENUM types, for example by adding values.

~~~sql
ALTER TYPE order_status
ADD VALUE 'refunded';
~~~

But schema evolution around ENUMs can be less flexible than updating rows in a lookup table.

This is one reason you should reserve ENUM for genuinely stable domains.

---

## 17. What Are Custom Types?

PostgreSQL lets you define your own types.

ENUM itself is one type of user-defined type.

Another useful concept is a **composite type**.

Example:

~~~sql
CREATE TYPE address AS (
    street TEXT,
    city TEXT,
    postal_code TEXT
);
~~~

Now `address` represents a structured value.

Conceptually:

~~~text
address
 ├── street
 ├── city
 └── postal_code
~~~

PostgreSQL's type system is powerful, but custom composite types are not something you need for every application table.

Often, normal tables and columns remain simpler.

---

## 18. Composite Type vs Table

A composite type describes a structured **value**.

A table represents stored **entities/rows** and can have:

~~~text
primary keys
foreign keys
indexes
constraints
relationships
independent lifecycle
~~~

So do not use a composite type simply to avoid creating a table.

Example:

~~~text
Customer address snapshot inside another value
→ composite type may sometimes fit

Addresses as independently managed entities
→ table may be more appropriate
~~~

The correct choice depends on the data model.

---

## 19. DOMAIN — Reusable Constrained Type

Another PostgreSQL feature is a **domain**.

A domain creates a reusable type based on an existing type plus rules.

Example:

~~~sql
CREATE DOMAIN positive_money AS NUMERIC(12, 2)
CHECK (VALUE >= 0);
~~~

Use it:

~~~sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    price positive_money NOT NULL
);
~~~

Now the rule:

~~~text
value must be >= 0
~~~

is part of the reusable domain.

---

## 20. Why DOMAIN Can Be Useful

Imagine many tables contain monetary values that follow the same rule.

Without a domain:

~~~sql
price NUMERIC(12,2) CHECK (price >= 0)
shipping NUMERIC(12,2) CHECK (shipping >= 0)
fee NUMERIC(12,2) CHECK (fee >= 0)
~~~

A domain can centralize a reusable value rule.

Mental model:

~~~text
Base PostgreSQL type
        +
reusable constraint
        ↓
DOMAIN
~~~

Do not create domains for every tiny validation rule. Use them when a reusable semantic data type genuinely improves the schema.

---

## 21. DOMAIN vs ENUM

These solve different problems.

~~~text
ENUM
→ value must be one of a named fixed set

DOMAIN
→ value is based on another type and must satisfy reusable rules
~~~

Example:

~~~text
order_status ENUM
→ pending / paid / shipped / cancelled

positive_money DOMAIN
→ NUMERIC value >= 0
~~~

---

## 22. ARRAY vs JSONB

This is an important design decision.

Suppose you need tags:

~~~text
gaming
laptop
featured
~~~

An array can be natural:

~~~sql
tags TEXT[]
~~~

But suppose each product has flexible structured attributes:

~~~json
{
  "ram": "16GB",
  "storage": "1TB",
  "processor": "Intel i7"
}
~~~

JSONB is more natural.

~~~text
ARRAY
→ multiple values of the same type

JSONB
→ flexible key/value or nested structure
~~~

---

## 23. ARRAY vs Separate Table

Suppose users can have roles.

You could store:

~~~text
roles = ['admin', 'editor']
~~~

But if roles have their own:

~~~text
id
name
permissions
description
created_at
~~~

then roles are entities.

Use:

~~~text
users
  │
  ▼
user_roles
  ▲
  │
roles
~~~

Again:

~~~text
simple values
→ ARRAY can fit

real entities
→ table
~~~

---

## 24. ENUM vs JSONB

These are usually solving completely different problems.

~~~text
ENUM
→ one value from a fixed allowed set

JSONB
→ flexible structured document
~~~

Example:

~~~sql
status order_status
~~~

vs:

~~~sql
attributes JSONB
~~~

Do not use JSONB merely to avoid defining a simple stable status column.

---

## 25. The Most Important Decision Table

~~~text
Requirement                         Better starting choice
--------------------------------------------------------------
One normal scalar value             Normal column
Several same-type simple values     ARRAY
One value from stable fixed set     ENUM or CHECK
Flexible nested attributes          JSONB
Reusable constrained scalar type    DOMAIN
Structured reusable value           Composite type
Real entity / relationship          TABLE + PK/FK
~~~

This table is more important than memorizing every PostgreSQL function.

---

## 26. Practical ShopHub Example

A product might use several PostgreSQL features together:

~~~sql
CREATE TYPE product_state AS ENUM (
    'draft',
    'active',
    'archived'
);

CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category_id BIGINT NOT NULL REFERENCES categories(id),
    name TEXT NOT NULL,
    state product_state NOT NULL DEFAULT 'draft',
    tags TEXT[] NOT NULL DEFAULT '{}',
    attributes JSONB NOT NULL DEFAULT '{}'::jsonb,
    price NUMERIC(12,2) NOT NULL CHECK (price >= 0)
);
~~~

Meaning:

~~~text
category_id
→ real relationship → foreign key

state
→ small stable set → ENUM

tags
→ simple same-type values → ARRAY

attributes
→ flexible structured metadata → JSONB

price
→ stable scalar → normal NUMERIC column
~~~

This is the kind of decision-making that matters in production database design.

---

## 27. Common Mistakes

## Mistake 1 — Using Arrays for Real Relationships

Bad:

~~~text
order.product_ids BIGINT[]
~~~

Better:

~~~text
order_items table
~~~

when products are real entities in a many-to-many relationship.

## Mistake 2 — Using ENUM for Dynamic Business Data

If admins constantly add and manage values, a lookup table may be easier to evolve.

## Mistake 3 — Using JSONB for a Simple Fixed Status

If status is just one stable value, a normal typed column with ENUM/CHECK is clearer.

## Mistake 4 — Creating Custom Types Everywhere

PostgreSQL supports powerful types, but simpler schemas are often easier to maintain.

## Mistake 5 — Forgetting PostgreSQL Array Indexing

PostgreSQL arrays normally start at index 1, unlike JavaScript arrays.

## Mistake 6 — Assuming GIN Makes Every Array Query Fast

Index usefulness depends on query operators, selectivity, table size, and planner decisions.

---

## Interview Revision

## What is a PostgreSQL ARRAY?

A column type that can store multiple values of the same underlying PostgreSQL type.

## What index does a PostgreSQL array normally start from?

Normally 1.

## How can you check whether a value is in an array?

One common approach is `value = ANY(array_column)`.

## What is an ENUM?

A user-defined PostgreSQL type containing a predefined set of named values.

## ENUM vs CHECK?

ENUM creates a dedicated type for a stable value set. CHECK keeps a normal column type and applies a constraint. Both can enforce allowed values and have different schema-evolution tradeoffs.

## ENUM vs lookup table?

Use ENUM for small stable predefined sets. A lookup table is often better when values are dynamic business data with their own attributes/lifecycle.

## What is a DOMAIN?

A reusable user-defined type based on an existing PostgreSQL type with optional constraints.

## What is a composite type?

A PostgreSQL user-defined structured value containing multiple named fields.

## ARRAY vs JSONB?

ARRAY is suited to multiple homogeneous values. JSONB is suited to flexible key/value and nested structures.

## ARRAY vs table?

Use arrays for simple values. Use tables when items are real entities or relationships requiring keys, constraints, attributes, and independent querying.

---

## Quick Revision

~~~text
ARRAY
→ many same-type values

ENUM
→ one value from fixed named set

DOMAIN
→ reusable base type + rule

COMPOSITE TYPE
→ reusable structured value

JSONB
→ flexible nested/key-value data

TABLE
→ real entities and relationships
~~~

### Important Array Concepts

~~~text
PostgreSQL array index
→ normally starts at 1

ANY
→ compare against array elements

@>
→ containment

array_append
→ add element

array_remove
→ remove matching element

GIN
→ common index option for supported array searches
~~~

### Most Important Design Rule

~~~text
Is it a simple collection?
→ ARRAY may fit

Is it a stable fixed choice?
→ ENUM/CHECK may fit

Is it flexible structured metadata?
→ JSONB may fit

Is it a real entity or relationship?
→ TABLE + PK/FK
~~~

---

## Key Takeaway
> **PostgreSQL gives you ARRAY, ENUM, DOMAIN, composite types, JSONB, and relational tables for different modeling problems. The important skill is choosing the simplest type that correctly represents the meaning and lifecycle of the data.**

---

[← Previous: Lesson 16 — JSON & JSONB](./16-json-and-jsonb.md) | [Back to Roadmap](../README.md) | [Next: Lesson 18 — Views & Materialized Views →](./18-views-and-materialized-views.md)