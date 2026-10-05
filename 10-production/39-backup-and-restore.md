# Lesson 39 — PostgreSQL Backup & Restore

## Goal of This Lesson

A production database is not safe merely because it is running correctly today.

You need a recovery strategy for events such as:

~~~text
accidental DELETE / DROP
bad deployment or migration
database corruption
server/storage failure
cloud-region incident
human error
~~~

The core principle is:

> **A backup is useful only if you can successfully restore it.**

PostgreSQL backup strategies broadly include:

~~~text
Logical backup
→ pg_dump / pg_dumpall

Physical backup
→ copy/base backup of PostgreSQL cluster files

Continuous WAL archiving
→ enables Point-in-Time Recovery when combined with a suitable base backup
~~~

---

# 1. Backup vs High Availability

Do not confuse these concepts.

~~~text
Backup
→ recover historical data/state after loss or mistake

High Availability
→ keep service available when a server/component fails
~~~

A replica is not automatically a replacement for backups.

Example:

~~~text
Accidental DELETE
      ↓
primary database
      ↓ replication
replica also receives DELETE
~~~

A historical backup/PITR strategy can let you recover a state from before the mistake.

---

# 2. Logical Backup

A logical backup represents database objects and/or data in a logical form that PostgreSQL tools can recreate.

Typical tool:

~~~text
pg_dump
~~~

Conceptually:

~~~text
PostgreSQL database
      ↓
pg_dump
      ↓
logical backup file/archive
~~~

Logical backups are useful for:

~~~text
individual databases
selected schemas/tables
migration between compatible PostgreSQL environments
portable restore workflows
development/test copies
~~~

---

# 3. Physical Backup

A physical backup copies PostgreSQL's physical cluster representation rather than exporting SQL-level objects.

Conceptually:

~~~text
PostgreSQL cluster files
      ↓
physical/base backup
      ↓
recoverable cluster copy
~~~

Physical backup is especially important for:

~~~text
large databases
whole-cluster recovery
replication setup
Point-in-Time Recovery
~~~

Physical backups are more tightly coupled to PostgreSQL server/cluster compatibility than logical dumps.

---

# 4. pg_dump

`pg_dump` creates a logical backup of **one PostgreSQL database**.

Basic plain-text dump:

~~~bash
pg_dump -d shophub > shophub.sql
~~~

or with connection information:

~~~bash
pg_dump -h localhost -U postgres -d shophub > shophub.sql
~~~

The output can contain SQL statements needed to recreate database objects and data, depending on selected options.

---

# 5. Plain SQL Format

Example:

~~~bash
pg_dump -d shophub -f shophub.sql
~~~

Plain format is human-readable SQL.

Restore with `psql`:

~~~bash
psql -d shophub_restore -f shophub.sql
~~~

Mental model:

~~~text
pg_dump
   ↓
shophub.sql
   ↓
psql
   ↓
restored database
~~~

---

# 6. Custom Format

For production logical backups, PostgreSQL's custom archive format is very useful.

Create:

~~~bash
pg_dump -Fc -d shophub -f shophub.dump
~~~

`-F c` / `-Fc` means custom format.

Restore using:

~~~bash
pg_restore -d shophub_restore shophub.dump
~~~

Custom format provides more restore flexibility than a plain SQL script.

---

# 7. Why Custom Format Is Useful

Custom archive format supports capabilities such as:

~~~text
selective restore
reordering by pg_restore
parallel restore where supported
compressed archive behavior
object listing
~~~

Inspect archive contents:

~~~bash
pg_restore -l shophub.dump
~~~

This lets you see objects available in the archive.

---

# 8. Directory Format

PostgreSQL also supports directory-format dumps.

Example:

~~~bash
pg_dump -Fd -j 4 -d shophub -f shophub_backup
~~~

`-j 4` uses parallel dump jobs where supported by the format.

Directory format is useful for larger logical backups and parallel workflows.

Restore can also use parallel jobs:

~~~bash
pg_restore -j 4 -d shophub_restore shophub_backup
~~~

---

# 9. Tar Format

Another archive option is tar format:

~~~bash
pg_dump -Ft -d shophub -f shophub.tar
~~~

For interviews, you mainly need to remember:

~~~text
Plain
→ restore with psql

Custom / Directory / Tar archive
→ restore with pg_restore
~~~

Custom format is a common practical choice.

---

# 10. pg_dump Creates a Consistent Snapshot

