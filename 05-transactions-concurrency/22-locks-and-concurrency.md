# Lesson 22 — Locks & Concurrency

## First Understand the Problem

In Lesson 21, you learned how isolation levels control what concurrent transactions can see.

Now consider a different question:

> What happens when two transactions try to modify the same database resource at the same time?

Example:

~~~text
Product stock = 1

Customer A
tries to buy it

Customer B
tries to buy it
at almost the same time
~~~

PostgreSQL needs a way to coordinate these operations safely.

One of the main mechanisms is **locking**.

---

## 1. What Is a Lock?

A lock coordinates concurrent access to database resources.

Simple mental model:

~~~text
Transaction A
    ↓
locks a row
    ↓
changes the row

Transaction B
    ↓
tries conflicting operation
    ↓
must wait / fail / skip
depending on query
~~~

Locks help PostgreSQL prevent incompatible operations from corrupting concurrent work.

---

## 2. Isolation vs Locks

These concepts are related but different.

~~~text
ISOLATION LEVEL
→ controls transaction visibility and concurrency guarantees

LOCK
→ controls conflicting access to database resources
~~~

Think:

~~~text
Transaction
   │
   ├── Isolation
   │     What can I see?
   │
   └── Locks
         What can concurrently modify/access this resource?
~~~

You often need to understand both.

---

# PostgreSQL Automatically Uses Locks

## 3. You Already Use Locks

You do not have to write `LOCK` manually for every operation.

PostgreSQL automatically acquires locks when required.

For example:

~~~sql
UPDATE products
SET stock = stock - 1
WHERE id = 10;
~~~

PostgreSQL obtains the necessary row-level lock while modifying the row.

If another transaction tries a conflicting update on the same row, it may need to wait.

~~~text
Transaction A
UPDATE product 10
      ↓
row locked

Transaction B
UPDATE product 10
      ↓
waits

Transaction A COMMIT
      ↓
Transaction B can continue
~~~

This automatic locking is fundamental to safe concurrent writes.

---

# Row-Level Locks

## 4. Why Row Locks Matter

Imagine a table with one million products.

Transaction A updates:

~~~text
product id = 10
~~~

Ideally, this should not prevent unrelated updates to:

~~~text
product id = 999
~~~

Row-level locking allows PostgreSQL to coordinate access at a fine-grained level.

~~~text
products

id 10   🔒 Transaction A
id 11   available
id 12   available
id 999  available
~~~

This allows high concurrency.

---

## 5. SELECT FOR UPDATE

Sometimes your application needs to:

1. Read a row.
2. Make a business decision.
3. Update that row.

A normal SELECT does not reserve the row against conflicting writers.

PostgreSQL provides:

~~~sql
SELECT ...
FOR UPDATE;
~~~

Example:

~~~sql
BEGIN;

SELECT id, stock
FROM products
WHERE id = 10
FOR UPDATE;

-- business logic

UPDATE products
SET stock = stock - 1
WHERE id = 10;

COMMIT;
~~~

Mental model:

~~~text
SELECT ... FOR UPDATE
        ↓
find row
        ↓
acquire row lock
        ↓
other conflicting writers wait
        ↓
current transaction updates
        ↓
COMMIT / ROLLBACK
        ↓
lock released
~~~

---

## 6. Normal SELECT vs SELECT FOR UPDATE

Normal SELECT:

~~~sql
SELECT *
FROM products
WHERE id = 10;
~~~

usually does not block another transaction from updating that row.

~~~text
Transaction A
normal SELECT

Transaction B
UPDATE same row
→ can proceed according to MVCC/locking rules
~~~

With:

~~~sql
SELECT *
FROM products
WHERE id = 10
FOR UPDATE;
~~~

Transaction A explicitly acquires a strong row lock for later modification.

~~~text
Transaction A
SELECT ... FOR UPDATE
        ↓
row locked

Transaction B
conflicting UPDATE
        ↓
waits
~~~

Important:

> `FOR UPDATE` does not mean ordinary SELECT queries can no longer read the row.

PostgreSQL's MVCC allows ordinary readers to continue reading an appropriate visible row version.

---

# ShopHub Inventory Example

## 7. Locking the Last Product

Suppose:

~~~text
stock = 1
~~~

Transaction A:

~~~sql
BEGIN;

SELECT stock
FROM products
WHERE id = 10
FOR UPDATE;
~~~

It sees:

~~~text
stock = 1
~~~

Transaction B runs the same:

~~~sql
BEGIN;

