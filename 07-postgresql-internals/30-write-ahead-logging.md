# Lesson 30 — Write-Ahead Logging (WAL)

## What Is WAL?

WAL stands for **Write-Ahead Logging**.

It is one of PostgreSQL's core mechanisms for **durability, crash recovery, replication, and point-in-time recovery**.

The central rule is simple:

> **PostgreSQL writes the log describing a change to durable WAL storage before the corresponding changed data page is allowed to be written to durable database storage.**

That is why it is called **write-ahead** logging.

~~~text
Change data
    ↓
Generate WAL record
    ↓
Persist required WAL
    ↓
Changed table/index pages may be flushed later
~~~

---

# 1. Why Does PostgreSQL Need WAL?

Imagine an order is created.

~~~sql
INSERT INTO orders (user_id, total)
VALUES (10, 4999);
~~~

PostgreSQL changes database pages in memory.

What if the machine crashes before those changed pages are written to their final data files?

Without a recovery mechanism, committed work could be lost or database files could be left inconsistent.

WAL solves this by recording enough information about changes so PostgreSQL can recover after a crash.

---

# 2. The Write-Ahead Rule

The important ordering is:

~~~text
1. Modify page in memory
        ↓
2. Generate WAL record
        ↓
3. WAL must reach durable storage as required
        ↓
4. COMMIT can be acknowledged
        ↓
5. Dirty data page can be written later
~~~

The crucial relationship is:

~~~text
WAL first
Data page later
~~~

PostgreSQL does **not** need to synchronously flush every modified table page to its final file at each normal commit.

---

# 3. WAL and ACID Durability

Recall ACID:

~~~text
A → Atomicity
C → Consistency
I → Isolation
D → Durability
~~~

WAL is a major part of PostgreSQL's durability implementation.

After a normally durable commit is acknowledged, PostgreSQL can use WAL during crash recovery even if some changed data pages had not yet reached their final table/index files.

~~~text
COMMIT
   ↓
required WAL durable
   ↓
power failure
   ↓
restart PostgreSQL
   ↓
replay WAL as needed
   ↓
recover committed changes
~~~

---

# 4. WAL Records

PostgreSQL generates WAL records describing changes to database state.

At a high level, operations such as these can generate WAL:

~~~text
INSERT
UPDATE
DELETE
index changes
page changes
transaction commit/abort information
some schema/storage operations
~~~

WAL is an internal binary log format.

It is **not** a normal SQL audit log such as:

~~~text
UPDATE products SET stock = 4 WHERE id = 10
~~~

Do not think of WAL as a readable history of SQL statements.

---

# 5. WAL Buffers

From Lesson 27: PostgreSQL has shared-memory **WAL buffers**.

Conceptually:

~~~text
Backend process
      ↓
generates WAL record
      ↓
WAL buffers
      ↓
WAL flushed
      ↓
WAL files
~~~

WAL buffers reduce the need for every generated WAL record to immediately perform its own separate storage write.

---

# 6. WAL Files and Segments

PostgreSQL stores WAL in segment files under its WAL storage area, normally `pg_wal` inside the data directory.

Conceptually:

~~~text
pg_wal/
  ├── WAL segment A
  ├── WAL segment B
  ├── WAL segment C
  └── ...
~~~

In standard PostgreSQL installations, WAL segment size commonly defaults to **16 MB**, although it can be chosen differently when a cluster is initialized.

You normally should not manually edit these files.

---

# 7. WAL Sequence

WAL records have positions in the WAL stream.

These positions are represented using **LSNs — Log Sequence Numbers**.

Think:

~~~text
WAL stream
------------------------------------------------>
LSN A          LSN B          LSN C          LSN D
~~~

An LSN identifies a location in WAL.

This becomes important for:

- replication progress
- recovery
- backups
- monitoring

Example PostgreSQL function:

~~~sql
SELECT pg_current_wal_lsn();
~~~

---

# 8. What Happens During COMMIT?

Simplified default mental model:

~~~text
Transaction changes rows
        ↓
WAL records generated
        ↓
COMMIT record generated
        ↓
required WAL flushed to durable storage
        ↓
