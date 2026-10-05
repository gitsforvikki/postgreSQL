# Lesson 34 — ORM vs Raw SQL (Prisma, Drizzle & pg)

## Goal of This Lesson

When a Node.js or Next.js application uses PostgreSQL, there are several ways to access the database.

Three common approaches are:

~~~text
Application
   │
   ├── Raw SQL with pg
   ├── Drizzle ORM / query builder
   └── Prisma ORM
           ↓
       PostgreSQL
~~~

The important question is not:

> Which tool is always best?

The better question is:

> Which level of abstraction is appropriate for this application and query?

---

# 1. What Is Raw SQL?

Raw SQL means writing SQL directly.

Example with `pg`:

~~~ts
const result = await pool.query(
  `
    SELECT id, name, email
    FROM users
    WHERE id = $1
  `,
  [userId]
 );
~~~

You directly control:

~~~text
SELECT
JOIN
WHERE
GROUP BY
CTE
window functions
indexes used by query shape
PostgreSQL-specific features
~~~

`pg` is a PostgreSQL driver, not an ORM.

---

# 2. What Is an ORM?

ORM stands for **Object-Relational Mapping**.

An ORM provides an application-level abstraction for working with relational database data.

Instead of always writing:

~~~sql
SELECT id, name, email
FROM users
WHERE id = $1;
~~~

you may write an ORM query such as:

~~~ts
const user = await prisma.user.findUnique({
  where: { id: userId },
});
~~~

The ORM translates application-level operations into database queries.

---

# 3. ORM Does Not Replace PostgreSQL

This is the most important concept in this lesson.

~~~text
ORM
   ↓
generates / sends SQL
   ↓
PostgreSQL
   ↓
query planner
   ↓
indexes / joins / locks / MVCC / WAL
~~~

Even when using an ORM, PostgreSQL still decides how queries execute.

You still need to understand:

- indexes
- joins
- transactions
- constraints
- isolation
- locks
- query plans
- normalization
- PostgreSQL data types

An ORM changes how your application **expresses database operations**. It does not remove database fundamentals.

---

# 4. Raw SQL with pg

Example:

~~~ts
const result = await pool.query(
  `
    SELECT
      u.id,
      u.name,
      COUNT(o.id)::int AS order_count
    FROM users u
    LEFT JOIN orders o
      ON o.user_id = u.id
    GROUP BY u.id, u.name
    ORDER BY order_count DESC
    LIMIT $1
  `,
  [10]
 );
~~~

Advantages:

~~~text
Maximum SQL control
PostgreSQL features directly available
Easy to reason about exact query
Excellent for complex reporting
No ORM abstraction to fight
~~~

Tradeoffs:

~~~text
More SQL written manually
More repetitive CRUD code
Manual result typing/validation strategy
Schema/migration tooling must be chosen separately
~~~

---

# 5. What Is Drizzle?

Drizzle is a TypeScript-oriented ORM/query toolkit that stays relatively close to SQL.

Typical mental model:

~~~text
TypeScript schema
      +
SQL-like query builder
      ↓
Drizzle
      ↓
PostgreSQL
~~~

Example schema:

~~~ts
import { pgTable, bigint, text } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: bigint('id', { mode: 'number' })
    .primaryKey()
    .generatedAlwaysAsIdentity(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
});
~~~

---

# 6. Drizzle Query Example

~~~ts
import { eq } from 'drizzle-orm';
import { users } from '@/db/schema';

const result = await db
  .select({
    id: users.id,
    name: users.name,
    email: users.email,
  })
  .from(users)
  .where(eq(users.id, userId));
~~~

If you know SQL, this structure often feels familiar:

~~~text
select()
from()
where()
orderBy()
limit()
~~~

---

# 7. What Is Prisma?

Prisma is a higher-level ORM/tooling ecosystem centered around a declarative data model and generated type-safe database APIs.

Conceptually:

~~~text
Prisma schema / data model
        ↓
Prisma tooling/client
        ↓
typed model-oriented API
        ↓
PostgreSQL
~~~

Example model:

~~~prisma
model User {
  id    Int    @id @default(autoincrement())
  name  String
  email String @unique
}
~~~

Application query:

~~~ts
const user = await prisma.user.findUnique({
  where: { id: userId },
  select: {
    id: true,
    name: true,
    email: true,
  },
});
~~~

---

# 8. Same Query — Three Styles

Suppose we need user `42`.

## pg