SELECT stock
FROM products
WHERE id = 10
FOR UPDATE;
~~~

Transaction B must wait because Transaction A holds the conflicting row lock.

Transaction A:

~~~sql
UPDATE products
SET stock = stock - 1
WHERE id = 10;

COMMIT;
~~~

Now Transaction B continues and sees the current state according to the command/isolation semantics.

The application can detect that stock is no longer available.

Conceptually:

~~~text
A locks product
      ↓
A checks stock = 1
      ↓
B tries to lock
      ↓
B waits
      ↓
A updates stock = 0
      ↓
A COMMIT
      ↓
B continues
      ↓
B sees no available stock
~~~

---

## 8. Do You Always Need FOR UPDATE for Inventory?

No.

This is very important.

For a simple inventory decrement, an atomic conditional UPDATE can often be cleaner:

~~~sql
UPDATE products
SET stock = stock - $1
WHERE id = $2
  AND stock >= $1
RETURNING id, stock;
~~~

PostgreSQL performs the condition check and update as part of the write operation.

Then:

~~~text
row returned
→ stock successfully reduced

no row returned
→ insufficient stock or product missing
~~~

Compare:

~~~text
Simple conditional state change
→ atomic UPDATE may be enough

Need to read row
→ make several decisions
→ perform multiple dependent operations
→ FOR UPDATE may be appropriate
~~~

Do not add explicit locks when one safe SQL statement can express the operation.

---

# Other Row Lock Modes

## 9. PostgreSQL Row Lock Modes

PostgreSQL provides several row-level locking clauses:

~~~text
FOR UPDATE
FOR NO KEY UPDATE
FOR SHARE
FOR KEY SHARE
~~~

You do not need to memorize every conflict rule immediately.

High-level understanding:

### FOR UPDATE

Strong row lock used when you intend to modify the row in ways that conflict with other writers/lockers.

### FOR NO KEY UPDATE

Slightly weaker than FOR UPDATE and used by some updates that do not modify key values involved in certain relationships.

### FOR SHARE

Allows compatible shared locking while preventing certain conflicting modifications.

### FOR KEY SHARE

A weaker shared row lock particularly related to protecting key values.

For interviews at your current level, focus primarily on:

~~~text
FOR UPDATE
→ lock rows you intend to work with/update
~~~

Know that PostgreSQL provides finer-grained alternatives.

---

# Waiting Behavior

## 10. What Happens When a Row Is Already Locked?

By default, a transaction requesting a conflicting lock normally waits.

~~~text
Transaction A
holds lock
    ↓

Transaction B
requests conflicting lock
    ↓
WAIT
    ↓
A COMMIT / ROLLBACK
    ↓
B continues
~~~

Waiting is not automatically an error.

It is normal concurrency behavior.

---

## 11. Lock Wait vs Deadlock

These are different.

### Lock wait

~~~text
A holds resource

B waits for A

A eventually finishes

B continues
~~~

This is normal.

### Deadlock

~~~text
A waits for B
AND
B waits for A
~~~

Neither can continue without intervention.

PostgreSQL detects deadlocks and aborts one transaction.

---

# NOWAIT

## 12. FOR UPDATE NOWAIT

Sometimes you do not want to wait for a locked row.

Use:

~~~sql
SELECT *
FROM products
WHERE id = 10
FOR UPDATE NOWAIT;
~~~

Behavior:

~~~text
row free
→ lock it immediately

row already locked incompatibly
→ return error immediately
~~~

Mental model:

~~~text
NOWAIT
→ "Give me the lock now or fail."
~~~

This can be useful when the application would rather return a quick response or retry later than wait.

---

# SKIP LOCKED

## 13. FOR UPDATE SKIP LOCKED

Sometimes you want to ignore rows already being processed by another worker.

Use:

~~~sql
SELECT *
FROM jobs
WHERE status = 'pending'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;
~~~

Mental model:

~~~text
Job 1 🔒 Worker A
Job 2 available
Job 3 available

Worker B uses SKIP LOCKED
        ↓
skip Job 1
        ↓
take Job 2
~~~

This is extremely useful for database-backed worker queues.

---

## 14. Worker Queue Example

Suppose:

~~~text
jobs

1 pending
2 pending
3 pending
4 pending
~~~

Worker A:

~~~sql
BEGIN;

SELECT id
FROM jobs
WHERE status = 'pending'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;
~~~

gets:

~~~text
job 1
~~~

Worker B runs at the same time and can get:

~~~text
job 2
~~~

Flow:

~~~text
Worker A ──→ Job 1 🔒
Worker B ──→ skips Job 1 ──→ Job 2 🔒
Worker C ──→ skips 1 & 2 ──→ Job 3 🔒
~~~

This allows multiple workers to process different jobs concurrently.

---

## 15. SKIP LOCKED Is Not for Every Query

Because locked rows are skipped, the returned rows do not necessarily represent a complete consistent view of all matching work.

Therefore:

~~~text
Worker queue
→ excellent use case

Normal customer-facing report
→ usually not appropriate
~~~

Use it when skipping currently claimed work is exactly the desired behavior.

---

# Table-Level Locks

## 16. PostgreSQL Also Has Table Locks

PostgreSQL uses table-level lock modes too.

Operations such as:

~~~text
SELECT
INSERT
UPDATE
DELETE
ALTER TABLE
CREATE INDEX
TRUNCATE
~~~

can acquire different table lock modes.

The important idea is not memorizing every mode yet.

Understand:

~~~text
Row lock
→ protects/coordin­ates individual rows

Table lock
→ coordinates operations involving a table as a database object
~~~

Some schema operations need stronger locks than normal DML.

---

## 17. Explicit LOCK TABLE

PostgreSQL allows:

~~~sql
LOCK TABLE products IN ACCESS EXCLUSIVE MODE;
~~~

But explicit strong table locks should be used carefully.

Why?

~~~text
strong table lock
      ↓
many concurrent operations may wait
      ↓
throughput drops
~~~

Most application CRUD should rely on PostgreSQL's normal row-level and automatically acquired locking rather than manually locking whole tables.

---

# Deadlocks

## 18. What Is a Deadlock?

A deadlock occurs when transactions wait on each other in a cycle.

Classic example:

~~~text
Transaction A locks Product 1

Transaction B locks Product 2

Transaction A wants Product 2
→ waits for B

Transaction B wants Product 1
→ waits for A
~~~

Diagram:

~~~text
Transaction A
holds Row 1
wants Row 2
    │
    ▼
waits for B

Transaction B
holds Row 2
wants Row 1
    │
    ▼
waits for A
~~~

Neither can proceed.

---

## 19. PostgreSQL Deadlock Detection

PostgreSQL does not let the deadlock wait forever.

It detects the cycle and aborts one transaction.

~~~text
A waits for B
B waits for A
      ↓
PostgreSQL detects deadlock
      ↓
one transaction aborted
      ↓
other can continue
~~~

The aborted application transaction may need to be retried.

---

## 20. How to Reduce Deadlocks

One of the most important techniques is:

> **Acquire locks in a consistent order.**

Bad:

~~~text
Transaction A
locks product 1
then product 2

Transaction B
locks product 2
then product 1
~~~

Better:

~~~text
Transaction A
locks product 1
then product 2

Transaction B
locks product 1
then product 2
~~~

Both follow the same order.

~~~text
Consistent lock order
        ↓
reduces circular waiting
        ↓
fewer deadlocks
~~~

---

## 21. Example: Multiple Product Checkout

Suppose an order contains:

~~~text
Product 10
Product 20
Product 30
~~~

Another order contains the same products.

If transactions lock them in arbitrary order, deadlock risk increases.

A strategy can be:

~~~text
Sort product IDs

10
20
30
 ↓
lock/update in this consistent order
~~~

This does not eliminate every possible deadlock, but consistent ordering is a major prevention technique.

---

# Lock Timeouts

## 22. lock_timeout

You may not want a query to wait indefinitely for a lock.

PostgreSQL supports:

~~~sql
SET lock_timeout = '2s';
~~~

Then a statement waiting too long for a lock can fail.

Mental model:

~~~text
request lock
    ↓
wait
    ↓
2 seconds exceeded
    ↓
error
~~~

The correct timeout depends on the application.

Do not choose arbitrary production values without measuring workload behavior.

---

## 23. deadlock_timeout

PostgreSQL also has a setting called:

~~~text
deadlock_timeout
~~~

It influences how long PostgreSQL waits before checking for a deadlock.

This is primarily a database configuration/operations concern.

Do not confuse:

~~~text
lock_timeout
→ how long your statement is willing to wait for a lock

deadlock_timeout
→ when PostgreSQL performs deadlock checking
~~~

---

# Observing Locks

## 24. pg_locks

PostgreSQL exposes lock information through:

~~~text
pg_locks
~~~

Example:

~~~sql
SELECT *
FROM pg_locks;
~~~

In production troubleshooting, lock information can help answer:

~~~text
Which session holds a lock?

Which session is waiting?