`pg_dump` can back up a live database while other clients continue using it.

It produces a transactionally consistent view of the database as of its snapshot.

Conceptually:

~~~text
Application writes continue
       ↓
PostgreSQL MVCC
       ↓
pg_dump sees a consistent snapshot
~~~

You normally do not need to shut down the application just to run a logical `pg_dump`.

However, schema changes during a dump can create operational complications, so coordinate backup/migration workflows carefully.

---

# 11. pg_dump Does Not Block Normal Readers/Writers Like a Full Shutdown

`pg_dump` takes locks needed to protect its view of schema objects, but it does not require exclusive locking of the entire database for ordinary use.

Important nuance:

~~~text
backup can coexist with normal activity
but
some conflicting DDL can be blocked or cause issues
~~~

Plan large backups with production workload in mind.

---

# 12. Schema-Only Backup

To back up database definitions without table data:

~~~bash
pg_dump --schema-only -d shophub -f schema.sql
~~~

This can include objects such as:

~~~text
tables
constraints
indexes
sequences
functions
views
~~~

Useful for inspecting/recreating structure.

---

# 13. Data-Only Backup

To dump data without schema definitions:

~~~bash
pg_dump --data-only -d shophub -f data.sql
~~~

This is useful in specialized migration/restore workflows.

Be careful: the target schema must be compatible with the data being restored.

---

# 14. Backup a Specific Table

~~~bash
pg_dump -d shophub -t users -f users.sql
~~~

Or custom format:

~~~bash
pg_dump -Fc -d shophub -t users -f users.dump
~~~

This is useful when only one table needs logical export.

---

# 15. Backup a Schema

~~~bash
pg_dump -d shophub -n public -f public_schema.sql
~~~

`-n` selects a schema.

PostgreSQL logical tools can therefore back up at different scopes:

~~~text
database
schema
table
~~~

---

# 16. Excluding Objects

Sometimes large or disposable tables should be excluded from a logical backup.

Example:

~~~bash
pg_dump -d shophub --exclude-table=temporary_logs -f shophub.sql
~~~

Use exclusions carefully.

If the excluded data is actually required for recovery, your backup is incomplete.

---

# 17. pg_dumpall

`pg_dump` handles one database.

`pg_dumpall` can dump all databases in a PostgreSQL cluster into a SQL script and can also capture cluster-wide objects such as roles depending on options.

Example:

~~~bash
pg_dumpall -f cluster.sql
~~~

Restore:

~~~bash
psql -f cluster.sql postgres
~~~

For large production systems, a physical backup/PITR strategy may be more appropriate than relying only on `pg_dumpall`.

---

# 18. Roles Are Important

A database dump and cluster-wide role configuration are different concerns.

To dump global objects such as roles/tablespaces:

~~~bash
pg_dumpall --globals-only > globals.sql
~~~

This matters because restored database objects may reference owners or privileges associated with roles.

Production recovery planning should account for:

~~~text
database objects
data
roles
ownership
privileges
tablespaces where relevant
~~~

---

# 19. Restoring a Plain SQL Dump

Create a target database:

~~~bash
createdb shophub_restore
~~~

Then:

~~~bash
psql -d shophub_restore -f shophub.sql
~~~

Or using SQL:

~~~sql
CREATE DATABASE shophub_restore;
~~~

Then run the dump through `psql`.

---

# 20. Restoring a Custom Archive

~~~bash
createdb shophub_restore
pg_restore -d shophub_restore shophub.dump
~~~

Useful options include:

~~~text
--clean
--create
--jobs
--schema
--table
--no-owner
~~~

Understand what an option does before using it against production.

---

# 21. --clean

`pg_restore --clean` attempts to drop database objects before recreating them.

Example:

~~~bash
pg_restore --clean -d shophub_restore shophub.dump
~~~

This can be destructive if pointed at the wrong database.

Production rule:

> Verify the target database before using destructive restore options.

---

# 22. --create

A custom archive can be restored with database creation behavior:

~~~bash
pg_restore --create -d postgres shophub.dump
~~~

`-d postgres` here is the initial database connection used while `pg_restore` creates/connects to the archived database as required.

Exact ownership/privilege behavior depends on the archive and restore options.

---

# 23. --no-owner

When restoring into a different environment, original object owners may not exist.

Example:

~~~bash
pg_restore --no-owner -d shophub_restore shophub.dump
~~~

