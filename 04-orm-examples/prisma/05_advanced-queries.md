## Prisma Advanced Queries

### Complex Filters

```ts
const posts = await prisma.post.findMany({
  where: {
    AND: [
      { published: true },
      {
        OR: [
          { title:   { contains: 'orm', mode: 'insensitive' } },
          { content: { contains: 'orm', mode: 'insensitive' } },
        ],
      },
    ],
    NOT: { author: { role: 'ADMIN' } },
  },
});
```

**Build filters dynamically** from query parameters. `undefined` values are
ignored:

```ts
import type { Prisma } from './generated/prisma/client.js';

function searchPosts(q: { search?: string; authorId?: number; tag?: string }) {
  const where: Prisma.PostWhereInput = {
    published: true,
    authorId: q.authorId,                                    // skipped if undefined
    title: q.search ? { contains: q.search, mode: 'insensitive' } : undefined,
    tags: q.tag ? { some: { name: q.tag } } : undefined,
  };
  return prisma.post.findMany({ where });
}
```

---

### Cursor Pagination

Offset pagination (`skip: 100000`) makes the DB read and discard every
skipped row. Cursor pagination jumps straight to the right place using an
index.

```ts
async function feed(cursor?: number, take = 20) {
  const posts = await prisma.post.findMany({
    where: { published: true },
    orderBy: { id: 'desc' },
    take: take + 1,                         // fetch one extra to know if there's more
    ...(cursor && { cursor: { id: cursor }, skip: 1 }), // skip the cursor row itself
  });

  const hasMore = posts.length > take;
  const items = hasMore ? posts.slice(0, take) : posts;
  return { items, nextCursor: hasMore ? items.at(-1)!.id : null };
}
```

| | Offset (`skip`/`take`) | Cursor |
|---|---|---|
| Jump to page N | ✅ | ❌ |
| Speed on deep pages | Gets slower | Constant |
| Stable when rows are inserted | ❌ duplicates/skips | ✅ |
| Best for | Admin tables | Infinite scroll, feeds, APIs |

The cursor field must be **unique and sortable** (`id`, or a unique
`createdAt` + `id` pair).

---

### Aggregations

```ts
const stats = await prisma.post.aggregate({
  where: { published: true },
  _count: { _all: true },
  _sum:   { views: true },
  _avg:   { views: true },
  _max:   { createdAt: true },
});
// { _count: { _all: 42 }, _sum: { views: 9120 }, _avg: { views: 217.1 }, _max: { createdAt: … } }
```

### Group By

```ts
// Top authors by total views, only those with 5+ published posts
const top = await prisma.post.groupBy({
  by: ['authorId'],
  where: { published: true },
  _count: { _all: true },
  _sum: { views: true },
  having: { authorId: { _count: { gte: 5 } } },
  orderBy: { _sum: { views: 'desc' } },
  take: 10,
});
```

```sql
SELECT author_id, COUNT(*), SUM(views) FROM posts
WHERE published = true
GROUP BY author_id HAVING COUNT(author_id) >= 5
ORDER BY SUM(views) DESC LIMIT 10;
```

`groupBy` returns IDs only. Load names with a second query
(`findMany({ where: { id: { in: ids } } })`).

### Distinct

```ts
await prisma.post.findMany({ distinct: ['authorId'], select: { authorId: true } });
```

---

### Raw SQL (the Escape Hatch)

Use raw SQL for window functions, CTEs, full-text search and bulk updates.

**`$queryRaw`: SELECT, returns rows.** The tagged template sends
`${values}` as parameters, so it is safe from injection.

```ts
const since = new Date('2026-01-01');

const ranked = await prisma.$queryRaw<{ author_id: number; posts: number; rank: bigint }[]>`
  SELECT author_id,
         COUNT(*)::int                           AS posts,
         RANK() OVER (ORDER BY COUNT(*) DESC)    AS rank
  FROM posts
  WHERE created_at > ${since}
  GROUP BY author_id`;
```

**`$executeRaw`: INSERT/UPDATE/DELETE, returns the affected row count.**

```ts
const n = await prisma.$executeRaw`
  UPDATE posts SET published = false WHERE created_at < ${cutoff}`;
```

**Dynamic pieces** with `Prisma.sql`, `Prisma.join`, `Prisma.empty`:

```ts
import { Prisma } from './generated/prisma/client.js';

const ids = [1, 2, 3];
const onlyPublished = true;

const rows = await prisma.$queryRaw`
  SELECT id, title FROM posts
  WHERE id IN (${Prisma.join(ids)})
  ${onlyPublished ? Prisma.sql`AND published = true` : Prisma.empty}`;
```