What resource is involved?
~~~

You normally combine lock information with PostgreSQL activity/session information for deeper investigation.

At this stage, remember:

~~~text
pg_locks
→ inspect current locking state
~~~

---

# Advisory Locks

## 25. What Is an Advisory Lock?

Sometimes the resource you want to coordinate is an **application concept**, not simply a database row.

Examples:

~~~text
Generate monthly invoice for customer 10

Run one scheduled report

Process one logical resource only once
~~~

PostgreSQL provides **advisory locks**.

They are application-defined locks identified by numeric keys.

Conceptually:

~~~text
Application resource
      ↓
choose advisory lock key
      ↓
PostgreSQL coordinates ownership
~~~

Unlike normal row locks, PostgreSQL does not automatically know what business object the key represents.

Your application defines that meaning.

---

## 26. Advisory Lock Example

One form is:

~~~sql
SELECT pg_advisory_lock(1001);
~~~

Release:

~~~sql
SELECT pg_advisory_unlock(1001);
~~~

There are also transaction-level advisory lock variants.

High-level use:

~~~text
Worker A
acquires logical lock 1001
       ↓
performs protected operation

Worker B
requests same lock
       ↓
waits / uses try-lock variant
~~~

Use advisory locks only when normal row/table constraints and locking do not naturally model the resource.

---

# Node.js Example

## 27. SELECT FOR UPDATE with node-postgres

~~~js
const client = await pool.connect();

try {
  await client.query("BEGIN");

  const result = await client.query(
    "SELECT id, stock FROM products WHERE id = $1 FOR UPDATE",
    [productId]
  );

  if (result.rowCount === 0) {
    throw new Error("Product not found");
  }

  const product = result.rows[0];

  if (product.stock < quantity) {
    throw new Error("Insufficient stock");
  }

  await client.query(
    "UPDATE products SET stock = stock - $1 WHERE id = $2",
    [quantity, productId]
  );

  await client.query("COMMIT");
} catch (error) {
  await client.query("ROLLBACK");
  throw error;
} finally {
  client.release();
}
~~~

Again, every transaction query uses the **same client**.

---

## 28. Prefer Atomic UPDATE When the Logic Is Simple

The previous example is useful when you genuinely need to read and make multiple decisions.

But for a simple stock decrement, this may be better:

~~~js
const result = await client.query(
  `
    UPDATE products
    SET stock = stock - $1
    WHERE id = $2
      AND stock >= $1
    RETURNING id, stock
  `,
  [quantity, productId]
);

if (result.rowCount === 0) {
  throw new Error("Insufficient stock");
}
~~~

Why?

~~~text
One statement
    ↓
condition + modification together
    ↓
less read-then-write complexity
~~~

This is an important production design skill:

> Do not solve with explicit locking what can be expressed safely as one atomic SQL statement.

---

# Putting Transactions, Isolation, and Locks Together

## 29. Complete Mental Model

~~~text
TRANSACTION
→ groups related operations
→ BEGIN / COMMIT / ROLLBACK

ISOLATION LEVEL
→ controls what concurrent transactions can observe
→ READ COMMITTED / REPEATABLE READ / SERIALIZABLE

LOCK
→ coordinates conflicting access to resources
→ automatic locks / FOR UPDATE / etc.

ATOMIC SQL
→ performs safe state transition in one statement
~~~

These tools work together.

They are not replacements for one another.

---

## 30. ShopHub Decision Example

Requirement:

> Reduce inventory only when enough stock exists.

Start with:

~~~sql
UPDATE products
SET stock = stock - $1
WHERE id = $2
  AND stock >= $1
RETURNING id, stock;
~~~

If the business workflow instead requires:

~~~text
read product
↓
inspect several values
↓
make multiple dependent decisions
↓
write several related rows
~~~

then you may need:

~~~text
Transaction
+
SELECT ... FOR UPDATE
+
other DB operations
~~~

If the business invariant spans complex concurrent transactions, isolation level may also become important.

---

# Common Mistakes

## 31. Thinking Every SELECT Needs FOR UPDATE

No.

Use it when you actually need to lock rows for a concurrency-sensitive operation.

---

## 32. Thinking FOR UPDATE Blocks All Reads

Ordinary PostgreSQL SELECTs can generally continue reading appropriate visible row versions because of MVCC.

---

## 33. Holding Locks While Calling External APIs

Bad:

~~~text
BEGIN
 ↓
FOR UPDATE
 ↓
call payment API
 ↓
wait...
 ↓
COMMIT
~~~

