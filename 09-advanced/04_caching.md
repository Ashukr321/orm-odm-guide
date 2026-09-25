## Caching

A cache keeps the result of an expensive read somewhere faster, so the next
request skips the database. It's the most effective scaling tool and the
easiest way to serve stale or wrong data.

Background: [Query Optimization](02_query-optimization.md) ·
[Database Performance](07_database-performance.md)

---

### Where Caches Live

```
Browser / CDN  →  API server (in-process)  →  Redis / Memcached  →  Database
  HTTP cache        LRU map per instance       shared cache          buffer cache
  ms, per user      ~µs, not shared            ~0.5 ms, shared       automatic
```

| Layer | Good for | Watch out for |
|---|---|---|
| HTTP / CDN (`Cache-Control`, `ETag`) | Public pages, assets, API GETs | Personalized data served to the wrong user |
| In-process (LRU map) | Tiny, hot, rarely changing data (config, feature flags) | Each instance has its own copy; memory grows |
| Shared (Redis) | Query results, sessions, computed views | Invalidation, network hop, one more system |
| Database buffer cache | Everything, automatically | Nothing to do except give it RAM |

Fix the query and add the index **before** adding a cache. A cache hides a
slow query; it doesn't fix it.

---

### Cache-Aside (Lazy Loading)

The application checks the cache, falls back to the database on a miss, and
stores the result. This is the default pattern.

```js
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

async function getPost(id) {
  const key = `post:${id}:v1`;

  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);                 // hit

  const post = await prisma.post.findUnique({            // miss
    where: { id },
    select: { id: true, title: true, content: true, author: { select: { name: true } } },
  });
  if (post) await redis.set(key, JSON.stringify(post), 'EX', 300);   // 5-minute TTL
  return post;
}
```

- Only data that is actually read gets cached.
- If Redis goes down, reads fall back to the database (slower, still correct).
- The first request after a miss or an expiry pays the full cost.

---

### Invalidation Strategies

| Strategy | How | Trade-off |
|---|---|---|
| TTL only | `EX 300` and accept up to 5 minutes of staleness | Simplest; stale until it expires |
| Delete on write | `redis.del(key)` after the update commits | Fresh; you must find every write path |
| Versioned keys | Bump `post:42:v7` → `v8`; old keys expire | No delete race; wastes some memory |
| Write-through | Update the cache in the same code path as the DB | Always warm; writes are slower |
| Write-behind | Write to the cache, flush to the DB later | Fast writes; risk of data loss |

```js
async function updatePost(id, data) {
  const post = await prisma.post.update({ where: { id }, data });
  await redis.del(`post:${id}:v1`);        // after the write succeeds
  return post;
}
```

Always combine delete-on-write **with** a TTL, so a missed invalidation
heals itself.

---

### Invalidate After Commit

Deleting the key inside a transaction lets another request refill the cache
with the **old** row before your transaction commits.

```
T1: BEGIN; UPDATE post 42; DEL post:42
T2:                           GET post:42 → miss → SELECT (sees the OLD row) → SET post:42
T1: COMMIT                    ← cache now holds stale data until the TTL
```

```js
// Delete after the transaction resolves, not inside it
await prisma.$transaction(async (tx) => {
  await tx.post.update({ where: { id }, data });
  await tx.auditLog.create({ data: { postId: id, action: 'update' } });
});
await redis.del(`post:${id}:v1`);
```

For stronger guarantees, publish invalidations from the database change
stream (PostgreSQL logical replication / Debezium, MongoDB change streams)
or an outbox table.

---

### Cache Stampede

When a hot key expires, hundreds of requests miss at once and all hit the
database with the same query.

| Fix | Idea |
|---|---|
| TTL jitter | `300 + random(0..60)` seconds, so keys don't expire together |
| Single flight / lock | Only one request rebuilds; others wait or get the old value |
| Stale-while-revalidate | Serve the expired value and refresh it in the background |
| Pre-warming | Refresh popular keys on a schedule before they expire |

```js
// Single flight within one process: concurrent misses share one DB call
const inflight = new Map();
function once(key, load) {
  if (!inflight.has(key)) inflight.set(key, load().finally(() => inflight.delete(key)));
  return inflight.get(key);
}
```

Across many instances, use a short Redis lock (`SET lock:key 1 NX EX 10`) so
only one rebuilds.

---

### What to Cache (and What Not To)

| Cache it | Don't cache it |
|---|---|
| Read-heavy data that changes rarely (product pages, profiles) | Balances, stock counts, anything you check before a write |
| Expensive aggregates (dashboards, leaderboards) | Data that must be read-your-own-writes consistent |
| Results of slow external APIs | Per-user data under a shared key |
| Rendered fragments, computed permissions | Huge objects you only read a field of |

**Key design:** include everything that changes the result:
`feed:user:42:page:1:v3`. Never build keys from raw user input without
normalizing it, or you'll cache the same data many times.

---

### ORM-Level Caching

| Tool | Built-in result cache | Notes |
|---|---|---|
| Prisma | No (Prisma Accelerate adds `cacheStrategy: { ttl, swr }`) | Otherwise use cache-aside around the client |
| Sequelize | No | Cache-aside, or community plugins |
| TypeORM | Yes: `cache: true` / `cache: 60000` on queries | Database table or Redis as the store |
| Mongoose | No | Cache `lean()` results; documents are heavy |

```js
// Prisma Accelerate (managed proxy)
await prisma.post.findMany({
  where: { published: true },
  cacheStrategy: { ttl: 60, swr: 30 },     // fresh for 60 s, serve stale 30 s more while refreshing
});
```

Keep caching **explicit** in a service or repository layer, so it's obvious
which reads can be stale.

---

### Interview-Ready Summary

- Caches sit at the **CDN, in-process, shared (Redis)** and **database**
  layers; fix slow queries before caching them.
- **Cache-aside** is the default: check the cache, load on a miss, store with
  a TTL.
- Invalidate by **TTL**, **delete on write** (after commit) or **versioned
  keys**; always keep a TTL as a safety net.
- Prevent **stampedes** with jitter, single-flight locks and
  stale-while-revalidate.
- Don't cache data you check before writing (balances, stock) or per-user
  data under shared keys.
- Most ORMs have no result cache; TypeORM and Prisma Accelerate are the
  exceptions.
