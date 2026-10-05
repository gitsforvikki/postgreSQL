# Lesson 21 — Transaction Isolation Levels

## First Understand the Problem

In Lesson 20, you learned that a transaction groups several database operations together.

But transactions can run **at the same time**.

Imagine ShopHub has only one laptop left:

~~~text
stock = 1
~~~

Two customers try to buy it at nearly the same moment:

~~~text
Customer A                 Customer B
    │                          │
    ├── starts transaction     ├── starts transaction
    │                          │
    ├── reads stock = 1        ├── reads stock = 1
    │                          │
    └── tries to buy           └── tries to buy
~~~

Now we have an important question:

> What should one transaction be allowed to see while another transaction is running?

That is what **transaction isolation** controls.

---

## 1. What Is Isolation?

Isolation controls how concurrent transactions observe and interact with database changes.

~~~text
Transaction A ──┐
                ├── PostgreSQL
Transaction B ──┘
~~~

The database needs to balance two goals:

~~~text
Correctness
    +
Concurrency / performance
~~~

If every transaction behaved as though it were the only transaction in the system, reasoning would be easier—but concurrency could become more expensive.

Isolation levels define different guarantees.

---

## 2. PostgreSQL Isolation Levels

SQL defines four common isolation-level names:

1. READ UNCOMMITTED
2. READ COMMITTED
3. REPEATABLE READ
4. SERIALIZABLE

In PostgreSQL:

~~~text
READ UNCOMMITTED
→ behaves like READ COMMITTED

READ COMMITTED
→ PostgreSQL default

REPEATABLE READ
→ stronger stable-snapshot behavior

SERIALIZABLE
→ strongest isolation level
~~~

For PostgreSQL interviews, this detail is important:

> PostgreSQL accepts `READ UNCOMMITTED`, but internally it behaves as `READ COMMITTED`.

---

# Problems Isolation Levels Are Designed Around

Before comparing the levels, understand the concurrency problems.

---

## 3. Dirty Read

A dirty read means one transaction reads data written by another transaction **before that other transaction commits**.

Example:

~~~text
Transaction A                 Transaction B

BEGIN
UPDATE account
balance = 0
                              reads balance = 0
ROLLBACK
~~~

Transaction B saw a value that never became committed.

That is a dirty read.

~~~text
uncommitted change
      ↓
another transaction reads it
      ↓
DIRTY READ
~~~

### PostgreSQL behavior

PostgreSQL does **not** allow dirty reads, even if you request `READ UNCOMMITTED`.

---

## 4. Non-Repeatable Read

Suppose Transaction A reads the same row twice.

First read:

~~~text
price = 50000
~~~

Between the reads, Transaction B updates and commits:

~~~text
price = 45000
~~~

Transaction A reads again and sees:

~~~text
price = 45000
~~~

Diagram:

~~~text
Transaction A                    Transaction B

SELECT price
→ 50000

                                 UPDATE price = 45000
                                 COMMIT

SELECT price again
→ 45000
~~~

The same query for the same row returned a different committed value during Transaction A.

That is a **non-repeatable read**.

---

## 5. Phantom Read

A phantom involves the set of rows matching a condition changing.

Transaction A:

~~~sql
SELECT COUNT(*)
FROM orders
WHERE status = 'pending';
~~~

Result:

~~~text
10
~~~

Transaction B inserts another pending order and commits.

Transaction A executes the condition again and might see:

~~~text
11
~~~

The new matching row is the **phantom**.

Mental model:

~~~text
First query
→ 10 matching rows

Another transaction commits a matching row

Second query
→ 11 matching rows
~~~

---

## 6. Three Problems to Remember

~~~text
Dirty Read
→ saw another transaction's uncommitted data

Non-Repeatable Read
→ same row changed between reads

Phantom Read
→ matching row set changed between reads
~~~

Now isolation levels become easier to understand.

---

# READ COMMITTED

## 7. READ COMMITTED — PostgreSQL Default

PostgreSQL's default isolation level is:

~~~text
READ COMMITTED
~~~

Start explicitly:

~~~sql
BEGIN ISOLATION LEVEL READ COMMITTED;
~~~

