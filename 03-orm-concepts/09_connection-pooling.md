## Connection Pooling

### Why Pooling Exists

Opening a database connection is **expensive**: a TCP handshake, often a
TLS handshake, authentication, and a new backend process on the server
(PostgreSQL forks one process per connection). That can take tens of
milliseconds, and each connection uses memory on the database server.

A **connection pool** opens a small set of connections **once** and lends
them out:

```
Request A ─┐                     ┌─ conn 1 ─┐
Request B ─┼─► acquire ─► Pool ──┼─ conn 2 ─┼──► Database
Request C ─┘   (wait if all      └─ conn 3 ─┘
                are busy)   ◄── release after the query
```

Every ORM uses a pool: Prisma's query engine, Sequelize and TypeORM (via
the driver's pool: `pg.Pool`, `mysql2`), and Mongoose (via the MongoDB
driver).

---

### How a Pool Behaves

| Setting | Meaning | Typical value |
|---|---|---|
| **max** | Most open connections at once | 5–20 per app instance |
| **min** | Connections kept open even when idle | 0–2 |
| **acquire / pool timeout** | How long a request waits for a free connection before failing | 10–30 s |
| **idle timeout** | Close a connection unused for this long | 10–30 s |

When all connections are busy, new requests **queue**. If they wait longer
than the acquire timeout, they fail with a pool timeout error, even though
the database itself is healthy.

---

### Configuring the Pool in Each ORM

```bash
# Prisma (classic query engine): pool options in the connection string
DATABASE_URL="postgresql://user:pass@host:5432/app?connection_limit=10&pool_timeout=20"
# default connection_limit = (number of CPUs × 2) + 1
```

```js
// Sequelize
new Sequelize(process.env.DATABASE_URL, {
  pool: { max: 10, min: 0, acquire: 30000, idle: 10000 },
});

// TypeORM (PostgreSQL): options passed to pg.Pool
new DataSource({
  type: 'postgres',
  url: process.env.DATABASE_URL,
  extra: { max: 10, idleTimeoutMillis: 30000, connectionTimeoutMillis: 5000 },
});
```

> With Prisma **driver adapters** (e.g. `@prisma/adapter-pg`), the pool
> comes from the driver you pass in, so you configure it on the
> `pg.Pool` itself.

---

### Sizing the Pool

The limit that matters is the **database's**, not your app's:

```
pool max × number of app instances  ≤  DB max_connections − reserved (admin, migrations, cron)

Example: PostgreSQL max_connections = 100, reserve 10
         6 app instances  →  pool max ≈ 90 / 6 = 15 each
```

- **Bigger is not faster.** A database has a limited number of CPU cores.
  Past a point, extra connections just add contention. The PostgreSQL
  wiki's starting point: `connections ≈ (CPU cores × 2) + effective
  spindles`.
- If requests wait for the pool, first make **queries faster** and
  **transactions shorter**. Raising `max` comes after that.
- **Autoscaling multiplies pools.** Scaling from 6 to 20 instances can
  exceed `max_connections` without any config change.

---

### Serverless: Many Tiny Pools

On serverless platforms (Vercel, AWS Lambda), each concurrent invocation
can be a separate instance with its **own pool**:

```
50 concurrent functions × pool of 10 = 500 connections  →  "too many connections"
```

Solutions:

| Fix | How |
|---|---|
| **External pooler** | PgBouncer, RDS Proxy, Supabase / Neon pooler endpoints |
| **Small pool per instance** | `connection_limit=1` (Prisma) or `max: 1–3` |
| **HTTP / edge drivers** | `@neondatabase/serverless`, PlanetScale, Prisma Accelerate |
| **Reuse the client** | Create it once per instance (module scope / `globalThis`), not per request |

> **PgBouncer transaction mode** doesn't support session-level features
> like named prepared statements. Older Prisma versions need
> `?pgbouncer=true` in the URL. Use the pooler URL for the app and a
> **direct** URL for migrations.

---

### One Client per Process

```js
// ❌ A new client (and a new pool) on every request
app.get('/users', async (req, res) => {
  const prisma = new PrismaClient();
  res.json(await prisma.user.findMany());
});

// ✅ One shared client for the whole process
// lib/prisma.js
const globalForPrisma = globalThis;
export const prisma = globalForPrisma.prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;   // survives hot reload
```

The `globalThis` trick stops Next.js dev hot reload from creating a new
pool on every file save. See
[Database Drivers](../01-database-foundation/database-drivers.md).

---

### Connection Leaks and Monitoring

A **leak** is a connection that is checked out and never released. The
pool slowly runs dry, and then every request times out.

```js
// pg: always release in finally
const client = await pool.connect();
try { await client.query('SELECT 1'); }
finally { client.release(); }
```

Common causes: a manually acquired client without `release()`, a
transaction that never commits or rolls back, and very long-running
queries.

```sql
-- PostgreSQL: who is connected, and what are they doing?
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
SELECT pid, now() - query_start AS runtime, state, query
FROM pg_stat_activity WHERE state <> 'idle' ORDER BY runtime DESC;
```

Many `idle in transaction` connections point to transactions that were
never closed.

---

### Interview-Ready Summary

- Connections are expensive. A **pool** opens a few and reuses them;
  requests **queue** when it is full.
- Know the settings: **max, min, acquire timeout, idle timeout**. Prisma
  sets them in the URL (`connection_limit`), Sequelize in `pool`, TypeORM
  in `extra`.
- Size it against the database: **pool max × instances ≤ max_connections
  − reserved**. Bigger is not faster.
- **Serverless** multiplies pools: use an external pooler (PgBouncer, RDS
  Proxy, Neon / Supabase), small pools, or HTTP drivers.
- Keep **one client per process** (`globalThis` in Next.js dev), always
  release connections, and watch `pg_stat_activity`.