PostgreSQL reports COMMIT success
~~~

The dirty table/index pages themselves may still be in shared buffers.

They can be flushed later.

This separates **transaction durability** from immediately writing every changed data page to its final relation file.

---

# 9. Dirty Pages

A dirty page is a page in memory that has been changed but whose latest state has not yet been written to its durable table/index file.

~~~text
Shared Buffers

Page A → clean
Page B → dirty
Page C → clean
~~~

WAL makes it safe for dirty data pages to be written later, provided PostgreSQL obeys the write-ahead rule.

---

# 10. Crash Recovery

Suppose PostgreSQL crashes after COMMIT but before a changed data page has been flushed.

Before crash:

~~~text
WAL
→ committed change safely recorded

Data page
→ latest version still only in memory
~~~

After restart:

~~~text
PostgreSQL starts
      ↓
finds recovery starting point
      ↓
reads WAL
      ↓
replays required changes
      ↓
database returns to consistent state
~~~

This process is called **crash recovery**.

---

# 11. WAL Redo

WAL allows PostgreSQL to **redo** changes during recovery when necessary.

Conceptually:

~~~text
Data file missing a committed change
        +
WAL contains required record
        ↓
replay/redo change
        ↓
recover page state
~~~

This is why WAL must be persisted before the corresponding changed data page can safely reach durable storage.

---

# 12. Checkpoints

If PostgreSQL had to replay WAL from the beginning of the database's lifetime after every crash, recovery would be impractical.

PostgreSQL therefore performs **checkpoints**.

A checkpoint establishes a recovery reference point and coordinates writing dirty pages.

~~~text
WAL history
────────────────────────────────────>

          CHECKPOINT
              │
              ▼
Recovery can start from an appropriate
checkpoint-related location instead of
replaying unlimited historical WAL.
~~~

Checkpoints therefore help bound crash-recovery work.

---

# 13. Checkpoint Flow

High-level:

~~~text
Dirty pages in shared buffers
          ↓
checkpoint activity
          ↓
required dirty pages written
          ↓
checkpoint WAL/control information
          ↓
new recovery baseline
~~~

Exact checkpoint internals are more complex, but this is the useful conceptual model.

---

# 14. WAL vs Data Files

Do not confuse these.

~~~text
DATA FILES
→ current table/index page contents

WAL
→ sequential log records describing changes needed for recovery
~~~

Both are important.

PostgreSQL cannot replace normal database storage with WAL alone.

---

# 15. WAL Is Sequential

WAL is designed around largely sequential log writing.

Why is this useful?

Randomly flushing many changed table pages at every commit could be expensive.

Instead:

~~~text
many database changes
      ↓
append WAL sequentially
      ↓
acknowledge durable commit
      ↓
data pages flushed later
~~~

This is a major architectural benefit of write-ahead logging.

---

# 16. WAL and UPDATE

Suppose:

~~~sql
UPDATE products
SET stock = stock - 1
WHERE id = 10;
~~~

Simplified flow:

~~~text
Find visible tuple
      ↓
create/update tuple version under MVCC
      ↓
change relevant pages in memory
      ↓
generate WAL
      ↓
commit WAL durability
      ↓
data pages can be flushed later
~~~

This connects:

~~~text
MVCC
 +
WAL
 +
Transactions
~~~

into one system.

---

# 17. WAL and Transactions

Suppose a transaction performs:

~~~sql
BEGIN;

INSERT INTO orders (...);
UPDATE products SET stock = stock - 1 WHERE id = 10;

COMMIT;
~~~

WAL records are generated for the relevant changes.

If the transaction commits durably, recovery can use WAL to restore its effects after a crash.

If the transaction does not commit, PostgreSQL's recovery/visibility mechanisms ensure it is not treated as a successfully committed transaction.

---

# 18. WAL and Rollback

A common misconception is:

~~~text
ROLLBACK means PostgreSQL must immediately reverse every changed disk page
~~~

Under PostgreSQL's MVCC model, aborted transaction versions can simply remain invisible and later be cleaned up.

WAL and transaction-status information work with MVCC to maintain correctness.

Conceptually:

~~~text
Transaction creates row versions
        ↓
ROLLBACK
        ↓
those versions are not visible as committed data
        ↓
VACUUM can eventually clean obsolete versions
~~~

---

# 19. WAL and Replication

WAL is also fundamental to PostgreSQL physical replication.

Primary server:

~~~text
Application writes
      ↓
Primary PostgreSQL
      ↓
generates WAL
      ↓
WAL stream
      ↓
Standby PostgreSQL
      ↓
replays WAL
~~~

The standby can reproduce changes made on the primary.

This is the foundation of **streaming replication**, covered later in Lesson 40.

---

# 20. Primary and Standby

High-level architecture:

~~~text
              writes
Application ──────────→ Primary
                         │
                         │ WAL stream
                         ▼
                       Standby
~~~

The standby continually receives and replays WAL records.

Depending on replication configuration, the standby can be used for failover and/or read workloads.

---

# 21. WAL and Point-in-Time Recovery (PITR)

WAL can also be archived for recovery purposes.

Suppose:

~~~text
10:00 → base backup
10:10 → transactions
10:20 → transactions
10:30 → accidental DELETE
~~~

With an appropriate base backup and archived WAL, PostgreSQL can potentially recover to a chosen point before the mistake.

Conceptually:

~~~text
Base backup
    +
Archived WAL
    ↓
Replay changes
    ↓
Stop at desired recovery target
~~~

This is called **Point-in-Time Recovery (PITR)**.

Lesson 39 covers backup and restore in more depth.

---

# 22. WAL Archiving

When WAL archiving is configured, completed WAL segments can be copied to durable archive storage.

High-level:

~~~text
Primary PostgreSQL
      ↓
pg_wal
      ↓
archive process/command
      ↓
WAL archive storage
~~~

Archived WAL can support PITR and recovery workflows.

Do not store the only copy of critical backups on the same machine as the production database.

---

# 23. Is WAL a Backup?

**No — WAL by itself is not a complete normal backup strategy.**

For PITR you generally need:

~~~text
Base backup
    +
continuous WAL archive
~~~

Why?

WAL describes changes relative to database state. You need a valid starting database backup/state from which to replay those changes.

Interview answer:

> WAL supports backup/recovery strategies, but WAL alone should not be described as your database backup.

---

# 24. synchronous_commit

PostgreSQL has a setting called:

~~~text
synchronous_commit
~~~

It controls aspects of how a transaction waits for WAL-related durability/replication conditions before reporting success.

With normal durable settings, commit waits for the required local WAL flush.

Some configurations can trade stronger immediate durability guarantees for lower commit latency.

For interview purposes:

> Do not disable or weaken durability settings casually. Understand the business durability requirement first.

---

# 25. fsync

`fsync` is another important durability setting.

When enabled, PostgreSQL tries to ensure updates are physically written to durable storage in the required order.

Disabling it can improve apparent performance but risks unrecoverable corruption/data loss after OS or hardware failure.

Production rule:

> Do not disable `fsync` just to make benchmarks look faster.

---

# 26. full_page_writes

PostgreSQL may write full-page images to WAL after checkpoints when a page is first modified.

Why?

A system crash can occur while the operating system/storage is writing a database page, potentially leaving a partially written page.

Full-page images help PostgreSQL recover from this type of torn-page risk.

High-level:

~~~text
first page modification after checkpoint
        ↓
full-page image may be logged
        ↓
later recovery can restore safe page image
~~~

You do not need to memorize the detailed implementation for full-stack interviews.

---

# 27. WAL Volume

Heavy write workloads generate significant WAL.

Examples:

~~~text
bulk INSERTs
frequent UPDATEs
large DELETE workloads
index changes
table rewrites
~~~

More indexes can also increase WAL volume because writes may need to update multiple indexes.

~~~text
one row UPDATE
    ↓
heap change
 +
index changes if required
    ↓
additional WAL
~~~

This is another reason unnecessary indexes have a write cost.

---

# 28. WAL and Performance

WAL provides durability but also introduces I/O.

Commit-heavy workloads may be sensitive to WAL flush latency.

Conceptually:

~~~text
Transaction
   ↓
