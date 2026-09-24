## Database Drivers

### What Is a Database Driver

A **database driver** is the low-level library that speaks a database's wire
protocol directly — it opens a TCP connection, authenticates, sends the
query bytes, and parses the raw response back into language-native values.

```
Your Code → ORM/ODM (optional) → Database Driver → Network → Database Server
```

Everything above the driver (Prisma, Sequelize, Mongoose, TypeORM, etc.) is
built **on top of** a driver. No ORM/ODM talks to the database directly —
they all delegate the actual connection and query execution to one.

- **Relational (SQL) databases** — each database has its own driver:
  `pg` (PostgreSQL), `mysql2` (MySQL), `sqlite3` / `better-sqlite3` (SQLite),
  `tedious` (MSSQL, via `mssql` package).
- **Document / NoSQL databases** — e.g. the official `mongodb` driver for
  MongoDB.

An ODM like Mongoose is literally a schema/validation layer wrapped around
the `mongodb` driver. An ORM like Prisma ships its own Rust query engine but
still ultimately opens connections using the same protocol-level logic a
driver would.

---

### Why Drivers Matter (Even If You Use an ORM)

1. **Performance-critical raw queries** — ORMs generate SQL that isn't
   always optimal; dropping to the driver (`pool.query(...)`) skips
   ORM overhead for hot paths.
2. **Debugging** — knowing what the driver does under the hood explains ORM
   behavior: connection pooling, query timeouts, prepared statements,
   transaction isolation.
3. **Features the ORM doesn't expose** — LISTEN/NOTIFY in Postgres, cursors,
   streaming large result sets, database-specific extensions (JSONB
   operators, full-text search).
4. **Interview signal** — "I use Prisma" is a tooling answer; "I know
   Prisma sits on connection pooling the same way `pg.Pool` does" is a
   systems answer. Interviewers probe this to check you understand what the
   ORM is hiding from you.

---

### Node.js: Using Drivers Directly

#### PostgreSQL — `pg`

```bash
npm install pg
```

```js
// db.js
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL, // postgres://user:pass@host:5432/db
  max: 10,                // max clients in the pool
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

module.exports = pool;
```

```js
// usage
const pool = require('./db');

async function getUserById(id) {
  const { rows } = await pool.query('SELECT * FROM users WHERE id = $1', [id]);
  return rows[0];
}
```

Key points:
- `Pool`, not `Client`, in any real app — a single `Client` handles one
  connection; a `Pool` reuses a set of connections across requests.
- Parameterized queries (`$1`, `$2`) — never string-concatenate values into
  SQL (SQL injection).
- `pool.query()` acquires a client, runs the query, releases it
  automatically. For multi-statement transactions, check out a client
  manually with `pool.connect()` and call `client.release()` yourself.

#### MySQL — `mysql2`

```bash
npm install mysql2
```

```js
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  password: 'secret',
  database: 'app_db',
  waitForConnections: true,
  connectionLimit: 10,
});

async function getUserById(id) {
  const [rows] = await pool.query('SELECT * FROM users WHERE id = ?', [id]);
  return rows[0];
}
```

`mysql2` is the modern, faster, Promise-native successor to the older
`mysql` package — always prefer `mysql2` in new code.

#### MongoDB — `mongodb` (native driver)

```bash
npm install mongodb
```

```js
const { MongoClient } = require('mongodb');

const client = new MongoClient(process.env.MONGO_URI);
let db;

async function connectDB() {
  if (db) return db;
  await client.connect();
  db = client.db('app_db');
  return db;
}

async function getUserById(id) {
  const database = await connectDB();
  return database.collection('users').findOne({ _id: id });
}
```

`MongoClient` already manages an internal connection pool — you don't build
your own pool on top of it, you just reuse one `MongoClient` instance for
the app's lifetime (this becomes important in serverless, see below).

---

### Connection Pooling — The Concept That Matters Most

Opening a TCP/TLS connection to a database is expensive (handshake, auth).
A **connection pool** keeps a set of already-open connections ready to be
reused, instead of opening/closing one per query.

- Too small a pool → requests queue up waiting for a free connection.
- Too large a pool → the database server itself has a hard connection
  limit (Postgres default is 100) and can be overwhelmed — a huge pool per
  app instance, multiplied across many instances, is a common production
  outage.
- Rule of thumb: pool size relates to available DB connections ÷ number of
  app instances, not "as high as possible."

This single concept — driver-level connection pooling — is *the* reason
Next.js on serverless needs special handling (next section).

---

### Next.js: Driver Usage Gotchas

Next.js complicates driver usage for one structural reason: **API
routes / Server Actions / Route Handlers don't run like a long-lived
Node.js server.** They run per-request, and in dev mode with hot reload
they get re-evaluated on every file save.

#### Problem 1 — Hot reload creates a new pool on every save (dev only)

