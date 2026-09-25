## N Plus One Problem

The N+1 problem is the most common ORM performance bug: one query loads a
list, then one more query runs for every item in that list. It looks harmless
in code and falls over in production.

Background: [Eager vs Lazy Loading](../03-orm-concepts/06_eager-vs-lazy-loading.md) ·
[Relationships](../03-orm-concepts/03_relationships.md)

---

### What N+1 Looks Like

```js
// 1 query: load 50 posts
const posts = await prisma.post.findMany({ take: 50 });

// N queries: one per post to get its author
for (const post of posts) {
  post.author = await prisma.user.findUnique({ where: { id: post.authorId } });
}
// Total: 1 + 50 = 51 round trips
```

```sql
SELECT * FROM posts LIMIT 50;
SELECT * FROM users WHERE id = 7;
SELECT * FROM users WHERE id = 3;
SELECT * FROM users WHERE id = 7;   -- same author, fetched again
-- ... 47 more
```

---

### Why It Hurts

| Posts | Queries | At 2 ms per round trip |
|---|---|---|
| 10 | 11 | ~22 ms |
| 100 | 101 | ~200 ms |
| 1,000 | 1,001 | ~2 s |

The cost is **round trips**, not the queries themselves. Each query is fast;
the latency adds up linearly with the list size.

---

### How to Spot It

Turn on query logging and look for the same query shape repeated with
different IDs.

```js
// Prisma
const prisma = new PrismaClient({ log: ['query'] });

// Sequelize
const sequelize = new Sequelize(url, { logging: console.log, benchmark: true });

// Mongoose
mongoose.set('debug', true);
```

Signs in production:

- Endpoint latency grows with the **page size** or number of children.
- APM traces (Datadog, New Relic, OpenTelemetry) show dozens of identical
  spans under one request.
- `pg_stat_statements` shows a trivial query with a huge `calls` count.

---

### Fix 1: Eager Loading

Ask the ORM for the relation up front. It loads all children in one or two
queries instead of N.

```js
// Prisma: 2 queries (posts, then users WHERE id IN (...))
const posts = await prisma.post.findMany({ take: 50, include: { author: true } });

// Sequelize: 1 query with a LEFT OUTER JOIN
const posts = await Post.findAll({ limit: 50, include: [{ model: User, as: 'author' }] });

// Mongoose: 2 queries (posts, then users with $in)
const posts = await Post.find().limit(50).populate('author');
```

```sql
-- What Prisma sends
SELECT * FROM posts LIMIT 50;
SELECT * FROM users WHERE id IN (7, 3, 12, ...);
```

Prisma's `relationLoadStrategy: 'join'` (behind the `relationJoins` preview
flag) switches `include` to a single JOIN-based query on PostgreSQL and MySQL.

---

### Fix 2: Batch It Yourself with `IN`

When the relation isn't modelled, or you're stitching data from two sources,
collect the IDs and load them in one query.

```js
const posts = await prisma.post.findMany({ take: 50 });

const authorIds = [...new Set(posts.map((p) => p.authorId))];
const authors = await prisma.user.findMany({ where: { id: { in: authorIds } } });

const byId = new Map(authors.map((a) => [a.id, a]));
const result = posts.map((p) => ({ ...p, author: byId.get(p.authorId) }));
// Total: 2 queries, no matter how many posts
```

This is exactly what eager loading does internally. Deduplicate the IDs with
a `Set` and keep very large `IN` lists in chunks of a few thousand.

---

### Fix 3: DataLoader (GraphQL)

In GraphQL each field resolver runs on its own, so N+1 is the default.
DataLoader collects every `load(id)` made in the same tick into one batch.

```js
import DataLoader from 'dataloader';

// Create per request, so the cache never leaks between users
const userLoader = new DataLoader(async (ids) => {
  const users = await prisma.user.findMany({ where: { id: { in: [...ids] } } });
  const byId = new Map(users.map((u) => [u.id, u]));
  return ids.map((id) => byId.get(id) ?? null);   // same order as ids
});

const resolvers = {
  Post: { author: (post, _args, ctx) => ctx.userLoader.load(post.authorId) },
};
```

Prisma already batches `findUnique()` calls made in the same tick, which is
why the fluent API (`prisma.post.findUnique(...).author()`) is safe inside
resolvers.

---

### Fix 4: Aggregate in the Database

If you only need a **count** or **sum** per parent, don't load the children
at all.

```js
// N+1: loads every post just to count them
for (const u of users) u.postCount = (await u.getPosts()).length;

// Prisma: one query with a correlated count
const users = await prisma.user.findMany({
  include: { _count: { select: { posts: true } } },
});

// Sequelize: GROUP BY
const counts = await Post.findAll({
  attributes: ['authorId', [sequelize.fn('COUNT', sequelize.col('id')), 'postCount']],
  group: ['authorId'],
  raw: true,
});
```

---

### Hidden N+1 Sources

The loop isn't always visible in your code.

| Where it hides | Example | Fix |
|---|---|---|
| Lazy getters | Sequelize `await post.getAuthor()` inside `.map()` | `include` in the parent query |
| Serializers | `toJSON()` or a DTO mapper that loads a relation | Load relations before serializing |
| Template loops | A view rendering `post.author.name` per row | Eager load in the controller |
| `Promise.all` | `Promise.all(posts.map((p) => load(p)))` | Still N queries, just in parallel; batch instead |
| Nested populate | `populate('comments')` then populate each comment's author | `populate({ path: 'comments', populate: 'author' })` |
| Validation hooks | A `beforeSave` hook that queries per row in a bulk save | Validate in one query before the loop |

`Promise.all` makes the requests concurrent, but it still opens N queries and
can exhaust the connection pool.

---

### Catch It in Tests

Count queries in an integration test so a regression fails CI.

```js
test('GET /posts runs a constant number of queries', async () => {
  const queries = [];
  prisma.$on('query', (e) => queries.push(e.query));   // log: [{ emit: 'event', level: 'query' }]

  await request(app).get('/posts?limit=50');

  expect(queries.length).toBeLessThanOrEqual(3);
});
```

Seed more than one row. A test with a single post can't tell 1+1 queries
from 1+N.

---

### Interview-Ready Summary

- **N+1** = one query for the list plus one per item. Cost grows linearly
  with the list because of **round trips**.
- Spot it with **query logs**, APM traces, or a query that has a huge call
  count in `pg_stat_statements`.
- Fix with **eager loading** (`include`, `populate`), **batching** with
  `WHERE id IN (...)`, **DataLoader** in GraphQL, or **aggregating** in the
  database with `_count` / `GROUP BY`.
- Watch for hidden sources: lazy getters, serializers, nested populate and
  `Promise.all` over a list.
- Guard it with a **query-count assertion** in integration tests.
