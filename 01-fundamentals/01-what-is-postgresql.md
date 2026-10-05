# Lesson 1 — What is PostgreSQL?

## 1. What is a Database?

A **database** is an organized collection of data that allows applications to store, manage, update, and retrieve information efficiently.

For example, an e-commerce application may store:

- Users
- Products
- Orders
- Payments
- Reviews

Instead of storing this information manually in files, applications commonly use a **Database Management System (DBMS)**.

---

## 2. What is a DBMS?

A **DBMS (Database Management System)** is software used to create, manage, access, and control databases.

Examples include:

- PostgreSQL
- MySQL
- Microsoft SQL Server
- Oracle Database
- MongoDB

PostgreSQL is a **relational/object-relational database management system**.

---

## 3. What is PostgreSQL?

**PostgreSQL** is a powerful, open-source relational database management system.

It stores structured data mainly in **tables** and allows relationships to be created between those tables.

Example:

```text
Users
  |
  | 1 : N
  v
Orders
```

One user can have many orders.

PostgreSQL provides features such as:

- SQL querying
- Transactions
- Constraints
- Relationships
- Indexes
- Views
- JSON/JSONB
- Functions and procedures
- Concurrency control
- Security and roles
- Extensions

---

## 4. Why is PostgreSQL Called Relational?

In a relational database, data is organized into tables.

Example:

### users

| id | name | email |
|---|---|---|
| 1 | Vikash | vikash@example.com |
| 2 | Rahul | rahul@example.com |

### orders

| id | user_id | amount |
|---|---|---|
| 101 | 1 | 2500 |
| 102 | 1 | 1500 |

The `user_id` column connects an order with a user.

```text
users
+----+--------+
| id | name   |
+----+--------+
| 1  | Vikash |
+----+--------+
       |
       | id = user_id
       |
       v
orders
+-----+---------+--------+
| id  | user_id | amount |
+-----+---------+--------+
| 101 |    1    | 2500   |
| 102 |    1    | 1500   |
+-----+---------+--------+
```

This relationship between tables is one of the main ideas behind relational databases.

---

## 5. SQL vs PostgreSQL

A common interview question is:

> What is the difference between SQL and PostgreSQL?

### SQL

**SQL (Structured Query Language)** is a language used to communicate with relational databases.

Example:

```sql
SELECT *
FROM users;
```

### PostgreSQL

PostgreSQL is the **database management system** that understands and executes SQL.

Think of it as:

```text
SQL
  |
  | commands
  v
PostgreSQL
  |
  v
Database
```

So:

```text
SQL        = Language
PostgreSQL = Database Management System
```

---

## 6. PostgreSQL in a Full-Stack Application

A typical application architecture may look like:

```text
React / Next.js
       |
       | HTTP request
       v
Node.js / Express
       |
       | SQL query
       v
PostgreSQL
       |
       v
Stored Data
```

The frontend should normally **not connect directly to PostgreSQL**.

Instead, the backend/server-side application communicates with PostgreSQL.

---

## 7. PostgreSQL Server Hierarchy

A PostgreSQL server can manage multiple databases.

A simplified hierarchy is:

```text
PostgreSQL Server
│
├── Database 1
│   │
│   ├── Schema
│   │   │
│   │   ├── Table
│   │   ├── View
│   │   └── Function
│   │
│   └── Other schemas
│
├── Database 2
│
└── Database 3
```

For the table itself:

```text
Table
│
├── Columns
│
└── Rows
```

The important hierarchy to remember is:

```text
PostgreSQL Server
        ↓
     Database
        ↓
      Schema
        ↓
      Table
        ↓
 Rows + Columns
```

We explore these concepts in detail in Lesson 2.

---

## 8. What is psql?

`psql` is PostgreSQL's command-line client.

It allows us to connect to PostgreSQL and execute SQL commands directly from the terminal.

Example:

```bash
psql -U postgres
```

Once connected, we can execute SQL:

```sql
SELECT *
FROM users;
```

We can also execute **psql meta-commands**.

Examples:

```text
\l
\c database_name
\dt
\d users
\q
```

### Common psql Commands

| Command | Purpose |
|---|---|
| `\l` | List databases |
| `\c database_name` | Connect to a database |
| `\dt` | List tables |
| `\d users` | Describe the users table |
| `\q` | Exit psql |

---

## 9. SQL Commands vs psql Commands

These are different.

### SQL command

```sql
SELECT *
FROM users;
```

SQL is sent to PostgreSQL for execution.

### psql command

```text
\dt
```

This is a command understood by the **psql client**.

So remember:

```text
SELECT * FROM users;
        ↓
      SQL

\dt
 ↓
psql meta-command
```

---

## 10. PostgreSQL's Role in an Application

PostgreSQL is responsible for much more than simply storing rows.

It can help enforce:

- Data types
- Primary keys
- Foreign keys
- Unique values
- Constraints
- Transactions
- Concurrency
- Security
- Indexes

Example:

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);
```

Here PostgreSQL itself helps guarantee that:

- `id` uniquely identifies a user.
- `name` cannot be NULL.
- `email` cannot be NULL.
- duplicate emails are rejected.

These concepts are covered in later lessons.

---

## 11. Important Mental Model

Do not think:

```text
PostgreSQL = place where data is saved
```

A better mental model is:

```text
PostgreSQL
    │
    ├── Stores data
    ├── Queries data
    ├── Enforces relationships
    ├── Protects data integrity
    ├── Handles transactions
    ├── Handles concurrent users
    ├── Controls permissions
    └── Optimizes data access
```

---

# Interview Revision

## What is PostgreSQL?

PostgreSQL is an open-source relational/object-relational database management system used to store, manage, query, and maintain structured data reliably.

## What is a database?

A database is an organized collection of data that applications can store, retrieve, update, and manage.

## What is a DBMS?

A DBMS is software that manages databases and provides mechanisms for storing, querying, securing, and maintaining data.

## SQL vs PostgreSQL?

```text
SQL        → Language
PostgreSQL → Database Management System
```

## What does relational mean?

Relational databases organize data into tables and allow relationships between those tables using keys.

## What is psql?

`psql` is PostgreSQL's command-line client used to connect to PostgreSQL and execute SQL and psql-specific commands.

---

# Quick Revision

```text
Database
→ Organized collection of data

DBMS
→ Software that manages databases

PostgreSQL
→ Open-source relational/object-relational DBMS

SQL
→ Language used to communicate with relational databases

psql
→ PostgreSQL command-line client

Basic hierarchy:

PostgreSQL Server
      ↓
   Database
      ↓
    Schema
      ↓
    Table
      ↓
Rows + Columns
```

---

## Key Takeaway

> **PostgreSQL is the database management system, SQL is the language used to communicate with it, and psql is one of the tools we can use to interact with PostgreSQL.**

---

[← Back to PostgreSQL Learning Roadmap](../README.md)
