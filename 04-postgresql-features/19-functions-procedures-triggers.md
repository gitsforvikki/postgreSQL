# Lesson 19 — Functions, Procedures & Triggers

## First Understand the Big Picture

PostgreSQL can execute logic **inside the database**, not only store data.

Three important tools are:

| Feature | Main idea | How it runs |
|---|---|---|
| **Function** | Reusable database logic that returns a value/result | Called from SQL |
| **Procedure** | Performs an operation | Called with `CALL` |
| **Trigger** | Runs automatically when a database event happens | Fired by PostgreSQL |

A simple mental model:

~~~text
FUNCTION
You ask PostgreSQL to run it
        ↓
Returns a result

PROCEDURE
You explicitly CALL it
        ↓
Performs an operation

TRIGGER
INSERT / UPDATE / DELETE happens
        ↓
PostgreSQL runs it automatically
~~~

The most important part of this lesson is understanding **when each one should be used**.

---

## 1. What Is a PostgreSQL Function?

A function is reusable logic stored inside PostgreSQL.

A function can:

- accept parameters
- execute SQL
- perform calculations
- query tables
- return one value
- return multiple rows

Simple example:

~~~sql
CREATE FUNCTION add_numbers(a INTEGER, b INTEGER)
RETURNS INTEGER
LANGUAGE SQL
AS $$
    SELECT a + b;
$$;
~~~

Call it:

~~~sql
SELECT add_numbers(10, 20);
~~~

Result:

~~~text
30
~~~

Flow:

~~~text
SELECT add_numbers(10, 20)
          ↓
PostgreSQL function
          ↓
10 + 20
          ↓
30
~~~

---

## 2. Understanding the Function Syntax

Look again:

~~~sql
CREATE FUNCTION add_numbers(a INTEGER, b INTEGER)
RETURNS INTEGER
LANGUAGE SQL
AS $$
    SELECT a + b;
$$;
~~~

The important pieces are:

~~~text
CREATE FUNCTION
→ create the function

add_numbers
→ function name

a INTEGER, b INTEGER
→ parameters

RETURNS INTEGER
→ return type

LANGUAGE SQL
→ function body is written using SQL

$$ ... $$
→ dollar-quoted function body
~~~

---

## 3. Function That Reads a Table

Functions become more useful when they work with database data.

Suppose ShopHub has:

~~~sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC(12,2) NOT NULL
);
~~~

We can create:

~~~sql
CREATE FUNCTION get_product_price(product_id BIGINT)
RETURNS NUMERIC
LANGUAGE SQL
AS $$
    SELECT price
    FROM products
    WHERE id = product_id;
$$;
~~~

Use it:

~~~sql
SELECT get_product_price(101);
~~~

The function hides the repeated query behind a reusable name.

---

## 4. Functions Can Return Tables

A PostgreSQL function is not limited to returning one value.

Example:

~~~sql
CREATE FUNCTION get_expensive_products(min_price NUMERIC)
RETURNS TABLE (
    id BIGINT,
    name TEXT,
    price NUMERIC
)
LANGUAGE SQL
AS $$
    SELECT p.id, p.name, p.price
    FROM products p
    WHERE p.price >= min_price;
$$;
~~~

Call it:

~~~sql
SELECT *
FROM get_expensive_products(50000);
~~~

Conceptually:

~~~text
min_price = 50000
       ↓
function queries products
       ↓
matching rows
       ↓
table-like result
~~~

---

## 5. SQL Functions vs PL/pgSQL Functions

PostgreSQL supports multiple procedural languages.

Two important ones are:

### SQL function

Useful when the logic is mainly SQL.

~~~sql
CREATE FUNCTION get_order_count(customer_id BIGINT)
RETURNS BIGINT
LANGUAGE SQL
AS $$
    SELECT COUNT(*)
    FROM orders
    WHERE user_id = customer_id;
$$;
~~~

### PL/pgSQL function

Useful when you need procedural logic such as:

- variables
- IF/ELSE
- loops
- exception handling
- multiple statements

Example:

~~~sql
CREATE FUNCTION stock_status(product_id BIGINT)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    current_stock INTEGER;
BEGIN
    SELECT stock
    INTO current_stock
    FROM products
    WHERE id = product_id;

    IF current_stock > 0 THEN
        RETURN 'IN_STOCK';
    ELSE
        RETURN 'OUT_OF_STOCK';
    END IF;
END;
$$;
~~~

Mental model:

~~~text
Simple SQL logic
→ LANGUAGE SQL

Procedural logic
→ LANGUAGE plpgsql
~~~

---

## 6. What Is a Stored Procedure?

A procedure is another reusable database program.

Create one with:

~~~sql
CREATE PROCEDURE archive_old_orders()
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE orders
    SET archived = true
    WHERE created_at < CURRENT_DATE - INTERVAL '2 years';
END;
$$;
~~~

Run it with:

~~~sql
CALL archive_old_orders();
~~~

Notice the important difference:

~~~text
FUNCTION
SELECT some_function(...)

PROCEDURE
CALL some_procedure(...)
~~~

---

## 7. Function vs Procedure

For interview purposes, understand the conceptual difference first.

| Function | Procedure |
|---|---|
| Called as part of SQL expressions/queries | Invoked using `CALL` |
| Has a declared return type | Does not use the function-style `RETURNS` clause |
| Often used to calculate/query/return data | Often used to perform an operation/workflow |
| Can participate in SQL expressions where allowed | Not used like a normal expression in `SELECT` |
| Transaction control is restricted | Procedures can perform transaction control in permitted invocation contexts |

Do not memorize:

> "Functions read data and procedures modify data."

That is too simplistic.

Functions can perform data-changing operations too. The real distinction is their invocation model, return behavior, and transaction-control capabilities.

---

## 8. Procedure Transaction Control — Important Nuance

PostgreSQL procedures can use transaction-control statements such as `COMMIT` and `ROLLBACK` **only in contexts where PostgreSQL permits transaction control**.

So avoid the oversimplified statement:

~~~text
Procedure can always COMMIT
Function cannot
~~~

A safer interview explanation is:

> PostgreSQL procedures have transaction-control capabilities that functions do not, but whether a procedure can commit or roll back depends on how it was invoked and its execution context.

For most application code, your Node.js/Next.js transaction boundaries will still commonly be managed by the application.

---

# Triggers

## 9. What Is a Trigger?

A trigger is database logic that PostgreSQL runs **automatically** when a configured database event occurs.

Typical events include:

~~~text
INSERT
UPDATE
DELETE
TRUNCATE
~~~

Basic flow:

~~~text
Application
    ↓
UPDATE products ...
    ↓
PostgreSQL sees UPDATE
    ↓
Trigger fires automatically
    ↓
Trigger function executes
~~~

Unlike a normal function, your application does not normally call the trigger directly.

---

## 10. Trigger Has Two Main Parts

In PostgreSQL, a common trigger setup has:

### Part 1 — Trigger Function

Contains the logic.

### Part 2 — Trigger

Defines **when** PostgreSQL should execute that function.

Flow:

~~~text
Trigger Function
= WHAT should happen

Trigger
= WHEN should it happen
~~~

This distinction is very important.

---

## 11. Simple updated_at Trigger

Suppose:

~~~sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC(12,2) NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
~~~

We want `updated_at` to change automatically whenever the row is updated.

First create the trigger function:

~~~sql
CREATE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;
~~~

Then create the trigger:

~~~sql
CREATE TRIGGER products_set_updated_at
BEFORE UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
~~~

Now:

~~~sql
UPDATE products
SET price = 59999
WHERE id = 1;
~~~

automatically causes:

~~~text
UPDATE product
     ↓
BEFORE UPDATE trigger
     ↓
set_updated_at()
     ↓
NEW.updated_at = NOW()
     ↓
updated row saved
~~~

The application did not need to manually update the timestamp.

---

## 12. Understanding NEW and OLD

Trigger functions often use two special records:

~~~text
OLD
→ row before the operation

NEW
→ row after the proposed change
~~~

