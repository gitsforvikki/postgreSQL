# Lesson 36 — SQL Injection & Secure Queries

## Goal of This Lesson

SQL injection happens when **untrusted input is allowed to change the structure or meaning of an SQL statement**.

For a full-stack developer, the most important rule is:

> **Keep SQL structure separate from untrusted data.**

Core defense:

~~~text
Untrusted input
      ↓
Runtime validation
      ↓
Parameterized query
      ↓
PostgreSQL
~~~

---

# 1. What Is SQL Injection?

Suppose an application builds SQL using user input:

~~~js
const query = `
  SELECT id, email
  FROM users
  WHERE email = '${email}'
`;
~~~

The application expects `email` to be ordinary data.

But because it is inserted directly into the SQL string, malicious input may become part of the SQL syntax.

That is the core SQL injection problem.

---

# 2. Vulnerable Query Construction

Bad:

~~~js
const result = await pool.query(
  `SELECT id, name FROM users WHERE id = ${req.params.id}`
);
~~~

Also bad:

~~~js
const sql =
  "SELECT id, email FROM users WHERE email = '" +
  req.body.email +
  "'";
~~~

In both cases:

~~~text
SQL syntax
    +
untrusted input
    ↓
one executable SQL string
~~~

The input may influence SQL structure rather than remaining only a value.

---

# 3. Parameterized Queries

Correct Node.js `pg` pattern:

~~~js
const result = await pool.query(
  `
    SELECT id, name, email
    FROM users
    WHERE email = $1
  `,
  [email]
 );
~~~

Here:

~~~text
SQL structure
→ WHERE email = $1

Data
→ [email]
~~~

The driver sends the query structure and parameter value separately.

PostgreSQL treats the supplied value as data rather than executable SQL syntax.

---

# 4. Multiple Parameters

~~~js
const result = await pool.query(
  `
    SELECT id, name, email
    FROM users
    WHERE email = $1
      AND is_active = $2
  `,
  [email, true]
 );
~~~

Mapping:

~~~text
$1 → email
$2 → true
~~~

Use placeholders for every untrusted value.

---

# 5. INSERT Safely

Unsafe:

~~~js
const sql = `
  INSERT INTO users (name, email)
  VALUES ('${name}', '${email}')
`;
~~~

Safe:

~~~js
const result = await pool.query(
  `
    INSERT INTO users (name, email)
    VALUES ($1, $2)
    RETURNING id, name, email
  `,
  [name, email]
 );
~~~

`RETURNING` gives the created row without needing an unsafe second query.

---

# 6. UPDATE Safely

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

Both the new value and target ID are parameters.

---

# 7. DELETE Safely

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

Do not concatenate request parameters into DELETE statements.

---

# 8. Parameters Are for Values

This distinction is critical.

Parameterized values work for data:

~~~sql
WHERE id = $1
WHERE email = $1
LIMIT $1
~~~

But a parameter is not a general replacement for SQL syntax or identifiers.

For example, you cannot expect:

~~~sql
ORDER BY $1
~~~

to safely turn `$1` into a column identifier such as `created_at`.

---

# 9. Dynamic Identifiers Need an Allowlist

Suppose the API supports:

~~~text
?sort=name
?sort=price
?sort=newest
~~~

Bad:

~~~js
const sql = `
  SELECT id, name, price
  FROM products
  ORDER BY ${req.query.sort}
`;
~~~

Safe approach:

~~~js
const sortColumns = {
  name: 'name',
  price: 'price',
  newest: 'created_at',
};

const sortColumn = sortColumns[req.query.sort] ?? 'created_at';

const sql = `
  SELECT id, name, price
  FROM products
  ORDER BY ${sortColumn} DESC
`;
~~~

Only server-defined trusted identifiers enter the SQL structure.

---

# 10. Dynamic Sort Direction

Do not directly interpolate:

~~~js
`ORDER BY created_at ${req.query.direction}`
~~~

Instead map it:

~~~js
const direction =
  req.query.direction === 'asc' ? 'ASC' : 'DESC';
~~~

Now the SQL syntax comes from controlled application values.

---

# 11. Dynamic Table Names

Parameters cannot represent table names either.

Bad:

~~~js
const sql = `SELECT * FROM ${req.query.table}`;
~~~

Prefer fixed application queries.

If dynamic identifiers are truly required:

~~~text
1. Define allowed identifiers
2. Map user choices to trusted identifiers
3. Use identifier-safe tooling when necessary
4. Never concatenate arbitrary user text
~~~

