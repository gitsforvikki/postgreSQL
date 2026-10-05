# Lesson 33 — PostgreSQL + Next.js App Router

## Goal of This Lesson

In a modern Next.js App Router application, PostgreSQL should be accessed from **server-side code**, not directly from the browser.

Core architecture:

~~~text
Browser
  ↓
Next.js App Router
  ├── Server Component ──→ DAL ──→ PostgreSQL
  ├── Route Handler ─────→ DAL ──→ PostgreSQL
  └── Server Action ─────→ DAL ──→ PostgreSQL
~~~

A **Client Component must never receive database credentials or connect directly to PostgreSQL**.

---

# 1. Server Components Are the Default

In the App Router, components are Server Components unless you mark a client boundary with:

~~~tsx
'use client';
~~~

Server Components can perform server-side data fetching and can query your database through server-only modules.

This means your own server-rendered page often does **not** need an internal API request just to read PostgreSQL data.

~~~text
Avoid unnecessary hop:
Server Component → fetch('/api/products') → Route Handler → DB

Prefer when appropriate:
Server Component → DAL → DB
~~~

---

# 2. Recommended Structure

~~~text
src/
├── app/
│   ├── products/
│   │   └── page.tsx
│   ├── products/[id]/
│   │   └── page.tsx
│   └── api/products/
│       └── route.ts
├── lib/
│   ├── db.ts
│   └── data/
│       └── products.ts
└── actions/
    └── product-actions.ts
~~~

`lib/db.ts` owns connection infrastructure.

`lib/data/*` acts as a Data Access Layer (DAL).

Server Components, Route Handlers, and Server Actions reuse that DAL.

---

# 3. Server-Only Database Module

Install `pg`:

~~~bash
npm install pg
~~~

`src/lib/db.ts`:

~~~ts
import 'server-only';
import { Pool } from 'pg';

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});
~~~

`server-only` helps prevent this module from accidentally entering a Client Component dependency graph.

---

# 4. Protect DATABASE_URL

`.env.local`:

~~~env
DATABASE_URL=postgresql://username:password@localhost:5432/shophub
~~~

Use it only in server-side code:

~~~ts
process.env.DATABASE_URL
~~~

Never create:

~~~env
NEXT_PUBLIC_DATABASE_URL=...
~~~

`NEXT_PUBLIC_` variables are intended to be exposed to browser-side code.

~~~text
DATABASE_URL
→ server secret

NEXT_PUBLIC_*
→ browser-visible configuration
~~~

---

# 5. Data Access Layer (DAL)

`src/lib/data/products.ts`:

~~~ts
import 'server-only';
import { pool } from '@/lib/db';

export async function getProducts() {
  const result = await pool.query(`
    SELECT id, name, price, created_at
    FROM products
    ORDER BY created_at DESC
  `);

  return result.rows;
}
~~~

A DAL gives you one place for:

- SQL/data access
- authorization close to sensitive data
- parameterization
- minimal safe return shapes
- reusable database logic

---

# 6. Query PostgreSQL from a Server Component

`src/app/products/page.tsx`:

~~~tsx
import { getProducts } from '@/lib/data/products';

export default async function ProductsPage() {
  const products = await getProducts();

  return (
    <main>
      <h1>Products</h1>

      <ul>
        {products.map((product) => (
          <li key={product.id}>
            {product.name} — {product.price}
          </li>
        ))}
      </ul>
    </main>
  );
}
~~~

Flow:

~~~text
Request
  ↓
Server Component
  ↓
getProducts()
  ↓
PostgreSQL
  ↓
rendered result / RSC payload
  ↓
Browser
~~~

No custom `/api/products` call is required for this server-side read.

---

# 7. Why Client Components Cannot Query PostgreSQL Directly

Bad architecture:

~~~text
Browser Client Component
       ↓
PostgreSQL
~~~

This would expose or require access to:

~~~text
database hostname
credentials
database network access
privileged query capability
~~~

Correct architecture:

~~~text
Client Component
      ↓
server boundary
      ↓
Route Handler / Server Action / server-rendered parent
      ↓
DAL
      ↓
PostgreSQL
~~~

---

# 8. Passing Server-Fetched Data to a Client Component

Server Component:

~~~tsx
import ProductList from './product-list';
import { getProducts } from '@/lib/data/products';

export default async function Page() {
  const products = await getProducts();
  return <ProductList products={products} />;
}
~~~

Client Component:

~~~tsx
'use client';

export default function ProductList({ products }) {
  // interactive browser behavior
  return <div>{/* ... */}</div>;
}
~~~

The Client Component receives safe serialized data, **not a database connection**.

---

# 9. Route Handlers

Route Handlers create HTTP endpoints using the Web `Request`/`Response` APIs and Next.js extensions.

`src/app/api/products/route.ts`:

~~~ts
import { NextResponse } from 'next/server';
import { getProducts } from '@/lib/data/products';

export async function GET() {
  const products = await getProducts();
  return NextResponse.json({ data: products });
}
~~~

Route Handlers are useful when you actually need an HTTP API boundary.

---

# 10. When Should You Use a Route Handler?

Good cases include:

~~~text
Client-side HTTP fetching
Mobile application
External frontend
Webhook endpoint
Third-party integration
Public/private API consumer
~~~

For your own Server Component:

~~~text
Server Component → DAL → PostgreSQL
~~~

is often simpler than:

~~~text
Server Component → your own Route Handler → DAL → PostgreSQL
~~~

Do not create an internal API layer automatically when no HTTP boundary is needed.

---

# 11. POST Route Handler

~~~ts
import { NextRequest, NextResponse } from 'next/server';
import { pool } from '@/lib/db';

export async function POST(request: NextRequest) {
  const body = await request.json();
  const { name, price } = body;

  if (!name || typeof price !== 'number') {
    return NextResponse.json(
      { message: 'Invalid input' },
      { status: 400 }
    );
  }

  const result = await pool.query(
    `
      INSERT INTO products (name, price)
      VALUES ($1, $2)
      RETURNING id, name, price, created_at
    `,
    [name, price]
  );

  return NextResponse.json(
    { data: result.rows[0] },
    { status: 201 }
  );
}
~~~

Even inside Next.js, all untrusted SQL values should remain parameterized.

---

# 12. Dynamic Route Handler Params

Current App Router dynamic params are asynchronous.

`src/app/api/products/[id]/route.ts`:

~~~ts
import { NextResponse } from 'next/server';
import { pool } from '@/lib/db';

export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;

  const result = await pool.query(
    `
      SELECT id, name, price
      FROM products
      WHERE id = $1
    `,
    [id]
  );

  if (result.rowCount === 0) {
    return NextResponse.json(
      { message: 'Product not found' },
      { status: 404 }
    );
  }

  return NextResponse.json({ data: result.rows[0] });
}
~~~

Remember:

~~~text
params
→ Promise<{ ... }>
→ await params
~~~

---

# 13. Dynamic Page Params

The same async pattern applies to dynamic pages.

~~~tsx
export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  // fetch product from DAL
  return <main>{/* ... */}</main>;
}
~~~

This is the modern App Router pattern.

---

# 14. Query Parameters with NextRequest

For a Route Handler:

~~~ts
import { NextRequest } from 'next/server';

export async function GET(request: NextRequest) {
  const search = request.nextUrl.searchParams.get('search') ?? '';
  const page = Number(request.nextUrl.searchParams.get('page') ?? '1');

  // ...
}
~~~

Request:

~~~text
/api/products?search=laptop&page=2
~~~

Use `request.nextUrl.searchParams` when working with NextRequest.

---

# 15. Server Actions

A Server Action is an asynchronous server function that can be invoked through React/Next.js mutation flows.

Separate action file:

~~~ts
'use server';

export async function createProduct(formData: FormData) {
  // runs on the server
}
~~~

Server Actions are especially useful for form mutations.

~~~text
Form
 ↓
Server Action
 ↓
validate + authorize
 ↓
DAL / PostgreSQL
 ↓
revalidate
 ↓
updated UI
~~~

---

# 16. Server Action Example

`src/actions/product-actions.ts`:

~~~ts
'use server';

import { revalidatePath } from 'next/cache';
import { pool } from '@/lib/db';

