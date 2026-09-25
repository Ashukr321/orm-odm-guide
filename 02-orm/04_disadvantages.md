## Disadvantages of ORMs

### At a Glance

| Disadvantage | One-line explanation |
|---|---|
| Hidden SQL | You don't see the queries being run |
| N+1 queries | Loops over lazy relations fire one query per row |
| Over-fetching | `SELECT *` and deep `include`s load far more than needed |
| Inefficient SQL | Generated SQL can be worse than hand-written SQL |
| Leaky abstraction | Complex queries force you back to SQL anyway |
| Runtime overhead | Hydration, change tracking and query engines cost CPU and memory |
| Learning curve | Every ORM has its own API, quirks and pitfalls |
| Vendor lock-in | Your data layer is tied to one library's API |
| Lowest common denominator | Database-specific features are poorly supported |
| Migration risk | Auto-generated migrations can drop data |

---

### 1. The SQL Is Hidden

One innocent-looking line can run several queries, or one very large one.

```js
await prisma.user.findMany({ include: { posts: { include: { comments: true } } } });
// Runs 3 queries (users, posts, comments) and can return a huge amount of data.
```

**Fix:** turn on query logging during development and read it.

```js
const prisma = new PrismaClient({ log: ['query'] });
// Sequelize: new Sequelize(url, { logging: console.log })
// TypeORM:   new DataSource({ …, logging: true })
```

---

### 2. The N+1 Problem

The most common ORM performance bug. One query loads N parents, then N more
queries load each parent's children.

```js
// ❌ 1 + N queries
const users = await prisma.user.findMany();          // 1 query
for (const user of users) {
  user.posts = await prisma.post.findMany({          // N queries
    where: { authorId: user.id },
  });
}

// ✅ 2 queries total
const users = await prisma.user.findMany({ include: { posts: true } });
```

With 1,000 users the first version runs 1,001 queries. Lazy-loading ORMs
(TypeORM lazy relations, Sequelize getters, Hibernate) make this bug easy to
write by accident.

Deep dive: [N+1 Problem](../09-advanced/01_n-plus-one-problem.md) ·
[Eager vs Lazy Loading](../03-orm-concepts/06_eager-vs-lazy-loading.md)

---

### 3. Over-fetching

By default most ORMs select **every column**.

```js
// ❌ Loads password hashes, large bio text, and every other column
const users = await prisma.user.findMany();

// ✅ Only what the API response needs
const users = await prisma.user.findMany({ select: { id: true, name: true } });
```

Over-fetching wastes network bandwidth and memory, and can **leak
sensitive fields** (like `passwordHash`) into API responses.

---

### 4. Generated SQL Isn't Always Optimal

ORMs build SQL for the general case. For complex queries, the result can be
slow or awkward.

| Query type | Where ORMs struggle |
|---|---|
| Reports & analytics | `GROUP BY`, `HAVING`, window functions, CTEs |
| Bulk operations | Updating 100k rows one entity at a time |
| Complex joins | Joins on non-FK conditions, self-joins, lateral joins |
| Deep includes | Large cartesian products or many round trips |

```sql
-- Easy in SQL, painful (or impossible) in most ORM query APIs
SELECT author_id,
       COUNT(*)                                            AS posts,
       RANK() OVER (ORDER BY COUNT(*) DESC)                AS rank
FROM posts
WHERE created_at > now() - interval '30 days'
GROUP BY author_id;
```

**Fix:** use the ORM's raw SQL escape hatch for these, always parameterized.

---

### 5. Leaky Abstraction

The ORM promises "you don't need SQL", but the promise breaks exactly when
things get hard: slow queries, deadlocks, locking, indexing, isolation
levels. At that point you need to understand **both** SQL and how your ORM
translates to it.

> An ORM is a productivity tool for people who know SQL, not a replacement
> for knowing SQL.

---

### 6. Runtime Overhead

| Source | Cost |
|---|---|
| Hydration | Turning rows into objects or model instances costs CPU and memory |
| Change tracking | Unit-of-Work ORMs keep snapshots of every loaded entity |
| Query engine | Some ORMs add a translation layer between your code and the driver |
| Bundle size | Matters in serverless (cold starts) and edge runtimes |

For normal APIs this is negligible next to network and disk time. For
bulk jobs (processing millions of rows), use raw SQL, streaming or
cursors.

```js
// Sequelize: skip model instance creation when you just need data
await User.findAll({ raw: true });
```

---

### 7. Learning Curve and Quirks

Each ORM has its own concepts: sessions, entity managers, dirty checking,
cascades, eager/lazy defaults, hooks. The bugs they cause look like
"magic":

- An update that silently didn't save because the entity was detached.
- A cascade delete that removed more rows than expected.
- A hook running at a time you didn't expect.
- `findOne` returning `null` vs throwing, depending on the method.

---

### 8. Lock-in and Lowest Common Denominator

- Your data-access code is written in the ORM's API. Switching ORMs means
  rewriting that layer.
- To stay portable, ORMs support the features **all** databases share.
  Postgres-specific features (`JSONB` operators, partial indexes, `LISTEN/NOTIFY`,
  full-text search, row-level security) often need raw SQL or manual
  migrations.

---

### 9. Migration Risks

Auto-generated migrations are a diff between your models and the database.
The ORM can misread your intent:

```prisma
model User {
  // Renamed name → fullName
  fullName String?
}
```

```sql
-- What the ORM may generate: data in "name" is lost
ALTER TABLE "User" DROP COLUMN "name";
ALTER TABLE "User" ADD COLUMN "fullName" TEXT;

-- What you actually wanted
ALTER TABLE "User" RENAME COLUMN "name" TO "fullName";
```

**Fix:** always read the generated migration SQL before applying it, and
never use `sync({ force: true })` / `synchronize: true` in production.

---

### Mitigation Cheat Sheet

| Problem | Mitigation |
|---|---|
| Hidden SQL | Enable query logging; use `EXPLAIN ANALYZE` on slow queries |
| N+1 | Eager-load with `include`; use DataLoader in GraphQL |
| Over-fetching | Use `select` / `attributes`; paginate |
| Slow complex queries | Raw parameterized SQL, views, or a query builder |
| Bulk work | `createMany`, `updateMany`, raw SQL, streaming |
| Migration data loss | Review SQL; write rename/backfill steps by hand |
| Sensitive fields leaking | Explicit `select`, or omit fields in the API layer |

---

### Interview-Ready Summary

- ORMs **hide the SQL**, which leads to the **N+1 problem**, **over-fetching**
  and occasionally **inefficient generated queries**.
- The abstraction **leaks** on hard problems (performance, locking,
  complex reports), so you still need SQL knowledge.
- Costs also include **runtime overhead**, **learning curve**, **lock-in**,
  weak support for **database-specific features**, and **risky
  auto-generated migrations**.
- Mitigate with query logging, `include`/`select`, pagination, raw SQL for
  hot paths, and reviewing every migration.