Most APIs do not need arbitrary client-selected table names.

---

# 12. Safe IN Queries

A common requirement:

~~~text
Get users with IDs 1, 5, 8
~~~

Do not manually build an untrusted comma-separated SQL list.

With PostgreSQL, one clean pattern is:

~~~js
const result = await pool.query(
  `
    SELECT id, name
    FROM users
    WHERE id = ANY($1::bigint[])
  `,
  [[1, 5, 8]]
 );
~~~

Or:

~~~js
const ids = req.body.ids;

const result = await pool.query(
  `SELECT id, name FROM users WHERE id = ANY($1::bigint[])`,
  [ids]
 );
~~~

Validate the array contents before querying.

---

# 13. Safe Search with LIKE / ILIKE

Suppose a user searches for:

~~~text
phone
~~~

Safe:

~~~js
const result = await pool.query(
  `
    SELECT id, name
    FROM products
    WHERE name ILIKE $1
  `,
  [`%${search}%`]
 );
~~~

The `%` wildcard is part of the **parameter value**, not concatenated into SQL syntax.

---

# 14. LIKE Wildcards Are Not SQL Injection

`%` and `_` have pattern meaning inside `LIKE`/`ILIKE`.

That is different from SQL injection.

Parameterized input still remains a value.

However, if your application wants `%` or `_` to mean literal characters rather than wildcards, you may need to escape them for **search semantics**.

That is a separate concern from SQL injection prevention.

---

# 15. LIMIT and OFFSET

These can be parameterized:

~~~js
const result = await pool.query(
  `
    SELECT id, name
    FROM products
    ORDER BY created_at DESC
    LIMIT $1 OFFSET $2
  `,
  [limit, offset]
 );
~~~

But still validate them:

~~~js
const limit = Math.min(
  Math.max(Number(req.query.limit) || 20, 1),
  100
);
~~~

Why validate if parameterization is already safe?

~~~text
Parameterization
→ protects SQL structure

Validation
→ protects application/business/resource rules
~~~

---

# 16. Parameterization vs Validation

These are different security controls.

Suppose:

~~~text
price = -100000
~~~

A parameterized query may safely treat it as a number.

But your business rules may still reject it.

~~~text
Parameterized SQL
→ prevents input becoming SQL syntax

Validation
→ checks whether the value is acceptable
~~~

You need both.

---

# 17. Parameterization vs Authorization

Another important distinction.

Safe SQL:

~~~js
await pool.query(
  'DELETE FROM orders WHERE id = $1',
  [orderId]
 );
~~~

This may be protected from SQL injection.

But can the current user delete **any** order?

Security requires authorization too.

Better:

~~~js
await pool.query(
  `
    DELETE FROM orders
    WHERE id = $1
      AND user_id = $2
    RETURNING id
  `,
  [orderId, authenticatedUserId]
 );
~~~

Now ownership is enforced in the operation.

---

# 18. Authentication, Authorization and SQL Safety

Think in layers:

~~~text
Authentication
→ Who is calling?
      ↓
Authorization
→ May they perform this operation?
      ↓
Validation
→ Is the input acceptable?
      ↓
Parameterized SQL
→ Can input alter SQL structure?
      ↓
DB constraints
→ Is stored data valid?
~~~

One layer does not replace another.

---

# 19. Express Example

Route:

~~~js
router.get('/users/:id', requireAuth, getUserById);
~~~

Controller:

~~~js
export async function getUserById(req, res, next) {
  try {
    const user = await userService.getUserById(
      req.params.id,
      req.user.id
    );

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    return res.json({ data: user });
  } catch (error) {
    next(error);
  }
}
~~~

Service:

~~~js
export async function getUserById(id, authenticatedUserId) {
  const result = await pool.query(
    `
      SELECT id, name, email
      FROM users
      WHERE id = $1
        AND id = $2
    `,
    [id, authenticatedUserId]
  );

  return result.rows[0] ?? null;
}
~~~

The example combines authorization and parameterized SQL.

---

# 20. Next.js Route Handler Example

~~~ts
import { NextRequest, NextResponse } from 'next/server';
import { pool } from '@/lib/db';

export async function GET(request: NextRequest) {
  const search = request.nextUrl.searchParams.get('search') ?? '';

  const result = await pool.query(
    `
      SELECT id, name, price
      FROM products
      WHERE name ILIKE $1
      LIMIT $2
    `,
    [`%${search}%`, 20]
  );

  return NextResponse.json({ data: result.rows });
}
~~~

