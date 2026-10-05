# Lesson 44 — Production E-Commerce Database Design

## Goal of This Lesson

This lesson combines the PostgreSQL concepts from Lessons 1–43 into a realistic e-commerce database similar to **ShopHub**.

We will design the database around real production requirements rather than simply creating a few tables.

By the end, you should understand:

~~~text
requirements → entities → relationships → constraints → indexes
→ transactions → concurrency → payments → production queries
~~~

---

# 1. Start With Business Requirements

Before writing SQL, identify what the system must do.

For ShopHub, assume we need:

~~~text
Users
Products
Categories
Product inventory
Guest/authenticated carts
Addresses
Orders
Order items
Payments
Order status
Payment status
Historical pricing
~~~

Important business rules:

~~~text
email must be unique
product price cannot be negative
stock cannot be negative
an order contains one or more items
order item price must preserve purchase-time price
payment amount must match the intended order/payment flow
checkout must not oversell stock
~~~

Database design begins with business invariants.

---

# 2. High-Level ER Diagram

~~~text
USERS
  │
  ├────< ADDRESSES
  │
  ├────< CARTS ────< CART_ITEMS >──── PRODUCTS
  │                                      │
  │                                      │
  └────< ORDERS ───< ORDER_ITEMS >───────┘
             │
             ├────< PAYMENTS
             │
             └──── shipping/billing snapshots

CATEGORIES ────< PRODUCTS
~~~

Key relationships:

~~~text
User 1:N Addresses
User 1:N Orders
Cart 1:N Cart Items
Product 1:N Cart Items
Order 1:N Order Items
Product 1:N Order Items
Order 1:N Payments
Category 1:N Products
~~~

---

# 3. Why Order Items Need Their Own Table

Orders and products have a many-to-many relationship.

One order can contain many products, and one product can appear in many orders.

Therefore:

~~~text
ORDERS
   ↓
ORDER_ITEMS
   ↑
PRODUCTS
~~~

`order_items` is the junction table and also stores relationship-specific historical data such as quantity and purchase-time price.

---

# 4. UUID vs BIGINT IDs

For a public-facing application, UUIDs are a reasonable choice for many entity IDs.

Example:

~~~sql
id UUID PRIMARY KEY DEFAULT gen_random_uuid()
~~~

This requires suitable UUID generation support such as PostgreSQL's built-in `gen_random_uuid()` on supported versions/configurations.

BIGINT identity keys are also completely valid.

The important point is to choose intentionally and consistently.

UUIDs do **not** replace authorization.

---

# 5. Users Table

~~~sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'customer',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT users_role_check
        CHECK (role IN ('customer', 'admin'))
);
~~~

For case-insensitive email uniqueness, one option is:

~~~sql
CREATE UNIQUE INDEX users_email_lower_uidx
ON users (lower(email));
~~~

Then application lookups should use the matching expression:

~~~sql
SELECT id, email, name
FROM users
WHERE lower(email) = lower($1);
~~~

---

# 6. Authentication Data

If your authentication library maintains its own user/account/session schema, do not duplicate fields unnecessarily.

Conceptually:

~~~text
Auth tables
→ credentials / OAuth accounts / sessions

Application profile
→ business-specific user data
~~~

For example, Better Auth may manage authentication tables while CareerLoop/ShopHub adds application-specific profile data around them.

Follow the authentication library's schema contract rather than forcing this illustrative `users` table onto it.

---

# 7. Categories

~~~sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    parent_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
~~~

`parent_id` enables hierarchical categories:

~~~text
Electronics
   ├─ Phones
   └─ Laptops
~~~

This is a self-referencing foreign key.

---

# 8. Products

~~~sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    description TEXT,
    price NUMERIC(12,2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT products_price_check CHECK (price >= 0),
    CONSTRAINT products_stock_check CHECK (stock >= 0)
);
~~~

This uses normal relational columns for core searchable/business fields and JSONB only for flexible metadata.

---

# 9. Why NUMERIC for Money?

Use exact numeric values for money-like amounts:

~~~sql
price NUMERIC(12,2)
~~~

rather than floating-point types.

Another common architecture stores monetary values as integer minor units:

~~~text
₹499.99
→ 49999 paise
~~~

Both approaches can be valid if used consistently.

