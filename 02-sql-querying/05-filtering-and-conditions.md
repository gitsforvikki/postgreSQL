# Lesson 5 — Filtering and Conditions

## 1. What is Filtering?

Filtering means selecting only the rows that match a condition.

In SQL, filtering is mainly done using:

```sql
WHERE
```

Example:

```sql
SELECT *
FROM users
WHERE age >= 18;
```

Conceptually:

```text
users table
    ↓
WHERE condition
    ↓
matching rows only
```

---

# 2. WHERE Clause

Suppose we have:

```text
users
+----+--------+-----+-----------+
| id | name   | age | is_active |
+----+--------+-----+-----------+
| 1  | Vikash | 25  | true      |
| 2  | Rahul  | 17  | true      |
| 3  | Aman   | 30  | false     |
| 4  | Neha   | 22  | true      |
+----+--------+-----+-----------+
```

Query:

```sql
SELECT *
FROM users
WHERE age >= 18;
```

PostgreSQL evaluates the condition for each row and returns the matching rows.

---

# 3. Comparison Operators

Important comparison operators:

| Operator | Meaning |
|---|---|
| `=` | Equal |
| `<>` | Not equal |
| `!=` | Not equal |
| `>` | Greater than |
| `<` | Less than |
| `>=` | Greater than or equal |
| `<=` | Less than or equal |

Examples:

```sql
SELECT *
FROM products
WHERE price > 1000;
```

```sql
SELECT *
FROM users
WHERE age <= 30;
```

```sql
SELECT *
FROM users
WHERE name = 'Vikash';
```

---

# 4. AND Operator

`AND` requires **all conditions** to be true.

```sql
SELECT *
FROM users
WHERE age >= 18
  AND is_active = true;
```

Mental model:

```text
Condition A = true
       +
Condition B = true
       ↓
row returned
```

If either condition is false, the row does not satisfy the combined condition.

---

# 5. OR Operator

`OR` requires at least one condition to be true.

```sql
SELECT *
FROM users
WHERE age < 18
   OR age > 60;
```

Conceptually:

```text
Condition A true
       OR
Condition B true
       ↓
row may be returned
```

---

# 6. NOT Operator

`NOT` negates a condition.

Example:

```sql
SELECT *
FROM users
WHERE NOT is_active;
```

Another example:

```sql
SELECT *
FROM products
WHERE NOT price > 5000;
```

---

# 7. Parentheses with AND and OR

When combining `AND` and `OR`, parentheses make the intended logic clear.

Example:

```sql
SELECT *
FROM users
WHERE is_active = true
  AND (age < 25 OR age > 40);
```

Mental model:

```text
            age < 25
              OR
            age > 40
               │
               ▼
          grouped result
               │
               AND
               │
      is_active = true
```

Without careful grouping, a query may return rows you did not intend.

---

# 8. IN Operator

Instead of writing:

```sql
SELECT *
FROM orders
WHERE status = 'pending'
   OR status = 'paid'
   OR status = 'shipped';
```

you can write:

```sql
SELECT *
FROM orders
WHERE status IN ('pending', 'paid', 'shipped');
```

`IN` is useful when a value can match one of several options.

Mental model:

```text
status
   ↓
Is it inside this set?
[pending, paid, shipped]
```

---

# 9. NOT IN

To exclude several values:

```sql
SELECT *
FROM orders
WHERE status NOT IN ('cancelled', 'refunded');
```

This means:

```text
return rows whose status
is not one of those values
```

Be careful when `NULL` values are involved because SQL uses three-valued logic. This becomes especially important with `NOT IN` subqueries, which we cover in Lesson 8.

---

# 10. BETWEEN

`BETWEEN` checks whether a value falls inside a range.

```sql
SELECT *
FROM products
WHERE price BETWEEN 1000 AND 5000;
```

Important:

> `BETWEEN` is inclusive.

Conceptually:

```text
1000 <= price <= 5000
```

So both boundary values can match.

Equivalent:

```sql
SELECT *
FROM products
WHERE price >= 1000
  AND price <= 5000;
```

---

# 11. NOT BETWEEN

