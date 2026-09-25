## Relational Database

### What Is a Relational Database

A **relational database** stores data in **tables** made of **rows** and
**columns**. Tables are linked to each other through **keys**. You read and
write the data with **SQL** (Structured Query Language).

```
users                     orders
+----+----------+         +----+---------+--------+
| id | name     |         | id | user_id | status |
+----+----------+         +----+---------+--------+
|  1 | Ashutosh | ◄────── | 10 |    1    | paid   |
|  2 | Priya    |         | 11 |    1    | pending|
+----+----------+         +----+---------+--------+
                           user_id is a FOREIGN KEY → users.id
```

Each fact is stored **once**, in one table, and you **JOIN** tables together
when you read. The database enforces the rules between tables: no order can
point to a user that doesn't exist.

Popular relational databases (RDBMS):
- **PostgreSQL**: feature-rich, strict, strong JSON support. A great default.
- **MySQL / MariaDB**: very widely deployed, common in web hosting.
- **SQLite**: a single file with no server. Used in mobile apps, tests and
  small tools.
- **SQL Server**, **Oracle**: common in enterprise environments.

---

### Terminology

| Term | Also called | Meaning |
|---|---|---|
| Table | Relation | A set of rows with the same columns |
| Row | Tuple, record | One entry (one user, one order) |
| Column | Attribute, field | One property, with a fixed type (`TEXT`, `INT`, `TIMESTAMPTZ`) |
| Schema | — | The structure of the tables: columns, types, constraints |
| Primary key (PK) | — | The column(s) that uniquely identify a row |
| Foreign key (FK) | — | A column that points to a PK in another table |
| Composite key | — | A key made of more than one column |
| Index | — | A lookup structure that makes searches fast |

---

### An Example Schema (PostgreSQL)

```sql
CREATE TABLE users (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email       TEXT NOT NULL UNIQUE,
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE products (
  id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name         TEXT NOT NULL,
  price_minor  BIGINT NOT NULL CHECK (price_minor >= 0)   -- paise/cents, never FLOAT
);

CREATE TABLE orders (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id     BIGINT NOT NULL REFERENCES users(id),
  status      TEXT NOT NULL DEFAULT 'pending'
              CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled')),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX orders_user_idx ON orders(user_id);

-- Join table for the many-to-many between orders and products
CREATE TABLE order_items (
  order_id          BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id        BIGINT NOT NULL REFERENCES products(id),
  qty               INT    NOT NULL CHECK (qty > 0),
  unit_price_minor  BIGINT NOT NULL,          -- snapshot of the price at purchase time
  PRIMARY KEY (order_id, product_id)          -- composite key
);
```

---

### Constraints: the Database Protects Your Data

| Constraint | What it guarantees | Example |
|---|---|---|
| `PRIMARY KEY` | Every row is unique and identifiable | `id` |
| `NOT NULL` | The value is required | `email TEXT NOT NULL` |
| `UNIQUE` | No duplicates | One account per email |
| `FOREIGN KEY` | No orphan rows | An order's `user_id` must exist in `users` |
| `CHECK` | The value follows a rule | `qty > 0`, a limited list of statuses |
| `DEFAULT` | A value is filled in when none is given | `created_at DEFAULT now()` |

The database enforces these for **every** client: your API, a script, an
admin tool, or a teammate running SQL by hand. App-level validation can be
bypassed. Constraints can't.

**`ON DELETE` choices for foreign keys**

