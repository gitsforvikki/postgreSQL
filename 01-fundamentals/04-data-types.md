# Lesson 4 — PostgreSQL Data Types

## 1. What is a Data Type?

A **data type** defines what kind of value a PostgreSQL column can store.

Example:

```sql
CREATE TABLE users (
    id BIGINT,
    name TEXT,
    age INTEGER,
    is_active BOOLEAN,
    created_at TIMESTAMPTZ
);
```

Here:

```text
id         → BIGINT
name       → TEXT
age        → INTEGER
is_active  → BOOLEAN
created_at → TIMESTAMPTZ
```

Choosing the correct data type improves:

- Data integrity
- Storage
- Query behavior
- Validation
- Performance
- Code clarity

---

# 2. Main PostgreSQL Data Type Categories

Important categories include:

```text
PostgreSQL Data Types
│
├── Numeric
├── Character / String
├── Boolean
├── Date & Time
├── UUID
├── JSON / JSONB
├── Arrays
└── Specialized / Custom Types
```

We focus here on the types most useful for application development.

---

# 3. Integer Types

Integers store whole numbers.

Common types include:

```text
SMALLINT
INTEGER
BIGINT
```

Example:

```sql
CREATE TABLE products (
    id BIGINT,
    stock INTEGER
);
```

Use integers for values such as:

- IDs
- Quantities
- Counts
- Stock

Example:

```sql
INSERT INTO products (id, stock)
VALUES (1, 50);
```

---

# 4. INTEGER vs BIGINT

`INTEGER` supports a smaller range than `BIGINT`.

Conceptually:

```text
INTEGER
   ↓
normal integer range

BIGINT
   ↓
much larger integer range
```

For identifiers that may grow significantly over time, `BIGINT` is commonly used.

Example:

```sql
id BIGINT
```

Later we learn how identity columns automatically generate these IDs.

---

# 5. Decimal / Exact Numeric Types

PostgreSQL provides:

```text
NUMERIC
DECIMAL
```

They are useful when **exact decimal precision** matters.

Example:

```sql
CREATE TABLE products (
    id BIGINT,
    name TEXT,
    price NUMERIC(10, 2)
);
```

`NUMERIC(10, 2)` means conceptually:

```text
maximum precision → 10 digits
decimal scale     → 2 digits
```

Example value:

```text
2499.99
```

---

# 6. Money and Floating-Point Values

For monetary calculations, avoid blindly using floating-point types.

Why?

Floating-point numbers represent many decimal values approximately.

For financial values where exact decimal behavior matters, prefer:

```sql
NUMERIC
```

or:

```sql
DECIMAL
```

Example:

```sql
price NUMERIC(12, 2)
```

Mental model:

```text
Approximate scientific/calculation values
             ↓
        floating point

Exact decimal values
             ↓
       NUMERIC / DECIMAL
```

For application money systems, another common design is storing the smallest currency unit as an integer, such as paise or cents. The important point is to avoid accidental floating-point precision problems.

---

# 7. Character / String Types

Common PostgreSQL string types include:

```text
TEXT
VARCHAR(n)
CHAR(n)
```

## TEXT

```sql
name TEXT
```

Stores variable-length text.

## VARCHAR

```sql
username VARCHAR(50)
```

Allows variable-length text with a specified maximum length.

For many application fields, PostgreSQL `TEXT` is perfectly appropriate unless a length limit is a real business rule.

---

# 8. TEXT vs VARCHAR

Example:

```sql
name TEXT
```

versus:

```sql
name VARCHAR(100)
```

Use `VARCHAR(n)` when the maximum length itself is meaningful.

For example:

```text
username must not exceed 50 characters
```

Otherwise, `TEXT` is often simpler.

Do not assume that `VARCHAR` is automatically faster than `TEXT` in PostgreSQL.

---

# 9. BOOLEAN

PostgreSQL has a real Boolean type:

```sql
BOOLEAN
```

Example:

```sql
CREATE TABLE users (
    id BIGINT,
    name TEXT,
    is_active BOOLEAN
);
```

Insert:

```sql
INSERT INTO users (id, name, is_active)
VALUES (1, 'Vikash', true);
```

Query:

```sql
SELECT *
FROM users
WHERE is_active = true;
```

A Boolean value can conceptually be:

```text
TRUE
FALSE
NULL
```

Remember:

> `NULL` is not the same as `FALSE`.

`NULL` represents an unknown or missing value.

---

# 10. Date and Time Types

Important PostgreSQL date/time types include:

