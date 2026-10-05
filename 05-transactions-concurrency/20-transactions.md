# Lesson 20 — Transactions

## First Understand the Problem

Imagine a ShopHub checkout requires three database operations:

1. Create the order.
2. Create the order items.
3. Reduce product stock.

What if this happens?

~~~text
Create order        ✅
Create order items  ✅
Reduce stock        ❌
~~~

Without a transaction, the database could contain an order even though inventory was never updated.

What we want is:

~~~text
EVERYTHING succeeds
        ↓
save all changes

OR

ANY important step fails
        ↓
undo all changes
~~~

That is the purpose of a **transaction**.

---

## 1. What Is a Transaction?

A transaction is a group of database operations treated as **one logical unit of work**.

~~~sql
BEGIN;

-- operation 1
-- operation 2
-- operation 3

COMMIT;
~~~

If the operation fails:

~~~sql
ROLLBACK;
~~~

Mental model:

~~~text
BEGIN
  ↓
operation 1
  ↓
operation 2
  ↓
operation 3
  ↓
all successful?
 /           \
yes           no
 ↓             ↓
COMMIT       ROLLBACK
 ↓             ↓
save          undo
~~~

---

## 2. BEGIN, COMMIT, and ROLLBACK

### BEGIN

Starts an explicit transaction.

~~~sql
BEGIN;
~~~

Think:

~~~text
BEGIN
→ these operations belong together
~~~

### COMMIT

Successfully finishes the transaction.

~~~sql
COMMIT;
~~~

### ROLLBACK

Aborts the transaction and undoes its uncommitted changes.

~~~sql
ROLLBACK;
~~~

Example:

~~~sql
BEGIN;

INSERT INTO orders (user_id, status)
VALUES (10, 'pending');

UPDATE products
SET stock = stock - 1
WHERE id = 20;

COMMIT;
~~~

If an important operation fails, the application should roll the transaction back.

---

## 3. When Do We Need a Transaction?

Transactions are useful when several database changes represent one business action.

Examples:

~~~text
E-commerce checkout
Bank transfer
Booking
Inventory reservation
Subscription update
Multi-table account creation
~~~

Ask:

> If the third operation fails, is it acceptable for the first two operations to remain saved?

If the answer is **no**, those operations probably belong in one transaction.

---

# ACID Properties

## 4. What Is ACID?

Transactions are commonly explained with four properties:

~~~text
A → Atomicity
C → Consistency
I → Isolation
D → Durability
~~~

These are fundamental interview concepts.

---

## 5. Atomicity

Atomicity means **all or nothing**.

~~~text
Create order
Create items
Reduce stock
      ↓
one step fails
      ↓
ROLLBACK
      ↓
transaction changes are undone
~~~

Memory:

~~~text
Atomicity
→ all or nothing
~~~

---

## 6. Consistency

Consistency means a successful transaction takes the database from one valid state to another valid state while enforced rules remain satisfied.

Rules can include:

~~~text
PRIMARY KEY
FOREIGN KEY
UNIQUE
CHECK
NOT NULL
business invariants
~~~

Example:

~~~sql
stock INTEGER CHECK (stock >= 0)
~~~

A transaction does not invent correct business rules. You still need correct logic and constraints.

Memory:

~~~text
Consistency
→ valid state → valid state
~~~

---

## 7. Isolation

Multiple transactions can run at the same time.

~~~text
Customer A buys last laptop
Customer B buys last laptop
~~~

Isolation describes how concurrent transactions observe and interact with each other.

~~~text
Transaction A ──┐
                ├── PostgreSQL concurrency control
Transaction B ──┘
~~~

Isolation levels are covered deeply in Lesson 21.

Memory:

~~~text
Isolation
→ controlled concurrent behavior
~~~

---

## 8. Durability

Once PostgreSQL successfully commits a transaction, its changes are intended to survive failures according to PostgreSQL's durability guarantees.

~~~text
COMMIT succeeds
      ↓
committed changes persist
~~~

PostgreSQL uses mechanisms such as WAL to support durability.

Memory:

~~~text
Durability
→ committed means persisted
~~~

---

## 9. ACID in One Diagram