Your ShopHub payment integration already benefits from integer minor-unit calculations at the payment boundary.

---

# 10. Do Not Trust Client Prices

Bad checkout request:

~~~json
{
  "productId": "...",
  "price": 1,
  "quantity": 2
}
~~~

The client must not be authoritative for price.

Correct flow:

~~~text
Client sends product + quantity
        ↓
Server loads authoritative product price
        ↓
Server calculates totals
        ↓
Database stores trusted amounts
~~~

Frontend totals are for display, not authority.

---

# 11. Product Metadata With JSONB

Flexible category-specific attributes can use JSONB:

~~~json
{
  "brand": "Example",
  "color": "Black",
  "ram": "16GB"
}
~~~

Why not make the entire product JSONB?

~~~text
price
stock
category_id
is_active
~~~

are core relational/business fields requiring strong constraints, indexing, and predictable querying.

Use a hybrid model.

---

# 12. Product Images

If a product can have multiple images, model them separately:

~~~sql
CREATE TABLE product_images (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    image_url TEXT NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    alt_text TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT product_images_sort_check CHECK (sort_order >= 0)
);
~~~

Do not store actual large image binaries in the normal product row unless there is a specific requirement.

Typically the database stores object-storage/CDN references.

---

# 13. Addresses

Users may have saved addresses:

~~~sql
CREATE TABLE addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    recipient_name TEXT NOT NULL,
    phone TEXT,
    line1 TEXT NOT NULL,
    line2 TEXT,
    city TEXT NOT NULL,
    state TEXT NOT NULL,
    postal_code TEXT NOT NULL,
    country_code CHAR(2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
~~~

But an order should generally preserve the address used at purchase time even if the user later edits their saved address.

---

# 14. Historical Address Snapshot

Suppose:

~~~text
Monday
User orders to Address A

Tuesday
User edits saved address to Address B
~~~

The Monday order must still show Address A.

Therefore an order can store a snapshot of the checkout address rather than relying only on the mutable `addresses` row.

Options include:

~~~text
dedicated order_addresses table
or
structured JSONB snapshot
or
snapshot columns on orders
~~~

For this lesson we will use JSONB snapshots for compactness.

---

# 15. Carts

~~~sql
CREATE TABLE carts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    guest_token TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT carts_owner_check CHECK (
        user_id IS NOT NULL OR guest_token IS NOT NULL
    )
);
~~~

This supports authenticated and guest carts conceptually.

Your application must define whether a user/guest can have one active cart or multiple carts and enforce that invariant appropriately.

---

# 16. Cart Items

~~~sql
CREATE TABLE cart_items (
    cart_id UUID NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    quantity INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (cart_id, product_id),
    CONSTRAINT cart_items_quantity_check CHECK (quantity > 0)
);
~~~

Composite primary key means the same product appears at most once per cart.

Updating quantity modifies the existing row instead of creating duplicates.

---

# 17. Cart Price vs Product Price

A cart may display the current product price, but the cart should not be treated as a permanent historical price record.

Conceptually:

~~~text
Cart
→ temporary intent

Order item
→ permanent purchase snapshot
~~~

At checkout, re-read authoritative product data and calculate totals on the server.

---

# 18. Orders

~~~sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    payment_status TEXT NOT NULL DEFAULT 'pending',
    currency CHAR(3) NOT NULL DEFAULT 'INR',
    subtotal NUMERIC(12,2) NOT NULL,
    tax_amount NUMERIC(12,2) NOT NULL DEFAULT 0,
    shipping_amount NUMERIC(12,2) NOT NULL DEFAULT 0,
    discount_amount NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount NUMERIC(12,2) NOT NULL,
    shipping_address JSONB NOT NULL,
    billing_address JSONB,
    idempotency_key TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT orders_status_check CHECK (
        status IN ('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled')
    ),
    CONSTRAINT orders_payment_status_check CHECK (
        payment_status IN ('pending', 'paid', 'failed', 'refunded', 'partially_refunded')
    ),
    CONSTRAINT orders_amounts_check CHECK (
        subtotal >= 0
        AND tax_amount >= 0
        AND shipping_amount >= 0
        AND discount_amount >= 0
        AND total_amount >= 0
    )
);
~~~

The exact statuses should match your application's real state machine.

---

# 19. Order Total Invariant

Conceptually:

~~~text
total_amount
=
subtotal
- discount_amount
+ tax_amount
+ shipping_amount
~~~

Your server should calculate this from authoritative values.

You may additionally enforce suitable database checks where the formula is stable.

Do not independently calculate one total in React and a different total in the order service.

Use one authoritative pricing pipeline.

---

# 20. Idempotency Key

Checkout/payment requests can be retried because of network failures.

Without idempotency:

~~~text
Request
  ↓
order created
  ↓
response lost
  ↓
client retries
  ↓
second order created
~~~

An idempotency key can protect the operation.

Example index:

~~~sql
CREATE UNIQUE INDEX orders_idempotency_key_uidx
ON orders(idempotency_key)
WHERE idempotency_key IS NOT NULL;
~~~

This is a partial unique index.

---

# 21. Order Items

~~~sql
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID REFERENCES products(id) ON DELETE SET NULL,
    product_name TEXT NOT NULL,
    unit_price NUMERIC(12,2) NOT NULL,
    quantity INTEGER NOT NULL,
    line_total NUMERIC(12,2) GENERATED ALWAYS AS (unit_price * quantity) STORED,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT order_items_price_check CHECK (unit_price >= 0),
    CONSTRAINT order_items_quantity_check CHECK (quantity > 0)
);
~~~

`product_name` and `unit_price` are historical snapshots.

---

# 22. Why Duplicate Product Name and Price?

Normally normalization reduces duplication.

But orders are historical business records.

Suppose product changes:

~~~text
At purchase
Name: Keyboard Pro
Price: ₹4,999

Later
Name: Keyboard Pro 2
Price: ₹5,999
~~~

The old order should still display:

~~~text
Keyboard Pro
₹4,999
~~~

This is **intentional denormalization for historical correctness**.

---

# 23. Why product_id Can Be Nullable in order_items

If a product is later removed from the catalog, historical orders should remain valid.

Using:

~~~sql
product_id UUID REFERENCES products(id) ON DELETE SET NULL
~~~

allows the order snapshot to survive product deletion.

Another production strategy is soft-deleting/deactivating products rather than physically deleting them.

---

# 24. Prefer Product Deactivation

For commerce systems, products often should not be physically deleted once referenced by business records.

Instead:

~~~sql
UPDATE products
SET is_active = false
WHERE id = $1;
~~~

Benefits:

~~~text
historical references remain
auditability improves
accidental destructive deletes reduce
~~~

---

# 25. Payments

An order may have multiple payment attempts.

Therefore:

~~~text
Order 1:N Payments
~~~

Example:

~~~sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE RESTRICT,
    provider TEXT NOT NULL,
    provider_order_id TEXT,
    provider_payment_id TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    amount NUMERIC(12,2) NOT NULL,
    currency CHAR(3) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT payments_amount_check CHECK (amount >= 0),
    CONSTRAINT payments_status_check CHECK (
        status IN ('pending', 'authorized', 'paid', 'failed', 'refunded', 'partially_refunded')
    )
);
~~~

Provider-specific IDs should generally be unique when the provider guarantees uniqueness.

---

# 26. Payment Provider IDs

Example:

~~~sql
CREATE UNIQUE INDEX payments_provider_payment_uidx
ON payments(provider, provider_payment_id)
WHERE provider_payment_id IS NOT NULL;
~~~

This protects against processing the same provider payment identifier more than once.

Your exact constraint depends on the gateway contract.

---

# 27. Never Trust Payment Success From the Browser

Bad flow:

~~~text
Browser says "payment success"
      ↓
database marks order paid
~~~

Correct flow:

~~~text
Payment provider
      ↓
verified server callback / webhook or server verification
      ↓
cryptographic/provider verification
      ↓
server updates payment/order state
~~~

The browser is not a trusted payment authority.

---

# 28. Webhook Idempotency

Payment providers can deliver the same webhook more than once.

Store an event identifier when available:

~~~sql
CREATE TABLE payment_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider TEXT NOT NULL,
    provider_event_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    processed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider, provider_event_id)
);
~~~

Then duplicate delivery can be recognized safely.

---

# 29. Inventory Is a Concurrency Problem

Suppose:

~~~text
stock = 1

User A checks out
User B checks out
at the same time
~~~

Naive flow:

~~~text
A reads stock = 1
B reads stock = 1
A decrements
B decrements
→ overselling risk
~~~

MVCC alone does not solve the business race.

---

# 30. Atomic Inventory Update

A powerful pattern is:

~~~sql
UPDATE products
SET stock = stock - $1
WHERE id = $2
  AND stock >= $1
RETURNING id, stock;
~~~

If no row is returned:

~~~text
product missing
or
insufficient stock
~~~

The condition and decrement happen atomically inside PostgreSQL.

---

# 31. Multiple Products Require a Transaction

Checkout may contain:

~~~text
Product A × 2
Product B × 1
Product C × 3
~~~

You need all required inventory/order writes to succeed together.

Conceptually:

~~~text
BEGIN
  ↓
create order
  ↓
for each item: authoritative product lookup + stock decrement
  ↓
insert order_items snapshots
  ↓
calculate/store authoritative totals
  ↓
COMMIT
~~~

If any required step fails:

~~~text
ROLLBACK
~~~

---

# 32. Node.js Transaction Rule

Use the same checked-out client for the entire transaction:

~~~js
const client = await pool.connect();

try {
  await client.query("BEGIN");

  // all transaction queries use client.query(...)

  await client.query("COMMIT");
} catch (error) {
  await client.query("ROLLBACK");
  throw error;
} finally {
  client.release();
}
~~~

Do not mix independent `pool.query()` calls into the transaction.

---

# 33. External Payment APIs and Transactions

Do **not** hold a PostgreSQL transaction open while waiting several seconds for an external payment gateway.

Bad:

~~~text
BEGIN
  ↓
lock/decrement stock
  ↓
call payment API
  ↓
wait...
  ↓
COMMIT
~~~

This creates long transactions and lock pressure.

Database and payment provider cannot be made one normal PostgreSQL ACID transaction.

---

# 34. Better Checkout State Flow

A simplified architecture:

~~~text
1. Validate cart
2. Calculate authoritative pricing
3. Create pending order/payment intent safely
4. Interact with payment provider
5. Verify provider result/webhook
6. Transition payment/order state
7. Apply inventory reservation/decrement according to chosen business design
~~~

The exact point at which inventory is reserved depends on the commerce requirements.

Do not pretend there is one universal checkout algorithm.

---

# 35. Inventory Reservation Model — Optional Advanced Design

For limited inventory, you may model reservations separately:

~~~text
available stock
reserved stock
confirmed sale
reservation expiry
~~~

Example entity:

~~~sql
CREATE TABLE inventory_reservations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
~~~

This is more complex and should be added only when business behavior requires it.

---

# 36. Order Status Is a State Machine

Do not think of `status` as arbitrary text.

Possible flow:

~~~text
pending
   ↓
confirmed
   ↓
processing
   ↓
shipped
   ↓
delivered
~~~

Other transitions:

~~~text
pending → cancelled
confirmed → cancelled
~~~

Your service layer should enforce valid transitions.

A database CHECK constrains allowed values but does not by itself enforce every allowed transition between values.

---

# 37. Payment Status Is Separate

Order fulfillment status and payment status are different concerns.

Example:

~~~text
order.status = confirmed
payment_status = paid
~~~

or:

~~~text
order.status = cancelled
payment_status = refunded
~~~

Do not overload one status column with every business concept.

---

# 38. Indexing Strategy

Indexes should follow actual access patterns.

Typical ShopHub queries:

~~~text
find product by slug
list active products by category
get newest products
get user's recent orders
get order items by order
lookup provider payment ID
get cart items
~~~

Design indexes around these queries.

---

# 39. Product Listing Index

Example query:

~~~sql
SELECT id, name, price, created_at
FROM products
WHERE category_id = $1
  AND is_active = true
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

Possible partial composite index:

~~~sql
CREATE INDEX products_active_category_created_idx
ON products(category_id, created_at DESC, id DESC)
WHERE is_active = true;
~~~

This aligns with filtering and keyset pagination.

---

# 40. User Orders Index

Query:

~~~sql
SELECT id, status, total_amount, created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

Index:

~~~sql
CREATE INDEX orders_user_created_idx
ON orders(user_id, created_at DESC, id DESC);
~~~

This is a common production access path.

---

# 41. Foreign-Key Indexes

