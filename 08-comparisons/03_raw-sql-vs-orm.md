## Raw SQL vs ORM

### The Spectrum

```
Raw driver (pg, mysql2)  →  Query builder (Kysely, Knex)  →  ORM (Prisma, Sequelize, TypeORM)
more control, more code                                     less code, more abstraction
```

| | Raw SQL | Query builder | ORM |
|---|---|---|---|
| You write | SQL strings | SQL via chained functions | Model method calls |
| Results | Plain rows | Plain rows (typed with Kysely) | Typed objects, relations attached |
| Type safety | None, unless you add it | Good (Kysely) | Good to excellent |
| Relations | Manual JOINs + mapping | Manual JOINs | `include` / `relations` |
| Migrations | Separate tool | Often built in | Built in |
| Performance control | Total | High | Lower; inspect the SQL |
| Learning | SQL | SQL + builder API | ORM API (+ SQL for debugging) |

Background: [What Is ORM](../02-orm/01_what-is-orm.md) ·
[Query Builder](../03-orm-concepts/05_query-builder.md)

---

### Five Tasks, Both Ways

Examples use `pg` for raw SQL and Prisma for the ORM, with the blog schema
from [Prisma Schema](../04-orm-examples/prisma/02_schema.md).

#### 1. Simple CRUD

```js
// Raw
const { rows: [user] } = await pool.query(
  'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING id, email, name, created_at',
  [email, name]
);
// user.created_at: snake_case; map it yourself

// ORM
const user = await prisma.user.create({ data: { email, name } });   // typed, camelCase
```

**Winner: ORM.** Less code, types, and no mapping.

#### 2. Parent with Children

```js
// Raw: JOIN, then group rows into objects
const { rows } = await pool.query(
  `SELECT u.id, u.email, p.id AS post_id, p.title
     FROM users u LEFT JOIN posts p ON p.author_id = u.id
    WHERE u.id = $1`,
  [id]
);
const user = rows.length
  ? { id: rows[0].id, email: rows[0].email,
      posts: rows.filter((r) => r.post_id).map((r) => ({ id: r.post_id, title: r.title })) }
  : null;

// ORM
const user = await prisma.user.findUnique({ where: { id }, include: { posts: { select: { id: true, title: true } } } });
```

**Winner: ORM.** Relation stitching is exactly what ORMs are for.

#### 3. A Report (Window Functions, CTE)

```sql
-- Raw: natural in SQL
WITH monthly AS (
  SELECT author_id, date_trunc('month', created_at) AS month, COUNT(*) AS posts
  FROM posts WHERE published
  GROUP BY 1, 2
)
SELECT author_id, month, posts,
       posts - LAG(posts) OVER (PARTITION BY author_id ORDER BY month) AS change
FROM monthly
ORDER BY author_id, month;
```

```js
// ORM: groupBy can't do LAG or date_trunc; you'd fetch rows and compute in JS.
// So use the ORM's raw escape hatch:
const rows = await prisma.$queryRaw`WITH monthly AS (…) SELECT …`;
```