~~~text
TRANSACTION
    │
    ├── Atomicity
    │   → all or nothing
    │
    ├── Consistency
    │   → valid state to valid state
    │
    ├── Isolation
    │   → concurrent behavior
    │
    └── Durability
        → committed changes persist
~~~

---

# Practical ShopHub Transaction

## 10. Order Creation Flow

A simplified checkout transaction:

~~~sql
BEGIN;

INSERT INTO orders (user_id, status)
VALUES (10, 'pending')
RETURNING id;

INSERT INTO order_items (
    order_id,
    product_id,
    quantity,
    price_at_order
)
VALUES (101, 20, 2, 5000);

UPDATE products
SET stock = stock - 2
WHERE id = 20
  AND stock >= 2;

COMMIT;
~~~

Business flow:

~~~text
Checkout
   ↓
BEGIN
   ↓
Create order
   ↓
Create order items
   ↓
Update inventory
   ↓
Everything valid?
 /            \
yes            no
 ↓              ↓
COMMIT        ROLLBACK
~~~

---

## 11. Safer Inventory Update

This condition prevents stock from being reduced below the requested quantity:

~~~sql
UPDATE products
SET stock = stock - 2
WHERE id = 20
  AND stock >= 2
RETURNING id, stock;
~~~

If no row is returned:

~~~text
Product does not exist
OR
Not enough stock
~~~

The application can then throw an error and roll back.

This combines:

~~~text
Transaction
+
atomic UPDATE
+
RETURNING
~~~

---

## 12. Autocommit

When individual SQL statements are executed without an explicit transaction block, clients normally operate in autocommit mode.

Conceptually, one statement behaves like:

~~~text
BEGIN
  ↓
statement
  ↓
COMMIT
~~~

Why use explicit BEGIN?

Because these:

~~~text
statement 1
statement 2
statement 3
~~~

must sometimes become **one transaction**, rather than three independent transactions.

---

# Savepoints

## 13. What Is a SAVEPOINT?

A savepoint is a checkpoint inside a transaction.

~~~sql
BEGIN;

INSERT INTO orders (...);

SAVEPOINT after_order;

INSERT INTO order_items (...);

ROLLBACK TO SAVEPOINT after_order;

COMMIT;
~~~

Mental model:

~~~text
BEGIN
  ↓
Operation A
  ↓
SAVEPOINT
  ↓
Operation B
  ↓
Operation C fails
  ↓
ROLLBACK TO SAVEPOINT
  ↓
keep earlier transaction work
~~~

---

## 14. RELEASE SAVEPOINT

When a savepoint is no longer required:

~~~sql
RELEASE SAVEPOINT after_order;
~~~

Memory:

~~~text
SAVEPOINT x
→ create checkpoint

ROLLBACK TO SAVEPOINT x
→ undo work after checkpoint

RELEASE SAVEPOINT x
→ remove checkpoint
~~~

Savepoints are useful when partial recovery inside a larger transaction genuinely makes sense.

---

# Node.js Transactions

## 15. The Most Important Node.js Rule

When using node-postgres:

> **Every query belonging to one transaction must use the same checked-out database client.**

A pool contains multiple connections:

~~~text
Pool
 ├── Connection A
 ├── Connection B
 ├── Connection C
 └── Connection D
~~~

A PostgreSQL transaction belongs to one database connection/session.

Therefore this is wrong conceptually:

~~~text
BEGIN  → Connection A
INSERT → Connection B
UPDATE → Connection C
COMMIT → Connection A
~~~

That is not one transaction.

---

## 16. Correct Node.js Transaction Pattern

~~~js
const client = await pool.connect();

try {
  await client.query("BEGIN");

  const orderResult = await client.query(
    "INSERT INTO orders (user_id, status) VALUES ($1, $2) RETURNING id",
    [userId, "pending"]
  );

  const orderId = orderResult.rows[0].id;

  await client.query(
    "INSERT INTO order_items (order_id, product_id, quantity, price_at_order) VALUES ($1, $2, $3, $4)",
    [orderId, productId, quantity, price]
  );

  const stockResult = await client.query(
    "UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1 RETURNING id, stock",
    [quantity, productId]
  );

  if (stockResult.rowCount === 0) {
    throw new Error("Insufficient stock");
  }

  await client.query("COMMIT");

  return orderId;
} catch (error) {
  await client.query("ROLLBACK");
  throw error;
} finally {
  client.release();
}
~~~

