# Lesson 28 — MVCC (Multi-Version Concurrency Control)

## Why Does PostgreSQL Need MVCC?

Imagine two users access the same product at the same time.

~~~text
Transaction A
→ reading product stock

Transaction B
→ updating product stock
~~~

A simple database design could force readers and writers to block each other frequently.

PostgreSQL instead uses **MVCC — Multi-Version Concurrency Control**.

The core idea is:

> PostgreSQL can keep multiple versions of a row so different transactions can see the row version that is valid for their own snapshot.

This greatly improves concurrency.

---

# 1. What Does MVCC Mean?

MVCC stands for:

~~~text
Multi
Version
Concurrency
Control
~~~

### Multi-Version

More than one version of a logical row can temporarily exist.

### Concurrency Control

PostgreSQL controls what concurrent transactions are allowed to see and modify.

High-level mental model:

~~~text
Logical row
   │
   ├── older version
   └── newer version

Different transactions
may see different versions
depending on visibility rules
~~~

---

# 2. The Main Benefit

One of PostgreSQL MVCC's major benefits is that ordinary reads generally do not block ordinary writes, and ordinary writes generally do not block ordinary reads.

Conceptually:

~~~text
Transaction A
SELECT row
    │
    │ can continue reading its visible version
    │
Transaction B
UPDATE row
~~~

This does **not** mean PostgreSQL has no locks.

Writers can still conflict with other writers, explicit locks can block, and schema-level operations can block other work.

MVCC and locking work together.

---

# 3. PostgreSQL Does Not Usually Update a Row In Place

Suppose the table contains:

~~~text
id | name   | stock
10 | Laptop | 5
~~~

Then you execute:

~~~sql
UPDATE products
SET stock = 4
WHERE id = 10;
~~~

Conceptually, PostgreSQL creates a new row version.

~~~text
Old version
id=10, stock=5

New version
id=10, stock=4
~~~

The old version does not necessarily disappear immediately.

It may still be needed by another transaction whose snapshot should see the old state.

---

# 4. Row Versions Are Called Tuples

In PostgreSQL internals, a stored row version is commonly called a **tuple**.

So:

~~~text
Application language
row

PostgreSQL internals
tuple / row version
~~~

An UPDATE can therefore create a new tuple version while leaving the previous tuple version behind temporarily.

---

# 5. Transaction IDs

PostgreSQL transactions are associated with transaction IDs, often abbreviated as **XIDs**.

Conceptually:

~~~text
Transaction A → XID 100
Transaction B → XID 101
Transaction C → XID 102
~~~

PostgreSQL uses transaction-related metadata to determine which row versions are visible to which transactions.

You do not need to memorize the complete internal transaction-ID machinery yet.

---

# 6. xmin and xmax — Conceptual View

Heap tuples contain system metadata that includes transaction-related fields such as `xmin` and `xmax`.

At a simplified learning level:

~~~text
xmin
→ transaction that created this tuple version

xmax
→ transaction that deleted/invalidated this tuple version
  or marks related row-version state
~~~

Example conceptual row version:

~~~text
id = 10
stock = 5
xmin = 100
xmax = 105
~~~

This suggests the tuple version was created by transaction 100 and later made obsolete/deleted as part of transaction 105.

Important: `xmax` has internal uses and flags beyond the simplified explanation, so do not treat it as a simple application-level deleted_by field.

---

# 7. UPDATE Under MVCC

Suppose:

~~~text
Original row version
stock = 5
xmin = 100
~~~

Transaction 105 executes:

~~~sql
UPDATE products
SET stock = 4
WHERE id = 10;
~~~

Conceptually:

~~~text
OLD VERSION
stock = 5
xmin = 100
xmax = 105

NEW VERSION
stock = 4
xmin = 105
xmax = ...
~~~

PostgreSQL can now decide which version is visible to a transaction based on its snapshot and transaction state.

---

# 8. What Is a Snapshot?