```sql
SELECT *
FROM products
WHERE price NOT BETWEEN 1000 AND 5000;
```

This selects values outside that range, subject to normal SQL `NULL` behavior.

---

# 12. LIKE

`LIKE` performs pattern matching.

Two important wildcards are:

```text
% → zero or more characters
_ → exactly one character
```

Example:

```sql
SELECT *
FROM users
WHERE name LIKE 'Vik%';
```

Possible matches:

```text
Vikash
Vikas
Vikki
```

---

# 13. LIKE Patterns

Starts with:

```sql
WHERE name LIKE 'Vi%'
```

Ends with:

```sql
WHERE name LIKE '%sh'
```

Contains:

```sql
WHERE name LIKE '%kas%'
```

Exactly one wildcard character:

```sql
WHERE name LIKE 'V_kash'
```

Here `_` represents exactly one character.

---

# 14. ILIKE — PostgreSQL Feature

PostgreSQL provides:

```sql
ILIKE
```

for case-insensitive pattern matching.

Example:

```sql
SELECT *
FROM users
WHERE name ILIKE 'vik%';
```

It can match values such as:

```text
Vikash
VIKASH
vikash
```

Useful memory:

```text
LIKE
→ pattern matching

ILIKE
→ case-insensitive pattern matching
→ PostgreSQL-specific operator
```

---

# 15. NULL

`NULL` represents a missing or unknown value.

It is not equivalent to:

```text
0
''
false
```

Example:

```text
phone_number = NULL
```

may mean that the phone number is unknown or has not been stored.

---

# 16. Wrong NULL Comparison

Do not write:

```sql
SELECT *
FROM users
WHERE phone = NULL;
```

This does not work like normal equality.

Likewise:

```sql
WHERE phone != NULL
```

is not the correct way to test for a non-NULL value.

---

# 17. IS NULL

Correct:

```sql
SELECT *
FROM users
WHERE phone IS NULL;
```

For non-NULL values:

```sql
SELECT *
FROM users
WHERE phone IS NOT NULL;
```

Remember:

```text
= NULL      ❌

IS NULL     ✅

!= NULL     ❌

IS NOT NULL ✅
```

---

# 18. Why NULL Behaves Differently

SQL uses **three-valued logic**.

A condition can evaluate to:

```text
TRUE
FALSE
UNKNOWN
```

Comparisons involving `NULL` often produce:

```text
UNKNOWN
```

For example, conceptually:

```text
NULL = 10
→ UNKNOWN

NULL = NULL
→ UNKNOWN
```

because `NULL` means an unknown value.

That is why SQL provides:

```text
IS NULL
IS NOT NULL
```

for NULL testing.

---

# 19. TRUE, FALSE and UNKNOWN

Suppose:

```text
age = NULL
```

Then:

```sql
age > 18
```

does not evaluate to true or false in the normal sense.

It evaluates to:

```text
UNKNOWN
```

A `WHERE` clause returns rows where the condition evaluates to **TRUE**.

Rows where the condition is FALSE or UNKNOWN are filtered out.

This explains many surprising NULL behaviors.

---

# 20. Combining Filters

Real queries usually combine multiple conditions.

Example:

```sql
SELECT id, name, age
FROM users
WHERE is_active = true
  AND age BETWEEN 18 AND 35
  AND name ILIKE 'v%'
ORDER BY age DESC;
```

Conceptual flow:

```text
users
  ↓
is_active = true
  ↓
age between 18 and 35
  ↓
name starts with v
  ↓
sort by age
  ↓
result
```

---

# 21. Filtering Dates

Suppose:

```text
created_at
```

is a timestamp.

You can filter using comparisons:

```sql
SELECT *
FROM orders
WHERE created_at >= '2026-01-01';
```

Or a range:

```sql
SELECT *
FROM orders
WHERE created_at >= '2026-01-01'
  AND created_at < '2027-01-01';
```

For timestamp ranges, using a half-open range like:

```text
>= start
< next boundary
```

is often clearer than trying to manually represent the final instant of a day/month/year.

---

# 22. Backend Filtering Flow

Suppose the frontend requests:

```text
GET /products?minPrice=1000&maxPrice=5000
```

The flow is:

```text
Frontend
   ↓
Query parameters
   ↓
Node.js / Express / Next.js
   ↓
Validate values
   ↓
Parameterized SQL WHERE conditions
   ↓
PostgreSQL
   ↓
Matching rows
   ↓
API response
```

Conceptually:

```sql
SELECT *
FROM products
WHERE price >= $1
  AND price <= $2;
```

with parameter values supplied separately.

Filtering logic is therefore directly connected to real API development.

---

# 23. Example: Product Search

Suppose we want:

- Active products
- Price between 1000 and 5000
- Name containing "phone"
- Stock available

Query:

```sql
SELECT id, name, price, stock
FROM products
WHERE is_active = true
  AND price BETWEEN 1000 AND 5000
  AND name ILIKE '%phone%'
  AND stock > 0
ORDER BY price ASC;
```

This is a realistic e-commerce filtering query.

---

# 24. Common Mistakes

## Using = NULL

Wrong:

```sql
WHERE email = NULL
```

Correct:

```sql
WHERE email IS NULL
```

---

## Forgetting Parentheses

Potentially confusing:

```sql
WHERE is_active = true
  AND age < 18
  OR age > 60
```

Clearer:

```sql
WHERE is_active = true
  AND (age < 18 OR age > 60)
```

Use parentheses to express the intended logic clearly.

---

## Forgetting BETWEEN is Inclusive

```sql
price BETWEEN 100 AND 500
```

includes both:

```text
100
500
```

---

## Confusing LIKE and ILIKE

```text
LIKE
→ pattern matching

ILIKE
→ case-insensitive pattern matching in PostgreSQL
```

---

## Treating NULL as an Empty String

These are different:

```text
NULL
''
```

`NULL` represents missing/unknown data.

An empty string is an actual string value with zero characters.

---

# Interview Revision

## What is WHERE?

`WHERE` filters rows according to a condition.

## Difference between AND and OR?

```text
AND
→ all conditions must be true

OR
→ at least one condition must be true
```

## What does IN do?

It checks whether a value matches one of the values in a specified set.

## Is BETWEEN inclusive?

Yes. Both boundary values are included.

## LIKE vs ILIKE?

```text
LIKE
→ pattern matching

ILIKE
→ case-insensitive pattern matching in PostgreSQL
```

## What do % and _ mean in LIKE?

```text
%
→ zero or more characters

_
→ exactly one character
```

## How do you check for NULL?

```sql
IS NULL
```

or:

```sql
IS NOT NULL
```

## Why doesn't = NULL work?

Because `NULL` represents an unknown value. Normal comparisons involving NULL evaluate to `UNKNOWN`, so SQL provides `IS NULL` and `IS NOT NULL`.

## What is SQL three-valued logic?

SQL conditions can evaluate to:

```text
TRUE
FALSE
UNKNOWN
```

The `UNKNOWN` state primarily appears because of NULL values.

---

# Quick Revision

```text
WHERE
→ filter rows

=
→ equal

<>
→ not equal

AND
→ all conditions

OR
→ any condition

NOT
→ negate condition

IN
→ match one of several values

BETWEEN
→ inclusive range

LIKE
→ pattern matching

ILIKE
→ case-insensitive pattern matching

%
→ any number of characters

_
→ one character

IS NULL
→ value is NULL

IS NOT NULL
→ value is not NULL
```

### NULL Mental Model

```text
NULL
  ↓
unknown / missing
  ↓
SQL three-valued logic

TRUE
FALSE
UNKNOWN
```

### Backend Mental Model

```text
Query Params
     ↓
Validation
     ↓
WHERE conditions
     ↓
PostgreSQL
     ↓
Filtered Results
```

---

## Key Takeaway

> **The WHERE clause controls which rows a query operates on. Master comparison operators, logical operators, IN, BETWEEN, pattern matching, and especially NULL handling because these form the foundation of real-world SQL filtering.**

---

[← Previous: Lesson 4 — PostgreSQL Data Types](../01-fundamentals/04-data-types.md) | [Back to Roadmap](../README.md) | [Next: Lesson 6 — Aggregate Functions →](./06-aggregate-functions.md)