Flow:

~~~text
pool.connect()
      ↓
one client
      ↓
BEGIN
      ↓
all transaction queries
      ↓
COMMIT or ROLLBACK
      ↓
client.release()
~~~

---

## 17. Why pool.query() Is Wrong for a Multi-Statement Transaction

Do not write:

~~~js
await pool.query("BEGIN");
await pool.query("INSERT INTO orders ...");
await pool.query("UPDATE products ...");
await pool.query("COMMIT");
~~~

Each pool call may use a different connection.

Correct approach:

~~~text
pool.connect()
     ↓
get ONE client
     ↓
client.query("BEGIN")
     ↓
client.query(...)
     ↓
client.query(...)
     ↓
client.query("COMMIT")
     ↓
client.release()
~~~

This is a very important backend interview point.

---

## 18. Why finally Is Important

~~~js
finally {
  client.release();
}
~~~

The pool has a limited number of connections.

If checked-out clients are never released:

~~~text
requests take connections
        ↓
connections never return
        ↓
pool becomes exhausted
        ↓
requests wait or fail
~~~

Always release the client.

---

# Transaction Boundaries

## 19. What Belongs Inside the Transaction?

Database operations such as:

~~~text
Create order
Create order_items
Update inventory
~~~

may belong together.

But external operations are different:

~~~text
Call payment gateway
Send email
Call shipping API
~~~

A PostgreSQL transaction cannot roll back another company's service.

---

## 20. Do Not Hold a Transaction Open While Waiting on External APIs

Bad flow:

~~~text
BEGIN
  ↓
insert order
  ↓
call payment gateway
  ↓
wait on network
  ↓
update inventory
  ↓
COMMIT
~~~

While waiting, the transaction may:

- hold locks longer
- occupy a connection
- increase contention
- increase timeout/deadlock risk
- reduce scalability

Important rule:

> **Keep database transactions as short as reasonably possible.**

Real payment flows require additional patterns such as idempotency, retries, events, or outbox processing.

---

## 21. Database Transaction Is Not a Distributed Transaction

A PostgreSQL transaction can make PostgreSQL operations atomic.

It cannot automatically make all these systems atomic:

~~~text
PostgreSQL
+
Cashfree / Razorpay
+
Email provider
+
Shipping service
~~~

Example:

~~~text
PostgreSQL COMMIT succeeds
        ↓
Email sending fails
~~~

PostgreSQL cannot roll back the external email system.

Mental model:

~~~text
DB transaction
→ atomicity inside PostgreSQL

External systems
→ require separate coordination
~~~

---

## 22. Transactions and Errors

If a statement fails inside a transaction, the transaction can enter a failed state.

Normal backend flow:

~~~text
BEGIN
  ↓
query
  ↓
query fails
  ↓
catch
  ↓
ROLLBACK
  ↓
release client
~~~

Robust application code must handle rollback.

---

## 23. Transactions Do Not Replace Constraints

A transaction groups operations.

A constraint defines valid data.

~~~text
TRANSACTION
→ operations belong together

CONSTRAINT
→ data must obey rules
~~~

Example:

~~~sql
stock INTEGER NOT NULL CHECK (stock >= 0)
~~~

Use both together.

---

## 24. Transactions Do Not Automatically Solve Concurrency

A common misconception is:

~~~text
"I used BEGIN and COMMIT,
so race conditions are impossible."
~~~

That is false.

Two transactions can still run concurrently:

~~~text
Transaction A reads stock = 1
Transaction B reads stock = 1
~~~

Correct behavior may depend on:

- isolation level
- row locks
- atomic SQL
- constraints
- retry logic

That is why the next lessons are:

~~~text
Lesson 21 → Isolation Levels
Lesson 22 → Locks & Concurrency
~~~

Transactions are the foundation, not the complete concurrency solution.

---

## 25. ShopHub Production Mental Model