This avoids commands that try to restore original ownership.

This is often useful for:

~~~text
development environments
different managed providers
migration to a different role setup
~~~

Still ensure the restoring role has enough privileges to create the required objects.

---

# 24. Selective Restore

With archive formats you can restore selected objects.

Example:

~~~bash
pg_restore -t users -d shophub_restore shophub.dump
~~~

Or list archive contents:

~~~bash
pg_restore -l shophub.dump > restore.list
~~~

Then edit/use a list for controlled restore workflows.

This is one major advantage over a simple monolithic SQL script.

---

# 25. Parallel Restore

For suitable archive formats:

~~~bash
pg_restore -j 4 -d shophub_restore shophub.dump
~~~

Parallel restore can reduce recovery time for large databases.

But performance depends on:

~~~text
CPU
storage I/O
number/size of tables
indexes
constraints
target server capacity
~~~

More jobs are not always faster.

---

# 26. Logical Backup Version Compatibility

A good operational rule is to use `pg_dump` from the same or a newer PostgreSQL major version than the server being dumped, following PostgreSQL's documented compatibility rules.

`pg_dump` cannot dump from a server newer than its own major version.

Logical dumps are generally designed to restore into newer PostgreSQL versions, but compatibility should be tested, especially across large version jumps or extension differences.

Do not treat backup compatibility as an assumption—test the actual migration path.

---

# 27. Extensions

A database may depend on extensions:

~~~sql
CREATE EXTENSION ...;
~~~

A restore environment must support required extensions and compatible versions.

Examples might include:

~~~text
pgcrypto
uuid-ossp
PostGIS
provider-specific extensions
~~~

Logical backup does not magically install missing operating-system/provider extension support.

---

# 28. Physical Base Backup

PostgreSQL provides `pg_basebackup` for taking a base backup of a running PostgreSQL cluster from a server configured to allow the required replication connection.

Conceptually:

~~~text
Running PostgreSQL cluster
       ↓
pg_basebackup
       ↓
physical base backup
~~~

This is a foundation for physical recovery and replication workflows.

---

# 29. Physical Backup Is Cluster-Level

A physical backup works with the PostgreSQL cluster/data directory rather than exporting one logical database.

Recall PostgreSQL terminology:

~~~text
PostgreSQL cluster
→ collection of databases managed by one server/data directory
~~~

This makes physical backup appropriate for full-instance recovery.

---

# 30. Why You Cannot Simply Copy a Live Data Directory

Naively copying PostgreSQL's live data directory while the server is changing files can produce an inconsistent backup unless you use a PostgreSQL-supported physical backup/snapshot procedure.

Bad idea:

~~~text
cp -r live-postgres-data/ backup/
~~~

without understanding consistency requirements.

Use PostgreSQL-supported base backup mechanisms or provider-supported storage snapshots with the required consistency guarantees.

---

# 31. WAL and Recovery

From Lesson 30:

~~~text
WAL
→ record of database changes needed for durability/recovery
~~~

A physical base backup plus archived WAL can allow PostgreSQL to replay changes after the base backup.

~~~text
Base Backup
     +
Archived WAL
     ↓
Recovery to later state
~~~

This is the foundation of Point-in-Time Recovery.

---

# 32. Point-in-Time Recovery (PITR)

PITR lets you recover the database to a selected point in time within your available backup/WAL history.

Example:

~~~text
10:00 database healthy
10:30 accidental DELETE
10:45 mistake discovered
~~~

Goal:

~~~text
Restore base backup
      ↓
replay WAL
      ↓
stop before destructive event
      ↓
recover state near 10:29
~~~

This is much more precise than restoring yesterday's nightly dump.

---

# 33. PITR Requires More Than WAL Files

You normally need:

~~~text
valid base backup
      +
continuous required WAL archive
      +
recovery configuration
~~~

If required WAL segments are missing, recovery cannot cross that gap.

Therefore WAL archiving must be monitored.

---

# 34. archive_mode and archive_command — High Level

Self-managed PostgreSQL can archive completed WAL segments using configuration such as:

~~~text
archive_mode
archive_command
~~~

Conceptually:

~~~text
PostgreSQL WAL
     ↓
archive process
     ↓
durable external WAL archive
~~~

Never design `archive_command` casually; failed archiving can cause WAL accumulation and recovery gaps.

Managed providers often implement PITR internally and expose retention/recovery controls instead of requiring you to configure these parameters directly.