export async function createProduct(formData: FormData) {
  const name = String(formData.get('name') ?? '').trim();
  const price = Number(formData.get('price'));

  if (!name || !Number.isFinite(price) || price < 0) {
    throw new Error('Invalid product');
  }

  await pool.query(
    `
      INSERT INTO products (name, price)
      VALUES ($1, $2)
    `,
    [name, price]
  );

  revalidatePath('/products');
}
~~~

Form:

~~~tsx
import { createProduct } from '@/actions/product-actions';

export default function ProductForm() {
  return (
    <form action={createProduct}>
      <input name="name" />
      <input name="price" type="number" />
      <button type="submit">Create</button>
    </form>
  );
}
~~~

---

# 17. Server Action Is Not a Database Technology

Do not confuse these layers.

~~~text
Server Action
→ mechanism for invoking server-side mutation logic

DAL
→ application data-access boundary

pg / Drizzle / Prisma
→ database access tool

PostgreSQL
→ database
~~~

You can change your database library without changing the basic meaning of a Server Action.

---

# 18. Validate on the Server

Client-side validation is useful for UX but is not a security boundary.

~~~text
Client validation
→ better feedback

Server validation
→ trusted enforcement before business logic

PostgreSQL constraints
→ final data-integrity enforcement
~~~

A malicious client can bypass your UI and invoke an exposed server endpoint/action directly.

---

# 19. Server Actions Need Authorization

`'use server'` does **not** mean:

~~~text
automatically authenticated
automatically authorized
automatically safe
~~~

A mutation should still:

~~~text
authenticate user
      ↓
authorize operation
      ↓
validate input
      ↓
perform parameterized DB operation
~~~

Treat Server Actions as externally invokable server entry points from a security perspective.

---

# 20. Async cookies()

Modern App Router code uses asynchronous `cookies()`.

~~~ts
import { cookies } from 'next/headers';

export async function getSessionToken() {
  const cookieStore = await cookies();
  return cookieStore.get('session')?.value;
}
~~~

Remember:

~~~text
cookies()
→ await cookies()
~~~

Do not use old synchronous examples as your default pattern.

---

# 21. Authorization Close to the Data

Suppose a user requests order `123`.

Do not only check authorization in the UI.

A DAL function can enforce ownership:

~~~ts
export async function getOrderForUser(orderId: string, userId: string) {
  const result = await pool.query(
    `
      SELECT id, status, total
      FROM orders
      WHERE id = $1
        AND user_id = $2
    `,
    [orderId, userId]
  );

  return result.rows[0] ?? null;
}
~~~

This prevents a caller from retrieving another user's order simply by changing an ID.

---

# 22. Cache Revalidation After Mutations

After changing PostgreSQL data, cached/rendered application data may need invalidation.

Path-based example:

~~~ts
import { revalidatePath } from 'next/cache';

revalidatePath('/products');
~~~

Conceptual flow:

~~~text
Server Action
   ↓
UPDATE PostgreSQL
   ↓
revalidatePath('/products')
   ↓
affected route can receive fresh data
~~~

---

# 23. Tag-Based Revalidation

When your application deliberately caches data with tags, tag invalidation can target related cached data.

High-level example:

~~~ts
import { revalidateTag } from 'next/cache';

revalidateTag('products', 'max');
~~~

Do not add caching merely because a database query exists.

Choose a caching strategy based on freshness and workload requirements.

---

# 24. Database Queries Are Not Automatically 'Your API Cache'

A common misconception is:

~~~text
Server Component query
=
automatically permanently cached database result
~~~

Do not reason about PostgreSQL data freshness using old blanket caching assumptions.

Understand the caching APIs and rendering behavior you explicitly use in your current Next.js version.

---

# 25. Transactions in Next.js

PostgreSQL transaction rules do not change because you use Next.js.

~~~ts
const client = await pool.connect();

