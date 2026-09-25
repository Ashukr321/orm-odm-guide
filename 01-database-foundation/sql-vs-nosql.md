## SQL vs NoSQL

### The Short Answer

- **SQL (relational)** databases store data in **tables with a fixed schema**,
  link them with keys, and query them with SQL. They are built for
  **correctness, relationships and flexible queries**.
- **NoSQL** ("not only SQL") covers several **non-relational** families:
  document, key-value, wide-column and graph. Each one is optimized for a
  **specific access pattern**, usually at large scale.

Neither is "better". The right question is: **what shape is the data, and
how will it be read and written?**

Deep dives: [Relational Database](relational-database.md) ·
[Document Database](document-database.md)

---

### NoSQL Is Four Different Things

"NoSQL" is not one technology. Compare SQL against the **specific family**
you're considering.

| Family | Data model | Examples | Great for |
|---|---|---|---|
| **Document** | JSON-like documents in collections | MongoDB, Firestore, Couchbase | Catalogs, profiles, CMS content, records with different shapes |
| **Key-value** | `key → value`, looked up by key only | Redis, DynamoDB, Memcached | Caching, sessions, rate limits, leaderboards |
| **Wide-column** | Rows partitioned by key, each with many columns | Cassandra, ScyllaDB, HBase | Huge write volume, time-series, IoT, messaging history |
| **Graph** | Nodes + edges (relationships are first-class) | Neo4j, Amazon Neptune | Social graphs, recommendations, fraud rings, many-hop traversals |

---

### Core Differences

| | SQL (relational) | NoSQL (typical) |
|---|---|---|
| **Data model** | Tables, rows, columns | Documents, key-value pairs, wide rows, graphs |
| **Schema** | **Schema-on-write**: defined up front and enforced | **Schema-on-read**: flexible, and the app decides what's valid |
| **Relationships** | Foreign keys + `JOIN` | Embedding / denormalization; joins are limited or absent |
| **Query language** | Standard SQL | Database-specific API (MongoDB queries, CQL, Cypher, Redis commands) |
| **Transactions** | ACID across many rows and tables, by design | Usually atomic per document / key; multi-item transactions are limited or cost more |
| **Consistency** | Strong by default | Often tunable: strong, or eventual for more speed and availability |
| **Scaling** | Vertical first, then read replicas; sharding is hard work | Horizontal (sharding / partitioning) is built in |
| **Ad-hoc queries** | Excellent: any column, any join, aggregations | Limited: you design around queries you know ahead of time |
| **Best when** | Related data, strict rules, reporting | Known access patterns, huge scale, flexible shapes |

---

### Same Data, Modeled Both Ways

**Requirement:** a user with addresses and orders.

**SQL: normalized into tables, joined when reading**

```sql
CREATE TABLE users     (id BIGINT PRIMARY KEY, name TEXT NOT NULL, email TEXT UNIQUE NOT NULL);
CREATE TABLE addresses (id BIGINT PRIMARY KEY, user_id BIGINT NOT NULL REFERENCES users(id), city TEXT NOT NULL);
CREATE TABLE orders    (id BIGINT PRIMARY KEY, user_id BIGINT NOT NULL REFERENCES users(id),
                        total_minor BIGINT NOT NULL, status TEXT NOT NULL);

-- Read a user with their addresses: a JOIN
SELECT u.name, a.city
FROM users u
JOIN addresses a ON a.user_id = u.id
WHERE u.id = 1;
```

**Document (MongoDB): shaped around the read**

```js
// users collection: addresses embedded (bounded, always read with the user)
{
  _id: ObjectId("..."),
  name: "Ashutosh",
  email: "ashu@example.com",
  addresses: [{ city: "Patna" }, { city: "Bengaluru" }]
}

// orders collection: referenced (grows without bound)
{ _id: ObjectId("..."), userId: ObjectId("..."), totalMinor: 249900, status: "paid" }

// Read a user with their addresses: one query, no join
db.users.findOne({ _id: userId });
```

**Key-value (Redis): only lookups by key**

```
SET session:9f2c1a  '{"userId":1,"role":"admin"}'  EX 3600   # expires in 1 hour
GET session:9f2c1a
```