Next.js does not automatically make raw SQL safe.

The same PostgreSQL rules apply.

---

# 21. Next.js Server Action Example

~~~ts
'use server';

import { pool } from '@/lib/db';

export async function updateProfile(formData: FormData) {
  const name = String(formData.get('name') ?? '').trim();
  const userId = await requireAuthenticatedUserId();

  if (name.length < 2 || name.length > 100) {
    throw new Error('Invalid name');
  }

  await pool.query(
    `
      UPDATE users
      SET name = $1
      WHERE id = $2
    `,
    [name, userId]
  );
}
~~~

Important:

~~~text
Server Action
≠ automatically authorized
≠ automatically validated
≠ automatically SQL-injection-proof if you write unsafe raw SQL
~~~

---

# 22. ORM Safety

ORM/query-builder APIs generally help keep values parameterized when used normally.

Example conceptual Drizzle query:

~~~ts
await db
  .select()
  .from(users)
  .where(eq(users.email, email));
~~~

Example Prisma query:

~~~ts
await prisma.user.findUnique({
  where: { email },
});
~~~

These structured APIs reduce the need to manually build SQL strings.

But:

> **Using an ORM does not make every possible query automatically safe.**

Raw SQL escape hatches and dynamic query construction still require care.

---

# 23. Unsafe Raw SQL Inside an ORM

Conceptually dangerous:

~~~ts
const query = `SELECT * FROM users WHERE email = '${email}'`;
// execute as raw SQL
~~~

The fact that an ORM executes the string does not remove the vulnerability.

Safe ORM raw-query APIs should use their documented parameter/tag mechanisms rather than concatenating untrusted input.

---

# 24. Manual Escaping Is Not the Primary Defense

Do not build your security strategy around:

~~~text
replace single quotes
remove semicolons
block certain words
strip '--'
~~~

Attack syntax can be more complex than a simple denylist.

Primary rule:

~~~text
Use parameterization
      +
allowlist identifiers
~~~

Manual string filtering is not a substitute for correct query construction.

---

# 25. Stored Procedures and Functions

Database functions are not automatically safe from SQL injection.

Safe static SQL:

~~~sql
SELECT id, email
FROM users
WHERE id = p_user_id;
~~~

can be safe when values are handled normally.

But a function that constructs dynamic SQL from untrusted text can introduce the same problem.

Conceptually:

~~~text
Dynamic SQL
   +
untrusted string
   ↓
possible SQL injection
~~~

If dynamic SQL is required in PL/pgSQL, use PostgreSQL's safe formatting/parameter mechanisms and strict identifier handling.

---

# 26. Least Privilege Reduces Blast Radius

Suppose an application vulnerability exists.

If the application connects as:

~~~text
SUPERUSER
~~~

the potential impact is enormous.

If it connects as:

~~~text
runtime role
→ SELECT/INSERT/UPDATE only required tables
→ no schema ownership
→ no CREATE ROLE
→ no DROP DATABASE
~~~

the damage that database credentials/query execution can cause is more limited.

Least privilege does **not prevent SQL injection**, but it can reduce impact.

---

# 27. Database Constraints as Defense in Depth

Suppose an attacker or buggy code attempts:

~~~text
quantity = -500
~~~

A database constraint can reject invalid data:

~~~sql
CHECK (quantity > 0)
~~~

Security/integrity layers:

~~~text
Validation
      ↓
Parameterized query
      ↓
Authorization
      ↓
Database constraints
~~~

Constraints do not replace injection protection, but they protect integrity.

---

# 28. Do Not Expose Raw Database Errors

Bad API response:

~~~text
duplicate key value violates unique constraint users_email_key
DETAIL: Key (email)=(...) already exists
internal table/schema details...
~~~

Better public response:

~~~json
{
  "message": "Email already registered"
}
~~~

Log appropriate internal details securely on the server.

Return controlled errors to clients.

---

# 29. Logging Sensitive Data

Do not casually log:

~~~text
database passwords
DATABASE_URL
access tokens
session cookies
full payment data
sensitive personal data
~~~

Security is not only query construction.

Logs can become another source of credential/data leakage.

---

# 30. SQL Injection vs XSS

These are different vulnerabilities.

~~~text
SQL Injection
→ attacks database query construction

XSS
→ injects script/content into browser-rendered output
~~~

Parameterized SQL protects against SQL injection.

It does not automatically protect the browser from XSS.

Use the correct defense for each layer.

---

# 31. SQL Injection vs Command Injection