~~~ts
const result = await pool.query(
  `SELECT id, name, email FROM users WHERE id = $1`,
  [42]
 );

const user = result.rows[0];
~~~

## Drizzle

~~~ts
const [user] = await db
  .select({
    id: users.id,
    name: users.name,
    email: users.email,
  })
  .from(users)
  .where(eq(users.id, 42));
~~~

## Prisma

~~~ts
const user = await prisma.user.findUnique({
  where: { id: 42 },
  select: {
    id: true,
    name: true,
    email: true,
  },
});
~~~

All three ultimately communicate with PostgreSQL.

---

# 9. Abstraction Level

~~~text
Higher abstraction
      ↑
   Prisma
      │
   Drizzle
      │
   Raw SQL / pg
      ↓
Lower abstraction / more direct SQL control
~~~

This is a useful mental model, but it is not a quality ranking.

Higher abstraction can improve productivity.

Lower abstraction can provide more direct control.

---

# 10. Type Safety

One major reason TypeScript developers use Drizzle or Prisma is type safety.

Example idea:

~~~text
Database/schema definition
       ↓
TypeScript understands columns/types
       ↓
editor autocomplete + compile-time checking
~~~

Raw `pg` does not automatically know the exact shape of every query result unless you provide your own typing strategy.

However:

> **Type safety is not runtime validation.**

TypeScript disappears at runtime.

You still need validation for untrusted request data.

---

# 11. Runtime Validation Still Matters

Suppose an API receives:

~~~json
{
  "price": "hello"
}
~~~

Your TypeScript function may expect:

~~~ts
price: number
~~~

but a malicious HTTP client does not care about your TypeScript types.

Correct layers:

~~~text
HTTP input
   ↓
Runtime validation
   ↓
ORM / parameterized query
   ↓
PostgreSQL constraints
~~~

ORM type safety does not replace request validation or database constraints.

---

# 12. Database Constraints Still Matter

Even if your ORM schema says an email is unique, the database should enforce the actual uniqueness constraint.

~~~sql
email TEXT NOT NULL UNIQUE
~~~

Why?

Because concurrent requests can race.

~~~text
Request A checks email
Request B checks email
       ↓
both think it is available
       ↓
both attempt INSERT
       ↓
PostgreSQL UNIQUE constraint decides safely
~~~

The database is the final integrity authority.

---

# 13. Migrations

ORM ecosystems often provide migration tooling.

Conceptually:

~~~text
Schema change
    ↓
Generate/write migration
    ↓
Review migration
    ↓
Apply to database
~~~

Examples of changes:

~~~text
CREATE TABLE
ALTER TABLE
ADD COLUMN
CREATE INDEX
ADD CONSTRAINT
~~~

Important:

> Migration tools automate execution and tracking; you should still understand the SQL/schema change they perform.

Never treat production migrations as a button you press without review.

---

# 14. Drizzle Migrations

Drizzle commonly pairs schema definitions with Drizzle Kit migration tooling.

High-level:

~~~text
TypeScript schema
      ↓
Drizzle Kit
      ↓
SQL migration
      ↓
PostgreSQL
~~~

This is especially attractive when you want TypeScript-first schema management while staying close to SQL.

---

# 15. Prisma Migrations

Prisma provides schema/migration workflows through its tooling.

High-level:

~~~text
Prisma data model
      ↓
Prisma migration tooling
      ↓
database migration
      ↓
PostgreSQL
~~~

Prisma is useful when a team prefers a model-oriented abstraction and generated client experience.

---

# 16. Complex Queries

Suppose you need:

~~~text
CTE
window function
JSONB operation
complex aggregation
PostgreSQL-specific operator
special reporting query
~~~

Raw SQL is often the clearest expression.

Example:

~~~sql
WITH ranked_orders AS (
  SELECT
    user_id,
    id,
    total,
    ROW_NUMBER() OVER (
      PARTITION BY user_id
      ORDER BY total DESC
    ) AS rn
  FROM orders
)
SELECT *
FROM ranked_orders
WHERE rn <= 3;
~~~

You should not distort a naturally SQL-shaped problem merely to avoid writing SQL.

---

# 17. Raw SQL Escape Hatches

Good database tools normally provide ways to execute SQL when their higher-level API is not the best fit.

Conceptually:

~~~text
ORM for normal CRUD
      +
raw SQL for specialized query
      ↓
same PostgreSQL database
~~~

This is a normal production approach.

You do **not** need to choose one style for every query forever.

---