COMMIT
   ↓
WAL flush
   ↓
durability acknowledgement
~~~

Storage performance, transaction batching, connection behavior, and durability configuration can therefore affect write throughput.

Do not optimize WAL settings blindly. Durability is more important than superficial benchmark speed.

---

# 29. Group Commit — High-Level Bonus

When many transactions commit around the same time, PostgreSQL can often amortize WAL flush work across multiple transactions.

Conceptually:

~~~text
Transaction A COMMIT ─┐
Transaction B COMMIT ─┼→ shared WAL flush work
Transaction C COMMIT ─┘
~~~

This is called **group commit** behavior.

It helps improve throughput under concurrent commit workloads.

---

# 30. WAL and Connection Pooling

Connection pooling does not remove WAL.

Each transaction using pooled connections still participates in PostgreSQL's durability mechanisms.

~~~text
HTTP requests
      ↓
connection pool
      ↓
transactions
      ↓
WAL generation / commit durability
~~~

Pooling solves connection-management overhead; WAL solves durability/recovery concerns.

They address different problems.

---

# 31. WAL and VACUUM

WAL and VACUUM are related to different concerns.

~~~text
WAL
→ durability + recovery + replication

VACUUM
→ MVCC cleanup + reusable space + visibility maintenance + XID safety
~~~

Some VACUUM-related operations can themselves generate WAL, but the concepts should not be confused.

---

# 32. WAL and MVCC

MVCC and WAL solve different parts of database correctness.

~~~text
MVCC
→ Which row version can this transaction see?

WAL
→ Can database changes survive/recover from a crash?
~~~

Together with locks and transactions:

~~~text
Transactions
    │
    ├── MVCC → visibility/isolation
    ├── Locks → conflicting-operation coordination
    └── WAL   → durability/recovery
~~~

This is an excellent interview mental model.

---

# 33. ShopHub Checkout Example

Suppose ShopHub performs:

~~~sql
BEGIN;

INSERT INTO orders (user_id, total)
VALUES ($1, $2);

UPDATE products
SET stock = stock - 1
WHERE id = $3
  AND stock > 0;

COMMIT;
~~~

Internal high-level flow:

~~~text
Order INSERT + stock UPDATE
          ↓
MVCC creates/modifies row versions
          ↓
WAL records generated
          ↓
COMMIT requested
          ↓
required WAL becomes durable
          ↓
success returned to backend
          ↓
dirty table/index pages may flush later
~~~

If PostgreSQL crashes after the durable commit but before all changed pages are flushed, crash recovery can use WAL to restore the committed changes.

---

# 34. What Happens After a Crash?

High-level restart sequence:

~~~text
PostgreSQL restarts
      ↓
detects previous shutdown was not clean
      ↓
starts crash recovery
      ↓
uses checkpoint information
      ↓
replays required WAL records
      ↓
reaches consistent state
      ↓
database becomes available
~~~

This happens automatically as part of PostgreSQL recovery.

---

# 35. WAL Monitoring — Useful Functions

Examples you may encounter:

~~~sql
SELECT pg_current_wal_lsn();
~~~

and functions/statistics for replication and WAL positions.

For a full-stack developer, understand the meaning of an LSN rather than memorizing every monitoring function.

~~~text
LSN
→ position in PostgreSQL WAL stream
~~~

---

# Common Mistakes

## 36. Thinking WAL Stores SQL Statements

WAL is an internal binary change log, not a normal SQL query history.

## 37. Thinking WAL Is the Same as Table Data

WAL records changes for recovery; table/index files store database pages.

## 38. Thinking WAL Alone Is a Backup

A proper PITR strategy needs a base backup plus the required WAL archive/stream.

## 39. Thinking COMMIT Requires Every Data Page to Be Written Immediately

PostgreSQL can acknowledge a durable commit after the required WAL is durable while dirty data pages are flushed later.

## 40. Disabling fsync for Production Performance

This can destroy durability guarantees and risk corruption/data loss after failure.

## 41. Thinking WAL and VACUUM Are the Same

WAL handles durability/recovery; VACUUM handles MVCC maintenance and transaction-age safety.