| Option | When the parent row is deleted… | Use for |
|---|---|---|
| `RESTRICT` / `NO ACTION` (default) | The delete fails if children exist | Most relationships (you can't delete a user who has orders) |
| `CASCADE` | The children are deleted too | Children that mean nothing without the parent (order items) |
| `SET NULL` | The child's FK becomes `NULL` | Optional links (an "assigned to" user who left) |

---

### Relationships

| Type | Example | How it's built |
|---|---|---|
| **One-to-one** | user ↔ profile | FK with `UNIQUE` on the child (`profiles.user_id UNIQUE`) |
| **One-to-many** | user → orders | FK on the "many" side (`orders.user_id`) |
| **Many-to-many** | orders ↔ products | A **join table** with two FKs (`order_items`), which can hold its own columns (`qty`, `unit_price_minor`) |

More in [Relationships](../03-orm-concepts/03_relationships.md).

---

### Normalization: Store Each Fact Once

Normalization means organising tables so the same fact isn't repeated.
Repeated facts get out of sync: you update the customer's email in one row
and forget the other twenty.

**Before (not normalized)**

| order_id | customer_email | customer_city | products |
|---|---|---|---|
| 10 | ashu@example.com | Patna | Keyboard, Mouse |
| 11 | ashu@example.com | Patna | Monitor |

Problems: `products` holds a list in one cell, and the customer's details
are repeated in every order.

| Normal form | Rule | Fix applied |
|---|---|---|
| **1NF** | Each cell holds one value; no lists or repeating groups | Split `products` into `order_items` rows |
| **2NF** | 1NF, and every non-key column depends on the **whole** key | In `order_items (order_id, product_id)`, `product_name` depends only on `product_id`, so move it to `products` |
| **3NF** | 2NF, and non-key columns depend **only on the key**, not on other non-key columns | `customer_city` depends on the customer, not the order, so move it to `users` |

**After:** `users`, `products`, `orders`, `order_items`, which is the schema
above.

> Aim for 3NF by default. Then denormalize **on purpose**, with a reason. For
> example, `unit_price_minor` on `order_items` is a snapshot of the price
> that was charged. See [How a Senior Developer Designs a Database](../12-db-design/01_senior-dev-thought-process.md).

---

### SQL: The Queries You Use Every Day

```sql
-- Filter + sort + paginate
SELECT id, name, email
FROM users
WHERE created_at >= now() - interval '30 days'
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

-- INNER JOIN: only orders that have a matching user
SELECT o.id, o.status, u.name
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'paid';

-- LEFT JOIN: every user, even users with zero orders
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;

-- Aggregation: revenue per product, only products that sold over ₹10,000
SELECT p.name, SUM(oi.qty * oi.unit_price_minor) AS revenue_minor
FROM order_items oi
JOIN products p ON p.id = oi.product_id
JOIN orders o   ON o.id = oi.order_id
WHERE o.status IN ('paid', 'shipped')
GROUP BY p.id, p.name
HAVING SUM(oi.qty * oi.unit_price_minor) > 1000000   -- in paise
ORDER BY revenue_minor DESC;
```

| JOIN type | Returns |
|---|---|
| `INNER JOIN` | Only rows that match on both sides |
| `LEFT JOIN` | Every row from the left table, plus matches (or `NULL`) from the right |
| `RIGHT JOIN` | The mirror of `LEFT` (rarely used; swap the tables instead) |
| `FULL OUTER JOIN` | Every row from both sides, matched where possible |

`WHERE` filters rows **before** grouping. `HAVING` filters groups **after**
grouping. More in [Joins](../03-orm-concepts/04_joins.md).

---

### ACID Transactions

A **transaction** groups several statements so they succeed or fail
together.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1 AND balance >= 500;
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;   -- or ROLLBACK; to undo everything
```

| Letter | Property | Meaning |
|---|---|---|
| **A** | Atomicity | All statements apply, or none do |
| **C** | Consistency | Constraints hold before and after the transaction |
| **I** | Isolation | Concurrent transactions don't see each other's half-done work |
| **D** | Durability | Once committed, the data survives a crash |

**Isolation levels** trade safety for concurrency:

| Level | Dirty read | Non-repeatable read | Phantom read | Default in |
|---|---|---|---|---|
| Read Uncommitted | possible* | possible | possible | — |
| **Read Committed** | ✅ prevented | possible | possible | **PostgreSQL**, SQL Server |
| **Repeatable Read** | ✅ | ✅ prevented | possible* | **MySQL (InnoDB)** |
| Serializable | ✅ | ✅ | ✅ prevented | — |

\* PostgreSQL never allows dirty reads, even at Read Uncommitted. Its
Repeatable Read also prevents phantom reads.

- **Dirty read:** you see another transaction's uncommitted change.
- **Non-repeatable read:** you read the same row twice and get different
  values.
- **Phantom read:** you run the same query twice and new rows appear.

More in [Transactions](../09-advanced/05_transactions.md) and
[Concurrency](../09-advanced/06_concurrency.md).

---

### Indexes

Without an index, the database reads every row to find a match (a
**sequential scan**). An index, usually a **B-tree**, finds rows in
logarithmic time.

```sql
CREATE INDEX orders_user_created_idx ON orders(user_id, created_at DESC);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC LIMIT 10;
-- Look for "Index Scan using orders_user_created_idx", not "Seq Scan"
```

- **Leftmost-prefix rule:** an index on `(user_id, created_at)` helps
  queries on `user_id`, or on `user_id + created_at`, but not on
  `created_at` alone.
- PostgreSQL automatically indexes `PRIMARY KEY` and `UNIQUE` columns, but
  **not** foreign key columns. Add those yourself when you join or filter
  on them.
- Every index slows down writes and uses disk and memory. Index your real
  query patterns, not every column.

More in [Indexing](../09-advanced/03_indexing.md).

---

### Node.js: Using a Relational Database With `pg`

```js
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 10 });

