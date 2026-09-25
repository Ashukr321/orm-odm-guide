## Database Performance

Query tuning fixes one slow statement. Database performance is the whole
system: round trips, connections, schema, memory, maintenance and how you
scale once one server isn't enough.

Background: [Connection Pooling](../03-orm-concepts/09_connection-pooling.md) ·
[Query Optimization](02_query-optimization.md) · [Indexing](03_indexing.md)

---

### Where the Time Goes

```
Request latency =  wait for a pooled connection
                +  network round trip(s) to the database
                +  query planning + execution (CPU, disk, locks)
                +  transferring rows
                +  ORM hydration (rows → objects)
```

| Symptom | Likely cause |
|---|---|
| Fast in `EXPLAIN`, slow in the app | Round trips, N+1, pool waits, huge result sets |
| Slow only under load | Pool exhaustion, lock contention, CPU saturation |
| Slow only sometimes | Cache misses, autovacuum, checkpoints, plan changes |
| Slow gets slower over time | Missing index on a growing table, bloat, offset pagination |

---

### Measure First

| Tool | What it tells you |
|---|---|
| APM / OpenTelemetry traces | Which endpoint and which query, with p95/p99 latency |
| `pg_stat_statements` | Total and mean time per normalized query |
| Slow query log | Every statement slower than a threshold |
| `pg_stat_activity` | What's running right now, and who is waiting on locks |
| MongoDB profiler / Atlas Performance Advisor | Slow operations and suggested indexes |

```sql
-- PostgreSQL: log statements slower than 200 ms
ALTER SYSTEM SET log_min_duration_statement = '200ms';
SELECT pg_reload_conf();

-- MySQL
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.2;
```

Track **p95 and p99**, not the average. The average hides the slow requests
users actually notice.

---

### Round Trips Add Up

A query that takes 0.3 ms in the database still costs a full network round
trip, often 1–5 ms across zones.

```js
// 3 sequential round trips
const user = await prisma.user.findUnique({ where: { id } });
const posts = await prisma.post.findMany({ where: { authorId: id } });
const count = await prisma.comment.count({ where: { authorId: id } });

// Independent reads: run them concurrently
const [user2, posts2, count2] = await Promise.all([
  prisma.user.findUnique({ where: { id } }),
  prisma.post.findMany({ where: { authorId: id } }),
  prisma.comment.count({ where: { authorId: id } }),
]);

// Or one query with nested relations
const profile = await prisma.user.findUnique({
  where: { id },
  include: { posts: true, _count: { select: { comments: true } } },
});
```

Keep the application and database in the **same region and zone**.

---

### Connections and Pooling

Every PostgreSQL connection is a process using several MB of RAM. More
connections than CPU cores mostly adds contention.

| Setup | Recommendation |
|---|---|
| Long-running server | One ORM client per process; pool size ≈ a few per CPU core |
| Many app instances | Sum of all pools must stay under `max_connections` |
| Serverless / edge | External pooler: PgBouncer (transaction mode), RDS Proxy, Prisma Accelerate |
| MongoDB | Driver pool per process (`maxPoolSize`, default 100) |

A request waiting for a connection looks like a slow query in your traces.
Check pool metrics before blaming the database. Details:
[Connection Pooling](../03-orm-concepts/09_connection-pooling.md).

---

### Reduce ORM Overhead on Hot Paths

| ORM | Lighter reads |
|---|---|
| Prisma | `select` only needed fields; `$queryRaw` / TypedSQL for heavy reports |
| Sequelize | `raw: true` returns plain objects instead of model instances |
| Mongoose | `.lean()` skips document hydration, getters and change tracking |
| TypeORM | `getRawMany()` on the query builder |

```js
// Mongoose: plain objects, often several times faster for large lists
const posts = await Post.find({ published: true }).select('title slug').lean();

// Sequelize
const rows = await Post.findAll({ attributes: ['id', 'title'], raw: true });
```

The ORM itself is rarely the bottleneck. Measure before you rewrite with raw
SQL.

---

### Schema Design for Speed