PostgreSQL does not automatically create indexes on referencing foreign-key columns.

Useful examples may include:

~~~sql
CREATE INDEX order_items_order_id_idx
ON order_items(order_id);

CREATE INDEX payments_order_id_idx
ON payments(order_id);

CREATE INDEX product_images_product_id_idx
ON product_images(product_id);
~~~

Index only where query/delete/update behavior benefits from it.

---

# 42. Cart Query

~~~sql
SELECT
    ci.product_id,
    p.name,
    p.price,
    p.stock,
    ci.quantity
FROM cart_items ci
JOIN products p ON p.id = ci.product_id
WHERE ci.cart_id = $1;
~~~

The primary key `(cart_id, product_id)` already begins with `cart_id`, so it can support lookup of cart items by `cart_id`.

Do not add a duplicate `cart_items(cart_id)` index without a measured reason.

---

# 43. Order Details Query

~~~sql
SELECT
    o.id,
    o.status,
    o.payment_status,
    o.total_amount,
    o.created_at,
    oi.product_name,
    oi.unit_price,
    oi.quantity,
    oi.line_total
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.id = $1
  AND o.user_id = $2;
~~~

Notice authorization is included in the query using `user_id`.

Knowing an order UUID must not automatically grant access to the order.

---

# 44. Authorization Near the Data

Bad:

~~~text
GET /orders/:id
      ↓
SELECT order WHERE id = :id
      ↓
return to whoever requested it
~~~

Better:

~~~text
authenticated user
      ↓
SELECT ... WHERE order.id = $1 AND order.user_id = $2
~~~

Admin access can use a separately authorized path.

UUIDs are identifiers, not access-control mechanisms.

---

# 45. Pagination

For shallow admin pages, OFFSET may be acceptable:

~~~sql
LIMIT 20 OFFSET 40;
~~~

For large user-facing feeds/catalogs, keyset pagination is often better:

~~~sql
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

Matching index:

~~~sql
(created_at DESC, id DESC)
~~~

or a composite index beginning with relevant equality filters.

---

# 46. Product Search

Simple search:

~~~sql
WHERE name ILIKE $1
~~~

may be acceptable initially.

As search requirements grow, consider:

~~~text
PostgreSQL full-text search
trigram indexes
dedicated search systems
~~~

Do not prematurely introduce Elasticsearch/OpenSearch for a small catalog.

---

# 47. Audit Trail

Important business actions may need history:

~~~text
order status changes
refunds
admin product changes
inventory adjustments
~~~

Example:

~~~sql
CREATE TABLE order_status_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    old_status TEXT,
    new_status TEXT NOT NULL,
    changed_by UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
~~~

Whether history is written by application code or carefully designed triggers depends on your architecture.

---

# 48. Inventory History

For important stock auditing, storing only current `products.stock` may not be enough.

Possible ledger:

~~~sql
CREATE TABLE inventory_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id),
    order_id UUID REFERENCES orders(id),
    change_quantity INTEGER NOT NULL,
    reason TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
~~~

Examples:

~~~text
-2  order_sale
+5  restock
+1  order_cancelled
~~~

A ledger improves traceability but adds complexity.

---

# 49. Soft Delete

For business entities such as products, consider:

~~~text
is_active BOOLEAN
or
deleted_at TIMESTAMPTZ
~~~

instead of physical deletion.

Advantages:

~~~text
history
recovery
references remain valid
~~~

But soft deletion also complicates queries because every relevant access path must respect the active/deleted state.

---

# 50. Timestamps

Use:

~~~sql
TIMESTAMPTZ
~~~

for real-world instants such as:

~~~text
created_at
updated_at
paid_at
shipped_at
expires_at
~~~

Store timestamps consistently and convert to the user's display timezone at the application/UI boundary.

---

# 51. updated_at

`DEFAULT now()` only sets the value during INSERT.

It does not automatically change on UPDATE.

Options:

~~~text
application explicitly updates updated_at
or
database trigger manages updated_at
~~~

Choose one consistent approach.

---

# 52. Constraints Are the Final Safety Net

Application validation is important, but database constraints protect data regardless of which application code writes it.

Examples:

~~~text
NOT NULL
UNIQUE
FOREIGN KEY
CHECK price >= 0
CHECK quantity > 0
CHECK stock >= 0
~~~