# 18. Hybrid Approach

A practical architecture:

~~~text
Application
    │
    ├── Standard CRUD
    │      ↓
    │   ORM / query builder
    │
    └── Complex analytics / specialized PostgreSQL
           ↓
        Raw SQL
              ↓
          PostgreSQL
~~~

Example:

~~~text
Create user        → Drizzle/Prisma
Update profile     → Drizzle/Prisma
Simple relations   → Drizzle/Prisma

Revenue report     → Raw SQL
Window functions   → Raw SQL
Complex CTE        → Raw SQL
Special JSONB query→ Raw SQL when clearer
~~~

Use the simplest tool that keeps the query correct, understandable, and maintainable.

---

# 19. ORM Does Not Automatically Prevent N+1

Suppose:

~~~ts
const users = await getUsers();

for (const user of users) {
  user.orders = await getOrders(user.id);
}
~~~

Whether those functions use raw SQL or an ORM, this can still become:

~~~text
1 user query
100 order queries
=
101 queries
~~~

An ORM can make N+1 easier to accidentally hide behind convenient APIs.

Always understand generated query behavior.

---

# 20. Inspect Generated SQL

When performance matters, inspect what the ORM actually sends to PostgreSQL.

Ask:

~~~text
How many SQL queries?
Which JOINs?
Which WHERE conditions?
Is pagination happening in SQL?
Are unnecessary columns selected?
Can indexes support this query?
~~~

Then use PostgreSQL tools:

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
...
~~~

ORM code is not the final execution plan.

---

# 21. Indexes Are Still Your Responsibility

Suppose ORM code looks elegant:

~~~ts
await db.query.orders.findMany({
  where: ...,
  orderBy: ...,
});
~~~

But the underlying SQL needs:

~~~text
WHERE user_id = ?
ORDER BY created_at DESC
~~~

PostgreSQL may benefit from:

~~~sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
~~~

The ORM does not remove index design.

---

# 22. Transactions with an ORM

ORMs provide transaction APIs, but the database semantics are still PostgreSQL transactions.

Conceptually:

~~~text
ORM transaction API
      ↓
BEGIN
      ↓
query 1
query 2
query 3
      ↓
COMMIT / ROLLBACK
~~~

You still need to understand:

- atomicity
- isolation levels
- locks
- deadlocks
- serialization retries
- short transaction duration

ORM syntax changes; PostgreSQL behavior remains.

---

# 23. Raw pg Transaction

From Lesson 31:

~~~ts
const client = await pool.connect();

try {
  await client.query('BEGIN');
  // queries
  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
~~~

You manually manage the transaction connection.

---

# 24. Drizzle Transaction — Conceptual Example

~~~ts
await db.transaction(async (tx) => {
  await tx.insert(orders).values({
    userId,
    status: 'pending',
  });

  await tx
    .update(products)
    .set({ /* ... */ })
    .where(/* ... */);
});
~~~

Drizzle manages the transaction boundary through its API.

But PostgreSQL is still performing the actual transaction underneath.

---

# 25. Prisma Transaction — Conceptual Example

~~~ts
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({
    data: { /* ... */ },
  });

  await tx.product.update({
    where: { id: productId },
    data: { /* ... */ },
  });

  return order;
});
~~~

Again:

~~~text
Prisma API
   ↓
PostgreSQL transaction
~~~

Do not keep interactive transactions open while waiting on slow external APIs.

---

# 26. PostgreSQL-Specific Features

PostgreSQL has powerful features such as:

~~~text
JSONB
arrays
GIN / GiST / BRIN
partial indexes
expression indexes
RETURNING
UPSERT
window functions
CTEs
materialized views
advisory locks
~~~

Before choosing an abstraction, ask:

> Does this tool let us use the PostgreSQL features our system actually needs without making the code harder?

Sometimes the ORM supports the feature directly.

Sometimes raw SQL is clearer.

---

# 27. Performance

Do not say:

~~~text
Raw SQL is always faster than ORM.
~~~

or:

~~~text
ORM is always optimized.
~~~

The real performance depends on:

~~~text
generated SQL
query plan
indexes
data distribution
number of round trips
selected columns
transaction design
connection strategy
~~~

Two different application APIs can generate effectively identical SQL.

Measure the database work.

---

# 28. Developer Productivity

ORMs/query builders can reduce repetitive code.

Examples:

~~~text
typed schema
autocomplete
relation helpers
migration tooling
generated model types
standard CRUD
~~~

