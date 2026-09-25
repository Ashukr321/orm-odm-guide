## What Is a Database

### The Simple Definition

A **database** is an organized collection of data. It is stored so the data
can be **saved, found, updated and protected** reliably, even when many
users use it at the same time.

A **DBMS** (Database Management System) is the software that manages it:
PostgreSQL, MySQL, MongoDB, Redis and SQLite are all DBMSs. In everyday
speech, "database" often means both.

```
Your App  ──(query)──►  DBMS (PostgreSQL, MongoDB…)  ──►  Data on disk
          ◄─(result)──
```

---

### Why Not Just Use Files or a Spreadsheet?

You *can* store users in a JSON file or an Excel sheet. It works for a
while. Then the problems start:

| Problem | Files / spreadsheet | Database |
|---|---|---|
| Two users save at the same time | One overwrites the other | **Concurrency control**: both writes are handled safely |
| The server crashes mid-write | A half-written, corrupted file | **Durability**: committed data survives crashes (write-ahead log) |
| Find one user among 10M | Read the whole file | **Indexes** find it in milliseconds |
| Bad data (missing email, text in an age field) | Saved without complaint | **Constraints** reject it |
| "Move ₹500 from A to B" | The first step saved, the second lost | **Transactions**: all or nothing |
| Who can see salaries? | Whoever has the file | **Access control**: users, roles, permissions |
| Complex questions ("top 10 cities by revenue") | Custom code for each one | **A query language** (SQL, or MongoDB queries) |

A database solves these problems once, correctly, so every application
doesn't have to solve them again.

---

### Core Vocabulary

| Term | Meaning |
|---|---|
| **Data** | Raw facts: `"Ashutosh"`, `29`, `2026-09-25` |
| **Database** | An organized collection of related data |
| **DBMS** | The software that stores, queries and protects the data |
| **Schema** | The structure: what tables or collections exist, their fields and types |
| **Record** | One entry: a row (SQL) or a document (MongoDB) |
| **Query** | A request to read or change data |
| **CRUD** | The four basic operations: **C**reate, **R**ead, **U**pdate, **D**elete |
| **Index** | A lookup structure that makes searches fast |
| **Transaction** | A group of changes that succeed or fail together |
| **Constraint** | A rule the database enforces (unique email, required field) |

---

### CRUD: The Four Operations Every App Does

| Operation | SQL | MongoDB |
|---|---|---|
| **Create** | `INSERT INTO users (name) VALUES ('Ashu');` | `db.users.insertOne({ name: 'Ashu' })` |
| **Read** | `SELECT * FROM users WHERE id = 1;` | `db.users.findOne({ _id: id })` |
| **Update** | `UPDATE users SET name = 'Ashutosh' WHERE id = 1;` | `db.users.updateOne({ _id: id }, { $set: { name: 'Ashutosh' } })` |
| **Delete** | `DELETE FROM users WHERE id = 1;` | `db.users.deleteOne({ _id: id })` |

Almost every feature you build (sign up, add to cart, edit profile,
cancel order) is one or more of these four.

---

### Types of Databases

| Type | Stores data as | Examples | Typical use |
|---|---|---|---|
| **Relational (SQL)** | Tables of rows and columns, linked by keys | PostgreSQL, MySQL, SQLite, SQL Server | Most business apps: users, orders, payments |
| **Document** | JSON-like documents | MongoDB, Firestore | Catalogs, profiles, content |
| **Key-value** | `key → value` | Redis, DynamoDB | Cache, sessions, rate limits |
| **Wide-column** | Rows partitioned by key, with many columns | Cassandra, ScyllaDB | Huge write volume, time-series |
| **Graph** | Nodes and edges | Neo4j | Social networks, recommendations |
| **Search** | Inverted indexes over text | Elasticsearch, OpenSearch | Full-text search, log search |
| **Time-series** | Timestamped measurements | TimescaleDB, InfluxDB | Metrics, IoT sensors |
| **Vector** | Numeric embeddings | pgvector, Pinecone | Semantic search, AI / RAG |

Deep dives: [Relational Database](relational-database.md) ·
[Document Database](document-database.md) · [SQL vs NoSQL](sql-vs-nosql.md)

---

### How a Database Works Inside (Simplified)

```
          SQL / query
               │
     ┌─────────▼──────────┐
     │  Parser            │  checks syntax: "is this valid SQL?"
     ├────────────────────┤
     │  Query planner     │  picks the fastest plan: index or full scan? which join order?
     ├────────────────────┤
     │  Executor          │  runs the plan
     ├────────────────────┤
     │  Buffer cache      │  keeps hot data pages in RAM (RAM is far faster than disk)
     ├────────────────────┤
     │  Write-ahead log   │  records every change first, so it can be replayed after a crash
     ├────────────────────┤
     │  Storage engine    │  data files on disk, in fixed-size pages
     └────────────────────┘
```