You may hold locks and a database connection during network delay.

Keep transactions and lock durations short.

---

## 34. Locking Rows in Random Order

This increases deadlock risk when transactions need multiple rows.

Use consistent ordering where possible.

---

## 35. Treating Lock Wait as a Deadlock

A normal lock wait can resolve when the lock holder commits.

A deadlock is a circular dependency.

---

## 36. Assuming Deadlocks Mean PostgreSQL Is Broken

Deadlocks are a normal possibility in concurrent systems.

PostgreSQL detects them and aborts one participant.

Applications should handle retryable failures where appropriate.

---

## 37. Using SKIP LOCKED for Normal Reports

SKIP LOCKED intentionally ignores locked rows.

That is excellent for worker queues but usually wrong for queries requiring a complete result set.

---

## 38. Explicitly Locking Whole Tables Without Need

Strong table locks can reduce concurrency dramatically.

Prefer the smallest appropriate locking strategy.

---

# Interview Revision

## What is a database lock?

A mechanism used to coordinate concurrent access to database resources and prevent incompatible operations from interfering incorrectly.

## Does PostgreSQL automatically use locks?

Yes. INSERT, UPDATE, DELETE, DDL, and many other operations automatically acquire required locks.

## What does SELECT FOR UPDATE do?

It selects rows and acquires row locks that conflict with other operations trying to modify/lock those rows incompatibly.

## Does FOR UPDATE block normal SELECT queries?

Normally no. PostgreSQL MVCC allows ordinary readers to read appropriate visible row versions.

## What is NOWAIT?

It tells PostgreSQL not to wait for a conflicting lock; fail immediately instead.

## What is SKIP LOCKED?

It skips rows that cannot immediately be locked, commonly useful for multi-worker job queues.

## Lock wait vs deadlock?

~~~text
Lock wait
→ B waits for A
→ A eventually finishes
→ B continues

Deadlock
→ A waits for B
→ B waits for A
→ circular dependency
~~~

## What does PostgreSQL do with a deadlock?

It detects the deadlock and aborts one transaction so the other can proceed.

## How can deadlocks be reduced?

Acquire required resources in a consistent order, keep transactions short, and avoid unnecessary locking.

## What is lock_timeout?

A limit on how long a statement waits to acquire a lock before failing.

## What is pg_locks?

A PostgreSQL system view exposing information about current locks.

## What is an advisory lock?

An application-defined PostgreSQL lock for coordinating logical resources that are not naturally represented by normal row/table locking.

## FOR UPDATE vs atomic UPDATE for inventory?

If the operation can be safely expressed as one conditional UPDATE, that is often simpler. FOR UPDATE is useful when the application must read, make multiple decisions, and then perform dependent operations.

---

# Quick Revision

~~~text
LOCK
→ coordinates concurrent access

ROW LOCK
→ fine-grained row coordination

FOR UPDATE
→ lock selected rows for update-style work

NOWAIT
→ lock immediately or fail

SKIP LOCKED
→ ignore already locked rows

DEADLOCK
→ circular waiting

pg_locks
→ inspect locking state

ADVISORY LOCK
→ application-defined logical lock
~~~

### Lock Wait

~~~text
A holds row
   ↓
B requests conflicting lock
   ↓
B waits
   ↓
A COMMIT
   ↓
B continues
~~~

### Deadlock

~~~text
A holds Row 1
A wants Row 2
      ↑
      │
B wants Row 1
B holds Row 2

→ circular wait
→ PostgreSQL aborts one transaction
~~~

### Worker Queue

~~~text
Worker A → Job 1 🔒

Worker B
FOR UPDATE SKIP LOCKED
       ↓
skips Job 1
       ↓
Job 2 🔒
~~~

### Inventory

~~~text
Simple stock decrement?
→ atomic conditional UPDATE

Complex read-decide-write workflow?
→ transaction + possibly FOR UPDATE
~~~

### Most Important Rule

~~~text
Keep transactions short
        +
lock only what you need
        +
acquire multiple locks consistently
        +
prefer atomic SQL when possible
~~~

---

## Key Takeaway

> **PostgreSQL locks coordinate concurrent access to data. Use atomic SQL for simple state changes, SELECT FOR UPDATE when a multi-step workflow truly needs to reserve rows, keep locks short, and reduce deadlocks by acquiring resources in a consistent order.**

---

[← Previous: Lesson 21 — Transaction Isolation Levels](./21-isolation-levels.md) | [Back to Roadmap](../README.md) | Next: Section 6 — Performance & Query Optimization