or simply:

~~~sql
BEGIN;
~~~

when the session/default configuration has not been changed.

The key idea is:

> Each statement sees a snapshot of data committed before that statement begins, with PostgreSQL's normal visibility rules.

This means two SELECT statements in the same transaction can see different committed database states.

---

## 8. READ COMMITTED Example

Transaction A:

~~~sql
BEGIN;

SELECT price
FROM products
WHERE id = 1;
~~~

Result:

~~~text
50000
~~~

Transaction B:

~~~sql
UPDATE products
SET price = 45000
WHERE id = 1;

COMMIT;
~~~

Transaction A queries again:

~~~sql
SELECT price
FROM products
WHERE id = 1;
~~~

It can now see:

~~~text
45000
~~~

Why?

Because the second SELECT is a new statement and therefore can use a newer committed snapshot.

Mental model:

~~~text
READ COMMITTED

Statement 1
   ↓
Snapshot A

other transaction commits

Statement 2
   ↓
Snapshot B
~~~

---

## 9. What READ COMMITTED Prevents

Dirty reads are prevented.

But non-repeatable reads can occur because later statements may see newer committed data.

For ordinary application workloads, READ COMMITTED is a practical default.

Do not assume that "default" means "safe for every concurrency requirement."

---

# READ UNCOMMITTED

## 10. READ UNCOMMITTED in PostgreSQL

SQL has a level called:

~~~sql
BEGIN ISOLATION LEVEL READ UNCOMMITTED;
~~~

But PostgreSQL treats it like READ COMMITTED.

Therefore:

~~~text
PostgreSQL READ UNCOMMITTED
            ↓
behaves as
            ↓
READ COMMITTED
~~~

Why?

PostgreSQL's MVCC architecture does not expose dirty reads in the way the SQL standard's weakest level permits.

Interview answer:

> PostgreSQL supports the READ UNCOMMITTED name, but it behaves as READ COMMITTED, so dirty reads are still prevented.

---

# REPEATABLE READ

## 11. What Is REPEATABLE READ?

Start it:

~~~sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
~~~

The important mental model is:

> Once the transaction establishes its snapshot, ordinary reads continue to use a stable transaction-level snapshot rather than taking a fresh snapshot for each statement.

Example:

~~~text
Transaction A                    Transaction B

BEGIN REPEATABLE READ

SELECT price
→ 50000

                                 UPDATE price = 45000
                                 COMMIT

SELECT price again
→ still 50000
~~~

Transaction A keeps seeing the database according to its transaction snapshot.

---

## 12. READ COMMITTED vs REPEATABLE READ

This is one of the most important comparisons.

### READ COMMITTED

~~~text
Statement 1 → Snapshot A

another transaction commits

Statement 2 → Snapshot B
~~~

### REPEATABLE READ

~~~text
Transaction begins/establishes snapshot
            ↓
        Snapshot A

Statement 1 → Snapshot A

another transaction commits

Statement 2 → Snapshot A
~~~

Easy memory:

~~~text
READ COMMITTED
→ snapshot can change between statements

REPEATABLE READ
→ stable snapshot across transaction
~~~

---

## 13. PostgreSQL REPEATABLE READ and Phantom Reads

The SQL standard allows some phantom behavior at REPEATABLE READ.

PostgreSQL's implementation is stronger than the minimum standard requirement.

Because PostgreSQL REPEATABLE READ uses a stable transaction snapshot, ordinary repeated queries do not suddenly see rows committed later by another transaction.

Example:

~~~text
Transaction A:
pending orders = 10

Transaction B:
inserts pending order
COMMIT

Transaction A:
pending orders = 10
~~~

So for interview purposes:

> PostgreSQL REPEATABLE READ prevents ordinary non-repeatable reads and phantom reads through its stable snapshot behavior.

However, a stable snapshot does **not** mean every possible business-level serialization anomaly is impossible.

That is why SERIALIZABLE exists.

---

## 14. Snapshot Does Not Mean No Conflicts

Suppose two REPEATABLE READ transactions both make decisions based on their snapshots and later attempt conflicting writes.

PostgreSQL may reject one transaction rather than allow an inconsistent update pattern.

