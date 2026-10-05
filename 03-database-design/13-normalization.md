# Lesson 13 — Normalization

## First Understand the Problem

Before learning definitions like **1NF, 2NF, and 3NF**, understand why normalization exists.

Imagine we store e-commerce data like this:

~~~text
orders

+----------+---------------+---------------------+------------------+
| order_id | customer_name | customer_phone      | products         |
+----------+---------------+---------------------+------------------+
| 101      | Vikash        | 9876543210          | Laptop, Mouse    |
| 102      | Vikash        | 9876543210          | Keyboard         |
| 103      | Rahul         | 9123456780          | Monitor          |
+----------+---------------+---------------------+------------------+
~~~

At first this may look convenient, but it creates problems.

Vikash's name and phone number are repeated in every order. Multiple products are also stored inside one field.

If the phone number changes, we may need to update many rows.

~~~text
Same information
      ↓
stored repeatedly
      ↓
more chances of inconsistent data
~~~

**Normalization is the process of organizing relational data so unnecessary duplication and data anomalies are reduced.**

---

# 1. Why Do We Normalize Data?

The main goals are:

- Reduce unnecessary duplication
- Prevent inconsistent data
- Make updates safer
- Make relationships clearer
- Improve database design and maintainability

A useful mental model:

~~~text
Badly structured table
        ↓
Repeated / mixed data
        ↓
Normalization
        ↓
Separate related entities
        ↓
Connect them using PK + FK
~~~

Normalization does **not** mean removing every repeated value. Some repetition is natural and necessary. The goal is to remove unnecessary dependency and duplication.

---

# 2. What Problems Does Bad Design Cause?

Three classic problems are called **anomalies**.

## Update Anomaly

Suppose Vikash appears in 50 order rows with the same phone number.

If the phone changes:

~~~text
50 rows need updating
        ↓
49 updated correctly
1 accidentally missed
        ↓
same customer now has two phone numbers
~~~

That is an **update anomaly**.

Better design:

~~~text
customers
customer_id | name   | phone
1           | Vikash | 9999999999

orders
order_id | customer_id
101      | 1
102      | 1
~~~

Now the phone exists in one appropriate place.

## Insert Anomaly

Suppose one large table stores customer and order information together.

What if a customer registers but has not placed an order yet?

If the design requires order fields, storing the customer becomes awkward or impossible.

That is an **insert anomaly**.

Separate tables solve it:

~~~text
customers
→ customer can exist independently

orders
→ created only when an order exists
~~~

## Delete Anomaly

Suppose Rahul has exactly one order and his customer information exists only in that order row.

If we delete that order:

~~~text
delete Rahul's only order
        ↓
Rahul's customer information also disappears
~~~

We unintentionally lost unrelated information.

That is a **delete anomaly**.

### Remember

~~~text
UPDATE anomaly
→ same fact must be changed in many places

INSERT anomaly
→ cannot store one fact without another unrelated fact

DELETE anomaly
→ deleting one fact accidentally removes another
~~~

---

# 3. Functional Dependency — Simple Meaning

This term sounds difficult, but the idea is simple.

If knowing column A determines exactly one value of column B, we say B depends on A.

Example:

~~~text
customer_id → customer_name
customer_id → customer_email
~~~

If:

~~~text
customer_id = 10
~~~

identifies one customer, then it determines that customer's name and email.

We write:

~~~text
customer_id → customer_name
~~~

Read it as:

> customer_name depends on customer_id.

Functional dependency helps us decide whether data belongs in the same table.

---

# 4. Normal Forms

The main normal forms you should understand for interviews and practical application development are:

~~~text
1NF
 ↓
2NF
 ↓
3NF
~~~

Simple memory trick:

~~~text
1NF → one value per cell
2NF → depend on the whole key
3NF → depend only on the key
~~~

We will understand each using examples.

---

# 5. First Normal Form — 1NF

## Problem

Suppose:

~~~text
orders

+----------+-------------------------+
| order_id | products                |
+----------+-------------------------+
| 101      | Laptop, Mouse, Keyboard |
+----------+-------------------------+
~~~

The `products` column contains multiple values.

This makes querying awkward.

