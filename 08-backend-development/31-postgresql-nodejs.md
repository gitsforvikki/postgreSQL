# Lesson 31 — PostgreSQL + Node.js

## Goal of This Lesson

So far, you have learned PostgreSQL itself.

Now we connect it to a real backend application.

High-level architecture:

~~~text
Client / Frontend
       ↓
Node.js Backend
       ↓
node-postgres (pg)
       ↓
PostgreSQL
~~~

In Node.js, one of the standard PostgreSQL drivers is **node-postgres**, installed through the `pg` package.

---

# 1. Install node-postgres

~~~bash
npm install pg
~~~

For a TypeScript project, depending on your setup and package versions, you may also use the relevant type support.

Then import from `pg`:

~~~js
const { Client, Pool } = require('pg');
~~~

or with ESM:

~~~js
import pg from 'pg';

const { Pool } = pg;
~~~

Your exact import style depends on your Node.js/module configuration.

---

# 2. Basic Client Connection

A simple direct connection can use `Client`:

~~~js
import pg from 'pg';

const { Client } = pg;

const client = new Client({
  host: 'localhost',
  port: 5432,
  user: 'postgres',
  password: 'your-password',
  database: 'shophub',
});

await client.connect();

const result = await client.query('SELECT NOW()');
console.log(result.rows);

await client.end();
~~~

Flow:

~~~text
new Client()
    ↓
connect()
    ↓
query()
    ↓
end()
~~~

This is useful for learning and scripts.

For a web server handling many requests, a connection pool is normally more appropriate.

---

# 3. Why Not Create a New Client for Every HTTP Request?

Bad architecture:

~~~text
Request 1 → connect → query → disconnect
Request 2 → connect → query → disconnect
Request 3 → connect → query → disconnect
...
~~~

Creating database connections has overhead.

PostgreSQL also normally uses a backend process for each active connection.

For web applications, reuse a controlled number of connections.

~~~text
Many HTTP requests
       ↓
Connection Pool
       ↓
Reusable PostgreSQL connections
~~~

---

# 4. Pool

Use `Pool` for normal backend applications:

~~~js
import pg from 'pg';

const { Pool } = pg;

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  user: 'postgres',
  password: 'your-password',
  database: 'shophub',
});
~~~

Then:

~~~js
const result = await pool.query('SELECT NOW()');

console.log(result.rows);
~~~

`pool.query()` checks out a connection internally, runs the query, and returns the connection to the pool automatically.

---

# 5. Use Environment Variables

Do not hard-code production credentials in source code.

Instead:

~~~env
DATABASE_URL=postgresql://username:password@localhost:5432/shophub
~~~

Then:

~~~js
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});
~~~

Architecture:

~~~text
.env / deployment secret
       ↓
process.env.DATABASE_URL
       ↓
Pool
       ↓
PostgreSQL
~~~

Never commit real production database credentials to Git.

---

# 6. A Reusable Database Module

A clean project can centralize the pool.

Example:

~~~text
src/
├── db/
│   └── index.js
├── services/
├── controllers/
├── routes/
└── server.js
~~~

`src/db/index.js`:

~~~js
import pg from 'pg';

const { Pool } = pg;

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});
~~~

Other modules import this shared pool instead of creating a new pool repeatedly.

---

# 7. Running a SELECT Query

~~~js
const result = await pool.query(
  'SELECT id, name, email FROM users'
 );
~~~

`result.rows` contains returned rows:

~~~js
console.log(result.rows);
~~~

Conceptually:

~~~text
Node.js
  ↓
pool.query(SQL)
  ↓
PostgreSQL
  ↓
rows
  ↓
result.rows
~~~

---

# 8. Parameterized Queries

This is one of the most important backend concepts.

Do **not** write:

~~~js
const result = await pool.query(
  `SELECT * FROM users WHERE email = '${email}'`
);
~~~

Untrusted input is being inserted directly into SQL syntax.

Instead use PostgreSQL parameters:

~~~js
const result = await pool.query(
  'SELECT id, name, email FROM users WHERE email = $1',
  [email]
 );
~~~

~~~text
SQL structure
SELECT ... WHERE email = $1
              +
Values
[email]
              ↓
node-postgres sends them separately
~~~

This is the primary defense against SQL injection for query values.

---

# 9. Multiple Parameters

~~~js
const result = await pool.query(
  `
    SELECT id, name, email
    FROM users
    WHERE is_active = $1
      AND created_at >= $2
  `,
  [true, startDate]
 );
~~~

Mapping:

~~~text
$1 → true
$2 → startDate
~~~

Parameters are positional.