A **snapshot** describes which transaction changes should be visible to a query/transaction at a point in its execution.

Think:

~~~text
Snapshot
   ↓
Which transactions are visible?
   ↓
Which tuple versions are visible?
   ↓
What rows should this query return?
~~~

Snapshots are a central part of MVCC.

---

# 9. Example: Reader and Writer

Initial value:

~~~text
stock = 5
~~~

Transaction A begins and obtains a snapshot that can see the old version.

Transaction B updates stock to 4 and commits.

Depending on Transaction A's isolation level and when its statement snapshot is taken, A may continue to see the version appropriate to its snapshot.

Conceptually:

~~~text
OLD VERSION              NEW VERSION
stock = 5                stock = 4
    ▲                         ▲
    │                         │
older snapshot           newer snapshot
may see this             may see this
~~~

This is how PostgreSQL can provide consistent views without making ordinary readers block writers.

---

# 10. MVCC and READ COMMITTED

PostgreSQL's default isolation level is **READ COMMITTED**.

At READ COMMITTED, each SQL statement gets a snapshot representing committed data as of the beginning of that statement.

Example:

~~~text
Transaction A
BEGIN;

SELECT stock;
→ sees 5

Transaction B
UPDATE stock = 4;
COMMIT;

Transaction A
SELECT stock again;
→ can see 4

COMMIT;
~~~

Why?

Because the second SELECT gets a new statement-level snapshot.

This connects directly to Lesson 21 on isolation levels.

---

# 11. MVCC and REPEATABLE READ

At PostgreSQL REPEATABLE READ, a transaction normally keeps the same transaction snapshot for its ordinary queries.

Example:

~~~text
Transaction A
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT stock;
→ sees 5

Transaction B
UPDATE stock = 4;
COMMIT;

Transaction A
SELECT stock again;
→ still sees 5

COMMIT;
~~~

Transaction A continues using a snapshot that predates B's committed change.

After A ends, a new transaction can see the newer committed version.

---

# 12. MVCC Does Not Mean No Locks

This is a very important interview point.

MVCC reduces reader/writer blocking, but PostgreSQL still uses locks.

Example:

~~~text
Transaction A
UPDATE product id=10
→ obtains row-level lock

Transaction B
UPDATE product id=10
→ may wait
~~~

Why?

Two writers cannot independently modify the same row at the same time without coordination.

So remember:

~~~text
MVCC
→ visibility + multiple versions

Locks
→ coordinate conflicting operations
~~~

---

# 13. Readers vs Writers

For an ordinary SELECT:

~~~sql
SELECT *
FROM products
WHERE id = 10;
~~~

and a concurrent UPDATE:

~~~sql
UPDATE products
SET stock = 4
WHERE id = 10;
~~~

the ordinary SELECT usually does not need to wait for the writer's row lock. It reads the version visible to its snapshot.

~~~text
Reader
   ↓
visible row version

Writer
   ↓
creates newer row version
~~~

This is one of MVCC's biggest practical advantages.

---

# 14. Writer vs Writer

Now consider:

~~~text
Transaction A
UPDATE products SET stock = 4 WHERE id = 10;

Transaction B
UPDATE products SET stock = 3 WHERE id = 10;
~~~

If A is still holding the conflicting row lock, B may have to wait.

~~~text
Writer A
holds row lock
     ↓
Writer B
waits
~~~

So MVCC does not remove write conflicts.

---

# 15. DELETE Under MVCC

Suppose:

~~~sql
DELETE FROM products
WHERE id = 10;
~~~

The tuple is not necessarily physically erased immediately from its heap page.

Conceptually:

~~~text
tuple exists
    ↓
DELETE marks it no longer visible to future applicable snapshots
    ↓
older transaction may still need old version
    ↓
later VACUUM can reclaim reusable space
~~~

This leads directly to the concept of **dead tuples**.

---

# 16. What Is a Dead Tuple?