// Always parameterize ($1, $2 …). Never build SQL with string concatenation.
async function getUserOrders(userId) {
  const { rows } = await pool.query(
    `SELECT o.id, o.status, o.created_at
       FROM orders o
      WHERE o.user_id = $1
      ORDER BY o.created_at DESC`,
    [userId]
  );
  return rows;
}

// A transaction needs ONE client for all its statements
async function placeOrder(userId, items) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const { rows: [order] } = await client.query(
      'INSERT INTO orders (user_id) VALUES ($1) RETURNING id',
      [userId]
    );
    for (const { productId, qty } of items) {
      // Copy the current price into the order item (price snapshot)
      const { rowCount } = await client.query(
        `INSERT INTO order_items (order_id, product_id, qty, unit_price_minor)
         SELECT $1::bigint, id, $3::int, price_minor FROM products WHERE id = $2`,
        [order.id, productId, qty]
      );
      if (rowCount === 0) throw new Error(`Product ${productId} not found`);
    }
    await client.query('COMMIT');
    return order.id;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();   // always return the client to the pool
  }
}
```

> Don't run `BEGIN` with `pool.query()`. Each `pool.query()` call can use a
> different connection, so your statements would run outside the
> transaction. Check out one client with `pool.connect()` and use it for
> every statement.

With an ORM, the same transaction is `prisma.$transaction(...)` or
`sequelize.transaction(...)`. See [Prisma Transactions](../04-orm-examples/prisma/06_transactions.md).
Connection setup and pooling are covered in [Database Drivers](database-drivers.md).

---

### Scaling a Relational Database

| Step | What it is | Solves |
|---|---|---|
| 1. Indexes + query tuning | Fix slow queries first | Most "the DB is slow" problems |
| 2. Connection pooling | PgBouncer / RDS Proxy | Too many connections |
| 3. Vertical scaling | Bigger machine (CPU, RAM, faster disk) | General load; simple and underrated |
| 4. Read replicas | Copies that serve reads | Read-heavy traffic (may be slightly behind) |
| 5. Partitioning | Split one big table by date or key, on one server | Huge tables, cheap deletion of old data |
| 6. Sharding | Split data across servers | Write volume beyond one machine; hard, so do it last |

Go through these in order. Most applications never need step 6.

---

### When to Use a Relational Database

**Good fit (most applications)**
- Data with clear relationships: users, orders, products, payments.
- Rules that must hold across rows and tables (money, inventory,
  bookings).
- Multi-step operations that need ACID transactions.
- Reporting and ad-hoc queries (SQL is excellent at these).

**Consider something else when**
- Records are self-contained and differ a lot in shape (see
  [Document Database](document-database.md)).
- You need massive write throughput with simple key lookups (key-value or
  wide-column stores).
- The data is a graph that you traverse many hops deep (graph databases).

See [SQL vs NoSQL](sql-vs-nosql.md) for the full comparison.

---

### Common Mistakes

| Mistake | Fix |
|---|---|
| Building SQL with string concatenation, which allows SQL injection | Parameterized queries (`$1`, `?`) or an ORM |
| `FLOAT` for money | Integer minor units (`BIGINT`) or `NUMERIC(12,2)` |
| No foreign keys ("the app handles it") | Declare FKs; the database enforces them for every client |
| Missing index on FK columns | Index the FK columns you join or filter on |
| `SELECT *` everywhere | Select only the columns you need |
| N+1 queries (one query per row in a loop) | A `JOIN`, or eager loading in your ORM. See [N+1 Problem](../09-advanced/01_n-plus-one-problem.md) |
| `BEGIN` through `pool.query()` | Check out one client with `pool.connect()` for the whole transaction |
| Comma-separated lists in one column | A separate table (1NF) |
| `TIMESTAMP` without a time zone | `TIMESTAMPTZ`, stored in UTC |

---

### Interview-Ready Summary

- A relational database stores data in **tables** linked by **primary and
  foreign keys**, and you query it with **SQL**.
- **Constraints** (`PK`, `FK`, `UNIQUE`, `NOT NULL`, `CHECK`) make the
  database enforce data rules for every client.
- **Normalize to 3NF** so each fact is stored once. Denormalize only on
  purpose, for example a price snapshot.
- **ACID transactions** make multi-step changes all-or-nothing. Know the
  isolation levels and their defaults: PostgreSQL uses Read Committed,
  MySQL uses Repeatable Read.
- **Indexes** (B-tree) turn full scans into fast lookups. Know the
  leftmost-prefix rule, and remember that PostgreSQL doesn't index FKs
  automatically.
- Scale in order: tune queries → pool connections → bigger machine →
  read replicas → partitioning → sharding last.
