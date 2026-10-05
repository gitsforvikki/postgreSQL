# Lesson 32 — PostgreSQL + Express API

## Goal of This Lesson

In Lesson 31, you connected Node.js directly to PostgreSQL using `pg`.

Now we place that database layer inside a clean Express API architecture.

~~~text
Frontend / Client
       ↓ HTTP
Express Route
       ↓
Controller
       ↓
Service
       ↓
pg Pool / Client
       ↓
PostgreSQL
~~~

The main goal is not only to make CRUD work, but to understand **where each responsibility belongs**.

---

# 1. Recommended Project Structure

One clean structure is:

~~~text
src/
├── config/
│   └── env.js
├── db/
│   └── index.js
├── routes/
│   └── user.routes.js
├── controllers/
│   └── user.controller.js
├── services/
│   └── user.service.js
├── validators/
│   └── user.validator.js
├── middlewares/
│   ├── auth.middleware.js
│   └── error.middleware.js
├── app.js
└── server.js
~~~

This is one possible architecture, not a PostgreSQL requirement.

The important idea is **separation of concerns**.

---

# 2. Responsibilities

~~~text
Route
→ Which URL + HTTP method calls which handler?

Controller
→ Read HTTP request and build HTTP response

Service
→ Business logic and database workflow

DB module
→ PostgreSQL connection infrastructure

Middleware
→ Shared request pipeline behavior
~~~

Keep this mental model:

~~~text
HTTP concerns
     ↓
Controller
     ↓
Business concerns
     ↓
Service
     ↓
Database concerns
     ↓
PostgreSQL
~~~

---

# 3. Install Dependencies

~~~bash
npm install express pg dotenv
~~~

A real project may also use a validation library, authentication library, logging library, security middleware, and other packages.

For this lesson, we keep the focus on Express + PostgreSQL.

---

# 4. Environment Variables

`.env`:

~~~env
PORT=3000
DATABASE_URL=postgresql://username:password@localhost:5432/shophub
~~~

Never commit real production credentials.

Load environment configuration according to your project's runtime/module setup.

---

# 5. PostgreSQL Pool

`src/db/index.js`:

~~~js
import pg from 'pg';

const { Pool } = pg;

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

pool.on('error', (error) => {
  console.error('Unexpected PostgreSQL pool error', error);
});
~~~

Create the pool once and reuse it.

~~~text
Express requests
      ↓
shared Pool
      ↓
controlled PostgreSQL connections
~~~

---

# 6. Express Application

`src/app.js`:

~~~js
import express from 'express';
import userRouter from './routes/user.routes.js';
import { errorHandler } from './middlewares/error.middleware.js';

export const app = express();

app.use(express.json());

app.use('/api/users', userRouter);

app.use(errorHandler);
~~~

`express.json()` parses JSON request bodies.

The error middleware is registered after routes so errors can flow to it.

---

# 7. Starting the Server

`src/server.js`:

~~~js
import 'dotenv/config';
import { app } from './app.js';

const port = Number(process.env.PORT) || 3000;

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
~~~

Why separate `app.js` and `server.js`?

~~~text
app.js
→ configure Express application

server.js
→ process startup / listen
~~~

This separation can make testing and application lifecycle management cleaner.

---

# 8. User Routes

`src/routes/user.routes.js`:

~~~js
import { Router } from 'express';
import {
  createUser,
  getUsers,
  getUserById,
  updateUser,
  deleteUser,
} from '../controllers/user.controller.js';

const router = Router();

router.post('/', createUser);
router.get('/', getUsers);
router.get('/:id', getUserById);
router.patch('/:id', updateUser);
router.delete('/:id', deleteUser);

export default router;
~~~

REST mapping:

~~~text
POST   /api/users      → create
GET    /api/users      → list
GET    /api/users/:id  → read one
PATCH  /api/users/:id  → update
DELETE /api/users/:id  → delete
~~~

---

# 9. Controller vs Service

A common beginner mistake is putting everything in the route/controller:

~~~text
parse request
validate
business logic
SQL
transaction
HTTP response
~~~

That becomes hard to maintain.

Instead:

~~~text
Controller
→ HTTP layer

Service
→ application/database workflow
~~~

Example:

~~~js
export async function getUserById(req, res, next) {
  try {
    const user = await userService.getUserById(req.params.id);

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    return res.status(200).json({ data: user });
  } catch (error) {
    next(error);
  }
}
~~~

The controller does not need to know the SQL.

---

# 10. Service Layer

`src/services/user.service.js`:

~~~js
import { pool } from '../db/index.js';

export async function getUserById(id) {
  const result = await pool.query(
    `
      SELECT id, name, email, created_at
      FROM users
      WHERE id = $1
    `,
    [id]
  );

  return result.rows[0] ?? null;
}
~~~

Flow:

~~~text
Controller
   ↓
userService.getUserById()
   ↓
pool.query()
   ↓
PostgreSQL
~~~

---

# 11. POST — Create User

Controller:

~~~js
export async function createUser(req, res, next) {
  try {
    const { name, email } = req.body;

    if (!name || !email) {
      return res.status(400).json({
        message: 'Name and email are required',
      });
    }

    const user = await userService.createUser({ name, email });

    return res.status(201).json({ data: user });
  } catch (error) {
    next(error);
  }
}
~~~

Service:

~~~js
export async function createUser({ name, email }) {
  const result = await pool.query(
    `
      INSERT INTO users (name, email)
      VALUES ($1, $2)
      RETURNING id, name, email, created_at
    `,
    [name, email]
  );

  return result.rows[0];
}
~~~

Important concepts:

~~~text
req.body
→ validation
→ service
→ parameterized INSERT
→ RETURNING
→ HTTP 201
~~~

---

# 12. GET — List Users

Controller:

~~~js
export async function getUsers(req, res, next) {
  try {
    const users = await userService.getUsers();

    return res.status(200).json({ data: users });
  } catch (error) {
    next(error);
  }
}
~~~

Service:

~~~js
export async function getUsers() {
  const result = await pool.query(`
    SELECT id, name, email, created_at
    FROM users
    ORDER BY created_at DESC
  `);

  return result.rows;
}
~~~

For real production tables, add pagination rather than returning an unlimited dataset.

---

# 13. PATCH — Update User

Controller:

~~~js
export async function updateUser(req, res, next) {
  try {
    const { id } = req.params;
    const { name } = req.body;

    if (!name) {
      return res.status(400).json({ message: 'Name is required' });
    }

    const user = await userService.updateUser(id, name);

    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }

    return res.status(200).json({ data: user });
  } catch (error) {
    next(error);
  }
}
~~~

Service:

~~~js
export async function updateUser(id, name) {
  const result = await pool.query(
    `
      UPDATE users
      SET name = $1
      WHERE id = $2
      RETURNING id, name, email, created_at
    `,
    [name, id]
  );

  return result.rows[0] ?? null;
}
~~~

`RETURNING` avoids a second SELECT.

---

# 14. DELETE — Delete User

Service:

~~~js
export async function deleteUser(id) {
  const result = await pool.query(
    `
      DELETE FROM users
      WHERE id = $1
      RETURNING id
    `,
    [id]
  );

  return result.rows[0] ?? null;
}
~~~

Controller can return:

~~~text
404 → user did not exist
204 → successfully deleted with no response body
~~~

Example:

~~~js
export async function deleteUser(req, res, next) {
  try {
    const deleted = await userService.deleteUser(req.params.id);

    if (!deleted) {
      return res.status(404).json({ message: 'User not found' });
    }

    return res.status(204).send();
  } catch (error) {
    next(error);
  }
}
~~~

---

# 15. req.params, req.query, req.body

These are fundamental Express concepts.

### Path Parameter

~~~text
GET /api/users/42
~~~

~~~js
req.params.id
~~~

### Query Parameters

~~~text
GET /api/users?page=2&limit=20
~~~