## 42. Deleting WAL Files Manually

Never manually delete files from `pg_wal` to solve disk-space problems. Doing so can make the database unrecoverable.

Investigate why WAL is being retained and fix the underlying replication/archive/configuration issue.

---

# Interview Revision

## What is WAL?

WAL stands for Write-Ahead Logging. PostgreSQL records changes in WAL before corresponding changed data pages are allowed to reach durable storage.

## Why is WAL needed?

For durability and crash recovery, and as a foundation for features such as physical replication and point-in-time recovery.

## What does write-ahead mean?

The relevant WAL record must be safely written before the associated changed data page can be safely persisted.

## Does COMMIT write every changed table page to disk immediately?

No. With normal durable settings, PostgreSQL ensures the required WAL is durable; dirty data pages can be flushed later.

## What is crash recovery?

After an unclean shutdown, PostgreSQL replays required WAL from an appropriate recovery point to restore a consistent database state.

## What is a checkpoint?

A recovery reference point that coordinates writing dirty pages and helps bound the amount of WAL that must be replayed after a crash.

## What is an LSN?

A Log Sequence Number identifying a position in the WAL stream.

## How does WAL support replication?

A standby can receive WAL generated by the primary and replay it to reproduce the primary's changes.

## How does WAL support PITR?

A base backup can be restored and archived WAL replayed until a desired recovery target.

## Is WAL itself a backup?

No. WAL is part of recovery/backup strategies; PITR generally requires a base backup plus the necessary WAL.

## WAL vs MVCC?

MVCC manages row-version visibility and concurrency; WAL provides durability and recovery.

## WAL vs VACUUM?

WAL records changes for durability/recovery; VACUUM maintains obsolete MVCC tuples, visibility metadata, and transaction-ID safety.

---

# Quick Revision

~~~text
WRITE-AHEAD LOGGING
        ↓
Change database page in memory
        ↓
Generate WAL
        ↓
Persist required WAL
        ↓
COMMIT acknowledged
        ↓
Data page may flush later
~~~

### Crash Recovery

~~~text
Crash
 ↓
Restart
 ↓
Checkpoint
 ↓
Replay WAL
 ↓
Consistent database
~~~

### Replication

~~~text
Primary
   ↓ WAL stream
Standby
   ↓
Replay
~~~

### PITR

~~~text
Base Backup
     +
Archived WAL
     ↓
Replay
     ↓
Desired point in time
~~~

### Core Differences

~~~text
MVCC
→ concurrency + visibility

Locks
→ conflicting-operation coordination

VACUUM
→ obsolete tuple maintenance

WAL
→ durability + recovery + replication
~~~

### Golden Rule

~~~text
WAL FIRST
DATA PAGE LATER
~~~

---

# Section 7 Summary — PostgreSQL Internals

You have now connected four major PostgreSQL internals concepts:

~~~text
PostgreSQL Architecture
        ↓
backend processes + shared memory + storage
        ↓
MVCC
        ↓
multiple row versions + snapshots
        ↓
VACUUM
        ↓
clean/reuse obsolete tuple space
        ↓
WAL
        ↓
durability + crash recovery
~~~

Together:

~~~text
Client Request
     ↓
Backend Process
     ↓
Transaction
 ┌────┼───────────┐
 ↓    ↓           ↓
MVCC Locks       WAL
 ↓    ↓           ↓
visibility       durability
     coordination
     ↓
VACUUM later maintains obsolete versions
~~~

This is the core PostgreSQL internal architecture a full-stack developer should understand.

---

## Key Takeaway

> **WAL is PostgreSQL's write-ahead durability mechanism: changes are logged before corresponding data pages are persisted. This allows PostgreSQL to acknowledge commits without immediately flushing every changed table page, recover committed work after crashes, stream changes to standby servers, and support point-in-time recovery when combined with proper backups.**

---

[← Previous: Lesson 29 — VACUUM & Autovacuum](./29-vacuum-and-autovacuum.md) | [Back to Roadmap](../README.md) | [Next: Lesson 31 — PostgreSQL + Node.js →](../08-backend-integration/31-postgresql-nodejs.md)