A dead tuple is an old row version that is no longer needed by any active transaction and can eventually be reclaimed.

Updates can create dead tuples:

~~~text
version 1
   ↓ UPDATE
version 2

version 1 eventually becomes dead
~~~

Deletes can create reclaimable dead tuples as well.

These old versions explain why UPDATE/DELETE activity can increase table storage usage until maintenance occurs.

---

# 17. Why PostgreSQL Needs VACUUM

Because old tuple versions are not always removed immediately, PostgreSQL needs maintenance to reclaim/reuse space and maintain MVCC metadata.

~~~text
UPDATE / DELETE
      ↓
old tuple versions
      ↓
eventually no active snapshot needs them
      ↓
VACUUM
      ↓
space becomes reusable
~~~

Lesson 29 covers VACUUM and autovacuum deeply.

---

# 18. Long-Running Transactions Are Dangerous

Suppose Transaction A starts and remains open for a very long time.

~~~text
Transaction A snapshot
        │
        │ remains active
        ▼
old row versions may still be potentially visible
        │
        ▼
VACUUM cannot reclaim everything it otherwise could
~~~

Meanwhile thousands of UPDATEs happen.

Old versions can accumulate.

This can contribute to table/index bloat and maintenance pressure.

Practical rule:

> Do not leave transactions open unnecessarily.

---

# 19. MVCC and Autocommit

If you run individual SQL statements without an explicit `BEGIN`, PostgreSQL clients commonly operate in autocommit behavior where each statement is its own transaction.

~~~text
UPDATE ...;
→ transaction begins
→ statement executes
→ transaction commits
~~~

But if your backend explicitly does:

~~~sql
BEGIN;
~~~

and forgets to COMMIT or ROLLBACK, that transaction can remain open while the connection stays alive.

This is one reason transaction cleanup in backend code is critical.

---

# 20. Node.js Transaction Example

Correct pattern with `pg`:

~~~js
const client = await pool.connect();