try {
  await client.query('BEGIN');

  // all transaction queries use client.query(...)

  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
~~~

Golden rule:

~~~text
ONE POSTGRESQL TRANSACTION
          =
ONE CHECKED-OUT DATABASE CONNECTION
~~~

Never use separate `pool.query()` calls for the statements of one transaction.

---

# 26. ShopHub Checkout Transaction

~~~text
Server Action / Route Handler
          ↓
Order service / DAL
          ↓
pool.connect()
          ↓
BEGIN
          ↓
create order
          ↓
create order items
          ↓
atomic inventory updates
          ↓
COMMIT
          ↓
revalidate affected UI
~~~

If a database step fails:

~~~text
ROLLBACK
   ↓
release client
   ↓
controlled application error
~~~

Do not keep the transaction open while waiting for a slow payment provider.

---

# 27. Route Handler vs Server Action

| Need | Prefer |
|---|---|
| Form/UI mutation inside your Next.js app | Server Action often fits well |
| Public/mobile/external HTTP API | Route Handler |
| Webhook endpoint | Route Handler |
| Browser code explicitly fetching HTTP JSON | Route Handler |
| Server-rendered read | Server Component + DAL |

These are architectural choices, not rigid laws.

---

# 28. Server Component vs Route Handler for Reads

### Own server-rendered page

~~~text
Server Component
      ↓
DAL
      ↓
PostgreSQL
~~~

### Browser/mobile/external HTTP consumer

~~~text
Client
  ↓ HTTP
Route Handler
  ↓
DAL
  ↓
PostgreSQL
~~~

The HTTP API layer is useful when there is an actual HTTP consumer boundary.

---

# 29. Client-Side Fetching Use Cases

A Client Component may need to fetch from a Route Handler for interactions such as:

~~~text
load more
browser-driven filtering
polling
live search
manual refresh
client-only interaction state
~~~

Flow:

~~~text
Client Component
     ↓ fetch()
Route Handler
     ↓
DAL
     ↓
PostgreSQL
~~~

The browser still never receives database credentials.

---

# 30. Existing Express Backend Is Still Valid

Next.js does not mean every application must remove Express.

Separate backend architecture can make sense when you need:

~~~text
multiple frontends
mobile clients
shared public API
independent backend deployment
microservices
separate backend team/lifecycle
~~~

Architecture:

~~~text
Next.js Frontend
       ↓ HTTP
Express API
       ↓
PostgreSQL
~~~

Use architecture based on system requirements, not framework hype.

---

# 31. Next.js Full-Stack Architecture

For an application owned entirely by Next.js:

~~~text
                    ┌→ Server Component ─┐
Browser / React ────┼→ Server Action ────┼→ DAL → PostgreSQL
                    └→ Route Handler ────┘
~~~

Each server entry point should reuse trusted data/business logic rather than duplicating SQL everywhere.

---

# 32. Error Handling

Database code can fail due to:

~~~text
constraint violation
connection problem
deadlock
serialization failure
invalid query
timeout
~~~

Handle expected errors intentionally and avoid exposing raw database internals to users.

For route UI failures, App Router error boundaries such as `error.tsx` can provide user-facing recovery UI for uncaught rendering errors.

API/Server Action code should still return/throw controlled application errors as appropriate.

---

# 33. SQL Injection Rules Still Apply

Safe:

~~~ts
await pool.query(
  'SELECT id, name FROM products WHERE id = $1',
  [id]
 );
~~~

Unsafe:

~~~ts
await pool.query(
  `SELECT * FROM products WHERE id = '${id}'`
);
~~~

Next.js does not automatically make raw SQL safe.

Use parameterization or the safe parameter mechanisms of your ORM/query library.

---

# 34. Do Not SELECT Sensitive Fields

DAL functions should return the minimum required data.

Prefer:

~~~sql
SELECT id, name, email
FROM users
WHERE id = $1;
~~~

rather than exposing:

~~~text
password_hash
internal flags
tokens
provider secrets
~~~

A server boundary protects secrets only if you also shape data correctly before returning it.

---

# 35. Complete Product Read Flow

~~~text
GET /products
    ↓
Next.js Server Component
    ↓
getProducts() DAL
    ↓
parameterized SQL / pg
    ↓
PostgreSQL
    ↓
rows
    ↓
Server Component renders
    ↓
RSC/HTML result sent to browser
~~~

This is the simplest important App Router + PostgreSQL read architecture.

---

# 36. Complete Mutation Flow

~~~text
User submits form
      ↓
Server Action
      ↓
authenticate
      ↓
authorize
      ↓
validate FormData
      ↓
DAL / transaction
      ↓
PostgreSQL
      ↓
revalidatePath / revalidateTag when needed
      ↓
updated UI
~~~

This is the mutation flow to remember for interviews.

---

# Common Mistakes

## 37. Importing pg into a Client Component

Database access belongs on the server.

## 38. Exposing DATABASE_URL with NEXT_PUBLIC_

Database credentials must remain server-side secrets.

## 39. Creating a Route Handler for Every Server Component Query

Server Components can call server-side data functions directly when no HTTP API boundary is needed.

## 40. Trusting Server Actions Automatically

Server Actions still require authentication, authorization, validation, and safe database access.

## 41. Using Old Synchronous Dynamic APIs

Use current async patterns such as `await params` and `await cookies()`.

## 42. Interpolating User Input into Raw SQL

Use parameterized queries.

## 43. Using pool.query() Across One Transaction

Check out one client and use it for every transaction statement.

## 44. Returning Entire Database Rows to the Browser

Return only fields the UI/API actually needs.

## 45. Calling Your Own API from a Server Component Without Need

Direct server-side DAL access avoids an unnecessary HTTP layer.

---

# Interview Revision

## Can a Next.js Server Component query PostgreSQL?

Yes. Server Components execute on the server and can call server-only database/DAL functions directly.

## Can a Client Component connect directly to PostgreSQL?

No. Browser code must not contain database credentials or direct privileged database access.

## What is a Route Handler?

A server-side HTTP endpoint defined with `route.ts` under the App Router, using handlers such as GET, POST, PATCH, and DELETE.

## What is a Server Action?

An asynchronous server function integrated with React/Next.js, commonly used for UI mutations such as form submissions.

## Server Action vs Route Handler?

A Server Action is convenient for internal application mutation flows; a Route Handler provides an explicit HTTP endpoint for HTTP consumers such as browser fetches, mobile apps, webhooks, or external clients.

## What is a DAL?

A Data Access Layer centralizes database access and can keep authorization, safe query logic, and minimal data shaping close to the data source.

## How are dynamic params accessed?

Use a Promise type and await it, for example `params: Promise<{ id: string }>` followed by `const { id } = await params`.

## How do you access cookies?

Use `const cookieStore = await cookies()` in current App Router server code.

## How do you access Route Handler query parameters?

With `NextRequest`, use `request.nextUrl.searchParams`.

## How do PostgreSQL transactions work in Next.js?

The same as other Node.js servers: check out one database client, run BEGIN/all statements/COMMIT or ROLLBACK on that client, then release it.

## Does Next.js remove the need for SQL injection protection?

No. Raw SQL values must still be parameterized.

## Do you always need an API route between a Server Component and PostgreSQL?

No. A Server Component can call a server-only DAL directly.

---

# Quick Revision

~~~text
SERVER COMPONENT
      ↓
DAL
      ↓
POSTGRESQL
~~~

~~~text
CLIENT COMPONENT
      ↓ HTTP when needed
ROUTE HANDLER
      ↓
DAL
      ↓
POSTGRESQL
~~~

~~~text
FORM
 ↓
SERVER ACTION
 ↓
AUTH + VALIDATION
 ↓
DAL / TRANSACTION
 ↓
POSTGRESQL
 ↓
REVALIDATE
~~~

### Modern App Router reminders

~~~text
params    → await params
cookies() → await cookies()
query     → request.nextUrl.searchParams
~~~

### Security

~~~text
DATABASE_URL
→ server only

Client Component
→ never direct DB access

Server Action
→ still authenticate + authorize + validate
~~~

### Golden Architecture

~~~text
UI
 ↓
trusted server boundary
 ↓
DAL
 ↓
PostgreSQL
~~~

---

## Key Takeaway

> **In the Next.js App Router, keep PostgreSQL behind server-only code. Server Components can read directly through a DAL, Server Actions are well suited to internal UI mutations, and Route Handlers provide explicit HTTP APIs. Client Components never connect directly to PostgreSQL, and every mutation still requires validation, authorization, parameterized queries, correct transaction handling, and deliberate cache revalidation.**

---

[← Previous: Lesson 32 — PostgreSQL + Express API](./32-postgresql-express-api.md) | [Back to Roadmap](../README.md) | [Next: Lesson 34 — ORM vs Raw SQL →](./34-orm-vs-raw-sql.md)