Defense in depth:

~~~text
UI validation
      ↓
server validation
      ↓
authorization
      ↓
database constraints
~~~

---

# 53. SQL Injection Protection

Use parameters:

~~~js
await client.query(
  `SELECT id, name, price
   FROM products
   WHERE id = $1`,
  [productId]
 );
~~~

Never concatenate untrusted values into SQL syntax.

Dynamic sort columns must be allowlisted.

---

# 54. Checkout Transaction — Conceptual SQL

One simplified stock-first checkout transaction could look like:

~~~sql
BEGIN;

INSERT INTO orders (...)
VALUES (...)
RETURNING id;

UPDATE products
SET stock = stock - $1
WHERE id = $2
  AND stock >= $1
RETURNING id, name, price, stock;

INSERT INTO order_items (...)
VALUES (...);

COMMIT;
~~~

If inventory update returns no row:

~~~sql
ROLLBACK;
~~~

In a real multi-item checkout, repeat the inventory/item work consistently for each item within the same transaction and calculate totals from authoritative product/pricing rules.

---

# 55. Lock Ordering

If multiple checkout transactions update several products, lock/update them in a deterministic order where practical.

Example:

~~~text
sort product IDs
      ↓
process A → B → C
~~~

rather than:

~~~text
Transaction 1 → A then B
Transaction 2 → B then A
~~~

Consistent ordering reduces deadlock risk.

PostgreSQL can still detect deadlocks, and applications should handle transaction failures correctly.

---

# 56. Payment Transaction Boundaries

Separate local atomicity from external workflow.

~~~text
PostgreSQL transaction
→ guarantees local DB operations

Payment gateway
→ external distributed system
~~~

You need state transitions, idempotency, verification, and recovery logic—not one giant database transaction around an HTTP payment request.

---

# 57. Webhook Processing Transaction

Conceptual verified webhook flow:

~~~text
Receive webhook
      ↓
verify signature/authenticity
      ↓
BEGIN
      ↓
insert provider event id if not processed
      ↓
lock/read payment/order if required
      ↓
update payment status
      ↓
update order status
      ↓
COMMIT
      ↓
acknowledge provider
~~~

Unique provider event IDs make duplicate webhook delivery safe.

---

# 58. Avoid Double Processing

Pattern:

~~~sql
INSERT INTO payment_events (provider, provider_event_id, event_type, payload)
VALUES ($1, $2, $3, $4)
ON CONFLICT (provider, provider_event_id) DO NOTHING
RETURNING id;
~~~

If no row is returned, the event may already have been recorded.

Your processing design must ensure the state change and event-recording behavior are transactionally coordinated.

---

# 59. Refunds

Refunds may be modeled as payment events or a dedicated table depending on requirements.

Example entity:

~~~text
refunds
  id
  payment_id
  provider_refund_id
  amount
  status
  reason
  created_at
~~~

Do not simply overwrite the original payment amount and lose history.

Financial state benefits from append/history-oriented modeling.

---

# 60. Coupons and Discounts

Do not store only:

~~~text
coupon_id
~~~

if the coupon can later change.

An order should preserve the actual applied result:

~~~text
discount_amount
coupon_code snapshot
discount rule/result if required for audit
~~~

Historical correctness is more important than perfectly eliminating duplication.

---

# 61. Taxes and Shipping

Store the amounts actually charged:

~~~text
subtotal
tax_amount
shipping_amount
discount_amount
total_amount
~~~

Why?

~~~text
tax rules can change
shipping rates can change
product prices can change
~~~

An old order must remain explainable.

---

# 62. Currency

Store currency explicitly:

~~~sql
currency CHAR(3) NOT NULL
~~~

Example:

~~~text
INR
USD
EUR
~~~

Never assume every stored amount is permanently in the application's current default currency.

---

# 63. Multi-Currency Complexity

If ShopHub later supports multiple currencies, decide whether prices are:

~~~text
stored independently per currency
or
converted from a base price
~~~

Orders must preserve the actual transaction currency and charged amounts.

Exchange rates used for historical transactions may also need snapshots/audit data.

---

# 64. Analytics Queries

Example monthly revenue:

~~~sql
SELECT
    date_trunc('month', created_at) AS month,
    SUM(total_amount) AS revenue