---

# 35. RPO — Recovery Point Objective

RPO asks:

> **How much data can the business afford to lose?**

Examples:

~~~text
RPO = 24 hours
→ losing up to one day may be acceptable

RPO = 5 minutes
→ at most roughly five minutes of data loss is acceptable

RPO near zero
→ requires much stronger continuous protection architecture
~~~

Your backup frequency and WAL strategy should come from business requirements.

---

# 36. RTO — Recovery Time Objective

RTO asks:

> **How long can the service remain unavailable while recovering?**

Examples:

~~~text
RTO = 4 hours
→ recovery may take hours

RTO = 10 minutes
→ recovery must be very fast
~~~

Backup architecture must consider both:

~~~text
RPO
→ acceptable data loss

RTO
→ acceptable recovery time
~~~

---

# 37. Backup Frequency

There is no universal answer such as:

~~~text
Always run pg_dump once per day
~~~

Frequency depends on:

~~~text
RPO
database size
write rate
restore time
storage cost
business importance
provider capabilities
~~~

A high-value production database may need continuous WAL/PITR plus periodic backups rather than only daily dumps.

---

# 38. Backup Retention

Do not keep only the newest backup.

A corruption or mistake may go unnoticed for days.

A retention strategy may conceptually include:

~~~text
recent daily backups
weekly backups
monthly backups
PITR WAL retention window
~~~

Exact policy depends on business/legal/storage requirements.

---

# 39. The 3-2-1 Backup Principle

A general backup principle is:

~~~text
3 copies of important data
2 different storage/media types or failure domains
1 copy offsite / isolated
~~~

For cloud systems, translate this into independent failure domains and durable backup storage rather than interpreting it too literally.

Do not keep the only backup on the same disk/server as PostgreSQL.

---

# 40. Backup Encryption

Backups contain production data.

They may contain:

~~~text
user information
emails
orders
business data
password hashes
tokens/secrets stored in DB
~~~

Protect backups with:

~~~text
encryption at rest
encrypted transport
strict access control
secure key management
~~~

A stolen unencrypted backup can be as damaging as a compromised live database.

---

# 41. Backup Access Control

Only required systems/people should be able to:

~~~text
create backups
read backups
download backups
delete backups
restore backups
~~~

Consider separate permissions for backup deletion so compromised application credentials cannot erase recovery copies.

---

# 42. Backup Integrity

A file existing in storage does not prove it is usable.

Check:

~~~text
backup command succeeded
expected file/archive exists
size is plausible
checksums/integrity mechanisms where applicable
logs contain no errors
restore test succeeds
~~~

The strongest test is a real restore into an isolated environment.

---

# 43. Restore Testing

This is one of the most important production practices.

Process:

~~~text
Take backup
    ↓
Create isolated restore environment
    ↓
Restore backup
    ↓
Run validation queries/tests
    ↓
Measure recovery time
    ↓
Record result
~~~

If you never test restore, you do not truly know your recovery capability.

---

# 44. What to Validate After Restore

Check:

~~~text
database starts/connects
important schemas exist
table counts are plausible
critical records exist
constraints/indexes exist
roles/permissions are correct
extensions work
application can connect
important API flows pass
~~~

For ShopHub, test:

~~~text
user login
product listing
orders
inventory
admin queries
~~~

---

# 45. Restore Into a Separate Database First

For logical restore testing:

~~~text
Production
   ↓ backup
Backup file
   ↓
Temporary restore DB
~~~

Do not practice destructive restore commands directly against the live production database.

Use isolated infrastructure whenever possible.

---

# 46. Backup Automation

Manual backups are easy to forget.

Production backup should normally be automated:

~~~text
scheduler/provider backup service
       ↓
backup job
       ↓
durable storage
       ↓
success/failure monitoring
~~~

Automation must include alerting.

A failed backup job that nobody notices is not a reliable backup system.

---

# 47. Example Logical Backup Script — Conceptual

~~~bash
#!/usr/bin/env bash
set -euo pipefail

timestamp=$(date +%Y%m%d_%H%M%S)
pg_dump -Fc "$DATABASE_URL" -f "shophub_${timestamp}.dump"
~~~

This is only the start.

A real production process also needs:

~~~text
secure storage upload
retention
encryption
monitoring
failure alerts
restore testing
~~~

Do not store database credentials directly in the script.

---

# 48. Password Handling

Avoid:

~~~bash
pg_dump postgresql://user:plaintext-password@host/db ...
~~~

in shell history/scripts where credentials can leak.

Use appropriate secure mechanisms such as:

~~~text
environment secrets
properly protected .pgpass where appropriate
cloud secret manager
provider-managed credentials
~~~

Protect process environments and logs as well.

---

# 49. Backup Before Risky Migration

Before a destructive/high-risk migration:

~~~text
verify recent backup
      ↓
verify restore path
      ↓
run migration
~~~

But do not use backups as an excuse for unsafe migrations.

Still use:

~~~text
tested migration scripts
staging rehearsal
transactional DDL where appropriate
roll-forward/rollback strategy
~~~

---

# 50. Logical Backup Is Not Always Enough for Huge Databases

For a very large database:

~~~text
pg_dump time
restore time
backup size
RTO
~~~

may become unacceptable.

Then physical backups, incremental/provider snapshot capabilities, continuous WAL archiving, replicas, or specialized backup tools may be more appropriate.

Choose recovery architecture from database size and RPO/RTO.

---

# 51. Managed PostgreSQL Backups

Managed services often provide:

~~~text
automatic backups
retention settings
PITR
snapshots
cross-region options
restore-to-new-instance workflows
~~~

Do not assume the defaults satisfy your requirements.

Verify:

~~~text
retention period
PITR window
RPO/RTO
backup location/failure domain
restore process
cost
deletion behavior
~~~

Provider-managed backup still requires your recovery plan.

---

# 52. Backup vs Replication

Replication:

~~~text
Primary
   ↓ continuous changes
Replica
~~~

Backup:

~~~text
Database state/history
   ↓
independent recovery copy
~~~

Replication helps availability/read scaling.

Backup helps recover historical or lost state.

Production systems often need both.

---

# 53. Backup vs Export

A CSV export is not equivalent to a complete PostgreSQL backup.

CSV may contain table rows but not necessarily:

~~~text
constraints
indexes
functions
views
sequences
permissions
types
relationships
~~~

Use database backup tools for database recovery.

Use CSV/export when the requirement is data interchange/reporting.

---

# 54. Restore Order and Dependencies

PostgreSQL objects have dependencies:

~~~text
types
tables
data
constraints
indexes
views
functions
~~~

`pg_restore` understands archive object dependencies and can order restoration appropriately.

This is another reason archive formats are useful for complex restores.

---

# 55. Sequence State After Restore

Logical dumps normally include sequence state as needed for dumped objects.

But when manually moving data outside normal dump/restore workflows, sequence values can become inconsistent.

Example problem:

~~~text
highest users.id = 500
sequence next value = 100
~~~

Then future INSERTs can conflict.

Use standard dump/restore tools or explicitly validate sequence state after custom migrations.

---

# 56. Ownership and Privileges After Restore

Restoring into a different environment can produce issues such as:

~~~text
role does not exist
permission denied
wrong owner
application cannot access table
~~~

Plan:

~~~text
roles
ownership
GRANTs
schema USAGE
runtime role
migration role
~~~

Backup recovery is not complete until the application can operate with the intended security model.

---

# 57. ShopHub Recovery Design

A reasonable conceptual architecture:

~~~text
Managed PostgreSQL
      │
      ├── automatic provider backups
      ├── PITR / WAL retention
      └── periodic independent logical dump
                 ↓
          protected backup storage
~~~

Then regularly:

~~~text
restore into temporary environment
      ↓
run ShopHub smoke tests
      ↓
record recovery time
~~~

This provides both provider-level recovery and a portable logical copy.

---

# 58. CareerLoop Recovery Design

For a smaller managed PostgreSQL application:

~~~text
Managed provider backup/PITR
        +
periodic logical dump if portability/extra protection is required
        ↓
tested restore
~~~

Do not over-engineer initially, but know:

~~~text
where backups are
how long they are retained
how to restore
how much data could be lost
how long recovery takes
~~~

Those questions matter more than merely saying "the provider has backups."

---

# 59. Disaster Recovery Runbook

A production team should document:

~~~text
Who declares recovery?
Which backup should be used?
Where are credentials/keys?
How is a restore started?
How is WAL/PITR target chosen?
How is application traffic stopped/switched?
How is restored data validated?
How is traffic returned?
Who communicates status?
~~~

During an incident, you should not be inventing the recovery process from memory.

---

# 60. Recovery Flow