---

# 10. INSERT from Node.js

~~~js
const result = await pool.query(
  `
    INSERT INTO users (name, email)
    VALUES ($1, $2)
    RETURNING id, name, email
  `,
  [name, email]
 );

const user = result.rows[0];
~~~

`RETURNING` is extremely useful in PostgreSQL backend code.

Without another SELECT, you immediately receive the inserted row.

~~~text
INSERT
  ↓
RETURNING
  ↓
result.rows[0]
~~~

---

# 11. UPDATE from Node.js

~~~js
const result = await pool.query(
  `
    UPDATE users
    SET name = $1
    WHERE id = $2
    RETURNING id, name, email
  `,
  [name, userId]
 );
~~~

Then:

~~~js
if (result.rowCount === 0) {
  // user not found
}
~~~

`rowCount` is useful when determining how many rows were affected/returned for commands where that meaning applies.

---

# 12. DELETE from Node.js

~~~js
const result = await pool.query(
  `
    DELETE FROM users
    WHERE id = $1
    RETURNING id
  `,
  [userId]
 );
~~~

Then:

~~~js
if (result.rowCount === 0) {
  // nothing was deleted
}
~~~

---

# 13. CRUD Mapping

~~~text
CREATE
→ INSERT

READ
→ SELECT

UPDATE
→ UPDATE

DELETE
→ DELETE
~~~

Node.js pattern:

~~~text
JavaScript values
      ↓
parameterized SQL
      ↓
pool.query()
      ↓
PostgreSQL
      ↓
result.rows / rowCount
~~~

---

# 14. Understanding the Query Result

A node-postgres result commonly exposes useful properties such as:

~~~text
result.rows
result.rowCount
result.command
result.fields
~~~

For normal application work, the two you will use most often are:

~~~js
result.rows
result.rowCount
~~~

Example:

~~~js
const result = await pool.query(
  'SELECT id, name FROM users WHERE id = $1',
  [userId]
 );

const user = result.rows[0];
~~~

---

# 15. pool.query() vs pool.connect()

This distinction is extremely important.

## Independent Query

Use:

~~~js
await pool.query(...);
~~~

for a normal query that does not need to stay on the same checked-out connection as other statements.

## Transaction

Use:

~~~js
const client = await pool.connect();
~~~

when multiple statements must run through the **same PostgreSQL connection**, such as a transaction.

Easy memory:

~~~text
One independent query
→ pool.query()

Transaction / same-session work
→ pool.connect()
→ client.query()
→ client.release()
~~~

---

# 16. Why a Transaction Must Use the Same Client

Suppose you incorrectly write:

~~~js
await pool.query('BEGIN');
await pool.query('INSERT ...');
await pool.query('UPDATE ...');
await pool.query('COMMIT');
~~~

This is unsafe.

Why?

`pool.query()` can use different pooled connections for different calls.

Conceptually:

~~~text
BEGIN  → Connection A
INSERT → Connection B
UPDATE → Connection C
COMMIT → Connection A
~~~

That is **not one transaction**.

A PostgreSQL transaction belongs to one database session/connection.

---

# 17. Correct Node.js Transaction Pattern

~~~js
const client = await pool.connect();