FROM orders
WHERE payment_status = 'paid'
GROUP BY date_trunc('month', created_at)
ORDER BY month;
~~~

For heavy repeated analytics, consider:

~~~text
read replica
materialized view
warehouse/analytics system
~~~

rather than making every dashboard query hit the transactional primary indefinitely.

---

# 65. Do Not Mix OLTP and Heavy Analytics Carelessly

Shop checkout is OLTP:

~~~text
small
fast
concurrent
transactional
~~~

Large historical reports may scan millions of rows.

If reporting starts hurting checkout performance, isolate or precompute analytics workloads.

---

# 66. N+1 Problem

Bad:

~~~text
SELECT 100 orders
then
100 separate SELECTs for order_items
~~~

Possible solutions:

~~~text
JOIN
batch query with ANY
ORM relation batching/eager loading
aggregation
~~~

Choose based on response shape and duplication cost.

---

# 67. Avoid SELECT *

For product cards:

~~~sql
SELECT id, name, slug, price
FROM products
...
~~~

not:

~~~sql
SELECT *
FROM products;
~~~

This reduces unnecessary data transfer and creates a clearer application contract.

---

# 68. Production Query Flow

~~~text
HTTP request
    ↓
authenticate
    ↓
authorize
    ↓
validate input
    ↓
service/business logic
    ↓
parameterized PostgreSQL query
    ↓
constraints + transaction
    ↓
result
    ↓
safe API response
~~~

This is the backend mental model to remember.

---

# 69. Suggested Schema Overview

~~~text
users
addresses

categories
products
product_images

carts
cart_items

orders
order_items
order_status_history

payments
payment_events
refunds (optional)

inventory_reservations (optional)
inventory_movements (optional)
~~~

Not every application needs every optional table on day one.

Start with required business invariants and evolve deliberately.

---

# 70. Complete Relationship Diagram

~~~text
users
 ├──< addresses
 ├──< carts
 │     └──< cart_items >── products ──> categories
 │                              │
 └──< orders                   ├──< product_images
       ├──< order_items >──────┘
       ├──< payments
       │      ├──< payment_events (conceptually provider/event history)
       │      └──< refunds
       ├──< order_status_history
       └──< inventory_reservations >── products

products ──< inventory_movements
~~~

Exact foreign-key placement can vary for event/refund/audit requirements.

---

# 71. Production Checkout Flow

~~~text
User clicks Checkout
        ↓
Server loads cart
        ↓
Validate products + quantities
        ↓
Calculate authoritative subtotal/tax/shipping/discount
        ↓
Create controlled order/payment state
        ↓
Safely reserve/decrement inventory according to business design
        ↓
Create payment request
        ↓
Payment provider
        ↓
Verified webhook/server verification
        ↓
Idempotent DB transaction updates payment/order
        ↓
Clear/close cart when appropriate
        ↓
Show confirmed order
~~~

The exact ordering varies with payment/inventory reservation strategy, but the invariants must remain correct.

---

# 72. Failure Scenarios to Design For

Ask what happens when:

~~~text
stock becomes unavailable during checkout
payment succeeds but browser closes
webhook is delivered twice
webhook arrives before browser callback
database transaction fails
payment API times out
same checkout request is retried
product price changes while item sits in cart
admin deactivates a product
refund occurs later
~~~

A production database design is judged by failure behavior, not only the happy path.

---

# 73. Backup and Recovery

Orders and payments are business-critical data.

Production requirements should include:

~~~text
automated backups
PITR
restore testing
retention policy
monitoring
~~~

Replication is not a replacement for backup.

---

# 74. Security

Production database:

~~~text
Application
→ least-privilege role

Migration pipeline
→ migration role

Administrator
→ privileged administrative role
~~~

Also enforce:

~~~text
TLS/network restrictions
secret management
parameterized SQL
authorization
payment verification
audit logging where required
~~~

---

# 75. Monitoring

Monitor ShopHub database for:

~~~text
checkout query latency
lock waits
deadlocks
connection usage
slow product queries
inventory contention
payment update errors
autovacuum
disk growth
WAL
replication lag if replicas exist
backup health
~~~

`pg_stat_statements` and `EXPLAIN ANALYZE` should guide query optimization.

---

# 76. Common Design Mistakes

## Trusting Price From the Client

Always calculate from authoritative server/database data.

