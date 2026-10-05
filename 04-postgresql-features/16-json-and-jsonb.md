# Lesson 16 — JSON & JSONB

## First Understand the Problem

PostgreSQL works best with normal relational columns when data has a stable structure.

~~~text
products

id | name   | price | stock
---+--------+-------+------
1  | Laptop | 60000 | 10
~~~

But different product types may need different extra attributes:

~~~text
Laptop → RAM, CPU, storage
T-Shirt → size, color, material
Mobile → battery, camera, screen size
~~~

Creating many mostly-empty columns can become awkward. PostgreSQL therefore supports JSON and JSONB for flexible structured data.

~~~text
Stable/core data → relational columns
Flexible/nested attributes → often JSONB
~~~

---

## 1. JSON

JSON is structured key/value data.

~~~json
{
  "brand": "Dell",
  "ram": "16GB",
  "storage": "512GB"
}
~~~

PostgreSQL can store it directly:

~~~sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    metadata JSON
);
~~~

---

## 2. JSON vs JSONB

PostgreSQL has both `JSON` and `JSONB`.

~~~text
JSON
→ primarily preserves the supplied JSON text representation

JSONB
→ decomposed binary representation
→ designed for efficient processing
→ supports powerful indexing
~~~

For most application JSON that you need to search/filter/index, **JSONB is usually the better default**.

~~~sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb
);
~~~

---

## 3. Why Not Put Everything in JSONB?

You could create only an ID plus one huge JSONB document, but that usually throws away relational advantages.

Core fields such as these are normally better as typed columns:

~~~text
id
name
price
stock
category_id
created_at
~~~

They benefit from clear types, constraints, foreign keys, relationships, and straightforward indexes.

A good design is often hybrid:

~~~sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category_id BIGINT NOT NULL REFERENCES categories(id),
    name TEXT NOT NULL,
    price NUMERIC(12,2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb
);
~~~

~~~text
Stable important data → relational columns
Flexible optional data → JSONB
~~~

---

## 4. Extracting Values: -> and ->>

`->` returns JSON/JSONB:

~~~sql
SELECT metadata -> 'brand'
FROM products;
~~~

`->>` returns text:

~~~sql
SELECT metadata ->> 'brand'
FROM products;
~~~

Memory:

~~~text
->  → JSON/JSONB
->> → text
~~~

This distinction is a common interview question.

---

## 5. Filtering JSONB

~~~sql
SELECT *
FROM products
WHERE metadata ->> 'brand' = 'Lenovo';
~~~

~~~sql
SELECT *
FROM products
WHERE metadata ->> 'ram' = '16GB';
~~~

Flow:

~~~text
metadata
   ↓
extract key
   ↓
text value
   ↓
compare
~~~

---

## 6. Nested JSON

Example:

~~~json
{
  "brand": "Lenovo",
  "specs": {
    "ram": "16GB",
    "storage": "1TB"
  }
}
~~~

Query nested RAM:

~~~sql
SELECT metadata -> 'specs' ->> 'ram'
FROM products;
~~~

Path:

~~~text
metadata → specs → ram
~~~

---

## 7. Path Operators #> and #>>

`#>` returns JSON/JSONB at a path:

~~~sql
SELECT metadata #> '{specs,ram}'
FROM products;
~~~

`#>>` returns text:

~~~sql
SELECT metadata #>> '{specs,ram}'
FROM products;
~~~

~~~text
#>  → JSON at path
#>> → text at path
~~~

---

## 8. JSON Arrays

JSON can contain arrays:

~~~json
{
  "colors": ["black", "silver", "blue"]
}
~~~

Arrays are useful inside flexible document attributes. But if the items represent real relational entities, a separate table is usually better.

For example, Order ↔ Product should normally use `order_items`, not a JSON array of product IDs.

---

## 9. Containment with @>

The JSONB containment operator `@>` asks whether the left JSONB value contains the right JSON structure.

~~~sql
SELECT *
FROM products
WHERE metadata @> '{"brand":"Lenovo"}';
~~~

Multiple attributes:

~~~sql
SELECT *
FROM products
WHERE metadata @> '{"brand":"Lenovo","ram":"16GB"}';
~~~

~~~text
metadata
   ↓
contains requested JSON structure?
   ↓
yes → match
~~~

---

## 10. Key Existence Operators

The question-mark operator checks whether a top-level key or array element exists.

Example concept:

~~~text
metadata contains top-level key "warranty"?
~~~

PostgreSQL also provides operators for asking whether **any** supplied keys exist or whether **all** supplied keys exist.

Interview memory:

~~~text
single question mark → key/element exists
question mark + pipe → any supplied key exists
question mark + ampersand → all supplied keys exist
~~~

---

## 11. Updating JSONB

`jsonb_set` can set or replace a value.

~~~sql
UPDATE products
SET metadata = jsonb_set(
    metadata,
    '{ram}',
    '"16GB"'::jsonb
)
WHERE id = 1;
~~~

Nested path:

~~~sql
UPDATE products
SET metadata = jsonb_set(
    metadata,
    '{specs,ram}',
    '"16GB"'::jsonb
)
WHERE id = 1;
~~~

~~~text
metadata
  ↓
specs
  ↓
ram
  ↓
replace value
~~~

---

## 12. Removing a Key

The minus operator can remove a top-level JSONB key.

~~~sql
UPDATE products
SET metadata = metadata - 'temporary_field'
WHERE id = 1;
~~~

---

## 13. JSONB Indexing

For frequent JSONB searches on large tables, PostgreSQL supports GIN indexes.

~~~sql
CREATE INDEX idx_products_metadata
ON products
USING GIN (metadata);
~~~

Simple mental model:

~~~text
JSONB document
      ↓
GIN index
      ↓
efficient supported searches
~~~

GIN stands for **Generalized Inverted Index**.

High-level comparison:

~~~text
B-tree
→ common scalar equality/range/order workloads

GIN
→ useful when one value contains many searchable elements
→ common with JSONB, arrays, full-text search
~~~

An index is not automatically useful for every query. Measure with `EXPLAIN ANALYZE`.

---

## 14. jsonb_ops vs jsonb_path_ops — High Level

JSONB GIN indexes support different operator classes.

~~~text
jsonb_ops
→ default
→ broader operator support

jsonb_path_ops
→ narrower/specialized support
→ often useful for containment-oriented workloads
~~~

Do not choose one merely because someone says it is faster. Choose according to the operators your real queries use.

---

## 15. JSONB vs Normal Columns

If every product has:

~~~text
name
price
stock
category_id
~~~

these normally belong in relational columns.

Then category-specific metadata can use JSONB:

~~~json
{
  "ram": "16GB",
  "storage": "1TB",
  "processor": "Intel i7"
}
~~~

This gives relational strength plus document-style flexibility.

---

## 16. JSONB vs Separate Table

Suppose reviews have:

~~~text
review_id
user_id
product_id
rating
comment
created_at
~~~

Reviews are real entities with their own identity, relationships, queries, and lifecycle.

Usually prefer:

~~~text
products
   │
   │ 1:N
   ▼
reviews
~~~

rather than embedding an ever-growing review collection inside product JSONB.

**Use JSONB because the data is genuinely flexible, not simply to avoid relational modeling.**

---

## 17. Decision Framework

Ask:

### Does it need a foreign key?
Prefer relational columns/tables.

### Is it a real entity with its own identity/lifecycle?
Prefer a separate table.

### Is the structure stable and important?
Usually prefer normal typed columns.

### Does the structure vary significantly between records?
JSONB may be useful.

### Is it flexible nested metadata?
JSONB is often a strong fit.

---

## 18. ShopHub Example

~~~sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category_id BIGINT NOT NULL REFERENCES categories(id),
    name TEXT NOT NULL,
    price NUMERIC(12,2) NOT NULL CHECK (price >= 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    attributes JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
~~~

Laptop attributes:

~~~json
{
  "ram": "16GB",
  "storage": "1TB",
  "processor": "Intel i7"
}
~~~

T-shirt attributes:

~~~json
{
  "size": "L",
  "color": "black",
  "material": "cotton"
}
~~~

Same product table, but flexible category-specific attributes.

Find 16GB products:

~~~sql
SELECT id, name, price
FROM products
WHERE attributes @> '{"ram":"16GB"}';
~~~

Extract color:

~~~sql
SELECT name, attributes ->> 'color' AS color
FROM products;
~~~

---

## 19. JSONB with Node.js

A JavaScript object maps naturally to JSON data.

~~~js
const attributes = {
  ram: "16GB",
  storage: "1TB",
  processor: "Intel i7"
};
~~~

Use parameterized SQL rather than concatenating JSON/user input into SQL strings.

~~~js
const result = await pool.query(
  "INSERT INTO products (name, attributes) VALUES ($1, $2) RETURNING *",
  [name, attributes]
);
~~~

---

## Common Mistakes

### Putting everything in JSONB
PostgreSQL is still relational. Keep core data relational.

### Using JSONB to avoid relationships
A real M:N relationship such as Orders ↔ Products should normally use a junction table.

### Confusing -> and ->>
`->` returns JSON/JSONB; `->>` returns text.

### Assuming JSONB automatically makes queries fast
Large workloads may still require suitable indexes and good query design.

### Adding GIN indexes blindly
Indexes cost storage and write work. Index real query patterns.

### Treating flexible data as uncontrolled data
JSONB should still have an application-level validation/schema strategy.

---

# Interview Revision

**What is JSONB?** PostgreSQL's decomposed binary JSON representation designed for efficient querying and indexing.

**JSON vs JSONB?** JSON primarily preserves the supplied representation; JSONB is optimized for processing and supports powerful indexing.

**`->` vs `->>`?** `->` returns JSON/JSONB; `->>` returns text.

**`#>` vs `#>>`?** `#>` returns JSON at a path; `#>>` returns text at a path.

**What does `@>` mean?** JSONB containment.

**How do you update JSONB?** `jsonb_set()` is one common approach.

**What index is commonly used for JSONB?** GIN.

**Should everything flexible be JSONB?** No. Keep core structured data and relationships relational.

**JSONB vs separate table?** If the data represents a real entity with identity, relationships, constraints, or its own lifecycle, a separate table is usually better.

---

# Quick Revision

~~~text
JSON
→ original JSON representation

JSONB
→ query-friendly binary representation
~~~

~~~text
->   → JSON
#>   → JSON at path
->>  → text
#>>  → text at path
@>   → contains
~~~

~~~text
jsonb_set()
→ set/replace JSONB value

minus operator
→ remove key

GIN
→ common JSONB index type
~~~

### Most Important Design Rule

~~~text
Stable core fields
→ normal columns

Relationships
→ tables + PK/FK

Flexible nested attributes
→ JSONB
~~~

---

## Key Takeaway
> **JSONB gives PostgreSQL document-style flexibility without abandoning relational design. Keep core fields and relationships relational, and use JSONB for flexible nested attributes that genuinely vary between records.**

---

[← Previous: Lesson 15 — PostgreSQL-Specific Features](./15-postgresql-specific-features.md) | [Back to Roadmap](../README.md) | [Next: Lesson 17 — Arrays, ENUMs & Custom Types →](./17-arrays-enums-custom-types.md)