**Winner: raw SQL** (run through the ORM's safe `$queryRaw`).

#### 4. Bulk Update

```js
// Raw: one statement
await pool.query(`UPDATE posts SET published = false WHERE created_at < $1`, [cutoff]);

// ORM: also one statement
await prisma.post.updateMany({ where: { createdAt: { lt: cutoff } }, data: { published: false } });

// ❌ ORM anti-pattern: load everything, update one by one
for (const p of await prisma.post.findMany()) await prisma.post.update({ where: { id: p.id }, data: { … } });
```

**Tie**, as long as you use `updateMany`. Updates with per-row values (for
example, from a `VALUES` list or a join) favour raw SQL.

#### 5. Upsert with Conflict Handling

```js
// (assumes tags has a `usage` counter column)
// Raw: full control over ON CONFLICT
await pool.query(
  `INSERT INTO tags (name) VALUES ($1)
   ON CONFLICT (name) DO UPDATE SET usage = tags.usage + 1
   RETURNING *`,
  [name]
);

// ORM
await prisma.tag.upsert({ where: { name }, create: { name }, update: { usage: { increment: 1 } } });
```

**Tie** for simple cases. Raw SQL wins for multi-row upserts with complex
conflict rules.

---

### Performance

| Factor | Raw SQL | ORM |
|---|---|---|
| Query quality | Exactly what you wrote | Generated; usually good, sometimes not |
| Overhead per query | Driver only | + query building + hydration (small for normal APIs) |
| Over-fetching | Only what you `SELECT` | `SELECT *` by default; fix with `select` |
| N+1 risk | You'd notice writing it | Easy to trigger with lazy loading / loops |
| Bulk / streaming | Cursors, `COPY`, batch statements | `createMany`/`updateMany`; streaming is limited |

For a typical API request, **network and index usage dominate**. ORM
overhead is rarely the bottleneck. The real ORM performance problems are
N+1 and over-fetching, and both are fixable. See
[N+1 Problem](../09-advanced/01_n-plus-one-problem.md).

---

### Safety

```js
// ❌ Raw SQL built by concatenation: SQL injection
await pool.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ Raw SQL with parameters
await pool.query('SELECT * FROM users WHERE email = $1', [email]);

// ✅ ORM: always parameterized
await prisma.user.findUnique({ where: { email } });

// ✅ ORM raw escape hatch, tagged template = parameterized
await prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}`;

// ❌ ORM "Unsafe" variant with interpolation: injection again
await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE email = '${email}'`);
```

Raw SQL is only as safe as the discipline of whoever writes it. An ORM
makes the safe path the default.

---

### Maintainability

| Concern | Raw SQL | ORM |
|---|---|---|
| Renaming a column | Search every SQL string | Change the schema; the compiler finds usages |
| Onboarding | Everyone reads SQL | Learn the ORM API |
| Code review | SQL is explicit and reviewable | Generated SQL is hidden; review the logs |
| Consistency | Varies by author | One pattern everywhere |
| Testing | Needs a real DB (or SQL mocks) | Mock the client/repository, or use a test DB |

---

### The Hybrid Approach (What Most Teams Do)

```
┌─────────────────────────────────────────┬──────────────────────────┐
│ ORM (~90%)                               │ Raw SQL (~10%)            │
│ CRUD, relations, nested writes,          │ Reports, CTEs, window     │
│ transactions, migrations                 │ functions, bulk jobs,     │
│                                          │ hot paths found by        │
│                                          │ profiling                 │
└─────────────────────────────────────────┴──────────────────────────┘
```

Keep raw SQL **contained** behind repository functions, **parameterized**,
and **typed**:

```ts
// repositories/reports.ts
export async function monthlyPostsByAuthor(since: Date) {
  return prisma.$queryRaw<{ author_id: number; month: Date; posts: number }[]>`
    SELECT author_id, date_trunc('month', created_at) AS month, COUNT(*)::int AS posts
    FROM posts WHERE created_at >= ${since}
    GROUP BY 1, 2 ORDER BY 1, 2`;
}
```

Options for **typed** raw SQL: Prisma TypedSQL (`.sql` files → generated
types), Kysely as a typed query builder next to your ORM, or `pgtyped`.

---

### Decision Checklist

| Use raw SQL when… | Use the ORM when… |
|---|---|
| The query needs CTEs, window functions or DB-specific features | It's CRUD, filtering, pagination or relations |
| Profiling shows the ORM's SQL is slow | Performance is fine (most of the time) |
| You're doing bulk ETL or data migrations | You want types and consistent code |
| It's a small script and the ORM setup isn't worth it | Many developers share the codebase |

---

### Interview-Ready Summary

- The spectrum runs **raw driver → query builder → ORM**, trading control for
  productivity and safety.
- **ORM wins** at CRUD, relations and consistency. **Raw SQL wins** at
  reports, CTEs, window functions and complex bulk operations. Simple bulk
  updates and upserts are a tie.
- ORM overhead is rarely the bottleneck; **N+1 and over-fetching** are.
- Raw SQL must be **parameterized**; never use `*Unsafe` with interpolated
  input.
- Use the **hybrid**: ORM by default, raw SQL behind typed repository
  functions for the rest.
