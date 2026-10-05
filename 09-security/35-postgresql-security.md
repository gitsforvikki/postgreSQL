# Lesson 35 — PostgreSQL Security

## Goal of This Lesson

Database security is not only about having a password.

A production PostgreSQL system needs multiple security layers:

~~~text
Application / Client
       ↓
Network security
       ↓
Authentication
       ↓
Authorization
       ↓
Database objects + data
~~~

The two most important questions are:

~~~text
Authentication
→ Who are you?

Authorization
→ What are you allowed to do?
~~~

---

# 1. PostgreSQL Roles

PostgreSQL uses **roles** for database identities and privilege grouping.

A role can represent:

~~~text
an application
a developer
an administrator
a read-only group
a migration process
~~~

Example:

~~~sql
CREATE ROLE shophub_app;
~~~

By default, that role does not necessarily have permission to log in.

---

# 2. LOGIN Roles

To allow a role to authenticate as a database user:

~~~sql
CREATE ROLE shophub_app
WITH LOGIN
PASSWORD 'strong-password';
~~~

or:

~~~sql
CREATE USER shophub_app
WITH PASSWORD 'strong-password';
~~~

In PostgreSQL, `CREATE USER` is essentially convenient syntax for creating a role with `LOGIN`.

Easy memory:

~~~text
ROLE
→ identity / privilege container

ROLE + LOGIN
→ can authenticate
~~~

---

# 3. Role Attributes

Roles can have attributes such as:

~~~text
LOGIN
SUPERUSER
CREATEDB
CREATEROLE
REPLICATION
~~~

Example:

~~~sql
CREATE ROLE migration_user
WITH LOGIN
CREATEDB
PASSWORD 'strong-password';
~~~

Do not grant powerful attributes unless they are genuinely required.

---

# 4. SUPERUSER

A PostgreSQL superuser can bypass most normal permission checks.

~~~text
SUPERUSER
   ↓
extremely powerful
   ↓
can access/modify almost everything
~~~

Your normal production application should generally **not** connect as a superuser.

Bad architecture:

~~~text
Web Application
      ↓
postgres / superuser
      ↓
Database
~~~

Better:

~~~text
Web Application
      ↓
limited application role
      ↓
only required database privileges
~~~

---

# 5. Principle of Least Privilege

**Least privilege** means:

> Give a role only the permissions it actually needs.

Example application requirement:

~~~text
users table
→ SELECT
→ INSERT
→ UPDATE

audit_logs
→ INSERT only

admin_settings
→ no access
~~~

Do not grant broad privileges simply because it is convenient.

---

# 6. GRANT

`GRANT` gives privileges.

Example:

~~~sql
GRANT SELECT, INSERT, UPDATE
ON TABLE users
TO shophub_app;
~~~

Now `shophub_app` can perform those operations on `users`, assuming other required database/schema access is also available.

---

# 7. REVOKE

`REVOKE` removes privileges.

~~~sql
REVOKE DELETE
ON TABLE users
FROM shophub_app;
~~~

Think:

~~~text
GRANT
→ give permission

REVOKE
→ remove permission
~~~

---

# 8. Common Table Privileges

Important table privileges include:

~~~text
SELECT
INSERT
UPDATE
DELETE
TRUNCATE
REFERENCES
TRIGGER
~~~

Example read-only role:

~~~sql
GRANT SELECT
ON TABLE products
TO reporting_user;
~~~

That role can read the table but does not automatically gain INSERT/UPDATE/DELETE.

---

# 9. Database CONNECT Privilege

A role may need permission to connect to a database.

~~~sql
GRANT CONNECT
ON DATABASE shophub
TO shophub_app;
~~~

Important:

> `CONNECT` means the role may connect to the database. It does **not** mean it can automatically SELECT every table.

Database access and object privileges are separate layers.

---

# 10. Schema Privileges

Objects live inside schemas.

A role commonly needs `USAGE` on a schema to access objects inside it.

Example:

~~~sql
GRANT USAGE
ON SCHEMA public
TO shophub_app;
~~~

Conceptually:

~~~text
Database CONNECT
      ↓
Schema USAGE
      ↓
Table SELECT / INSERT / UPDATE / DELETE
~~~

Each layer can matter.

---

# 11. CREATE on a Schema