- Use the **smallest correct type**: `int` vs `bigint`, `timestamptz`,
  `numeric` for money, `uuid` (not `varchar(36)`).
- **Normalize** first. Denormalize a counter or a copied field only when a
  measured hot read needs it, and update it in the same transaction.
- Keep wide, rarely read columns (large `TEXT`, blobs) in a separate table
  or object storage.
- Prefer time-ordered IDs (UUIDv7, ULID, identity) over random UUIDv4 for
  large tables; random keys fragment B-tree indexes.
- In MongoDB, **embed** what's read together and **reference** what grows
  without bound, keeping documents well below the 16 MB limit.

---

### Scaling Reads: Replicas

Send reads to replicas and writes to the primary.

```js
// Prisma
import { readReplicas } from '@prisma/extension-read-replicas';
const prisma = new PrismaClient().$extends(readReplicas({ url: process.env.REPLICA_URL }));
await prisma.post.findMany();                 // replica
await prisma.$primary().post.findMany();      // force primary

// Sequelize
const sequelize = new Sequelize('blog', null, null, {
  dialect: 'postgres',
  replication: {
    read: [{ host: 'replica-1', username: 'app', password: process.env.DB_PASS }],
    write: { host: 'primary', username: 'app', password: process.env.DB_PASS },
  },
});
```

Replicas lag behind the primary by milliseconds to seconds. Read from the
**primary right after a write** when the user must see their own change.

---

### Big Tables: Partitioning and Archiving

```sql
CREATE TABLE events (
  id         bigint GENERATED ALWAYS AS IDENTITY,
  created_at timestamptz NOT NULL,
  payload    jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_09 PARTITION OF events
  FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
```

- Queries that filter on the partition key read only the matching
  partitions (**partition pruning**).
- Dropping an old partition is instant, unlike a huge `DELETE`.
- Move cold data to an archive table or a warehouse, so the hot table and
  its indexes fit in memory.
- **Sharding** (splitting data across servers) is the last step. Citus,
  Vitess and MongoDB sharding do it, at a large operational cost.

---

### Keep the Database Healthy

| Task | Why |
|---|---|
| Autovacuum (PostgreSQL) | Removes dead row versions left by `UPDATE`/`DELETE`; prevents bloat and XID wraparound |
| `ANALYZE` / fresh statistics | The planner picks good plans only with accurate row estimates |
| Memory sizing | `shared_buffers` / `innodb_buffer_pool_size` should hold the hot data and indexes |
| Statement timeouts | `statement_timeout`, `lock_timeout` stop one bad query from taking everything down |
| Monitor disk and I/O | A full disk or saturated IOPS stalls every query |

```sql
-- Per-role safety net for the application user
ALTER ROLE app SET statement_timeout = '5s';
ALTER ROLE app SET idle_in_transaction_session_timeout = '30s';
```

---

### The Scaling Ladder

Climb only as far as you need.

| Step | Effort |
|---|---|
| 1. Fix N+1 and over-fetching | Low |
| 2. Add the right indexes | Low |
| 3. Tune the pool, add timeouts | Low |
| 4. Cache hot reads | Medium |
| 5. Bigger instance (vertical scaling) | Low, costs money |
| 6. Read replicas | Medium |
| 7. Partition and archive | Medium–high |
| 8. Shard | High |

Most applications never need to go past step 6.

---

### Interview-Ready Summary

- Latency = **pool wait + round trips + execution + transfer + hydration**.
  Measure with APM, `pg_stat_statements` and the slow query log; watch
  **p95/p99**.
- Cut **round trips**: batch, parallelize independent reads, eager load.
- Size **connection pools** against `max_connections`; use PgBouncer or RDS
  Proxy for serverless.
- Use `lean()`, `raw: true` and `select` on hot read paths.
- Scale reads with **replicas** (mind replication lag), big tables with
  **partitioning**, and keep autovacuum, statistics and timeouts healthy.
- Follow the **scaling ladder**: queries → indexes → pool → cache → bigger
  box → replicas → partitions → shards.