In dev, Next.js clears the module cache on file changes. If `db.js`
creates `new Pool()` at the top level, every hot reload creates *another*
pool, and old pools are never closed — you exhaust your local Postgres
connection limit within a few minutes of editing code.

**Fix — cache the pool on the global object** so it survives module
re-evaluation:

```js
// lib/db.js
import { Pool } from 'pg';

const globalForPg = globalThis;

export const pool =
  globalForPg.pgPool ??
  new Pool({ connectionString: process.env.DATABASE_URL, max: 10 });

if (process.env.NODE_ENV !== 'production') {
  globalForPg.pgPool = pool;
}
```

This is the exact same pattern the Prisma docs recommend for
`PrismaClient` in Next.js — it's not Prisma-specific, it's a Next.js dev
mode issue that applies to any driver/client you instantiate at module
scope.

#### Problem 2 — Serverless deployments (Vercel) spin up many isolated instances

Each serverless function invocation can be a **fresh execution
environment**. Under load, Vercel may run dozens of instances of your
route handler concurrently, and *each one* opens its own pool. A pool of
10 connections × 50 concurrent instances = 500 connections, which blows
past most managed Postgres plans' connection limits.

Mitigations, roughly in order of how they're used in practice:

1. **Use a pooling proxy in front of the database** — PgBouncer, or a
   managed equivalent (Neon, Supabase, and RDS Proxy all offer this).
   The app connects to the proxy with a small pool; the proxy multiplexes
   many app connections onto few real database connections.
2. **Keep `max` low per instance** (e.g. `max: 1`–`5`) since you're one of
   many concurrent instances, not the only client.
3. **Prefer HTTP-based/edge-compatible drivers where available** — e.g.
   Neon's `@neondatabase/serverless` or PlanetScale's driver, which talk
   to the database over HTTP/WebSocket instead of a raw TCP connection.
   This matters even more for the Edge Runtime (see below), which can't
   open raw TCP sockets at all.
4. **Reuse the client/connection across invocations when the runtime
   allows it** (Node.js runtime, not Edge) via the same `globalThis`
   caching pattern as Problem 1 — a warm serverless instance can reuse its
   existing pool for the next request instead of reconnecting.

#### Problem 3 — Edge Runtime can't use most Node.js drivers

Route Handlers/Middleware running on the **Edge Runtime**
(`export const runtime = 'edge'`) execute in a V8 isolate, not Node.js —
there's no raw TCP socket API, so `pg`, `mysql2`, and the standard
`mongodb` driver simply don't work there.

- Use the **Node.js runtime** (the default) for anything using a
  traditional driver.
- If you specifically need Edge (lower latency, geographically distributed
  execution), you need an HTTP-based driver built for it —
  `@neondatabase/serverless`, `@vercel/postgres`, `@planetscale/database`,
  or MongoDB's **Data API** instead of the native `mongodb` driver.

#### Minimal example — API Route with a cached pool

```js
// app/api/users/[id]/route.js
import { pool } from '@/lib/db'; // the globalThis-cached pool from above

export async function GET(_req, { params }) {
  const { rows } = await pool.query('SELECT * FROM users WHERE id = $1', [params.id]);
  if (!rows[0]) {
    return Response.json({ error: 'Not found' }, { status: 404 });
  }
  return Response.json(rows[0]);
}
```

---

### Driver vs ORM vs ODM — Where Each Fits

| Layer | Example | Talks to |
|---|---|---|
| Driver | `pg`, `mysql2`, `mongodb` | Database wire protocol directly |
| ORM | Prisma, Sequelize, TypeORM | Driver (or its own engine using the same protocol) |
| ODM | Mongoose | `mongodb` driver |

You almost never choose "driver vs ORM" as a global decision — most
real apps use an ORM/ODM for 95% of queries and drop to the raw driver
(often exposed by the ORM itself, e.g. `prisma.$queryRaw`, Sequelize's
`sequelize.query()`) for the remaining 5% that need raw SQL performance
or a feature the ORM doesn't model.

---

### Interview-Ready Summary

- A driver is the lowest-level library that implements a database's wire
  protocol; ORMs/ODMs are built on top of one.
- Always use a **connection pool**, not a single persistent connection —
  and size it relative to the database's connection limit, not per-app
  intuition.
- In Next.js, the two concrete failure modes to know cold:
  1. **Dev hot reload** creates a new pool per file save unless cached on
     `globalThis`.
  2. **Serverless scale-out** multiplies pools across instances — solved
     with a pooling proxy (PgBouncer/Neon/Supabase) or an HTTP-based
     serverless driver, not a bigger `max`.
- **Edge Runtime** can't use standard Node.js drivers at all — it needs an
  HTTP/WebSocket-based driver, or you run that route on the Node.js
  runtime instead.