- **Pages:** data is stored and read in fixed-size blocks (8 KB in
  PostgreSQL, 16 KB in MySQL InnoDB), not row by row.
- **Buffer cache:** frequently used pages stay in memory. That is why a
  database with enough RAM is dramatically faster.
- **Write-ahead log (WAL):** a change is written to an append-only log
  *before* the data files. After a crash, the database replays the log, so
  committed data is never lost.
- **Query planner:** you say *what* you want; the database decides *how* to
  get it. `EXPLAIN` shows you its plan.

---

### Where the Database Sits in a Web App

```
Browser ──HTTP──► API server (Node.js / Next.js) ──driver──► Database server
                   │                                  TCP, e.g. port 5432 (PostgreSQL),
                   │                                  3306 (MySQL), 27017 (MongoDB), 6379 (Redis)
                   └── ORM / ODM (Prisma, Mongoose) sits on top of the driver
```

1. The user clicks "Place order".
2. The API validates the request.
3. The API sends queries to the database through a **driver**, usually via
   an **ORM/ODM**, over a pooled connection.
4. The database runs them inside a **transaction** and returns the result.
5. The API sends the response back to the browser.

The database runs as its **own server process** (the client-server model).
SQLite is the exception: it is a library that reads a local file directly.

How your code connects: [Database Drivers](database-drivers.md).

---

### OLTP vs OLAP: Two Very Different Workloads

| | OLTP (transactional) | OLAP (analytical) |
|---|---|---|
| Purpose | Run the app | Analyze the business |
| Typical query | "Get order #123", "insert a payment" | "Revenue by city per month for 3 years" |
| Rows per query | A few | Millions |
| Users | Thousands at once | A few analysts or dashboards |
| Examples | PostgreSQL, MySQL, MongoDB | BigQuery, Snowflake, ClickHouse, Redshift |

Running heavy analytics on your production OLTP database slows the app down.
At scale, data is copied to an OLAP store for reporting.

---

### Important Database Properties

- **ACID** (transactions): **A**tomic, **C**onsistent, **I**solated,
  **D**urable. Details in [Relational Database](relational-database.md#acid-transactions).
- **Replication:** copies of the data on other servers, for high
  availability and to spread out reads.
- **Backups:** periodic snapshots plus the log, so you can restore to a
  point in time. *A backup you have never tested restoring is not a backup.*
- **Security:** authentication, roles and permissions, encryption in
  transit (TLS) and at rest, and parameterized queries to prevent SQL
  injection.

---

### Managed vs Self-Hosted

| | Self-hosted | Managed (cloud) |
|---|---|---|
| Examples | PostgreSQL on your own VM or Docker | AWS RDS, Neon, Supabase, MongoDB Atlas, PlanetScale |
| You handle | Install, upgrades, backups, failover, monitoring | Mostly your schema and queries |
| Cost | Cheaper machines, more of your time | Higher bill, less of your time |
| Good for | Learning, tight budgets, special needs | Most production apps |

Free tiers for learning: [Free Platforms & Databases](../resrouces/free-platforms-and-databases.md).

---

### Common Beginner Mistakes

| Mistake | Fix |
|---|---|
| Storing app data in JSON files | Use a real database (even SQLite) from day one |
| Building queries with string concatenation (SQL injection) | Parameterized queries, or an ORM |
| Saving passwords as plain text | Store only a hash (bcrypt / argon2), never the password |
| No backups, or backups never tested | Automated backups plus a regular test restore |
| Opening a new connection per request | A connection pool. See [Database Drivers](database-drivers.md) |
| Picking a database by hype | Choose by data shape and access pattern. See [SQL vs NoSQL](sql-vs-nosql.md) |

---

### Where to Go Next

1. [Relational Database](relational-database.md): tables, keys, SQL, ACID
2. [Document Database](document-database.md): documents, embed vs reference
3. [SQL vs NoSQL](sql-vs-nosql.md): how to choose
4. [Database Drivers](database-drivers.md): how Node.js connects
5. [What Is an ORM](../02-orm/01_what-is-orm.md) and [What Is an ODM](../05-odm/01_what-is-odm.md)

---

### Interview-Ready Summary

- A **database** is an organized collection of data. The **DBMS** is the
  software that stores, queries and protects it.
- Compared with files, it adds **concurrency control, durability,
  indexes, constraints, transactions, access control and a query
  language**.
- Every app is built from **CRUD**: create, read, update, delete.
- Main types: **relational, document, key-value, wide-column, graph**,
  plus search, time-series and vector stores.
- Inside: parser → **query planner** → executor, with a **buffer cache**
  in RAM and a **write-ahead log** for crash safety.
- **OLTP** runs the app with many small queries. **OLAP** analyzes the
  business with a few huge queries. Keep them separate at scale.