Also different:

~~~text
SQL injection
→ database query interpreter

Command injection
→ operating-system shell/command interpreter
~~~

General principle:

> Never let untrusted data become executable syntax in an interpreter.

Use context-appropriate safe APIs.

---

# 32. Second-Order SQL Injection — High Level

Sometimes malicious-looking text is safely stored as ordinary data first.

Later, another part of the application reads that stored value and unsafely concatenates it into dynamic SQL.

~~~text
Input
 ↓ safely stored
Database value
 ↓ later concatenated into SQL
Dynamic SQL
 ↓
vulnerability
~~~

Lesson:

> Treat data as data even if it originally came from your own database.

Do not assume stored strings are automatically safe SQL syntax.

---

# 33. Prepared Statements

Prepared statements and parameterized queries are related concepts, but do not confuse performance features with the fundamental security rule.

Example node-postgres named query:

~~~js
await pool.query({
  name: 'user-by-email',
  text: 'SELECT id, email FROM users WHERE email = $1',
  values: [email],
});
~~~

The security benefit comes from keeping the value parameterized.

Naming/preparing the query can have execution/reuse implications but is not required merely to parameterize values.

---

# 34. Transactions Do Not Prevent SQL Injection

Wrapping unsafe SQL in:

~~~sql
BEGIN;
...
COMMIT;
~~~

does not make it safe.

Transactions solve atomicity/isolation problems.

Parameterization solves SQL syntax/data separation.

Different concerns.

---

# 35. HTTPS Does Not Prevent SQL Injection

HTTPS encrypts traffic between client and server.

It does not fix unsafe server-side SQL construction.

~~~text
HTTPS
→ transport security

Parameterized SQL
→ query construction security
~~~

Both are needed for different reasons.

---

# 36. Input Length and Resource Abuse

Even parameterized queries can be expensive.

Example:

~~~text
search string = enormous input
limit = 1,000,000
complex filter = expensive query
~~~

Parameterization prevents SQL injection, but applications should also:

~~~text
validate sizes
cap pagination
use timeouts where appropriate
rate limit exposed APIs
design indexes
monitor expensive queries
~~~

Security includes availability as well as confidentiality/integrity.

---

# 37. ShopHub Search Example

Request:

~~~text
GET /api/products?search=phone&sort=price&direction=asc
~~~

Secure construction:

~~~js
const search = String(req.query.search ?? '').slice(0, 100);

const sortMap = {
  price: 'price',
  name: 'name',
  newest: 'created_at',
};

const sortColumn = sortMap[req.query.sort] ?? 'created_at';
const direction = req.query.direction === 'asc' ? 'ASC' : 'DESC';

const result = await pool.query(
  `
    SELECT id, name, price
    FROM products
    WHERE name ILIKE $1
    ORDER BY ${sortColumn} ${direction}
    LIMIT $2
  `,
  [`%${search}%`, 20]
 );
~~~

Why is this safe?

~~~text
search
→ parameter

limit
→ parameter

sortColumn
→ trusted allowlist value

direction
→ trusted server-selected ASC/DESC
~~~

No arbitrary client text becomes SQL syntax.

---

# 38. ShopHub Order Authorization Example

Bad:

~~~js
await pool.query(
  'SELECT * FROM orders WHERE id = $1',
  [orderId]
 );
~~~

if any logged-in user can supply any order ID.

Better:

~~~js
const result = await pool.query(
  `
    SELECT id, status, total, created_at
    FROM orders
    WHERE id = $1
      AND user_id = $2
  `,
  [orderId, authenticatedUserId]
 );
~~~

This combines:

~~~text
SQL injection protection
      +
object-level authorization
~~~

---

# 39. CareerLoop Example

Suppose CareerLoop supports:

~~~text
GET /api/applications?status=interview&sort=newest
~~~

Safe architecture:

~~~text
Next.js Route Handler
      ↓
authenticate current user
      ↓
validate status
      ↓
map sort to trusted column
      ↓
parameterized WHERE user_id = $1 AND status = $2
      ↓
PostgreSQL
~~~

Example:

~~~ts
const result = await pool.query(
  `
    SELECT id, company, role, status, created_at
    FROM applications
    WHERE user_id = $1
      AND status = $2
    ORDER BY created_at DESC
    LIMIT $3
  `,
  [userId, status, 50]
 );
~~~

The user's identity comes from trusted authentication context, not from a client-provided `userId`.

---

# 40. Secure Query Checklist

Before shipping a database query, ask:

~~~text
□ Is every untrusted VALUE parameterized?
□ Are dynamic identifiers allowlisted?
□ Is request input validated?
□ Is the caller authenticated where required?
□ Is the operation authorized?
□ Are DB constraints present?
□ Does the DB role have least privilege?
□ Are sensitive columns excluded from results?
□ Are errors controlled?
□ Are limits bounded?
~~~

If these are consistently answered, your data-access layer becomes much safer.

---

# Common Mistakes

## 41. Concatenating Request Data into SQL

Never directly concatenate `req.body`, `req.params`, `req.query`, FormData, cookies, headers, or other untrusted strings into SQL syntax.

## 42. Thinking Validation Alone Prevents Injection

Validation is useful, but parameterization is the primary defense for values.

## 43. Trying to Parameterize Column Names

Parameters are for values. Use allowlists for dynamic identifiers.

## 44. Assuming ORM Means Injection Is Impossible

Unsafe raw SQL and dynamic query construction can still create vulnerabilities.

## 45. Trusting Data Because It Came from the Database

Stored data can become dangerous if later inserted into dynamic SQL syntax.

## 46. Using a Superuser Application Role

Least privilege reduces the potential blast radius.

## 47. Forgetting Authorization

A perfectly parameterized query can still expose another user's data if ownership/permissions are not checked.

## 48. Exposing Raw PostgreSQL Errors

Return controlled application messages.

## 49. Assuming HTTPS Solves SQL Injection

HTTPS protects transport, not SQL construction.

## 50. Building Manual SQL Denylists

Do not depend on blocking quotes, semicolons, or keywords. Use safe query construction.

---

# Interview Revision

## What is SQL injection?

A vulnerability where untrusted input is interpreted as part of SQL syntax and changes the intended database query.

## What is the primary defense?

Parameterized queries/prepared parameter mechanisms that keep SQL structure separate from data values.

## What does `$1` mean in node-postgres?

It is a positional parameter placeholder whose value is supplied separately, for example `[email]`.

## Can `$1` represent a table or column name?

No. Parameters represent data values, not SQL identifiers. Dynamic identifiers should come from trusted allowlists or appropriate identifier-safe tooling.

## Is input validation enough?

No. Validation checks whether data is acceptable; parameterization prevents it from becoming SQL syntax.

## Does an ORM prevent all SQL injection?

No. Normal structured APIs help, but unsafe raw SQL or dynamic query construction can still introduce injection.

## Does authentication prevent SQL injection?

No. Authentication identifies the caller. Query construction must still be safe.

## Does parameterization replace authorization?

No. A query can be injection-safe and still expose or modify data the caller is not allowed to access.

## Why use least privilege?

It limits what compromised application credentials or exploitable query paths can do.

## Are stored procedures automatically safe?

No. Procedures/functions that build unsafe dynamic SQL can also be vulnerable.

## Why avoid raw DB errors in API responses?

They can leak internal schema, constraint, and implementation details and create poor security boundaries.

---

# Quick Revision

~~~text
BAD
User input
   ↓
string concatenation
   ↓
SQL syntax + data mixed
   ↓
SQL injection risk
~~~

~~~text
GOOD
User input
   ↓
validation
   ↓
parameter value
   ↓
SQL structure stays fixed
   ↓
PostgreSQL
~~~

### Values

~~~text
email
userId
price
limit
search text
     ↓
$1, $2, $3 ...
~~~

### Identifiers

~~~text
column
table
sort direction
     ↓
trusted allowlist / safe identifier handling
~~~

### Full Security Model

~~~text
Authentication
      ↓
Authorization
      ↓
Validation
      ↓
Parameterized SQL
      ↓
Least-privilege DB role
      ↓
PostgreSQL constraints
~~~

### Golden Rule

~~~text
UNTRUSTED DATA
must remain
DATA
not
SQL SYNTAX
~~~

---

## Key Takeaway

> **Prevent SQL injection by keeping SQL structure separate from untrusted values through parameterized queries. Validate inputs for business/resource rules, allowlist dynamic identifiers, enforce authorization independently, use least-privilege database roles, retain PostgreSQL constraints, and never assume an ORM, HTTPS, authentication, or a transaction makes unsafe SQL construction safe.**

---

[← Previous: Lesson 35 — PostgreSQL Security](./35-postgresql-security.md) | [Back to Roadmap](../README.md) | [Next: Lesson 37 — PostgreSQL Configuration →](../10-production/37-postgresql-configuration.md)