try {
  await client.query('BEGIN');

  await client.query(
    'UPDATE products SET stock = stock - 1 WHERE id = $1',
    [productId]
  );

  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
~~~

Why is the `finally` block important?

Because the checked-out connection must be returned to the pool.

Why are COMMIT/ROLLBACK important?

Because leaving transactions open harms correctness and can interfere with MVCC cleanup.

---

# 21. MVCC and Visibility

PostgreSQL does not simply ask:

~~~text
What is the newest physical row version?
~~~

It asks something closer to:

~~~text
Which version of this logical row
is visible under this query's snapshot?
~~~

This distinction is fundamental.

Two transactions can legitimately see different committed states depending on their isolation level and snapshot timing.

---

# 22. Simplified Visibility Example

Imagine:

~~~text
Version A
stock = 5
created by XID 100

Version B
stock = 4
created by XID 110
~~~

A snapshot taken before transaction 110 became visible may see:

~~~text
Version A
~~~

A later snapshot may see:

~~~text
Version B
~~~

PostgreSQL's real visibility rules account for transaction status, snapshot boundaries, command IDs, and other internal details.

For full-stack interviews, focus on the snapshot + row-version model.

---

# 23. MVCC and SELECT FOR UPDATE

Ordinary SELECT:

~~~sql
SELECT stock
FROM products
WHERE id = 10;
~~~

reads according to MVCC visibility and does not normally lock the row against writers.

But:

~~~sql
SELECT stock
FROM products
WHERE id = 10
FOR UPDATE;
~~~

explicitly requests a row lock for a read-modify-write workflow.

~~~text
Normal SELECT
→ read visible version

SELECT FOR UPDATE
→ read row + coordinate conflicting writers
~~~

This connects MVCC with Lesson 22's locking concepts.

---

# 24. MVCC and Atomic UPDATE

For inventory, you may not need to SELECT and then UPDATE separately.

Instead:

~~~sql
UPDATE products
SET stock = stock - 1
WHERE id = $1
  AND stock > 0
RETURNING stock;
~~~

PostgreSQL coordinates concurrent writes and evaluates the update safely according to its concurrency rules.

This often avoids an application-level read-then-write race.

~~~text
Bad pattern
SELECT stock
→ decide in Node.js
→ UPDATE later

Better when suitable
single conditional UPDATE
→ database performs condition + write atomically
~~~

---

# 25. MVCC and Indexes

Indexes point to heap tuples/row versions.

Because MVCC can leave old tuple versions temporarily, index scans still need to respect tuple visibility.

This is why an index entry existing does not automatically mean the corresponding tuple version is visible to your transaction.

~~~text
Index entry
    ↓
candidate tuple
    ↓
MVCC visibility check
    ↓
visible? return it
not visible? ignore it
~~~

---

# 26. MVCC and Index-Only Scans

From Lesson 24: PostgreSQL can sometimes perform an Index Only Scan.

But MVCC visibility still needs to be confirmed.

PostgreSQL uses the **visibility map** to identify heap pages where all tuples are known to be visible to all relevant transactions.

~~~text
Index contains needed columns
        +
visibility map says heap page is all-visible
        ↓
heap visit may be avoided
~~~

This is why VACUUM can indirectly help index-only scans by maintaining visibility information.

---

# 27. HOT Updates — High-Level Bonus

PostgreSQL has an optimization called **HOT — Heap-Only Tuple** update.

Under suitable conditions, particularly when indexed columns are not changed and there is space on the same heap page, PostgreSQL can create a new row version without adding new index entries for every index.

High-level:

~~~text
UPDATE non-indexed column
        ↓
conditions allow HOT
        ↓
new tuple version on same heap page
        ↓
avoid unnecessary index-entry updates
~~~

You do not need HOT internals for most junior/mid full-stack interviews.

Remember only:

> PostgreSQL has optimizations to reduce index-maintenance cost for some updates.

---

# 28. MVCC vs Lock-Based Thinking

Without MVCC, you might imagine:

~~~text
Reader locks row
     ↓
Writer waits

Writer locks row
     ↓
Reader waits
~~~

PostgreSQL's MVCC model instead allows ordinary readers to work from visible versions while writers create new versions.

~~~text
Reader
→ old/visible version

Writer
→ new version
~~~

This improves concurrency significantly for mixed read/write workloads.

---

# 29. ShopHub Example

Initial product:

~~~text
id = 10
stock = 5
~~~

Customer A is viewing the product.

Customer B completes an operation that updates stock:

~~~sql
UPDATE products
SET stock = stock - 1
WHERE id = 10
  AND stock > 0
RETURNING stock;
~~~

PostgreSQL creates a newer row version.

~~~text
Older tuple
stock = 5

New tuple
stock = 4
~~~

Customer A's query sees whichever version is valid for its statement/transaction snapshot.

Future statements under READ COMMITTED can see the committed stock=4 version.

This is MVCC in a real application context.

---

# 30. What MVCC Does NOT Solve

MVCC is powerful, but it does not automatically solve:

~~~text
lost business-level decisions
deadlocks
writer/writer conflicts
incorrect transaction boundaries
overselling caused by poor application logic
external API consistency
long-running transactions
~~~

You still need:

- correct transactions
- locks when appropriate
- atomic SQL operations
- suitable isolation levels
- retry handling where required
- good application design

---

# Common Mistakes

## 31. Thinking UPDATE Overwrites the Same Row In Place

PostgreSQL generally creates a new tuple version under MVCC.

## 32. Thinking Old Versions Disappear Immediately

They can remain until they are no longer needed and maintenance can reclaim/reuse their space.

## 33. Thinking MVCC Means No Locks

PostgreSQL still uses locks, especially for conflicting writes and explicit locking operations.

## 34. Thinking Every Transaction Always Sees the Latest Committed Value

Visibility depends on isolation level and snapshot timing.

## 35. Leaving Transactions Open

Long-running transactions can prevent old tuple versions from becoming reclaimable and can create operational problems.

## 36. Confusing Dead Tuple with Corrupt Data

Dead tuples are a normal consequence of PostgreSQL's MVCC design. VACUUM manages their reusable space.

## 37. Reading xmin/xmax as Simple Business Fields

They are internal transaction metadata with more nuanced semantics than simple created_by/deleted_by fields.

---

# Interview Revision

## What is MVCC?

Multi-Version Concurrency Control is PostgreSQL's mechanism for managing concurrent transactions by maintaining row versions and determining visibility through snapshots.

## Why does PostgreSQL use MVCC?

To provide transaction isolation and high concurrency while allowing ordinary readers and writers to proceed without unnecessarily blocking each other.

## Does UPDATE modify a row in place?

At a high level, PostgreSQL normally creates a new tuple version and leaves the old version until it can eventually be reclaimed.

## What is a tuple?

A PostgreSQL term commonly used for a stored row version.

## What is a snapshot?

A representation used to determine which transaction changes and tuple versions are visible to a query/transaction.

## What is xmin?

Conceptually, transaction metadata identifying the transaction that created a tuple version.

## What is xmax?

Conceptually, transaction metadata involved when a tuple version is deleted/invalidated or locked; its real internal semantics are more nuanced.

## Does MVCC eliminate locks?

No. Writers can still block conflicting writers, and PostgreSQL uses many kinds of locks.

## What is a dead tuple?

An old tuple version that is no longer needed by active transactions and can eventually have its space reclaimed/reused.

## Why is VACUUM necessary?

MVCC creates old row versions through UPDATE/DELETE. VACUUM makes space from obsolete versions reusable and performs other important maintenance.

## Why are long-running transactions problematic?

Their old snapshots can keep old row versions potentially visible, preventing PostgreSQL from reclaiming them as soon as it otherwise could.

## READ COMMITTED vs REPEATABLE READ snapshots?

READ COMMITTED generally gets a new snapshot for each statement; REPEATABLE READ normally uses a stable transaction snapshot for ordinary queries.

## What is HOT update?

An optimization that can avoid creating new index entries for certain updates when indexed columns are unchanged and other conditions are satisfied.

---

# Quick Revision

~~~text
MVCC
│
├── Multiple row versions
├── Transaction snapshots
├── Visibility rules
└── Better concurrency
~~~

### UPDATE

~~~text
Old tuple
   ↓ UPDATE
New tuple created

Old tuple
→ remains temporarily
→ later becomes dead
→ VACUUM can reclaim/reuse space
~~~

### Reader vs Writer

~~~text
Ordinary Reader
→ sees visible version
→ usually does not block writer

Writer
→ creates new version
~~~

### Writer vs Writer

~~~text
Writer A updates row
→ holds conflicting row lock

Writer B updates same row
→ may wait
~~~

### READ COMMITTED

~~~text
Statement 1 → snapshot A
Statement 2 → snapshot B
~~~

### REPEATABLE READ

~~~text
Transaction
   ↓
stable snapshot for ordinary queries
~~~

### MVCC → VACUUM Connection

~~~text
UPDATE / DELETE
      ↓
old tuple versions
      ↓
dead tuples
      ↓
VACUUM
      ↓
reusable space + visibility maintenance
~~~

---

## Key Takeaway

> **PostgreSQL MVCC allows concurrent transactions to work with different visible versions of rows instead of forcing ordinary readers and writers to block each other. UPDATE and DELETE leave old tuple versions behind, snapshots determine visibility, locks still coordinate conflicting writes, and VACUUM eventually maintains/reclaims obsolete versions.**

---

[← Previous: Lesson 27 — PostgreSQL Architecture](./27-postgresql-architecture.md) | [Back to Roadmap](../README.md) | [Next: Lesson 29 — VACUUM & Autovacuum →](./29-vacuum-and-autovacuum.md)