`CREATE` on a schema allows creating objects inside it.

Example:

~~~sql
GRANT CREATE
ON SCHEMA app
TO migration_user;
~~~

Your normal runtime application may not need schema creation privileges.

Separate schema-changing permissions from ordinary runtime access when possible.

---

# 12. PUBLIC

`PUBLIC` is a special PostgreSQL pseudo-role representing all roles.

Conceptually:

~~~text
PUBLIC
→ every PostgreSQL role
~~~

If a privilege is granted to `PUBLIC`, every role effectively receives it.

Therefore inspect broad `PUBLIC` privileges when hardening a database.

Do not assume every privilege is granted to PUBLIC by default. Defaults depend on the object type; for example, databases commonly grant some privileges such as CONNECT/TEMPORARY to PUBLIC, while ordinary table access is not automatically granted to everyone.

---

# 13. Group Roles

A role does not need to represent only one login.

You can create a role as a privilege group:

~~~sql
CREATE ROLE app_readonly;

GRANT SELECT
ON ALL TABLES IN SCHEMA public
TO app_readonly;
~~~

Then grant membership:

~~~sql
GRANT app_readonly TO developer1;
~~~

Architecture:

~~~text
app_readonly
   ↓ privileges
tables

developer1
   ↓ membership
app_readonly
~~~

This makes privilege management easier for teams.

---

# 14. Existing Tables vs Future Tables

This command:

~~~sql
GRANT SELECT
ON ALL TABLES IN SCHEMA public
TO app_readonly;
~~~

affects existing matching tables.

What about tables created later?

Use default privileges.

---

# 15. ALTER DEFAULT PRIVILEGES

Example:

~~~sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA public
GRANT SELECT ON TABLES
TO app_readonly;
~~~

This configures privileges for **future objects created by the relevant object-creating role**.

Important interview nuance:

> `ALTER DEFAULT PRIVILEGES` does not retroactively change existing tables.

Also remember that default privileges are associated with the role creating future objects, so configure them in the correct ownership context.

---

# 16. Runtime Role vs Migration Role

A strong production design separates responsibilities.

~~~text
Migration role
→ CREATE TABLE
→ ALTER TABLE
→ CREATE INDEX
→ schema changes

Application runtime role
→ SELECT
→ INSERT
→ UPDATE
→ DELETE
→ only required runtime objects
~~~

Why?

If your web application's credentials are compromised, the attacker should not automatically gain permission to drop or redesign your entire schema.

---

# 17. Example Role Design

~~~sql
CREATE ROLE shophub_runtime
WITH LOGIN
PASSWORD 'runtime-secret';

CREATE ROLE shophub_migration
WITH LOGIN
PASSWORD 'migration-secret';
~~~

Then grant only what each role needs.

Conceptually:

~~~text
CI/CD migration job
      ↓
shophub_migration
      ↓
schema privileges

Next.js / Express app
      ↓
shophub_runtime
      ↓
runtime data privileges
~~~

---

# 18. Authentication Is Separate from Authorization

Suppose a role successfully enters the correct password.

That proves:

~~~text
Authentication succeeded
~~~

It does **not** automatically mean:

~~~text
SELECT every table
DROP database
CREATE role
read admin data
~~~

Those abilities depend on role attributes and object privileges.

---

# 19. pg_hba.conf

`pg_hba.conf` is PostgreSQL's **Host-Based Authentication** configuration file.

It controls which clients can attempt to connect and which authentication method is used for matching connections.

Conceptually:

~~~text
Incoming connection
       ↓
pg_hba.conf rules
       ↓
matching rule
       ↓
authentication method
       ↓
success / failure
~~~

---

# 20. pg_hba.conf Rule Structure

A typical host rule resembles:

~~~text
TYPE  DATABASE  USER  ADDRESS         METHOD
host  shophub   app   127.0.0.1/32    scram-sha-256
~~~

Important fields:

~~~text
TYPE
→ local / host / hostssl etc.

DATABASE
→ which database(s)

USER
→ which role(s)

ADDRESS
→ allowed client network/address

METHOD
→ authentication mechanism
~~~

---

# 21. First Matching pg_hba.conf Rule

This is an important PostgreSQL behavior.

Rules are checked in order.

~~~text
Connection
   ↓
Rule 1 match? → no
   ↓