So:

~~~text
REPEATABLE READ
≠
"Every transaction always succeeds"
~~~

Higher isolation can trade some concurrency convenience for stronger correctness guarantees.

Your application must be prepared for transaction failures in concurrency-sensitive workflows.

---

# SERIALIZABLE

## 15. What Is SERIALIZABLE?

Start it:

~~~sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
~~~

SERIALIZABLE is PostgreSQL's strongest isolation level.

The goal is:

> Concurrent transactions should have an outcome equivalent to some valid serial (one-after-another) execution.

Imagine:

~~~text
Actual execution:

A ────────┐
          ├── overlap
B ────────┘

Required logical outcome:

A → B

or

B → A
~~~

PostgreSQL allows concurrency, but prevents the committed result from violating serializable behavior.

---

## 16. Serializable Does Not Mean PostgreSQL Literally Runs One Transaction at a Time

This is a common misunderstanding.

SERIALIZABLE does **not** simply mean:

~~~text
Transaction A finishes
then
Transaction B starts
~~~

PostgreSQL can still execute transactions concurrently.

It tracks dependencies and can detect when the concurrent execution would produce an unsafe serialization pattern.

When necessary, PostgreSQL aborts one transaction.

---

## 17. Serialization Failure

Suppose:

~~~text
Transaction A
reads data
makes decision

Transaction B
reads related data
makes conflicting decision
~~~

If PostgreSQL determines that both cannot safely commit while preserving serializable behavior, one may fail with a serialization error.

Conceptually:

~~~text
Transaction A ─┐
               ├── unsafe dependency pattern
Transaction B ─┘
        ↓
PostgreSQL aborts one
        ↓
application retries
~~~

This leads to an important production rule:

> Applications using SERIALIZABLE must be prepared to retry transactions that fail due to serialization conflicts.

---

## 18. Why Retry Is Correct

A serialization failure does not necessarily mean your SQL is invalid.

It can mean:

~~~text
"This transaction was valid,
but it could not safely commit
with the other concurrent transactions."
~~~

The application can retry the **entire transaction** using fresh database state.

Typical idea:

~~~text
try transaction
      ↓
serialization failure?
   /             \
 no               yes
 ↓                 ↓
done             retry
~~~

Retries should be bounded rather than infinite.

---

# Comparing the Levels

## 19. Main PostgreSQL Comparison

| Isolation level | PostgreSQL behavior |
|---|---|
| READ UNCOMMITTED | Behaves as READ COMMITTED |
| READ COMMITTED | Fresh committed snapshot per statement |
| REPEATABLE READ | Stable transaction snapshot |
| SERIALIZABLE | Strongest; prevents serialization anomalies by aborting conflicting transactions when needed |

The progression is:

~~~text
READ COMMITTED
      ↓
more stable view
      ↓
REPEATABLE READ
      ↓
strongest serializable behavior
      ↓
SERIALIZABLE
~~~

---

## 20. Problem Comparison

A useful PostgreSQL-oriented mental table:

| Problem | Read Committed | Repeatable Read | Serializable |
|---|---:|---:|---:|
| Dirty read | Prevented | Prevented | Prevented |
| Non-repeatable read | Possible | Prevented | Prevented |
| Ordinary phantom read | Possible | Prevented by PostgreSQL snapshot behavior | Prevented |
| Serialization anomaly | Possible | Possible | Prevented by abort/retry when necessary |

Remember that PostgreSQL's REPEATABLE READ is stronger than the minimum SQL-standard behavior commonly shown in generic database tables.

---

# ShopHub Example

## 21. Two Customers Buying the Last Product

Suppose:

~~~text
product stock = 1
~~~

A naive application does:

~~~text
1. SELECT stock
2. if stock > 0
3. UPDATE stock
4. create order
~~~

Two requests can overlap.

~~~text
Transaction A                  Transaction B

reads stock = 1               reads stock = 1

decides "available"            decides "available"

updates                        updates
~~~

Isolation level matters, but the best solution is not automatically:

> "Always use SERIALIZABLE."

Often the SQL operation itself should be designed safely.

For example:

~~~sql
UPDATE products
SET stock = stock - 1
WHERE id = $1
  AND stock > 0
RETURNING id, stock;
~~~

Then:

~~~text
row returned
→ stock was successfully claimed

no row returned
→ unavailable / insufficient stock
~~~

This is an **atomic conditional update**.

---

## 22. Isolation Level vs Atomic SQL

This distinction is extremely important.

~~~text
Isolation level
→ controls visibility/concurrent transaction guarantees

Atomic SQL
→ performs a state change safely in one database statement
~~~

For many inventory operations:

~~~sql
UPDATE ...
WHERE stock >= quantity
RETURNING ...
~~~

can be simpler and more robust than:

~~~text
SELECT stock
   ↓
application checks
   ↓
UPDATE later
~~~

Use isolation, locking, constraints, and SQL design together rather than relying on only one mechanism.

---

# Changing Isolation Level

## 23. Set Isolation When Starting the Transaction

Examples:

~~~sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
~~~

~~~sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
~~~

Another form is:

~~~sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
~~~

The transaction characteristics must be configured at the appropriate point before conflicting transactional work has made the change invalid.

For learning, prefer the clear form:

~~~sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
~~~

---

# Node.js Example

## 24. REPEATABLE READ with node-postgres

~~~js
const client = await pool.connect();

try {
  await client.query("BEGIN ISOLATION LEVEL REPEATABLE READ");

  const product = await client.query(
    "SELECT id, stock FROM products WHERE id = $1",
    [productId]
  );

  // other database work using the same client

  await client.query("COMMIT");
} catch (error) {
  await client.query("ROLLBACK");
  throw error;
} finally {
  client.release();
}
~~~

The same rule from Lesson 20 remains:

> Every query in the transaction must use the same checked-out client.

---

## 25. SERIALIZABLE Retry Concept in Node.js

High-level structure:

~~~js
async function runTransaction() {
  const client = await pool.connect();

  try {
    await client.query("BEGIN ISOLATION LEVEL SERIALIZABLE");

    // read + write operations

    await client.query("COMMIT");
  } catch (error) {
    await client.query("ROLLBACK");

    // if this is a serialization failure,
    // the caller may retry the whole transaction

    throw error;
  } finally {
    client.release();
  }
}
~~~

Do not retry only the last failed query.

Why?

Because the transaction's earlier reads and decisions may now be outdated.

Correct mental model:

~~~text
serialization failure
        ↓
ROLLBACK
        ↓
retry entire transaction
        ↓
fresh snapshot / fresh decisions
~~~

---

# Choosing an Isolation Level

## 26. Should You Always Use SERIALIZABLE?

No.

Stronger isolation can increase transaction aborts/retries under contention and may affect throughput.

Use the level required by the correctness needs of the operation.

General thinking:

~~~text
Normal CRUD
→ READ COMMITTED often sufficient

Need stable transaction-wide snapshot
→ REPEATABLE READ may fit

Need strongest protection from serialization anomalies
→ SERIALIZABLE
~~~

But the final choice depends on the actual business invariant and SQL pattern.

Do not choose isolation levels only from a generic table.

---

## 27. Isolation vs Locks

Isolation levels and locks are related but not identical.

~~~text
Isolation level
→ what concurrent transaction behavior/visibility is allowed

Locks
→ coordinate access to particular database resources
~~~

For example:

~~~sql
SELECT *
FROM products
WHERE id = 1
FOR UPDATE;
~~~

explicitly locks the selected row against conflicting row operations.

We cover this deeply in:

**Lesson 22 — Locks & Concurrency**

---

## 28. Isolation vs Transactions

A transaction is the container:

~~~text
BEGIN
...
COMMIT
~~~

Isolation is one property of how that transaction behaves relative to other transactions.

~~~text
TRANSACTION
    │
    ├── operations
    ├── commit/rollback
    └── isolation level
           ↓
      concurrency behavior
~~~

---

## 29. Common Mistakes

### 1. Thinking READ UNCOMMITTED Allows Dirty Reads in PostgreSQL

It does not. PostgreSQL treats it as READ COMMITTED.