For teams building many conventional CRUD features, this can significantly improve development speed.

That productivity benefit is real.

---

# 29. Raw SQL Maintainability

Raw SQL is not automatically unmaintainable.

Well-structured SQL can be:

~~~text
clear
testable
version-controlled
optimized
easy for SQL-skilled developers to inspect
~~~

Bad:

~~~text
SQL strings scattered randomly through UI/routes/controllers
~~~

Better:

~~~text
DAL / repository / service
       ↓
named query functions
       ↓
parameterized SQL
~~~

Architecture matters more than simply whether SQL is handwritten.

---

# 30. ORM Maintainability

ORM code can also become difficult if:

~~~text
business logic is mixed into every query
relations are loaded without understanding cost
generated SQL is never inspected
database constraints are ignored
migrations are applied blindly
~~~

An ORM improves developer experience, not architectural discipline automatically.

---

# 31. Drizzle vs Prisma — Mental Model

Use this simplified interview-level distinction:

~~~text
Drizzle
→ TypeScript-first
→ SQL-like API
→ stays relatively close to relational/SQL concepts

Prisma
→ model-oriented ORM
→ generated client/tooling experience
→ higher-level application abstraction
~~~

Both can work with PostgreSQL.

Neither makes PostgreSQL knowledge unnecessary.

---

# 32. pg vs Drizzle vs Prisma

| Area | pg / Raw SQL | Drizzle | Prisma |
|---|---|---|---|
| Abstraction | Low | Medium / SQL-like | Higher / model-oriented |
| Direct SQL control | Excellent | High | High with raw-query escape hatches |
| TypeScript experience | Manual/custom typing | Strong | Strong |
| Standard CRUD | More manual | Convenient | Very convenient |
| Complex SQL | Excellent | Good + raw SQL | ORM API + raw SQL when needed |
| Migration tooling | Separate/manual choice | Drizzle tooling | Prisma tooling |
| PostgreSQL knowledge needed | Yes | Yes | Yes |

This table is a conceptual comparison, not a benchmark.

---

# 33. When Raw SQL Is a Strong Choice

Prefer or strongly consider raw SQL when:

~~~text
query is naturally complex SQL
you need precise PostgreSQL behavior
analytics/reporting is heavy
window functions/CTEs dominate
special operators/features matter
you need maximum query transparency
~~~

Raw SQL is especially valuable when performance work requires direct control and easy EXPLAIN analysis.

---

# 34. When an ORM Is a Strong Choice

An ORM/query builder is attractive when:

~~~text
application has lots of standard CRUD
team values generated/inferred types
schema tooling is useful
relations are conventional
developer productivity matters strongly
~~~

Most business applications contain many operations that fit this category.

---

# 35. When Hybrid Is Best

Many production systems naturally become hybrid.

Example ShopHub:

~~~text
Users CRUD
Products CRUD
Addresses
Cart operations
       ↓
Drizzle / Prisma

Complex revenue report
Top-selling products
Inventory analytics
Window-function reports
       ↓
Raw SQL
~~~

This is not a failure of the ORM.

It is choosing the appropriate abstraction per query.

---

# 36. Your CareerLoop Context

For a Next.js + PostgreSQL application using Drizzle:

~~~text
Next.js Server Component / Action / Route Handler
                    ↓
                DAL/service
                    ↓
                 Drizzle
                    ↓
               PostgreSQL
~~~

You should still be able to inspect the SQL and understand why an index or transaction is needed.

For a specialized query, using Drizzle's SQL capabilities/raw SQL escape hatch can be completely reasonable.

---

# 37. ShopHub Decision Example

Suppose you need:

~~~text
Create product
~~~

An ORM query is concise and type-friendly.

Now suppose admin dashboard needs:

~~~text
monthly revenue
previous-month comparison
running total
rank category by revenue
~~~

This is naturally expressed with:

~~~text
GROUP BY
LAG
SUM() OVER
RANK
CTEs
~~~

Raw SQL may be substantially clearer.

Choose based on the operation, not ideology.

---

# 38. Learning Order

For becoming a strong backend/full-stack developer:

~~~text
1. Learn SQL
      ↓
2. Learn PostgreSQL behavior
      ↓
3. Learn indexes + transactions + EXPLAIN
      ↓
4. Learn ORM/query builder
      ↓
5. Know when to drop down to SQL
~~~

This is much stronger than:

~~~text
Learn ORM methods
→ never understand database
~~~