For example:

> Find every order containing product 15.

We would have to parse values stored inside one field.

## 1NF Idea

For the practical relational-design mental model used here, keep each field atomic for the model instead of storing a repeating list inside one cell.

Instead of:

~~~text
order_id | products
101      | Laptop, Mouse
~~~

design related rows:

~~~text
orders

order_id
101

order_items

order_id | product_id
101      | 10
101      | 20
~~~

Now each product relationship has its own row.

### 1NF Mental Model

~~~text
Bad
products = 'Laptop, Mouse, Keyboard'

Better relational design
one order
   ↓
many order_items
   ↓
one product reference per row
~~~

### Important Note

1NF is often summarized as **atomic values / no repeating groups**. In practice, the exact meaning of atomic depends on the data model. The important lesson is not to hide a real one-to-many relationship inside a comma-separated field.

---

# 6. Second Normal Form — 2NF

2NF becomes especially important when a table has a **composite key**.

Remember from Lesson 11:

~~~sql
PRIMARY KEY (order_id, product_id)
~~~

The complete key contains two columns.

## Problem Example

Suppose:

~~~text
order_items

+----------+------------+--------------+-------------+----------+
| order_id | product_id | product_name | order_date  | quantity |
+----------+------------+--------------+-------------+----------+
| 101      | 10         | Laptop       | 2026-10-01  | 1        |
| 101      | 20         | Mouse        | 2026-10-01  | 2        |
+----------+------------+--------------+-------------+----------+
~~~

Primary key:

~~~text
(order_id, product_id)
~~~

Now examine the dependencies:

~~~text
quantity
→ depends on order_id + product_id

product_name
→ depends only on product_id

order_date
→ depends only on order_id
~~~

This is the problem.

Some non-key columns depend on only **part** of the composite key.

This is called a **partial dependency**.

## Fix

Move facts to the entity they actually describe.

~~~text
orders
order_id | order_date

products
product_id | product_name

order_items
order_id | product_id | quantity
~~~

Now:

~~~text
order_date
→ belongs to order

product_name
→ belongs to product

quantity
→ belongs to the relationship between order and product
~~~

That is much cleaner.

## 2NF Definition

A table is in **Second Normal Form** when:

1. It is already in 1NF.
2. Non-key attributes do not depend on only part of a candidate key; in the common composite-key case, they depend on the whole key.

### Easy Memory

~~~text
Composite key = A + B

Wrong:
column depends only on A

Wrong:
column depends only on B

Correct for relationship-specific fact:
column depends on A + B
~~~

### Important Interview Point

If a table has a single-column candidate key, the classic partial-dependency problem cannot occur for that key.

---

# 7. Third Normal Form — 3NF

Now suppose our table is already in 2NF.

Consider:

~~~text
users

+---------+--------+-------------+---------------+
| user_id | name   | city_id     | city_name     |
+---------+--------+-------------+---------------+
| 1       | Vikash | 10          | Ahmedabad     |
| 2       | Rahul  | 20          | Bengaluru     |
+---------+--------+-------------+---------------+
~~~

Dependencies:

~~~text
user_id → city_id
city_id → city_name
~~~

Therefore:

~~~text
user_id
   ↓
city_id
   ↓
city_name
~~~

`city_name` does not directly describe the user identity. It describes the city.

This is a **transitive dependency**.

## Fix

Separate cities:

~~~text
users

user_id | name   | city_id
1       | Vikash | 10

cities

city_id | city_name
10      | Ahmedabad
~~~

Relationship:

~~~text
users.city_id
      ↓
cities.city_id
~~~

Now city information has one proper source.

## 3NF Definition

A practical interview definition is:

> A table in 2NF should not have non-key attributes depending on other non-key attributes in a way that creates a transitive dependency on the key.

### Easy Memory

~~~text
Wrong

Primary Key
    ↓
non-key column A
    ↓
non-key column B

Better

Primary Key → appropriate attributes

Separate entity key → its own attributes
~~~

---

# 8. The Famous Memory Rule

A common interview memory aid is:

> **The key, the whole key, and nothing but the key.**

Meaning:

~~~text
1NF
→ keep values properly structured / avoid repeating groups