## Storing Only product_id in order_items

Preserve purchase-time product name/price and other required historical details.

## Checking Stock With SELECT Then Updating Later Without Concurrency Protection

Use atomic conditional updates, row locking, or another deliberate inventory strategy.

## Holding a DB Transaction Open During Payment API Calls

Keep database transactions short; coordinate external systems with states/idempotency.

## Treating Payment Browser Redirect as Proof

Verify server-side/provider-side.

## No Idempotency

Retries and duplicate webhooks can create duplicate business effects.

## Deleting Products Referenced by Orders

Prefer deactivation/soft-delete or preserve nullable references plus snapshots.

## No Database Constraints

Application validation alone cannot protect against every writer or bug.

## Indexing Every Column

Index actual access patterns and measure.

## One Giant JSONB Document for the Entire Order

Use relational modeling for core entities/relationships; JSONB is useful for snapshots/flexible metadata, not as an excuse to avoid schema design.

---

# 77. Interview Questions

## Why is `order_items` required?

It resolves the many-to-many relationship between orders and products and stores relationship-specific historical information such as quantity and purchase-time price.

## Why copy product price into order_items?

Because product prices change. The order must preserve the price actually charged at purchase time.

## How do you prevent overselling?

Use a transaction plus concurrency-safe inventory logic such as an atomic conditional `UPDATE ... WHERE stock >= quantity RETURNING ...`, or appropriate row locking/reservation design.

## Why not trust the checkout total from React?

The browser is untrusted and may contain stale or manipulated data. The server must calculate totals from authoritative pricing rules/data.

## Why can an order have multiple payment rows?

Payment can involve retries, failed attempts, authorizations, captures, refunds, or provider lifecycle events. Modeling attempts/history separately preserves auditability.

## Why is idempotency important?

Network retries and duplicate provider callbacks can repeat requests. Idempotency ensures the same logical operation does not create duplicate effects.

## Why separate order status and payment status?

They represent different state machines: fulfillment and money movement.

## Why use a historical address snapshot?

A user's saved address can change later, but the order must retain the address used for that purchase.

## Why index `(user_id, created_at DESC, id DESC)` on orders?

It supports a common query: retrieving a user's recent orders efficiently with deterministic keyset pagination.

## Why should checkout use a transaction?

Related local database changes such as order creation, order items, and inventory operations must either succeed together or roll back together.

## Can the payment API be part of the PostgreSQL transaction?

No. It is an external system. Use short local transactions plus state machines, idempotency, verification, and recovery logic.

---

# 78. Interview-Ready Architecture Answer

If asked to design an e-commerce PostgreSQL database, explain:

~~~text
1. Identify entities and business invariants.
2. Normalize core relationships.
3. Use PK/FK/UNIQUE/CHECK constraints.
4. Preserve historical order/payment snapshots.
5. Calculate money server-side using exact values.
6. Protect inventory with transactions + concurrency-safe updates.
7. Make checkout/webhooks idempotent.
8. Index real filters, joins, sorting, and pagination.
9. Keep external payment calls outside long DB transactions.
10. Add backups, monitoring, security, and recovery for production.
~~~

That answer demonstrates database design plus production thinking.

---

# 79. Final Mental Model

~~~text
CATALOG
categories → products → images

SHOPPING
user/guest → cart → cart_items

CHECKOUT
cart
 ↓
authoritative pricing
 ↓
transaction / inventory protection
 ↓
order → order_items snapshots
 ↓
payment intent

PAYMENT
provider
 ↓
verified idempotent callback
 ↓
payments / order state

OPERATIONS
indexes + constraints + monitoring + backups + security
~~~

---

## Key Takeaway

> **A production e-commerce database is not just a set of CRUD tables. Core relational data should be normalized and protected with constraints; orders must preserve historical price/address/payment facts; inventory requires concurrency-safe transactions; payment workflows require server-side verification and idempotency; and indexes must follow real query patterns. Production correctness comes from designing for retries, concurrent users, mutable catalog data, external payment failures, and recovery—not merely the happy checkout path.**

---

[← Previous: Lesson 43 — Production Deployment](../10-production/43-production-deployment.md) | [Back to Roadmap](../README.md) | [Next: Lesson 45 — Production Database Optimization →](./45-production-database-optimization.md)