~~~text
Incident detected
      ↓
Protect evidence / stop harmful writes if necessary
      ↓
Choose recovery target
      ↓
Restore backup / PITR
      ↓
Validate database
      ↓
Validate application
      ↓
Switch traffic
      ↓
Monitor closely
      ↓
Post-incident review
~~~

Exact steps depend on the failure type.

---

# Common Mistakes

## 61. Taking Backups but Never Testing Restore

A backup that cannot be restored is not a useful recovery plan.

## 62. Keeping the Only Backup on the Database Server

A server/storage failure could destroy both production data and backup.

## 63. Assuming a Replica Is a Backup

Replication can propagate destructive changes.

## 64. Running pg_dumpall as the Only Strategy for a Huge Production Cluster

Choose backup technology according to size, RPO, and RTO.

## 65. Copying a Live Data Directory Naively

Use PostgreSQL-supported physical backup/snapshot procedures.

## 66. Ignoring Roles and Extensions

A data restore can still fail operationally if ownership, privileges, or required extensions are missing.

## 67. Hardcoding Backup Passwords

Use secure credential management.

## 68. Keeping Backups Forever Without a Retention Policy

Define retention based on recovery, compliance, privacy, and cost requirements.

## 69. Assuming Managed Backups Need No Testing

Know how to restore and test the provider's recovery workflow.

## 70. Confusing RPO and RTO

RPO is acceptable data loss; RTO is acceptable recovery time.

---

# Interview Revision

## What is pg_dump?

A PostgreSQL utility that creates a logical backup of one database.

## pg_dump vs pg_dumpall?

`pg_dump` dumps one database. `pg_dumpall` can dump all databases in a cluster and cluster-wide global objects such as roles, depending on options.

## Plain vs custom dump?

Plain format is SQL typically restored with `psql`; custom format is an archive restored with `pg_restore` and supports flexible/selective restore capabilities.

## Can pg_dump run while the database is live?

Yes. It uses PostgreSQL's MVCC snapshot behavior to produce a consistent logical dump while normal activity can continue, though conflicting DDL must still be considered.

## What is pg_restore?

A utility for restoring non-plain `pg_dump` archive formats such as custom, directory, and tar archives.

## What is a physical backup?

A backup of PostgreSQL's physical cluster representation rather than a logical export of database objects.

## What is pg_basebackup?

A PostgreSQL utility for taking a physical base backup of a running PostgreSQL cluster using the replication protocol.

## What is PITR?

Point-in-Time Recovery restores a physical base backup and replays archived WAL to recover to a chosen point within the available recovery history.

## What is RPO?

Recovery Point Objective: the maximum acceptable amount of data loss measured in time.

## What is RTO?

Recovery Time Objective: the maximum acceptable time to restore service.

## Is replication a backup?

No. Replication improves availability/read scaling but can replicate accidental destructive changes. Independent historical recovery is still needed.

## What is the most important backup practice?

Regularly test restores and validate that the recovered database/application actually works.

---

# Quick Revision

~~~text
LOGICAL BACKUP
PostgreSQL DB
    ↓
pg_dump
    ↓
plain SQL / custom archive
    ↓
psql / pg_restore
~~~

~~~text
PHYSICAL + PITR
Base backup
    +
Archived WAL
    ↓
restore + replay
    ↓
chosen recovery point
~~~

### Tools

~~~text
pg_dump
→ one database

pg_dumpall
→ cluster-wide logical script / globals

pg_restore
→ restore archive-format dumps

pg_basebackup
→ physical base backup
~~~

### Recovery Objectives

~~~text
RPO
→ How much data can we lose?

RTO
→ How long can recovery take?
~~~

### Golden Rule

~~~text
Backup created
≠
Recovery guaranteed

Backup restored and tested
=
Evidence that recovery works
~~~

---

## Key Takeaway

> **PostgreSQL recovery should combine the right backup type with business RPO/RTO requirements. Use `pg_dump`/`pg_restore` for portable logical backups, physical base backups plus WAL for full-cluster Point-in-Time Recovery, protect backup copies independently from production, automate and monitor backup jobs, and regularly perform real restore tests. A backup file is not proof of recoverability—the successful restore is.**

---

[← Previous: Lesson 38 — Connection Pooling & PgBouncer](./38-connection-pooling-and-pgbouncer.md) | [Back to Roadmap](../README.md) | [Next: Lesson 40 — Replication →](./40-replication.md)