2NF
→ attributes depend on the whole relevant key

3NF
→ non-key attributes should not depend transitively on other non-key attributes
~~~

Do not memorize this sentence without understanding the examples above.

---

# 9. Full E-Commerce Example

Imagine one badly designed table:

~~~text
order_data

+----------+---------------+----------------+--------------+--------------+----------+
| order_id | customer_name | customer_email | product_name | unit_price   | quantity |
+----------+---------------+----------------+--------------+--------------+----------+
| 101      | Vikash        | v@email.com    | Laptop       | 60000        | 1        |
| 101      | Vikash        | v@email.com    | Mouse        | 1000         | 2        |
| 102      | Vikash        | v@email.com    | Keyboard     | 2000         | 1        |
+----------+---------------+----------------+--------------+--------------+----------+
~~~

Problems:

~~~text
customer data repeated
product data repeated
order data repeated
different entities mixed together
~~~

A normalized design separates the entities.

~~~text
users
-----
id
name
email

orders
------
id
user_id
created_at

products
--------
id
name
current_price

order_items
-----------
order_id
product_id
quantity
price_at_order
~~~

Relationships:

~~~text
users
  │
  │ 1:N
  ▼
orders
  │
  │ 1:N
  ▼
order_items
  ▲
  │ N:1
  │
products
~~~

This design is easier to maintain and protects each entity's facts.

---

# 10. Is Repeated Data Always Bad?

No.

This is extremely important.

Consider:

~~~text
products.current_price = 65000
~~~

A customer purchased it yesterday for:

~~~text
60000
~~~

If `order_items` only references the product and always reads the current product price, old orders would appear to change when product prices change.

So storing:

~~~text
order_items.price_at_order = 60000
~~~

is useful.

This is not accidental duplication. It is a **historical snapshot** representing a different fact.

~~~text
products.current_price
→ price now

order_items.price_at_order
→ price agreed at purchase time
~~~

They have different meanings.

### Key Lesson

> Normalization removes unnecessary redundancy, not meaningful historical data.

---

# 11. Normalization vs Relationships

Normalization often creates separate tables.

Those tables are connected using:

~~~text
Primary Keys
     +
Foreign Keys
~~~

Example:

~~~text
users.id
   ▲
   │
orders.user_id
~~~

So Lessons 11–13 connect directly:

~~~text
Primary Key
     ↓
identify entity

Foreign Key
     ↓
connect entities

Normalization
     ↓
decide which facts belong to which entity
~~~

---

# 12. Should We Always Normalize Everything?

Not necessarily.

For most transactional application databases, a normalized design is an excellent starting point.

But sometimes systems intentionally duplicate or precompute data for performance or reporting.

This is called **denormalization**.

~~~text
Normalization
→ reduce unnecessary redundancy
→ stronger consistency
→ cleaner write model

Denormalization
→ intentionally duplicate/precompute some data
→ potentially simpler/faster reads
→ requires consistency strategy
~~~

Example: a reporting table may store precomputed daily sales totals instead of calculating millions of order rows on every dashboard request.

Do not denormalize just because joins look inconvenient. First design correctly, measure the real performance problem, and then optimize intentionally.

---

# 13. Normalization vs Performance

A common misconception is:

~~~text
More normalized tables
        ↓
More JOINs
        ↓
Therefore normalization is bad
~~~

That conclusion is too simplistic.

PostgreSQL is designed to join relational tables efficiently when schemas, queries, indexes, and statistics are appropriate.

A good workflow is:

~~~text
Design correctly
      ↓
Normalize important entities
      ↓
Add appropriate indexes
      ↓
Measure real queries
      ↓
Optimize if necessary
      ↓
Denormalize only when justified
~~~

---

# 14. How to Normalize a Table — Practical Method

When you see a large table, ask these questions.

### Step 1 — What are the real entities?

Example:

~~~text
User
Order
Product
Category
~~~

### Step 2 — Which columns describe each entity?

~~~text
User
→ name, email

Product
→ name, current_price

Order
→ user_id, created_at, status
~~~

### Step 3 — Are multiple values hidden in one field?

If yes, identify whether there is actually a separate one-to-many or many-to-many relationship.