try {
  await client.query('BEGIN');

  const orderResult = await client.query(
    `
      INSERT INTO orders (user_id, total)
      VALUES ($1, $2)
      RETURNING id
    `,
    [userId, total]
  );

  await client.query(
    `
      UPDATE products
      SET stock = stock - 1
      WHERE id = $1
        AND stock > 0
    `,
    [productId]
  );

  await client.query('COMMIT');

  return orderResult.rows[0];
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
~~~

Flow:

~~~text
pool.connect()
      ↓
Client / Connection A
      ↓
BEGIN
      ↓
INSERT
      ↓
UPDATE
      ↓
COMMIT or ROLLBACK
      ↓
client.release()
~~~

Every transaction statement runs on the same client.

---

# 18. Why client.release() Matters

When you call:

~~~js
const client = await pool.connect();
~~~

you have checked a connection out of the pool.

If you never release it:

~~~text
Pool
├── connection 1 → leaked
├── connection 2 → leaked
├── connection 3 → leaked
└── eventually no available connections
~~~

Therefore:

~~~js
finally {
  client.release();
}
~~~

is an important pattern.

---

# 19. Error Handling

Database operations can fail because of:

~~~text
unique constraint violation
foreign key violation
NOT NULL violation
CHECK violation
connection failure
deadlock
serialization failure
invalid SQL
~~~

Example:

~~~js
try {
  const result = await pool.query(
    `
      INSERT INTO users (name, email)
      VALUES ($1, $2)
      RETURNING id, name, email
    `,
    [name, email]
  );

  return result.rows[0];
} catch (error) {
  console.error('Database operation failed', error);
  throw error;
}
~~~

Do not expose raw internal database errors directly to end users.

Your API layer should translate known errors into appropriate application/HTTP responses.

---

# 20. PostgreSQL Error Codes

PostgreSQL errors have SQLSTATE codes that applications can inspect.

For example, a unique constraint violation commonly uses:

~~~text
23505
~~~

Conceptual backend handling:

~~~js
if (error.code === '23505') {
  // map to an application conflict error
}
~~~

This is better than trying to parse human-readable error strings.

---

# 21. Database Constraints Still Matter

Suppose your Node.js code validates:

~~~js
if (!email) {
  throw new Error('Email required');
}
~~~

You should still use database constraints where appropriate:

~~~sql
email TEXT NOT NULL UNIQUE
~~~

Defense in depth:

~~~text
Frontend validation
       ↓
Backend validation
       ↓
Parameterized SQL
       ↓
Database constraints
~~~

Application validation improves UX/business handling.

Database constraints protect data integrity.

---

# 22. A Simple Service Layer

Instead of placing SQL everywhere, centralize database operations.

Example:

~~~text
src/
├── db/
│   └── index.js
├── services/
│   └── user.service.js
└── ...
~~~

`user.service.js`:

~~~js
import { pool } from '../db/index.js';

export async function getUserById(id) {
  const result = await pool.query(
    `
      SELECT id, name, email
      FROM users
      WHERE id = $1
    `,
    [id]
  );

  return result.rows[0] ?? null;
}
~~~

Benefits:

~~~text
SQL centralized
business logic easier to test
controllers/routes stay smaller
database code easier to optimize later
~~~

---

# 23. Recommended Backend Flow

For a layered Node/Express application:

~~~text
HTTP Request
    ↓
Route
    ↓
Controller
    ↓
Service
    ↓
Database module / pg
    ↓
PostgreSQL
~~~

Responsibilities:

~~~text
Route
→ map URL/method

Controller
→ HTTP request/response concerns

Service
→ business logic + database workflow

DB module
→ pool / database connection infrastructure
~~~

Lesson 32 expands this into a complete Express API.

---

# 24. Dynamic SQL — Important Security Rule

Parameters work for **values**.

Example:

~~~sql
WHERE id = $1
~~~

But you cannot generally use a value parameter as an SQL identifier:

~~~sql
ORDER BY $1
~~~

and expect `$1` to safely become a column identifier such as `created_at`.

For dynamic columns/directions, use an allowlist.

Example:

~~~js
const allowedSortColumns = {
  newest: 'created_at',
  price: 'price',
  name: 'name',
};

const sortColumn = allowedSortColumns[sortBy] ?? 'created_at';

const query = `
  SELECT id, name, price
  FROM products
  ORDER BY ${sortColumn} DESC
`;
~~~

Why is this safe compared with directly inserting user input?

Because `sortColumn` comes from trusted server-defined values, not arbitrary client SQL text.

---

# 25. Dynamic Table Names

Same principle:

~~~text
$1 parameters
→ values

table / column / schema names
→ identifiers
~~~

If identifiers truly must be dynamic:

- prefer a fixed allowlist
- use an appropriate identifier-escaping approach/library when required
- never directly concatenate arbitrary user input

Most CRUD APIs do not need arbitrary table names at all.

---

# 26. Arrays as Parameters

Suppose you need several user IDs.

PostgreSQL can accept an array parameter:

~~~js
const result = await pool.query(
  `
    SELECT id, name
    FROM users
    WHERE id = ANY($1::bigint[])
  `,
  [userIds]
 );
~~~

This is often cleaner than manually constructing an unsafe comma-separated list.

---

# 27. Search with ILIKE

Safe search:

~~~js
const result = await pool.query(
  `
    SELECT id, name
    FROM products
    WHERE name ILIKE $1
    LIMIT $2
  `,
  [`%${search}%`, limit]
 );
~~~

The `%` wildcard is part of the parameter value.

The SQL structure is still separate from untrusted data.

---

# 28. Pagination

Offset pagination:

~~~js
const result = await pool.query(
  `
    SELECT id, name, created_at
    FROM products
    ORDER BY created_at DESC
    LIMIT $1 OFFSET $2
  `,
  [limit, offset]
 );
~~~

Even though values are parameterized, validate and cap `limit` in application code.

Example:

~~~js
const limit = Math.min(Number(requestedLimit) || 20, 100);
~~~

For large datasets/deep pages, consider keyset pagination from Lesson 26.

---

# 29. Prepared Statements — High Level

node-postgres supports named queries.

Example:

~~~js
const result = await pool.query({
  name: 'user-by-id',
  text: `
    SELECT id, name, email
    FROM users
    WHERE id = $1
  `,
  values: [userId],
});
~~~

The name allows prepared-statement behavior on each database connection where it is used.

Do not name every query simply because the feature exists.

Use it when repeated-query behavior and measurement justify it.

---

# 30. Pool Error Events

A pool can encounter errors on idle clients.

A useful production pattern is to register an error listener:

~~~js
pool.on('error', (error) => {
  console.error('Unexpected PostgreSQL pool error', error);
});
~~~

Your production application should also have appropriate logging, monitoring, and process-management behavior.

---

# 31. Graceful Shutdown

When your application is shutting down, you can close the pool:

~~~js
await pool.end();
~~~

Conceptually:

~~~text
Application shutting down
        ↓
stop accepting new work
        ↓
finish/terminate appropriate work
        ↓
pool.end()
        ↓
DB connections closed
~~~

Do not call `pool.end()` after every request.

It is for shutting down the pool when the application/process is ending.

---

# 32. NUMERIC and JavaScript Precision

PostgreSQL's `NUMERIC` type can represent exact decimal values with precision beyond JavaScript's normal `Number` safety.

Database drivers may therefore return some numeric types as strings rather than automatically converting them to JavaScript numbers.

Example conceptual result:

~~~js
{
  total: '9999999999999999.99'
}
~~~

Do not blindly call `Number()` on values that may exceed JavaScript's safe precision.

For money, define a consistent application strategy such as integer minor units where appropriate or a decimal library/type-aware conversion strategy.

---

# 33. Timestamps

Be deliberate about date/time handling.

For application events such as:

~~~text
created_at
paid_at
updated_at
~~~

`TIMESTAMPTZ` is commonly appropriate.

Your Node.js application should treat timestamps consistently, usually normalizing application logic around UTC and formatting for users at the presentation layer.

---

# 34. SSL in Production

Local development may connect directly to localhost.

Hosted PostgreSQL providers commonly require or recommend encrypted TLS connections.

Configuration depends on your provider.

Conceptually:

~~~text
Node.js backend
      ↓
TLS-encrypted PostgreSQL connection
      ↓
Managed PostgreSQL
~~~

Do not blindly disable certificate verification just to silence connection errors.

Use the provider's documented secure configuration.

---

# 35. Pool Size

Do not assume:

~~~text
more connections = more performance
~~~

Too many connections can hurt PostgreSQL.

Think:

~~~text
HTTP concurrency
      ↓
application pool
      ↓
controlled DB concurrency
      ↓
PostgreSQL
~~~

The correct pool size depends on:

- PostgreSQL capacity
- number of application instances
- workload
- query duration
- infrastructure/provider limits

Connection pooling is covered more deeply in Lesson 38.

---

# 36. Serverless Consideration

In serverless environments, many application instances can appear dynamically.

If every instance creates a large connection pool:

~~~text
many serverless instances
       ×
many DB connections each
       ↓
connection explosion
~~~

Use your deployment/database provider's recommended pooling architecture.

Depending on the platform, this may involve a provider pooler, PgBouncer, or a serverless-compatible driver/connection strategy.

---

# 37. ShopHub Example

Suppose ShopHub creates an order.

Architecture:

~~~text
React / Next.js UI
       ↓
Backend order service
       ↓
pg connection pool
       ↓
PostgreSQL transaction
       ↓
orders + order_items + inventory
~~~

Service flow:

~~~text
Validate request
     ↓
pool.connect()
     ↓
BEGIN
     ↓
INSERT order
     ↓
INSERT order items
     ↓
UPDATE inventory
     ↓
COMMIT
     ↓
release client
~~~

If anything fails:

~~~text
error
 ↓
ROLLBACK
 ↓
release client
 ↓
application error handling
~~~

This is the bridge between your PostgreSQL transaction knowledge and real Node.js backend code.

---

# 38. Example Order Transaction

~~~js
export async function createOrder({ userId, productId, total }) {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    const productResult = await client.query(
      `
        UPDATE products
        SET stock = stock - 1
        WHERE id = $1
          AND stock > 0
        RETURNING id, stock
      `,
      [productId]
    );

    if (productResult.rowCount === 0) {
      throw new Error('Product is out of stock');
    }

    const orderResult = await client.query(
      `
        INSERT INTO orders (user_id, total)
        VALUES ($1, $2)
        RETURNING id, user_id, total, created_at
      `,
      [userId, total]
    );

    await client.query('COMMIT');

    return orderResult.rows[0];
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
~~~

Important concepts combined:

~~~text
Pool
Parameterized SQL
Atomic conditional UPDATE
RETURNING
Transaction
ROLLBACK
Connection release
~~~

---

# Common Mistakes

## 39. Creating a New Database Connection for Every Request

Use a pool for normal backend workloads.

## 40. Creating Multiple Pools Accidentally

Centralize your pool in a database module rather than constructing a new pool throughout the application.

## 41. String-Interpolating User Input into SQL

Use parameterized queries for values.

## 42. Using pool.query() Across a Transaction

A transaction must stay on one checked-out client/connection.

## 43. Forgetting client.release()

This can leak pooled connections and eventually exhaust the pool.

## 44. Forgetting ROLLBACK

Transaction error paths must cleanly end the transaction before releasing/reusing the connection.

## 45. Hard-Coding Database Passwords

Use environment/deployment secrets.

## 46. Exposing Raw Database Errors to Clients

Log internal details securely and return controlled application errors.

## 47. Trusting Application Validation Alone

Use PostgreSQL constraints for data integrity.

## 48. Using Parameters for Identifiers

`$1` is for data values, not arbitrary SQL table/column identifiers. Use trusted allowlists for dynamic identifiers.

## 49. Calling pool.end() After Every Query

`pool.end()` closes the pool; use it during application shutdown, not ordinary request handling.

---

# Interview Revision

## What is node-postgres?

`pg` is a PostgreSQL client/driver for Node.js that allows applications to connect, send queries, manage pools, and run transactions.

## Client vs Pool?

`Client` represents a database connection. `Pool` manages reusable connections and is normally preferred for web applications.

## Why use connection pooling?

Creating PostgreSQL connections has overhead; pooling reuses a controlled number of connections across many application requests.

## What is a parameterized query?

A query where SQL structure and data values are sent separately using placeholders such as `$1`, `$2`. It is the primary SQL-injection defense for values.

## Why use RETURNING?

PostgreSQL can return inserted, updated, or deleted rows directly without requiring a separate SELECT.

## pool.query() vs pool.connect()?

Use `pool.query()` for independent queries. Use `pool.connect()` when you need one checked-out connection for a transaction or session-specific sequence of operations.

## Why must a transaction use the same client?

A PostgreSQL transaction belongs to one database session/connection. Different pooled connections cannot collectively form one transaction.

## Why use finally with client.release()?

To guarantee the checked-out connection is returned to the pool whether the operation succeeds or fails.

## Can `$1` represent a column name?

No. Parameters represent values, not SQL identifiers. Dynamic identifiers should come from trusted allowlists or appropriate identifier-safe tooling.

## Why keep DB constraints if Node validates input?

Application validation improves behavior/UX, while database constraints guarantee integrity regardless of which code path writes data.

## Why can NUMERIC come back as a string?

To avoid silently losing precision when PostgreSQL numeric values exceed what JavaScript Number can safely represent.

---

# Quick Revision

~~~text
Node.js
   ↓
pg Pool
   ↓
PostgreSQL
~~~

### Normal Query

~~~text
pool.query()
   ↓
pool chooses connection
   ↓
query
   ↓
connection automatically returned
~~~

### Transaction

~~~text
pool.connect()
   ↓
client
   ↓
BEGIN
   ↓
query 1
query 2
query 3
   ↓
COMMIT / ROLLBACK
   ↓
client.release()
~~~

### Security

~~~text
Never:
SQL + raw user input

Use:
SQL placeholders
$1, $2, $3
 +
values array
~~~

### Architecture

~~~text
Route
  ↓
Controller
  ↓
Service
  ↓
DB / Pool
  ↓
PostgreSQL
~~~

### Golden Transaction Rule

~~~text
ONE TRANSACTION
      =
ONE DATABASE CONNECTION
~~~

---

## Key Takeaway

> **In Node.js, use a shared PostgreSQL connection pool, parameterize all untrusted values, use `RETURNING` to simplify CRUD, and run every statement of a transaction through the same checked-out client with guaranteed COMMIT/ROLLBACK and `client.release()`.**

---

[← Previous: Lesson 30 — Write-Ahead Logging (WAL)](../07-postgresql-internals/30-write-ahead-logging.md) | [Back to Roadmap](../README.md) | [Next: Lesson 32 — PostgreSQL + Express API →](./32-postgresql-express-api.md)