Rule 2 match? → yes
   ↓
use Rule 2 authentication method
~~~

Once a record matches, PostgreSQL uses that record. If authentication under that matching record fails, PostgreSQL does not simply continue trying later records as fallback alternatives.

Rule ordering therefore matters.

---

# 22. local vs host

High-level:

~~~text
local
→ local Unix-domain socket connection

host
→ TCP/IP connection

hostssl
→ TCP/IP connection using SSL/TLS
~~~

Example:

~~~text
local   all   postgres                 peer
host    appdb appuser 127.0.0.1/32    scram-sha-256
~~~

Exact configuration depends on your operating system, PostgreSQL setup, and security requirements.

---

# 23. trust Authentication

A `trust` rule means a matching connection is allowed without PostgreSQL requiring a password or other authentication proof.

Example:

~~~text
host all all 0.0.0.0/0 trust
~~~

This would be extremely dangerous on an exposed server.

Rule:

> Never use broad `trust` rules in production simply to make connection errors disappear.

---

# 24. SCRAM-SHA-256

`scram-sha-256` is PostgreSQL's modern password authentication method and is preferred over older MD5-based password authentication for new secure configurations.

Example:

~~~text
host  shophub  shophub_app  10.0.0.0/24  scram-sha-256
~~~

Conceptually:

~~~text
Client
  ↓
SCRAM authentication exchange
  ↓
PostgreSQL verifies credentials
~~~

Passwords should not be stored or transmitted as plain application code constants.

---

# 25. pg_hba.conf vs GRANT

This distinction is frequently asked in interviews.

~~~text
pg_hba.conf
→ Can/how this connection authenticate?

GRANT / REVOKE
→ What may the authenticated role do?
~~~

Example:

~~~text
pg_hba.conf allows appuser connection
             ↓
appuser authenticates successfully
             ↓
SELECT users
             ↓
PostgreSQL checks table privileges
~~~

Both layers must permit the operation.

---

# 26. SSL/TLS

SSL/TLS protects PostgreSQL network traffic from being sent as readable plaintext across an untrusted network.

Architecture:

~~~text
Application
    ↓ encrypted connection
PostgreSQL
~~~

Without transport encryption on an untrusted network, sensitive traffic can be exposed.

Managed PostgreSQL providers commonly provide TLS connection requirements/options.

---

# 27. hostssl

`hostssl` rules match TCP/IP connections using SSL.

Example:

~~~text
hostssl shophub shophub_app 10.0.0.0/24 scram-sha-256
~~~

This combines:

~~~text
network/address rule
       +
SSL connection requirement
       +
SCRAM authentication
~~~

TLS and password authentication solve different problems.

---

# 28. Certificate Verification

Encryption alone is not the complete story.

A client should also verify that it is connecting to the intended server according to the security mode/configuration being used.

Bad production workaround:

~~~js
ssl: {
  rejectUnauthorized: false
}
~~~

used blindly just to silence TLS errors.

This can weaken server identity verification.

Use your PostgreSQL provider's documented certificate/SSL configuration.

---

# 29. Database Password vs Application User Password

Do not confuse these.

~~~text
DATABASE ROLE PASSWORD
→ Next.js / Express backend authenticates to PostgreSQL

APPLICATION USER PASSWORD
→ Customer logs into ShopHub / CareerLoop
~~~

Example architecture:

~~~text
Customer
   ↓ email/password/session
Application Backend
   ↓ DATABASE_URL credentials
PostgreSQL
~~~

These are completely different authentication layers.

---

# 30. Protect DATABASE_URL

Bad:

~~~ts
const DATABASE_URL = 'postgresql://admin:password@server/db';
~~~

committed to Git.

Better:

~~~text
Deployment secret / environment variable
       ↓
DATABASE_URL
       ↓
server-only application code
~~~

Never expose database credentials through `NEXT_PUBLIC_*` variables.

If credentials leak, rotate them.

---

# 31. Network Exposure

Prefer architectures where PostgreSQL is not unnecessarily reachable from the public internet.

~~~text
Internet
   ↓
Application
   ↓ private/restricted network
PostgreSQL
~~~

Instead of:

~~~text
Entire Internet
      ↓
Public PostgreSQL port
~~~

Use provider networking controls, firewalls/security groups, private networking, or IP restrictions where appropriate.