```text
DATE
TIME
TIMESTAMP
TIMESTAMPTZ
```

---

## DATE

Stores a calendar date.

```sql
birth_date DATE
```

Example:

```text
2026-10-05
```

Use it when only the date matters.

---

## TIMESTAMP

```sql
created_at TIMESTAMP
```

This represents a date and time **without time zone semantics**.

---

## TIMESTAMPTZ

```sql
created_at TIMESTAMPTZ
```

`TIMESTAMPTZ` means:

```text
TIMESTAMP WITH TIME ZONE
```

It is generally a strong choice for real-world instants such as:

- Account creation
- Order creation
- Payment completion
- Login time
- API events

Example:

```sql
created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
```

---

# 11. TIMESTAMP vs TIMESTAMPTZ

A useful mental model:

```text
TIMESTAMP
→ date + time without time-zone-aware instant semantics

TIMESTAMPTZ
→ represents an absolute point in time
→ PostgreSQL handles conversion according to session time zone
```

For events occurring at a real moment globally, such as:

```text
order created
payment completed
user logged in
```

`TIMESTAMPTZ` is usually preferable.

---

# 12. UUID

PostgreSQL has a native:

```sql
UUID
```

type.

Example:

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    name TEXT
);
```

A UUID looks like:

```text
550e8400-e29b-41d4-a716-446655440000
```

UUIDs are useful when identifiers need to be generated across distributed systems without relying on one central increasing sequence.

Common use cases:

- Public-facing IDs
- Distributed systems
- APIs
- Multiple services creating records

---

# 13. Integer ID vs UUID

Conceptually:

```text
Integer / BIGINT
→ small
→ simple
→ efficient
→ naturally sequential when generated by a sequence/identity

UUID
→ much larger identifier space
→ useful for distributed generation
→ non-sequential depending on UUID version
→ larger storage/index footprint
```

Neither is automatically correct for every application.

Choose based on requirements.

Also remember:

> A UUID is an identifier, not an authorization mechanism.

An unpredictable ID does not replace access-control checks.

---

# 14. JSON

PostgreSQL supports:

```sql
JSON
```

Example:

```sql
CREATE TABLE products (
    id BIGINT,
    metadata JSON
);
```

Example value:

```json
{
  "brand": "Logitech",
  "wireless": true
}
```

JSON is useful when data has a flexible or nested structure.

---

# 15. JSONB

PostgreSQL also provides:

```sql
JSONB
```

Example:

```sql
CREATE TABLE products (
    id BIGINT,
    name TEXT,
    metadata JSONB
);
```

Conceptually:

```text
JSON
→ stores JSON representation

JSONB
→ stores a decomposed binary representation
→ designed for efficient processing
→ supports powerful indexing/querying
```

For most application scenarios where JSON data needs to be queried or indexed, `JSONB` is commonly preferred.

We cover JSON and JSONB deeply in Lesson 16.

---

# 16. JSONB Example

Example product:

```json
{
  "brand": "Apple",
  "color": "black",
  "features": ["wireless", "bluetooth"]
}
```

It can be stored in:

```sql
metadata JSONB
```

This allows a hybrid design:

```text
Products
│
├── id        → BIGINT
├── name      → TEXT
├── price     → NUMERIC
├── stock     → INTEGER
└── metadata  → JSONB
```

Core business fields remain relational columns while flexible attributes can live in JSONB.

---

# 17. Arrays

PostgreSQL supports array columns.

Example:

```sql
CREATE TABLE products (
    id BIGINT,
    name TEXT,
    tags TEXT[]
);
```

Example value:

```text
{"electronics","keyboard","wireless"}
```

Arrays can be useful for simple collections of values.

But they should not replace proper relational modeling when the values represent real entities or relationships.

Arrays are covered deeply in Lesson 17.

---

# 18. NULL and Data Types

Any nullable column can contain:

```text
NULL
```

Example:

```sql
middle_name TEXT
```

If a user does not have a middle name stored:

```text
NULL
```

Remember:

```text
NULL
≠ 0
≠ ''
≠ false
```

It means:

```text
unknown / missing / not present
```

depending on the data model.

---

# 19. Choosing the Correct Type

Suppose we are designing an e-commerce product table.

```sql
CREATE TABLE products (
    id BIGINT,
    name TEXT,
    price NUMERIC(12, 2),
    stock INTEGER,
    is_active BOOLEAN,
    metadata JSONB,
    created_at TIMESTAMPTZ
);
```

Why?

```text
id
→ BIGINT
→ identifier

name
→ TEXT
→ string