### 2. Thinking READ COMMITTED Means One Snapshot for the Whole Transaction

In PostgreSQL, it generally uses a new committed snapshot for each statement.

### 3. Assuming Generic REPEATABLE READ Tables Exactly Describe PostgreSQL

PostgreSQL's REPEATABLE READ prevents ordinary phantom reads through stable snapshot behavior.

### 4. Thinking SERIALIZABLE Means Transactions Literally Run One by One

They can execute concurrently. PostgreSQL may abort one when required for serializable correctness.

### 5. Not Retrying Serialization Failures

Applications using SERIALIZABLE must be designed for retryable transaction failures.

### 6. Retrying Only the Last Statement

Retry the whole transaction because earlier reads/decisions may no longer be valid.

### 7. Using Higher Isolation Instead of Designing Good SQL

Atomic conditional updates, constraints, and locking can be just as important.

### 8. Thinking BEGIN Alone Solves Race Conditions

A transaction does not automatically remove concurrency anomalies.

---

# Interview Revision

## What is transaction isolation?

It controls how concurrent transactions observe and interact with each other's database changes.

## What is PostgreSQL's default isolation level?

`READ COMMITTED`.

## What happens with READ UNCOMMITTED in PostgreSQL?

It behaves as READ COMMITTED, so dirty reads are still prevented.

## What is a dirty read?

Reading another transaction's uncommitted data.

## What is a non-repeatable read?

Reading the same row twice in one transaction and seeing a different committed value because another transaction changed it.

## What is a phantom read?

Repeating a predicate query and seeing a changed set of matching rows because another transaction inserted/deleted matching data.

## READ COMMITTED mental model?

A new committed snapshot is generally taken for each statement.

## REPEATABLE READ mental model?

The transaction uses a stable snapshot, so repeated ordinary reads see a consistent transaction-level view.

## Does PostgreSQL REPEATABLE READ allow ordinary phantom reads?

No. PostgreSQL's implementation prevents ordinary phantom reads through its stable snapshot behavior.

## What is SERIALIZABLE?

The strongest PostgreSQL isolation level. It ensures committed concurrent execution is equivalent to some valid serial ordering, aborting transactions when necessary.

## Can SERIALIZABLE transactions fail?

Yes. PostgreSQL can raise serialization failures.

## What should the application do?

Rollback and retry the **entire transaction** when the error is retryable.

## Should every transaction use SERIALIZABLE?

No. Choose isolation based on correctness requirements and concurrency tradeoffs.

---

# Quick Revision

~~~text
READ UNCOMMITTED
→ PostgreSQL treats as READ COMMITTED

READ COMMITTED
→ default
→ new snapshot per statement
→ non-repeatable reads possible

REPEATABLE READ
→ stable transaction snapshot
→ prevents non-repeatable reads
→ PostgreSQL also prevents ordinary phantoms

SERIALIZABLE
→ strongest
→ equivalent to serial execution
→ may abort transaction
→ application retries
~~~

### Concurrency Problems

~~~text
Dirty Read
→ read uncommitted data

Non-Repeatable Read
→ same row, different value

Phantom Read
→ same condition, different row set
~~~

### ShopHub Inventory

~~~text
Avoid when possible:

SELECT stock
     ↓
app checks
     ↓
UPDATE later

Prefer atomic operation when suitable:

UPDATE products
SET stock = stock - quantity
WHERE stock >= quantity
RETURNING ...
~~~

### Most Important Mental Model

~~~text
Transaction
→ groups work

Isolation
→ controls concurrent visibility/behavior

Lock
→ coordinates access to resources

Atomic SQL
→ safely changes state in one statement
~~~

---

## Key Takeaway

> **Isolation levels control how concurrent PostgreSQL transactions see and affect one another. READ COMMITTED is the default, REPEATABLE READ provides a stable transaction snapshot, and SERIALIZABLE provides the strongest correctness guarantee but requires applications to handle retryable serialization failures.**

---

[← Previous: Lesson 20 — Transactions](./20-transactions.md) | [Back to Roadmap](../README.md) | [Next: Lesson 22 — Locks & Concurrency →](./22-locks-and-concurrency.md)