Notice the trade-off. SQL can answer *any* question later ("which cities
have the most users who ordered twice?"). The document model makes the
**planned** read fast. Redis can **only** fetch by key, but does it in well
under a millisecond.

---

### Schema: On-Write vs On-Read

| | Schema-on-write (SQL) | Schema-on-read (NoSQL) |
|---|---|---|
| When is structure checked? | When you insert or update | When your code reads the data |
| Adding a field | A migration (`ALTER TABLE`) | Just start writing it |
| Bad data (typo, wrong type) | Rejected by the database | Saved, and found later as a bug |
| Old records | All rows have the same shape | You handle several shapes in code |

Flexible schema speeds up early development, but the schema doesn't
disappear. It moves into your application code. Most teams add it back with
MongoDB's `$jsonSchema` validation or an ODM like Mongoose. See
[ODM](../05-odm/01_what-is-odm.md).

---

### Consistency: ACID vs BASE, and CAP

**ACID** (SQL default): **A**tomic, **C**onsistent, **I**solated,
**D**urable. A committed transaction is correct and permanent.

**BASE** (common in distributed NoSQL): **B**asically **A**vailable,
**S**oft state, **E**ventually consistent. Replicas may disagree for a
moment, then converge.

**CAP theorem, stated correctly:** when a **network partition** happens in
a distributed system, you must choose:
- **Consistency**: refuse or delay some requests so nobody reads stale
  data, or
- **Availability**: answer every request, even if some answers are stale.

It is **not** "pick any 2 of 3". Partitions will happen, so the real choice
is C or A *during* a partition. **PACELC** adds the everyday trade-off: even
with no partition, you trade **latency** against **consistency**.

| System | Typical behaviour |
|---|---|
| PostgreSQL / MySQL (single primary) | Strong consistency; the primary is the source of truth |
| MongoDB (default `w: "majority"`) | Leans consistent; reads from secondaries can be stale |
| Cassandra / DynamoDB | Tunable per query: eventual by default, strong when you ask for it (DynamoDB strongly consistent reads, Cassandra `QUORUM`) |

---

### Scaling: The Real Picture

- **SQL scales further than people think.** A single well-tuned PostgreSQL
  instance with indexes, pooling and read replicas handles most
  applications, even with millions of users.
- **NoSQL scales out easily** because it avoids cross-node joins and
  transactions. That is the price you pay for easy sharding.
- **Distributed SQL** (CockroachDB, YugabyteDB, Google Spanner) offers SQL
  and ACID with horizontal scaling, at the cost of higher latency per write
  and more operational complexity.

Rule of thumb: **don't choose NoSQL only for scale you don't have yet.**
Choose it because the **data model or access pattern** fits.

---

### Common Myths

| Myth | Reality |
|---|---|
| "NoSQL is faster." | It is faster for the access pattern it was designed for. It is slower, or can't do it at all, for others (ad-hoc joins, reporting). |
| "NoSQL has no schema." | The schema lives in your code instead of the database, and you still have to manage it. |
| "SQL can't scale." | Replicas, partitioning and distributed SQL go very far. Most apps never outgrow one primary. |
| "MongoDB has no transactions." | It has had multi-document ACID transactions since 4.0. |
| "SQL can't store JSON." | PostgreSQL `JSONB` stores, indexes and queries JSON very well. |

**Postgres JSONB: the best of both for flexible attributes**

```sql
CREATE TABLE products (
  id          BIGINT PRIMARY KEY,
  name        TEXT NOT NULL,
  price_minor BIGINT NOT NULL,
  attributes  JSONB NOT NULL DEFAULT '{}'   -- differs per product type
);
CREATE INDEX products_attributes_idx ON products USING GIN (attributes);

SELECT name FROM products WHERE attributes @> '{"color": "black", "wireless": true}';
```

Keep strict, relational fields as columns, and put the flexible, rarely
joined attributes in `JSONB`.

---

### How to Decide

Ask these questions in order:

1. **Is the data relational, with rules that span entities** (money,
   inventory, bookings)? → **SQL**
2. **Do you need ad-hoc queries or reporting** that you can't predict yet?
   → **SQL**
3. **Is each record self-contained, read as a whole, with shapes that
   differ?** → **Document**
4. **Is it only lookup by key, and needs to be very fast** (cache,
   sessions)? → **Key-value**
5. **Is it a huge volume of writes with a known query per partition**
   (events, time-series)? → **Wide-column**
6. **Are the relationships themselves the question** ("friends of friends
   who bought X")? → **Graph**
7. **Still unsure?** → **PostgreSQL.** It is the safest default, and
   `JSONB` covers most flexible-schema needs.

---

### Polyglot Persistence: Real Systems Use Both

Most production systems use **several** databases, each for what it does
best:

```
                ┌──────────────── PostgreSQL ── users, orders, payments (ACID)
                │
  Your API ─────┼──────────────── Redis ─────── cache, sessions, rate limits
                │
                ├──────────────── MongoDB ───── product catalog, CMS content
                │
                └──────────────── OpenSearch ── full-text product search
```

The cost: every extra database is one more thing to run, monitor, back up
and keep in sync. Add one only when it solves a real problem.

---

### SQL vs NoSQL in Node.js: ORM vs ODM

| Store | Tooling | Guide |
|---|---|---|
| SQL | **ORM**: Prisma, Sequelize, TypeORM, Drizzle | [ORM](../02-orm/01_what-is-orm.md) |
| MongoDB | **ODM**: Mongoose (Prisma also supports MongoDB) | [ODM](../05-odm/01_what-is-odm.md) |
| Any | Raw **driver**: `pg`, `mysql2`, `mongodb` | [Database Drivers](database-drivers.md) |

---

### Interview-Ready Summary

- SQL = tables, fixed schema, joins, **ACID**, excellent ad-hoc queries.
  NoSQL = **four families** (document, key-value, wide-column, graph), each
  optimized for a specific access pattern.
- **Schema-on-write vs schema-on-read**: with NoSQL, the schema moves into
  your code; it doesn't disappear.
- **CAP**: during a network partition, choose consistency or availability.
  It is not "pick 2 of 3". PACELC adds latency vs consistency in normal
  operation.
- SQL scales further than the myths say; NoSQL makes horizontal scaling
  easy by giving up joins and cross-item transactions.
- Choose based on **data shape and access patterns**, not hype. Default to
  PostgreSQL (with `JSONB` for flexible fields) when unsure.
- Real systems are often **polyglot**: SQL for core data, Redis for cache,
  a document or search store where it fits.