~~~text
Request
   ↓
Validate input
   ↓
Checkout DB client
   ↓
BEGIN
   ↓
Create order
   ↓
Create order_items
   ↓
Safely update inventory
   ↓
All DB operations successful?
     /             \
   yes              no
    ↓                ↓
 COMMIT           ROLLBACK
    \                /
     ↓              ↓
       release client
            ↓
         response
~~~

---

## 26. When Should You Use a Transaction?

Strong candidates include:

### Money movement

~~~text
debit account
+
credit account
~~~

### Order creation

~~~text
order
+
order_items
+
inventory
~~~

### Multi-table account creation

~~~text
user
+
profile
+
required settings
~~~

### Booking

~~~text
reserve resource
+
create booking
+
related database state
~~~

Simple rule:

> **If partial success would leave the database in an invalid or misleading state, consider a transaction.**

---

## Common Mistakes

### 1. Forgetting ROLLBACK

Handle transaction failure explicitly.

### 2. Using different pool connections

One transaction must stay on one PostgreSQL client/connection.

### 3. Keeping transactions open too long

Long transactions increase contention and connection usage.

### 4. Waiting on slow external APIs inside a transaction

Keep the database transaction focused and short.

### 5. Thinking transactions prevent every race condition

Isolation, locks, constraints, and atomic SQL still matter.

### 6. Forgetting client.release()

Always return checked-out connections to the pool.

### 7. Treating external APIs as part of PostgreSQL atomicity

PostgreSQL cannot roll back payment, email, or shipping services.

---

# Interview Revision

## What is a transaction?

A group of database operations treated as one logical unit of work.

## BEGIN?

Starts an explicit transaction.

## COMMIT?

Successfully completes the transaction.

## ROLLBACK?

Aborts the transaction and undoes its uncommitted changes.

## What is ACID?

~~~text
Atomicity
Consistency
Isolation
Durability
~~~

## Atomicity?

All-or-nothing behavior.

## Consistency?

A successful transaction moves the database between valid states while enforced rules remain satisfied.

## Isolation?

Controls how concurrent transactions observe and interact with each other.

## Durability?

Successfully committed changes persist according to the database's durability guarantees.

## What is a SAVEPOINT?

A named checkpoint inside a transaction that allows partial rollback.

## Why must Node.js transactions use one client?

Because a PostgreSQL transaction belongs to one connection/session.

## Why not use pool.query() for every transaction statement?

Because separate calls may execute on different pooled connections.

## Why should transactions be short?

Long transactions can hold resources and locks longer and increase contention.

## Does a database transaction include external APIs?

No. PostgreSQL atomicity applies to PostgreSQL work, not external services.

## Does BEGIN/COMMIT automatically prevent race conditions?

No. Isolation levels, locking, atomic SQL, constraints, and sometimes retries are still required.

---

# Quick Revision

~~~text
BEGIN
→ start transaction

COMMIT
→ save transaction

ROLLBACK
→ undo transaction

SAVEPOINT
→ internal checkpoint

ROLLBACK TO SAVEPOINT
→ partial rollback
~~~

### ACID

~~~text
A → Atomicity
    all or nothing

C → Consistency
    valid state → valid state

I → Isolation
    concurrent behavior

D → Durability
    committed changes persist
~~~

### Node.js Rule

~~~text
pool.connect()
      ↓
ONE client
      ↓
BEGIN
      ↓
all transaction queries
      ↓
COMMIT / ROLLBACK
      ↓
client.release()
~~~

### Most Important Production Rule

~~~text
Keep transactions:

SHORT
+
DATABASE-FOCUSED
+
ON ONE CONNECTION
~~~

---

## Key Takeaway

> **A transaction groups related database operations into one logical unit so they can succeed together or be rolled back together. In Node.js, every statement in the transaction must use the same checked-out PostgreSQL client, and the transaction should be kept as short as reasonably possible.**

---

[← Previous: Lesson 19 — Functions, Procedures & Triggers](../04-postgresql-features/19-functions-procedures-triggers.md) | [Back to Roadmap](../README.md) | [Next: Lesson 21 — Isolation Levels →](./21-isolation-levels.md)