---

# 32. Security Layers Together

Production architecture:

~~~text
Browser
  ↓ HTTPS
Next.js / Express
  ↓
Application authentication
  ↓
Authorization
  ↓
Parameterized SQL
  ↓ TLS/private network
PostgreSQL
  ↓
pg_hba.conf authentication
  ↓
Role privileges
  ↓
Constraints / data
~~~

No single layer replaces the others.

---

# 33. SQL Injection Is a Different Layer

Even a limited PostgreSQL role does not mean SQL injection is acceptable.

Use:

~~~text
parameterized queries
ORM safe APIs
allowlisted dynamic identifiers
~~~

Then also use least privilege.

~~~text
Prevent attack
      +
Reduce blast radius if something goes wrong
~~~

Lesson 36 covers SQL injection deeply.

---

# 34. Row-Level Security (RLS) — Introduction

PostgreSQL supports **Row-Level Security** policies.

RLS can restrict which rows a role may access or modify.

Conceptual example:

~~~text
orders table

User context A
→ rows belonging to A

User context B
→ rows belonging to B
~~~

RLS can be useful in multi-tenant/security-sensitive architectures.

But it requires careful policy and application design.

For now remember:

> Table privileges control access at the object/operation level; RLS can additionally restrict access at the row level.

---

# 35. Ownership

Every PostgreSQL object has an owner.

Owners have special control over their objects, including the ability to alter/drop them and manage privileges.

This is another reason not to make the normal application role the owner of everything unnecessarily.

Conceptually:

~~~text
schema owner / migration owner
      ↓
creates/manages objects

runtime app role
      ↓
receives only required privileges
~~~

---

# 36. Sequences and Permissions

If you use sequences directly or certain identity/sequence-backed patterns, sequence privileges can matter.

Example:

~~~sql
GRANT USAGE, SELECT
ON SEQUENCE some_sequence
TO shophub_app;
~~~

Do not memorize every sequence privilege now.

Remember that tables are not the only PostgreSQL objects with permissions.

---

# 37. Functions and Permissions

Functions also have privileges such as EXECUTE.

Be especially careful with security-sensitive functions and `SECURITY DEFINER`, because such functions can execute with the function owner's privileges.

High-level:

~~~text
SECURITY INVOKER
→ caller's privileges

SECURITY DEFINER
→ function owner's privileges
~~~

`SECURITY DEFINER` can be powerful and dangerous if written or configured incorrectly.

---

# 38. Inspect Roles

In `psql`:

~~~text
\du
~~~

This shows roles and important attributes.

Useful question:

~~~text
Is my application accidentally SUPERUSER?
Does this role have CREATEDB?
Which roles does it belong to?
~~~

---

# 39. Inspect Privileges

In `psql`, useful commands include:

~~~text
\l
\dn
\dt
\d users
\dp users
~~~

`\dp` / `\z` can help inspect object access privileges.

Use these tools when debugging permission problems.

---

# 40. Debugging Permission Errors

Suppose the application receives:

~~~text
permission denied for table orders
~~~

Debug systematically:

~~~text
1. Which DB role is the app actually using?
2. Can it connect under pg_hba.conf?
3. Does it have database CONNECT?
4. Does it have schema USAGE?
5. Does it have required table privilege?
6. Is privilege inherited through another role?
7. Is RLS involved?
~~~

Do not solve every permission error by granting SUPERUSER.

---

# 41. ShopHub Production Example

Architecture:

~~~text
Vercel / Backend
      ↓
DATABASE_URL for shophub_runtime
      ↓ TLS
Managed PostgreSQL
      ↓
pg_hba/network authentication
      ↓
runtime role
      ↓
SELECT / INSERT / UPDATE / DELETE
only where required
~~~

CI/CD migration:

~~~text
Deployment pipeline
      ↓
migration credentials
      ↓
shophub_migration
      ↓
ALTER / CREATE required schema objects
~~~

Separate credentials reduce unnecessary privilege.

---

# 42. CareerLoop Example

For a Next.js + Drizzle + PostgreSQL application:

~~~text
Browser
   ↓
Next.js server
   ↓
Drizzle
   ↓
limited runtime PostgreSQL role
   ↓
CareerLoop database
~~~

The browser never receives the database password.

Schema migrations can use a more privileged deployment-only role if your infrastructure supports that separation.