price
→ NUMERIC
→ exact decimal

stock
→ INTEGER
→ whole number

is_active
→ BOOLEAN
→ true / false

metadata
→ JSONB
→ flexible structured data

created_at
→ TIMESTAMPTZ
→ real-world instant
```

---

# 20. Common Mistakes

## Storing numbers as TEXT

Bad design:

```sql
price TEXT
```

Then values such as:

```text
"100"
"20"
"9"
```

are strings rather than numeric values.

Use a numeric type when the value is actually numeric.

---

## Storing dates as TEXT

Avoid:

```sql
created_at TEXT
```

when the value represents a date/time.

Prefer:

```sql
created_at TIMESTAMPTZ
```

when appropriate.

---

## Using floating point blindly for money

Financial values usually require exact decimal handling.

Prefer an appropriate exact representation such as:

```sql
NUMERIC
```

or an integer smallest-unit strategy.

---

## Using JSONB for everything

This defeats many advantages of relational modeling.

Avoid:

```text
users
└── everything JSONB
```

when the data has clear relational structure.

Prefer:

```text
normal columns
+
JSONB for genuinely flexible attributes
```

---

## Thinking NULL means false

```text
NULL ≠ FALSE
```

They represent different states.

---

# 21. Practical ShopHub Example

A simplified product table might look like:

```sql
CREATE TABLE products (
    id BIGINT,
    name TEXT,
    description TEXT,
    price NUMERIC(12, 2),
    stock INTEGER,
    is_active BOOLEAN,
    metadata JSONB,
    created_at TIMESTAMPTZ
);
```

Mental model:

```text
Product
│
├── ID             → BIGINT
├── Name           → TEXT
├── Description    → TEXT
├── Price          → NUMERIC
├── Stock          → INTEGER
├── Active?        → BOOLEAN
├── Extra metadata → JSONB
└── Created time   → TIMESTAMPTZ
```

Choosing correct types makes the database understand what each value actually represents.

---

# Interview Revision

## What is a PostgreSQL data type?

A data type defines what kind of value a column can store and what operations PostgreSQL can perform on that value.

## INTEGER vs BIGINT?

Both store whole numbers, but `BIGINT` supports a much larger range.

## NUMERIC vs floating point?

`NUMERIC` provides exact decimal arithmetic, while floating-point values are approximate.

For financial calculations requiring exact decimal values, `NUMERIC` is commonly preferred.

## TEXT vs VARCHAR?

Both store strings. `VARCHAR(n)` enforces a maximum length, while `TEXT` does not specify one.

## BOOLEAN values?

Conceptually:

```text
TRUE
FALSE
NULL
```

`NULL` is not equivalent to false.

## TIMESTAMP vs TIMESTAMPTZ?

`TIMESTAMP` stores date/time without time-zone-aware instant semantics.

`TIMESTAMPTZ` represents an absolute instant and is usually preferred for events such as creation or payment timestamps.

## What is UUID?

A UUID is a 128-bit universally unique identifier type, useful when identifiers need to be generated across distributed systems.

## JSON vs JSONB?

`JSON` stores the JSON representation, while `JSONB` stores a decomposed binary representation that is generally better suited to querying and indexing.

## Should JSONB replace relational tables?

No. Use normal relational columns/tables for structured business data and relationships, and JSONB where flexible nested attributes are genuinely useful.

---

# Quick Revision

```text
Whole numbers
→ INTEGER / BIGINT

Exact decimals
→ NUMERIC / DECIMAL

Strings
→ TEXT / VARCHAR

True / False
→ BOOLEAN

Date only
→ DATE

Date + time
→ TIMESTAMP

Real-world instant
→ TIMESTAMPTZ

Distributed identifier
→ UUID

Flexible structured data
→ JSONB

Simple same-type collection
→ ARRAY
```

### Production-oriented choices

```text
Money
→ NUMERIC or integer smallest currency unit
→ avoid accidental floating-point precision issues

created_at
→ usually TIMESTAMPTZ

Flexible metadata
→ often JSONB

Large generated numeric ID
→ often BIGINT + identity
```

---

## Key Takeaway

> **Choose a PostgreSQL data type based on what the value actually represents. Correct types improve data integrity, querying, application behavior, and long-term database design.**

---

[← Previous: Lesson 3 — Basic SQL CRUD](./03-basic-sql-crud.md) | [Back to Roadmap](../README.md) | [Next: Lesson 5 — Filtering and Conditions →](../02-sql-querying/05-filtering-and-conditions.md)