**Never** build SQL with string concatenation:

```ts
await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE email = '${email}'`);       // ❌ injection
await prisma.$queryRawUnsafe('SELECT * FROM users WHERE email = $1', email);         // ✅ if you must
```

Table and column names can't be parameters. If they come from input,
check them against an allow-list.

For fully typed, SQL-file-based queries, see Prisma's **TypedSQL**
(`prisma/sql/*.sql` + `prisma generate --sql`).

---

### Full-Text Search (PostgreSQL)

```ts
const results = await prisma.$queryRaw`
  SELECT id, title
  FROM posts
  WHERE to_tsvector('english', title || ' ' || coalesce(content, ''))
        @@ plainto_tsquery('english', ${term})
  LIMIT 20`;
```

Add a GIN index on the `tsvector` expression in a migration for speed.

---

### Client Extensions

Extensions add behavior to the client without wrapping every call.
(They replace the old `$use` middleware, which was removed.)

**Computed field:**

```ts
const xprisma = prisma.$extends({
  result: {
    user: {
      displayName: {
        needs: { name: true, email: true },
        compute: (u) => u.name ?? u.email.split('@')[0],
      },
    },
  },
});

const u = await xprisma.user.findFirst();
u?.displayName;
```

**Custom model method:**

```ts
const xprisma = prisma.$extends({
  model: {
    post: {
      publish(id: number) {
        return prisma.post.update({ where: { id }, data: { published: true } });
      },
    },
  },
});

await xprisma.post.publish(10);
```

**Query hook (timing / logging):**

```ts
const xprisma = prisma.$extends({
  query: {
    $allModels: {
      async $allOperations({ model, operation, args, query }) {
        const start = performance.now();
        const result = await query(args);
        console.log(`${model}.${operation} ${(performance.now() - start).toFixed(1)}ms`);
        return result;
      },
    },
  },
});
```

**Soft delete** (add `deletedAt DateTime?` to the model):

```ts
const xprisma = prisma.$extends({
  query: {
    post: {
      async findMany({ args, query }) {
        args.where = { ...args.where, deletedAt: null };
        return query(args);
      },
    },
  },
  model: {
    post: {
      softDelete(id: number) {
        return prisma.post.update({ where: { id }, data: { deletedAt: new Date() } });
      },
    },
  },
});
```

---

### Logging and Query Events

```ts
const prisma = new PrismaClient({
  adapter,
  log: [{ emit: 'event', level: 'query' }],
});

prisma.$on('query', (e) => {
  if (e.duration > 200) console.warn(`Slow query (${e.duration}ms): ${e.query}`, e.params);
});
```

Run slow queries through `EXPLAIN ANALYZE` in `psql` to see whether an
index is missing. See [Query Optimization](../../09-advanced/02_query-optimization.md).

---

### Reusable Query Shapes and Types

```ts
import { Prisma } from './generated/prisma/client.js';

const postCard = {
  id: true,
  title: true,
  author: { select: { name: true } },
  _count: { select: { tags: true } },
} satisfies Prisma.PostSelect;

type PostCard = Prisma.PostGetPayload<{ select: typeof postCard }>;

const cards: PostCard[] = await prisma.post.findMany({ select: postCard });
```

---

### Performance Checklist

| Do | Why |
|---|---|
| `select` only the fields you need | Less data, no leaking of sensitive fields |
| Cursor pagination on large tables | Constant speed on deep pages |
| `include` instead of loops | Avoids N+1 |
| `createMany` / `updateMany` for bulk work | One statement instead of N |
| `@@index` on filter/sort/FK columns | Prisma doesn't add FK indexes for you |
| Log and time queries | Find slow ones early |
| Raw SQL for reports | Window functions, CTEs, full-text search |

---

### Interview-Ready Summary

- Compose filters with `AND`/`OR`/`NOT` and build them dynamically (undefined
  values are ignored).
- Prefer **cursor pagination** (`cursor` + `skip: 1` + `take`) over deep offsets.
- `aggregate`, `groupBy` (with `having`) and `distinct` cover most analytics.
- `$queryRaw` / `$executeRaw` tagged templates are parameterized and safe;
  `*Unsafe` variants are not. Use `Prisma.sql` / `Prisma.join` for dynamic SQL.
- **Client extensions** add computed fields, model methods, query hooks and
  soft deletes (they replace middleware).
- Use `satisfies Prisma.XSelect` + `GetPayload` for reusable, typed query shapes.

Next: [Transactions](06_transactions.md)