~~~js
req.query.page
req.query.limit
~~~

### Request Body

~~~http
POST /api/users

{
  "name": "Vikash",
  "email": "vikash@example.com"
}
~~~

~~~js
req.body.name
req.body.email
~~~

---

# 16. Pagination

Controller:

~~~js
const page = Math.max(Number(req.query.page) || 1, 1);
const requestedLimit = Number(req.query.limit) || 20;
const limit = Math.min(Math.max(requestedLimit, 1), 100);
const offset = (page - 1) * limit;
~~~

Service:

~~~js
const result = await pool.query(
  `
    SELECT id, name, email, created_at
    FROM users
    ORDER BY created_at DESC
    LIMIT $1 OFFSET $2
  `,
  [limit, offset]
 );
~~~

Why cap the limit?

~~~text
Client asks limit=1000000
        ↓
without cap
        ↓
huge DB/API response
~~~

Always bound public API page sizes.

For large/deep datasets, consider keyset pagination from Lesson 26.

---

# 17. Filtering

Request:

~~~text
GET /api/users?active=true
~~~

Service:

~~~js
const result = await pool.query(
  `
    SELECT id, name, email
    FROM users
    WHERE is_active = $1
    ORDER BY created_at DESC
  `,
  [isActive]
 );
~~~

Do not fetch every row and filter in JavaScript when PostgreSQL can efficiently perform the filter.

---

# 18. Search

Request:

~~~text
GET /api/users?search=vik
~~~

Safe SQL:

~~~js
const result = await pool.query(
  `
    SELECT id, name, email
    FROM users
    WHERE name ILIKE $1
       OR email ILIKE $1
    LIMIT $2
  `,
  [`%${search}%`, limit]
 );
~~~

The wildcard is inside the parameter value, so untrusted search text does not become SQL syntax.

---

# 19. Dynamic Sorting

Request:

~~~text
GET /api/users?sort=name
~~~

You cannot safely treat an arbitrary user string as a SQL identifier parameter.

Use an allowlist:

~~~js
const sortColumns = {
  name: 'name',
  newest: 'created_at',
};

const sortColumn = sortColumns[req.query.sort] ?? 'created_at';
~~~

Then the service can construct SQL using only that trusted value.

~~~js
const query = `
  SELECT id, name, email, created_at
  FROM users
  ORDER BY ${sortColumn} DESC
  LIMIT $1
`;

const result = await pool.query(query, [limit]);
~~~

Never do:

~~~js
`ORDER BY ${req.query.sort}`
~~~

with arbitrary user input.

---

# 20. Input Validation

Validation should happen before business/database operations.

Examples:

~~~text
email format
required fields
string lengths
numeric ranges
enum values
pagination bounds
~~~

Architecture:

~~~text
Request
  ↓
Validation
  ↓
Controller / Service
  ↓
Database constraints
~~~

Use a validation library in larger projects if appropriate.

Important:

> Validation improves request handling, but database constraints remain the final integrity layer.

---

# 21. Defense in Depth

For an email:

~~~text
Frontend
→ user-friendly validation

Express/API
→ trusted server validation

SQL
→ parameterized query

PostgreSQL
→ NOT NULL + UNIQUE constraint
~~~

Each layer solves a different problem.

---

# 22. HTTP Status Codes

Useful API status codes:

| Status | Typical meaning |
|---:|---|
| 200 | Successful read/update |
| 201 | Resource created |
| 204 | Successful response with no body |
| 400 | Invalid request |
| 401 | Authentication required/invalid |
| 403 | Authenticated but forbidden |
| 404 | Resource not found |
| 409 | Conflict, such as duplicate unique value |
| 422 | Semantically invalid input in APIs that use this convention |
| 500 | Unexpected server error |

Choose a consistent API convention.

---

# 23. Centralized Error Handling

Instead of repeating error-response logic everywhere, use Express error middleware.

`src/middlewares/error.middleware.js`:

~~~js
export function errorHandler(error, req, res, next) {
  console.error(error);

  return res.status(500).json({
    message: 'Internal server error',
  });
}
~~~

Flow:

~~~text
Controller/Service throws
       ↓
next(error)
       ↓
Error middleware
       ↓
controlled HTTP response
~~~

Do not send raw PostgreSQL stack traces or credentials to clients.

---

# 24. Mapping PostgreSQL Errors

Suppose email has a UNIQUE constraint.

PostgreSQL may return SQLSTATE:

~~~text
23505
~~~

for a unique violation.

You can translate it into HTTP 409:

~~~js
export function errorHandler(error, req, res, next) {
  if (error.code === '23505') {
    return res.status(409).json({
      message: 'Resource already exists',
    });
  }

  console.error(error);

  return res.status(500).json({
    message: 'Internal server error',
  });
}
~~~

Production code may use custom application error classes to avoid coupling all HTTP logic directly to database codes.

---

# 25. Authentication Middleware

Authentication should generally happen before protected controllers execute.

~~~text
Request
  ↓
Auth middleware
  ↓
identity established
  ↓
Controller
  ↓
Service
  ↓
PostgreSQL
~~~

Example route:

~~~js
router.get('/me', requireAuth, getCurrentUser);
~~~

Middleware might set:

~~~js
req.user = {
  id: authenticatedUserId,
};
~~~

Then the controller/service can use that trusted authenticated identity.

---

# 26. Authentication Is Not Authorization

Authentication:

~~~text
Who are you?
~~~

Authorization:

~~~text
Are you allowed to perform this action?
~~~

Example:

~~~text
Authenticated user
       ↓
PATCH /orders/123
       ↓
Does order 123 belong to this user?
       ↓
yes → allow
no  → reject
~~~

Never trust a user ID from the request body to prove ownership.

Use the authenticated identity and enforce ownership/role rules on the server.

---

# 27. Secure Ownership Query

Instead of:

~~~sql
SELECT *
FROM orders
WHERE id = $1;
~~~

for a user-owned resource, you may enforce ownership in SQL:

~~~sql
SELECT id, status, total
FROM orders
WHERE id = $1
  AND user_id = $2;
~~~

Values:

~~~text
$1 → requested order ID
$2 → authenticated user ID
~~~

This can keep authorization close to the data access.

---

# 28. Transactions in Express

Suppose checkout requires:

~~~text
1. create order
2. create order items
3. decrement inventory
~~~

These operations should succeed or fail together.

Use one checked-out PostgreSQL client:

~~~js
const client = await pool.connect();