---

# Common Mistakes

## 43. Running the Web App as postgres/SUPERUSER

Use a least-privileged application role.

## 44. Granting ALL Everywhere

Grant only what the role needs.

## 45. Confusing LOGIN with Table Access

LOGIN allows authentication; object privileges determine permitted operations.

## 46. Confusing pg_hba.conf with GRANT

`pg_hba.conf` handles connection authentication rules; `GRANT`/`REVOKE` handle authorization.

## 47. Using Broad trust Rules

`trust` can allow matching clients without password authentication. Never expose it broadly in production.

## 48. Exposing DATABASE_URL to the Browser

Database credentials belong only in trusted server-side environments.

## 49. Blindly Disabling TLS Certificate Verification

Follow the provider's secure SSL/TLS configuration instead.

## 50. Using the Runtime Role for Migrations

Separate schema-changing and runtime privileges when practical.

## 51. Assuming Application Authentication Protects PostgreSQL

Your user-login system and PostgreSQL role authentication are separate layers.

## 52. Solving Permission Errors with SUPERUSER

Find the missing privilege instead of bypassing the security model.

---

# Interview Revision

## What is a PostgreSQL role?

A PostgreSQL identity and privilege container that can represent users, applications, or privilege groups.

## Role vs user?

PostgreSQL uses roles as the unified concept. A role with LOGIN can authenticate; `CREATE USER` is essentially shorthand for creating a LOGIN role.

## What is least privilege?

Grant only the permissions required for a role to perform its intended job.

## What does GRANT do?

It gives privileges on database objects or role membership.

## What does REVOKE do?

It removes previously available privileges.

## What does CONNECT mean?

It permits connecting to a database; it does not automatically grant table access.

## What does schema USAGE mean?

It allows access to objects within the schema, subject to the privileges on those objects.

## What is PUBLIC?

A special pseudo-role representing all PostgreSQL roles.

## What is pg_hba.conf?

PostgreSQL's host-based authentication configuration that determines which matching connections can authenticate and which method they must use.

## How are pg_hba.conf rules evaluated?

Top to bottom; the first matching record is used, and failed authentication does not fall through to later matching records.

## What is SCRAM-SHA-256?

A modern password authentication method supported by PostgreSQL and preferred over legacy MD5-based password authentication for new secure setups.

## pg_hba.conf vs GRANT?

`pg_hba.conf` controls connection authentication; `GRANT`/`REVOKE` control what an authenticated role may do.

## Why use SSL/TLS?

To encrypt database network traffic and, with appropriate verification, authenticate the server identity.

## Why separate migration and runtime roles?

The application normally needs data operations, while migrations need schema-changing privileges. Separation reduces the blast radius of compromised runtime credentials.

## What is RLS?

Row-Level Security lets PostgreSQL policies restrict which rows a role can access or modify.

---

# Quick Revision

~~~text
AUTHENTICATION
→ Who are you?

AUTHORIZATION
→ What can you do?
~~~

### Connection Security

~~~text
Client
  ↓ network rule
pg_hba.conf
  ↓ authentication
SCRAM / certificate / other configured method
  ↓
PostgreSQL role
~~~

### Authorization

~~~text
Role
 ↓
Database CONNECT
 ↓
Schema USAGE
 ↓
Table privileges
 ↓
Optional RLS
 ↓
Data
~~~

### Production Roles

~~~text
Migration role
→ schema changes

Runtime role
→ only application data operations

Admin role
→ operational administration
~~~

### Golden Rule

~~~text
Authenticate strongly.
Grant minimally.
Encrypt connections.
Keep DB credentials server-side.
~~~

---

## Key Takeaway

> **PostgreSQL security is layered: `pg_hba.conf` and authentication establish who may connect, roles and `GRANT`/`REVOKE` determine what authenticated identities may do, SSL/TLS protects network traffic, and least-privilege runtime roles reduce the impact of compromised application credentials. Keep migration/admin privileges separate from normal application access whenever practical.**

---

[← Previous: Lesson 34 — ORM vs Raw SQL](../08-backend-development/34-orm-vs-raw-sql.md) | [Back to Roadmap](../README.md) | [Next: Lesson 36 — SQL Injection & Secure Queries →](./36-sql-injection-and-secure-queries.md)