### Step 4 — Does a column depend on only part of a composite key?

If yes, investigate a 2NF violation.

### Step 5 — Does one non-key column determine another non-key fact?

If yes, investigate a transitive dependency / 3NF issue.

### Step 6 — Create relationships

Use primary and foreign keys to connect the separated entities.

### Step 7 — Preserve intentional historical facts

Do not remove values such as `price_at_order` merely because a similar current value exists elsewhere.

---

# 15. Common Mistakes

## Memorizing 1NF, 2NF, 3NF Without Understanding Dependencies

Instead ask:

> What real-world fact does this column describe, and what key determines it?

## Putting Comma-Separated Relationships in One Column

Bad relational design for a true product relationship:

~~~text
product_ids = '10,20,30'
~~~

Usually better:

~~~text
order_items
order_id | product_id
~~~

## Duplicating Customer Data in Every Order

Usually store the customer relationship with a foreign key rather than copying current profile fields everywhere.

However, some immutable historical snapshots such as billing/shipping details may intentionally be stored with an order depending on business requirements.

## Thinking Every Duplicate Value Violates Normalization

Repeated values are not automatically wrong. What matters is the dependency and meaning of the data.

## Denormalizing Before Measuring

Do not intentionally duplicate data for performance before identifying a real bottleneck.

---

# Interview Revision

## What is normalization?

Normalization is the process of organizing relational data to reduce unnecessary redundancy and prevent update, insert, and delete anomalies.

## Why normalize?

~~~text
reduce unnecessary duplication
improve consistency
avoid anomalies
clarify relationships
make data easier to maintain
~~~

## What are the three common anomalies?

~~~text
Update anomaly
Insert anomaly
Delete anomaly
~~~

## What is functional dependency?

If one attribute/key determines another attribute, the second is functionally dependent on the first.

Example:

~~~text
user_id → email
~~~

## What is 1NF?

For the practical model here, values are kept appropriately atomic and repeating groups such as comma-separated one-to-many relationships are avoided.

## What is 2NF?

The table is in 1NF and non-key attributes do not have improper partial dependency on only part of a candidate key, especially relevant with composite keys.

## What is 3NF?

The table is in 2NF and avoids transitive dependencies where non-key facts improperly depend on other non-key facts.

## What is partial dependency?

With a composite key, a non-key attribute depends on only part of that key.

## What is transitive dependency?

A key determines one non-key attribute, which then determines another non-key attribute.

~~~text
Key → A → B
~~~

## What is denormalization?

Intentional duplication or precomputation of data, usually for a measured performance/reporting reason, with a strategy to maintain correctness.

## Should databases always be fully normalized?

Normalization is a strong starting point for transactional design, but production systems may intentionally denormalize selected data when justified.

---

# Quick Revision

~~~text
NORMALIZATION
      ↓
organize data correctly
      ↓
reduce unnecessary redundancy
      ↓
avoid anomalies
~~~

### Normal Forms

~~~text
1NF
→ one properly modeled value per field / no repeating groups

2NF
→ whole key

3NF
→ only the appropriate key; avoid transitive dependency
~~~

### Anomalies

~~~text
UPDATE
→ same fact inconsistent across rows

INSERT
→ cannot add one fact independently

DELETE
→ removing one fact loses another
~~~

### Dependency Diagram

~~~text
2NF problem

(order_id, product_id)
      ↓
product_name depends only on product_id

3NF problem

user_id
  ↓
city_id
  ↓
city_name
~~~

### E-Commerce Design

~~~text
users
  │
orders
  │
order_items ─── products
~~~

### Important Exception

~~~text
products.current_price
≠
order_items.price_at_order

current fact
≠
historical fact
~~~

---

## Key Takeaway
> **Normalization is not mainly about memorizing 1NF, 2NF, and 3NF. It is about asking where each fact truly belongs, storing that fact in the right place, and connecting related entities with keys so the database remains consistent.**

---

[← Previous: Lesson 12 — Constraints](./12-constraints.md) | [Back to Roadmap](../README.md) | [Next: Lesson 14 — Relationships & Modeling →](./14-relationships-and-modeling.md)