try {
  await client.query('BEGIN');

  // query 1
  // query 2
  // query 3

  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
~~~

Never spread a transaction across independent `pool.query()` calls.

---

# 29. Order Transaction Service

Example:

~~~js
export async function createOrder({ userId, items }) {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    const orderResult = await client.query(
      `
        INSERT INTO orders (user_id, status)
        VALUES ($1, 'pending')
        RETURNING id, user_id, status, created_at
      `,
      [userId]
    );

    const order = orderResult.rows[0];

    for (const item of items) {
      const stockResult = await client.query(
        `
          UPDATE products
          SET stock = stock - $1
          WHERE id = $2
            AND stock >= $1
          RETURNING id, stock
        `,
        [item.quantity, item.productId]
      );

      if (stockResult.rowCount === 0) {
        throw new Error('Insufficient stock');
      }

      await client.query(
        `
          INSERT INTO order_items
            (order_id, product_id, quantity, unit_price)
          VALUES ($1, $2, $3, $4)
        `,
        [order.id, item.productId, item.quantity, item.unitPrice]
      );
    }

    await client.query('COMMIT');
    return order;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
~~~

This is conceptually correct transaction structure, though a production checkout should also calculate authoritative prices on the server and may batch operations rather than issuing one query per item.

---

# 30. Do Not Hold DB Transactions Open During External APIs

Bad checkout flow:

~~~text
BEGIN
 ↓
lock/decrement stock
 ↓
call payment provider
 ↓
wait 5 seconds
 ↓
COMMIT
~~~

This keeps:

~~~text
DB connection occupied
locks held longer
transaction snapshot open
~~~

External APIs cannot participate atomically in a normal PostgreSQL transaction.

Design payment/order workflows using appropriate state transitions, idempotency, verification/webhooks, and compensation/reservation strategies instead of holding a database transaction open across slow network calls.

---

# 31. N+1 Problem in Express

Bad endpoint:

~~~js
const users = await getUsers();

for (const user of users) {
  user.orders = await getOrdersByUser(user.id);
}
~~~

If 100 users are returned:

~~~text
1 users query
100 order queries
=
101 queries
~~~

Possible fixes:

- JOIN when appropriate
- batch with `ANY($1::bigint[])`
- aggregate
- redesign response shape

Do not automatically create one giant JOIN either; measure the complete data-fetching strategy.

---

# 32. API Response Shape

Do not expose raw database rows simply because they exist.

Database row:

~~~text
id
email
password_hash
internal_flags
created_at
updated_at
~~~

Public response may be:

~~~json
{
  "id": 10,
  "name": "Vikash",
  "email": "vikash@example.com"
}
~~~

Return only fields the client should receive.

---

# 33. Avoid SELECT * in Public APIs

Prefer:

~~~sql
SELECT id, name, email
FROM users
WHERE id = $1;
~~~

instead of:

~~~sql
SELECT *
FROM users
WHERE id = $1;
~~~

Benefits:

~~~text
less data
clear API contract
reduced accidental sensitive-field exposure
easier optimization
~~~

---

# 34. Service Reuse

Suppose both:

~~~text
REST endpoint
background job
~~~

need the same business operation.

If logic lives in the service:

~~~text
HTTP Controller ─┐
                 ├→ Service → PostgreSQL
Background Job ──┘
~~~

the business logic can be reused without pretending the background job is an HTTP request.

This is one reason controller/service separation is useful.

---

# 35. Controller Should Stay Thin

A healthy controller often looks like:

~~~text
read request
validate/normalize HTTP inputs
call service
map result to HTTP response
forward errors
~~~

Not:

~~~text
500 lines of SQL
payment integration
email sending
inventory algorithm
authorization scattered everywhere
~~~

Business logic belongs in reusable server-side layers.

---

# 36. Complete Request Flow

Example:

~~~text
POST /api/users
       ↓
Express router
       ↓
createUser controller
       ↓
validate body
       ↓
userService.createUser
       ↓
pool.query(
 INSERT ... VALUES ($1,$2) RETURNING ...
)
       ↓
PostgreSQL constraints
       ↓
returned row
       ↓
controller
       ↓
HTTP 201 JSON
~~~

This is the complete mental model to remember.

---

# 37. Production ShopHub Flow

~~~text
Browser
   ↓ HTTPS
Express API
   ↓
Auth middleware
   ↓
Controller
   ↓
Order Service
   ↓
PostgreSQL transaction
   ├── create order
   ├── insert items
   └── update inventory
   ↓
COMMIT
   ↓
HTTP response
~~~

Database errors flow in the opposite direction:

~~~text
PostgreSQL error
   ↓
Service throws
   ↓
Controller next(error)
   ↓
Error middleware
   ↓
controlled HTTP error
~~~

---

# 38. Security Checklist

~~~text
✓ HTTPS in production
✓ DATABASE_URL stored as secret
✓ Parameterized SQL
✓ Input validation
✓ Authentication middleware
✓ Authorization checks
✓ PostgreSQL constraints
✓ Least-privilege DB role
✓ Controlled error responses
✓ Pagination limits
✓ Allowlisted dynamic identifiers
✓ No sensitive fields in responses
~~~

Security is layered.

---

# Common Mistakes

## 39. SQL Directly in Every Route

This creates duplication and tightly couples HTTP handling to database logic.

## 40. Putting All Business Logic in Controllers

Keep controllers focused on HTTP concerns and move reusable workflows into services.

## 41. Returning Unlimited Rows

Use bounded pagination for collection endpoints.

## 42. Filtering Large Results in JavaScript

Push appropriate relational filtering into PostgreSQL.

## 43. Directly Interpolating req.query into ORDER BY

Use a trusted allowlist for SQL identifiers.

## 44. Trusting req.body.userId for Ownership

Use authenticated server-side identity and authorization rules.

## 45. Exposing Raw DB Errors

Translate/log errors safely.

## 46. Using pool.query() for Multi-Statement Transactions

Use one checked-out client.

## 47. Calling External APIs Inside a Long DB Transaction

Keep DB transactions short and design cross-system workflows explicitly.

## 48. Ignoring N+1 Queries

Measure endpoint query counts and batch/join appropriately.

## 49. Returning SELECT * from User Tables

Explicitly select safe fields.

---

# Interview Revision

## What is the request flow in an Express + PostgreSQL application?

Route → controller → service → database layer/pg → PostgreSQL, then the result flows back to the controller and HTTP response.

## What does a route do?

It maps an HTTP method/path to middleware and a handler.

## What does a controller do?

It handles HTTP-specific concerns such as request inputs, response status/body, and forwarding errors.

## What does a service do?

It contains reusable business logic and database workflows independent of HTTP details.

## Why separate controller and service?

To reduce coupling, improve reuse/testability, and keep HTTP logic separate from business/database workflows.

## How do you prevent SQL injection?

Use parameterized queries for values and allowlists/safe identifier handling for dynamic SQL identifiers.

## How do you handle duplicate email?

Enforce UNIQUE in PostgreSQL and map the known unique-violation error to an appropriate API conflict response.

## How do you paginate?

Validate/cap a page size and use LIMIT/OFFSET for simple pagination or keyset pagination for large/deep ordered datasets.

## Authentication vs authorization?

Authentication establishes identity; authorization decides whether that identity may perform the requested action.

## How do you run a PostgreSQL transaction in Express?

Check out one client with `pool.connect()`, run BEGIN and every transactional statement through that client, then COMMIT or ROLLBACK and release it in `finally`.

## Why not call a payment API inside an open DB transaction?

It keeps database locks/connections open while waiting on an external system that PostgreSQL cannot atomically control.

## What is N+1?

One initial query followed by one query per returned item, causing excessive database round trips.

---

# Quick Revision

~~~text
REQUEST
   ↓
ROUTE
   ↓
MIDDLEWARE
   ↓
CONTROLLER
   ↓
SERVICE
   ↓
PG POOL / CLIENT
   ↓
POSTGRESQL
   ↓
RESULT
   ↓
HTTP RESPONSE
~~~

### Responsibilities

~~~text
Route      → endpoint mapping
Controller → HTTP
Service    → business logic
DB         → connection infrastructure
PostgreSQL → integrity + data
~~~

### Security

~~~text
Auth
 ↓
Authorization
 ↓
Validation
 ↓
Parameterized SQL
 ↓
DB Constraints
~~~

### Transaction

~~~text
pool.connect()
 ↓
BEGIN
 ↓
all queries on same client
 ↓
COMMIT / ROLLBACK
 ↓
release()
~~~

### Golden Rule

~~~text
Controller knows HTTP.
Service knows business workflow.
PostgreSQL protects data integrity.
~~~

---

## Key Takeaway

> **A clean Express + PostgreSQL API separates HTTP routing/controllers from reusable service and database logic. Use a shared connection pool, parameterized queries, server-side validation and authorization, PostgreSQL constraints, centralized error handling, bounded pagination, and one checked-out client for every multi-statement transaction.**

---

[← Previous: Lesson 31 — PostgreSQL + Node.js](./31-postgresql-nodejs.md) | [Back to Roadmap](../README.md) | [Next: Lesson 33 — PostgreSQL + Next.js →](./33-postgresql-nextjs.md)