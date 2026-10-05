# Lesson 12 — Constraints

## Core Idea

A **constraint** is a rule enforced by PostgreSQL to protect data integrity.

Important rules include `PRIMARY KEY`, `FOREIGN KEY`, `NOT NULL`, `UNIQUE`, `CHECK`, and commonly used `DEFAULT`.

## Why Constraints Matter

Application validation can be bypassed or contain bugs. Database constraints provide the final integrity layer.

~~~text
Frontend validation
        ↓
Backend validation
        ↓
Database constraints
        ↓
Reliable stored data
~~~

## 1. NOT NULL

`NOT NULL` requires a value.

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL
);
~~~

`NULL` and an empty string are different: `NULL` means missing/unknown, while `''` is an actual zero-length string. `NOT NULL` alone does not reject empty strings.

To reject blank names as well:

~~~sql
name TEXT NOT NULL CHECK (length(trim(name)) > 0)
~~~

## 2. UNIQUE

`UNIQUE` prevents duplicate values according to the constraint's uniqueness semantics.

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email TEXT UNIQUE
);
~~~

If a value must be both present and unique:

~~~sql
email TEXT UNIQUE NOT NULL
~~~

### UNIQUE and NULL

By default, a normal PostgreSQL `UNIQUE` constraint can allow multiple NULL values. Therefore, `UNIQUE` does not imply `NOT NULL`.

### Multi-column UNIQUE

Sometimes the combination must be unique:

~~~sql
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    UNIQUE (user_id, role_id)
);
~~~

`user_id` and `role_id` can individually repeat, but the same pair cannot repeat.

## 3. CHECK

`CHECK` enforces a Boolean condition.

~~~sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    price NUMERIC(12, 2) CHECK (price >= 0),
    stock INTEGER CHECK (stock >= 0)
);
~~~

Allowed-status example:

~~~sql
status TEXT CHECK (
    status IN ('pending', 'paid', 'shipped', 'cancelled')
)
~~~

A check can also compare columns in the same row:

~~~sql
CREATE TABLE bookings (
    id BIGINT PRIMARY KEY,
    starts_at TIMESTAMPTZ NOT NULL,
    ends_at TIMESTAMPTZ NOT NULL,
    CHECK (ends_at > starts_at)
);
~~~

## 4. DEFAULT

`DEFAULT` supplies a value when an INSERT omits the column.

~~~sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    is_active BOOLEAN DEFAULT true
);
~~~

Useful defaults include:

~~~sql
is_active BOOLEAN NOT NULL DEFAULT true
stock INTEGER NOT NULL DEFAULT 0
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
status TEXT NOT NULL DEFAULT 'pending'
~~~

### DEFAULT does not mean NOT NULL

`DEFAULT` handles an omitted value. `NOT NULL` forbids NULL. They solve different problems.

## 5. Named Constraints

Constraints can have explicit names:

~~~sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    price NUMERIC(12, 2),
    CONSTRAINT products_price_non_negative
        CHECK (price >= 0)
);
~~~

Meaningful names help with migrations, schema inspection, and database errors.

## 6. Column-Level vs Table-Level

A simple single-column rule can be written next to the column:

~~~sql
price NUMERIC(12, 2) CHECK (price >= 0)
~~~

Table-level syntax is useful for named or multi-column rules:

~~~sql
CONSTRAINT valid_booking_period
    CHECK (ends_at > starts_at)
~~~

## 7. Adding and Removing Constraints

Add later:

~~~sql
ALTER TABLE products
ADD CONSTRAINT products_price_non_negative
CHECK (price >= 0);
~~~

Add uniqueness:

~~~sql
ALTER TABLE users
ADD CONSTRAINT users_email_unique
UNIQUE (email);
~~~

Remove:

~~~sql
ALTER TABLE products
DROP CONSTRAINT products_price_non_negative;
~~~

Existing data generally needs to be compatible with a newly enforced rule.

## 8. Combining Constraints

Real schemas combine multiple rules:

~~~sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    sku TEXT UNIQUE NOT NULL,
    price NUMERIC(12, 2) NOT NULL CHECK (price >= 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
~~~

This protects identity, required values, uniqueness, valid numeric ranges, and sensible defaults.

## 9. Constraints vs Application Validation

A strong production approach uses both:

~~~text
Frontend
→ user-friendly validation

Backend
→ business validation and authorization

Database
→ final data-integrity guarantees
~~~

Data can be written by APIs, scripts, background jobs, migrations, admin tools, or other services. Important invariants should not depend only on frontend validation.

## 10. Constraints vs Business Workflows

Good database constraint rules include:

- price must not be negative
- quantity must be positive
- email must be present
- SKU must be unique
- foreign-key reference must be valid
- end time must be after start time

Broader workflows such as sending emails or calling payment services normally belong in application/service logic.

## Real ShopHub Example

~~~sql
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled')),
    total NUMERIC(12, 2) NOT NULL CHECK (total >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
~~~

Even if application code has a bug, PostgreSQL still protects these important order invariants.

## Common Mistakes

### Thinking NOT NULL rejects empty strings
It does not. `NULL` and `''` are different.

### Thinking UNIQUE automatically means NOT NULL
It does not. Use `UNIQUE NOT NULL` when both rules are required.

### Thinking DEFAULT prevents NULL
It does not. `DEFAULT` handles omission; `NOT NULL` rejects NULL.

### Keeping every rule only in frontend code
Frontend validation improves UX but is not the final integrity boundary.

### Using CHECK for every workflow
Use constraints for durable data invariants, not external side effects or complex application workflows.

# Interview Revision

**What is a constraint?** A database-enforced rule that protects data integrity.

**What does NOT NULL do?** Prevents NULL values.

**Is an empty string NULL?** No.

**What does UNIQUE do?** Prevents duplicate values or duplicate column combinations according to its uniqueness semantics.

**Does UNIQUE imply NOT NULL?** No.

**Can a normal UNIQUE column contain multiple NULL values in PostgreSQL?** Yes, under the default behavior.

**What does CHECK do?** Requires a row to satisfy a Boolean condition.

**What does DEFAULT do?** Supplies a value when the column is omitted from an INSERT.

**Does DEFAULT imply NOT NULL?** No.

**Why validate in both the application and database?** Application validation gives useful business/user feedback; database constraints provide final integrity regardless of which client writes the data.

# Quick Revision

~~~text
NOT NULL
→ value required

UNIQUE
→ no duplicate value/combination

CHECK
→ condition must be satisfied

DEFAULT
→ value used when omitted

PRIMARY KEY
→ row identity

FOREIGN KEY
→ valid relationship
~~~

Important combinations:

~~~sql
email TEXT UNIQUE NOT NULL
stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0)
CHECK (ends_at > starts_at)
UNIQUE (user_id, role_id)
~~~

## Key Takeaway
> **Constraints move critical data rules into PostgreSQL itself. Combine them with application validation so invalid data is rejected even when another layer fails.**

---

[← Previous: Lesson 11 — Primary Keys & Foreign Keys](./11-primary-and-foreign-keys.md) | [Back to Roadmap](../README.md) | [Next: Lesson 13 — Normalization →](./13-normalization.md)