Your PostgreSQL lessons 1–33 give you the right foundation.

---

# Common Mistakes

## 39. Thinking ORM Means You Do Not Need SQL

ORM operations eventually become database queries. SQL knowledge remains essential for debugging and optimization.

## 40. Thinking ORM Automatically Prevents All SQL Injection

Safe ORM APIs help, but unsafe raw SQL or dynamically constructed identifiers can still introduce vulnerabilities.

## 41. Thinking Type Safety Replaces Runtime Validation

TypeScript cannot validate arbitrary HTTP input at runtime.

## 42. Removing Database Constraints Because ORM Validates

Keep NOT NULL, UNIQUE, CHECK, FK, and other integrity rules in PostgreSQL where appropriate.

## 43. Never Inspecting Generated SQL

Convenient application code can still generate inefficient database work.

## 44. Assuming Raw SQL Is Always Faster

Performance depends on the actual SQL/query plan and workload, not the label of the library.

## 45. Forcing Complex SQL into an Awkward ORM Expression

Use raw SQL when it is clearer and safer.

## 46. Using Raw SQL Everywhere Without Structure

Centralize queries in a DAL/repository/service rather than scattering SQL throughout the app.

## 47. Ignoring Transactions Because the ORM Has a Transaction Method

You still need to understand PostgreSQL transaction semantics.

---

# Interview Revision

## What is an ORM?

An Object-Relational Mapping layer provides application-level abstractions for working with relational database data and translates operations into database queries.

## What is raw SQL?

Writing SQL directly and sending it to PostgreSQL through a driver such as `pg`.

## Is pg an ORM?

No. node-postgres (`pg`) is a PostgreSQL driver/client library.

## Why use an ORM?

For developer productivity, type-safe application APIs, relation/schema tooling, migrations, and reducing repetitive CRUD code.

## Why use raw SQL?

For direct control, complex queries, reporting, PostgreSQL-specific functionality, and transparent performance tuning.

## Does an ORM replace SQL knowledge?

No. PostgreSQL still executes SQL and uses indexes, locks, transactions, MVCC, statistics, and query plans.

## Does ORM type safety replace runtime validation?

No. Untrusted HTTP/form input must still be validated at runtime.

## Does an ORM replace database constraints?

No. PostgreSQL constraints remain the final data-integrity layer.

## Can an ORM cause N+1 queries?

Yes. You must understand how many SQL queries the application generates.

## Can you mix ORM and raw SQL?

Yes. A hybrid approach is common: ORM/query builder for standard operations and raw SQL for complex or specialized queries.

## Drizzle vs Prisma?

Drizzle stays closer to SQL with a TypeScript-first query/schema approach; Prisma offers a more model-oriented generated ORM experience. Both still require database knowledge.

## What should you learn first?

SQL and PostgreSQL fundamentals first, then an ORM/query builder.

---

# Quick Revision

~~~text
RAW SQL / pg
→ maximum SQL control
→ more manual application code
~~~

~~~text
DRIZZLE
→ TypeScript-first
→ SQL-like abstraction
→ strong typed query experience
~~~

~~~text
PRISMA
→ model-oriented ORM
→ generated client/tooling
→ convenient CRUD/relations
~~~

### All Roads Lead to PostgreSQL

~~~text
pg ────────┐
Drizzle ───┼→ SQL/database protocol → PostgreSQL
Prisma ────┘
                         ↓
                planner + indexes
                MVCC + locks
                WAL + transactions
~~~

### Production-Friendly Approach

~~~text
Simple CRUD
     ↓
ORM / Query Builder

Complex Query
     ↓
Raw SQL

Both
     ↓
PostgreSQL
~~~

### Golden Rule

~~~text
ORM improves productivity.
SQL knowledge provides control.
PostgreSQL knowledge provides correctness.
~~~

---

## Key Takeaway

> **Raw SQL gives maximum control, while tools such as Drizzle and Prisma improve developer productivity and type-safe database access. An ORM does not replace SQL, indexes, constraints, transactions, EXPLAIN, or PostgreSQL knowledge. In production, using an ORM/query builder for ordinary CRUD and raw SQL for complex or PostgreSQL-specific queries is often the most practical approach.**

---

[← Previous: Lesson 33 — PostgreSQL + Next.js](./33-postgresql-nextjs.md) | [Back to Roadmap](../README.md) | [Next: Lesson 35 — PostgreSQL Security →](../09-security/35-postgresql-security.md)