For an UPDATE:

~~~text
OLD
price = 50000

UPDATE
price = 45000

NEW
price = 45000
~~~

This allows a trigger to compare what changed.

---

## 13. OLD and NEW by Operation

A useful mental model:

| Operation | OLD | NEW |
|---|---|---|
| INSERT | normally unavailable | inserted row |
| UPDATE | previous row | updated row |
| DELETE | deleted row | normally unavailable |

This becomes especially useful for auditing.

---

## 14. BEFORE vs AFTER Triggers

A trigger can run at different times.

### BEFORE

Runs before the database operation is completed.

~~~text
UPDATE requested
      ↓
BEFORE trigger
      ↓
operation continues
~~~

Useful when you need to modify or validate the row before it is stored.

Example:

~~~text
set updated_at
normalize a value
perform special validation
~~~

### AFTER

Runs after the operation has happened.

~~~text
UPDATE completes
      ↓
AFTER trigger
      ↓
trigger logic
~~~

Useful for things such as database audit records.

Easy memory:

~~~text
BEFORE
→ influence/process row before write

AFTER
→ react after write
~~~

---

## 15. Audit Log Example

Suppose you want to record every product price change.

Create an audit table:

~~~sql
CREATE TABLE product_price_audit (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id BIGINT NOT NULL,
    old_price NUMERIC(12,2),
    new_price NUMERIC(12,2),
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
~~~

Trigger function:

~~~sql
CREATE FUNCTION log_product_price_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF OLD.price IS DISTINCT FROM NEW.price THEN
        INSERT INTO product_price_audit (
            product_id,
            old_price,
            new_price
        )
        VALUES (
            OLD.id,
            OLD.price,
            NEW.price
        );
    END IF;

    RETURN NEW;
END;
$$;
~~~

Trigger:

~~~sql
CREATE TRIGGER product_price_audit_trigger
AFTER UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION log_product_price_change();
~~~

Now:

~~~text
Product price updated
       ↓
AFTER UPDATE trigger
       ↓
compare OLD.price and NEW.price
       ↓
price changed?
   ├── No  → nothing
   └── Yes → insert audit record
~~~

Notice `IS DISTINCT FROM`.

It behaves safely when NULL values are involved, unlike a simple `<>` comparison.

---

## 16. FOR EACH ROW vs FOR EACH STATEMENT

PostgreSQL supports different trigger levels.

### Row-level trigger

~~~sql
FOR EACH ROW
~~~

Runs once for each affected row.

Suppose:

~~~text
UPDATE affects 100 rows
~~~

Then a row trigger can execute:

~~~text
100 times
~~~

### Statement-level trigger

~~~sql
FOR EACH STATEMENT
~~~

Runs once for the SQL statement.

~~~text
UPDATE affects 100 rows
       ↓
statement trigger
       ↓
runs once
~~~

This matters greatly for performance and behavior.

---

## 17. INSTEAD OF Triggers

PostgreSQL also supports `INSTEAD OF` triggers for suitable views.

Conceptually:

~~~text
Application attempts operation on view
            ↓
INSTEAD OF trigger
            ↓
custom database logic performs the required work
~~~

This is more advanced.

For now remember:

~~~text
BEFORE
→ before operation

AFTER
→ after operation

INSTEAD OF
→ replace operation, commonly associated with views
~~~

---

## 18. When Are Triggers Useful?

Good trigger use cases can include:

### Automatic timestamps

~~~text
updated_at
~~~

### Audit logging

~~~text
who/what changed
old value
new value
time
~~~

### Database-centric consistency rules

When a rule must happen regardless of which application/client performs the write.

The key benefit is:

~~~text
Node.js app
Admin script
SQL console
Background worker
        ↓
all modify database
        ↓
same trigger can run
~~~

The logic lives close to the data.

---

## 19. Why Triggers Can Become Dangerous

Triggers are powerful because they run automatically.

That is also their biggest danger.

Imagine:

~~~text
UPDATE orders
~~~

but hidden behind that update:

~~~text
Trigger A
   ↓
updates another table
   ↓
Trigger B
   ↓
more database work
~~~

A developer may look only at the original UPDATE and not realize how much hidden work occurs.

Problems can include:

- unexpected side effects
- difficult debugging
- performance problems
- trigger chains
- hidden business logic
- accidental recursion

Therefore:

> Use triggers when automatic database-level behavior provides clear value, not simply because PostgreSQL supports them.

---

## 20. What Should Usually NOT Be Done in a Trigger?

Avoid putting external workflows directly inside database triggers.

Examples:

~~~text
send email
call payment gateway
call external REST API
send push notification
perform long network operation
~~~

Why?

Because database transactions should not depend on slow or unreliable external systems.

Bad mental flow:

~~~text
UPDATE order
    ↓
trigger
    ↓
external payment/email API
    ↓
network delay/failure
    ↓
database transaction affected
~~~

These workflows usually belong in application/background-job/event-processing layers.

---

## 21. Trigger vs Constraint

Suppose:

~~~text
price must never be negative
~~~

Do you need a trigger?

No.

Use a constraint:

~~~sql
price NUMERIC(12,2) CHECK (price >= 0)
~~~

General rule:

~~~text
Simple data validity rule
→ CONSTRAINT

Automatic database reaction
→ TRIGGER
~~~

Prefer the simpler declarative database feature when it can express the rule.

---

## 22. Trigger vs Application Logic

Suppose ShopHub must update `updated_at`.

You could do it in Node.js:

~~~sql
UPDATE products
SET
    price = $1,
    updated_at = NOW()
WHERE id = $2;
~~~

Or enforce it using a database trigger.

The difference:

~~~text
Application logic
→ explicit in application code
→ easier for app developers to see

Trigger
→ enforced at database level
→ runs regardless of which client modifies data
→ less visible from application code
~~~

Neither approach is automatically correct for every system.

Choose based on ownership of the rule and whether it must be guaranteed for all database writers.

---

## 23. Function vs Procedure vs Trigger

This is the most important comparison.

| Feature | Function | Procedure | Trigger |
|---|---|---|---|
| Explicitly invoked? | Yes | Yes | No, event-driven |
| Typical invocation | `SELECT function()` or SQL expression | `CALL procedure()` | Automatic |
| Returns value/result? | Yes, according to declared return type | Not with function-style `RETURNS` | Trigger function returns trigger-specific value |
| Main purpose | Reusable DB calculation/query/logic | Explicit database operation | Automatic reaction to DB event |
| Transaction control | Restricted | Possible in permitted contexts | Runs as part of triggering operation/transaction |

Easy memory:

~~~text
Need a reusable database result?
→ FUNCTION

Need to explicitly execute a stored operation?
→ PROCEDURE

Need something to happen automatically on INSERT/UPDATE/DELETE?
→ TRIGGER
~~~

---

## 24. ShopHub Practical Example

Suppose ShopHub has these requirements:

### Requirement 1

Calculate total revenue for a customer.

Possible function:

~~~text
get_customer_revenue(user_id)
~~~

### Requirement 2

Run a database maintenance/archive operation manually or from an admin job.

Possible procedure:

~~~text
CALL archive_old_orders()
~~~

### Requirement 3

Automatically record product price changes.

Possible trigger:

~~~text
UPDATE products
      ↓
price changed
      ↓
audit trigger
      ↓
product_price_audit
~~~

The choice comes from **how the logic should be invoked**.

---

## 25. Node.js / Next.js Perspective

Your backend can call PostgreSQL functions:

~~~sql
SELECT get_customer_revenue($1);
~~~

and procedures:

~~~sql
CALL archive_old_orders();
~~~

Triggers are different.

Your backend may only execute:

~~~sql
UPDATE products
SET price = $1
WHERE id = $2;
~~~

PostgreSQL itself notices the event and executes the trigger.

Architecture:

~~~text
Next.js / Node.js
      ↓
parameterized SQL
      ↓
PostgreSQL
      ↓
INSERT / UPDATE / DELETE
      ↓
Trigger automatically fires
~~~

Your application should still understand that these side effects exist.

---

## 26. Common Mistakes

### Mistake 1 — Thinking a Trigger Is Called Manually

A trigger is event-driven and fires automatically when its configured event occurs.

### Mistake 2 — Thinking Function and Procedure Are Identical

They overlap in capability, but have different invocation, return, and transaction-control semantics.

### Mistake 3 — Putting Every Business Rule in Triggers

This can make the application difficult to understand and debug.

### Mistake 4 — Using a Trigger Instead of a Simple Constraint

Use `NOT NULL`, `CHECK`, `UNIQUE`, PK/FK, etc. when they directly express the rule.

### Mistake 5 — Calling External APIs from Trigger Logic

Keep external workflows outside normal database trigger execution.

### Mistake 6 — Forgetting Row-Level Trigger Cost

A `FOR EACH ROW` trigger can run thousands of times when one statement affects thousands of rows.

### Mistake 7 — Forgetting OLD and NEW Availability

Which record exists depends on INSERT, UPDATE, or DELETE.

---

## Interview Revision

### What is a PostgreSQL function?

Reusable logic stored in PostgreSQL that has a declared return type and can be invoked from SQL.

### SQL function vs PL/pgSQL function?

SQL functions are convenient for SQL-centric logic. PL/pgSQL adds procedural features such as variables, conditions, loops, and exception handling.

### What is a stored procedure?

A stored database program invoked with `CALL`, commonly used for explicit operations. PostgreSQL procedures can have transaction-control capabilities in permitted contexts.

### What is a trigger?

Database logic that automatically executes when a configured event occurs.

### Trigger vs trigger function?

The trigger defines **when** to run. The trigger function defines **what** logic runs.

### What are OLD and NEW?

Special trigger records representing the previous and new row values where applicable.

### BEFORE vs AFTER?

`BEFORE` runs before the operation completes and can be useful for modifying the row. `AFTER` runs after the operation and is useful for reactions such as auditing.

### FOR EACH ROW vs FOR EACH STATEMENT?

Row-level triggers run once per affected row. Statement-level triggers run once per SQL statement.

### Should a trigger send emails or call payment APIs?

Usually no. External network workflows are better handled by the application, background jobs, or event-processing infrastructure.

### Constraint vs trigger?

Use declarative constraints for straightforward integrity rules. Use triggers when automatic event-driven database behavior is genuinely required.

---

## Quick Revision

~~~text
FUNCTION
→ reusable database logic
→ has declared return type
→ called from SQL
→ SQL or PL/pgSQL

PROCEDURE
→ explicit stored operation
→ CALL procedure_name(...)
→ transaction control possible in permitted contexts

TRIGGER
→ automatic
→ reacts to DB events
→ INSERT / UPDATE / DELETE etc.
→ uses trigger function
~~~

### Trigger Flow

~~~text
Application
    ↓
UPDATE products
    ↓
PostgreSQL
    ↓
BEFORE / AFTER trigger
    ↓
trigger function
    ↓
automatic DB-side behavior
~~~

### OLD vs NEW

~~~text
INSERT
OLD → unavailable
NEW → inserted row

UPDATE
OLD → previous row
NEW → updated row

DELETE
OLD → deleted row
NEW → unavailable
~~~

### Most Important Decision

~~~text
Simple integrity rule?
→ CONSTRAINT

Reusable database result/logic?
→ FUNCTION

Explicit stored operation?
→ PROCEDURE

Automatic DB event reaction?
→ TRIGGER

External email/payment/API workflow?
→ APPLICATION / JOB / EVENT SYSTEM
~~~

---

## Key Takeaway

> **Functions provide reusable database logic and results, procedures provide explicitly called database operations, and triggers automatically react to database events. Use triggers carefully because their automatic behavior can make important logic less visible.**

---

[← Previous: Lesson 18 — Views & Materialized Views](./18-views-and-materialized-views.md) | [Back to Roadmap](../README.md) | [Next: Lesson 20 — Transactions →](../05-transactions-concurrency/20-transactions.md)
