## Eager vs Lazy Loading

### The Question Both Answer

When you load a user, **when** should the ORM load the user's posts?

| Strategy | When related data is loaded | Queries |
|---|---|---|
| **Eager loading** | Up front, together with the parent | 1 (JOIN) or 2 (`WHERE id IN …`), no matter how many rows |
| **Lazy loading** | Later, the first time you access the relation | 1 per access, which becomes **N+1** inside a loop |
| **Explicit loading** | When you ask for it with a separate call | You decide |

---

### Eager Loading

You tell the ORM up front which relations you need.

```js
// Prisma
const users = await prisma.user.findMany({ include: { posts: true } });

// Sequelize
const users = await User.findAll({ include: [{ model: Post, as: 'posts' }] });

// TypeORM
const users = await userRepo.find({ relations: { posts: true } });
```

```sql
-- What Prisma sends: 2 queries in total, for 1 user or 10,000 users
SELECT * FROM users;
SELECT * FROM posts WHERE author_id IN (1, 2, 3, …);
```

✅ A predictable, small number of queries.
❌ Loads data even when a code path doesn't use it (over-fetching).

---

### Lazy Loading

The relation is loaded **on first access**, behind the scenes or through a
getter.

```js
// Sequelize: association getter
const users = await User.findAll();              // 1 query
for (const user of users) {
  const posts = await user.getPosts();           // +1 query per user
}
```

```ts
// TypeORM lazy relation: declared as a Promise
@OneToMany(() => Post, (post) => post.author)
posts: Promise<Post[]>;

const posts = await user.posts;                  // query runs here
```

✅ Loads nothing you don't use.
❌ Hidden queries. In a loop, that is the **N+1 problem**.

---

### The N+1 Problem

```
1 query   → SELECT * FROM users;                          (100 users)
+100      → SELECT * FROM posts WHERE author_id = 1;
            SELECT * FROM posts WHERE author_id = 2;
            … × 100
= 101 queries for one page
```

Each query may take only 1–2 ms, but 101 round trips add up to 100+ ms,
and under load they compete for pool connections.

**Fix it with eager loading or batching:**

```js
// 2 queries total instead of 101
const users = await prisma.user.findMany({ include: { posts: true } });
```

Deep dive: [N+1 Problem](../09-advanced/01_n-plus-one-problem.md)

---

### Prisma: No Lazy Loading

Prisma returns **plain objects**. Accessing `user.posts` never triggers a
query: the field is simply missing unless you `include` it. That rules out
accidental lazy loading.

```js
const user = await prisma.user.findUnique({ where: { id: 1 } });
user.posts;               // undefined (not loaded), and TypeScript flags it

// Explicit loading with the fluent API
const posts = await prisma.user.findUnique({ where: { id: 1 } }).posts();
```

> Prisma can still produce N+1: you can write a loop that calls
> `findMany` once per user. The fix is the same, `include` or a single
> `WHERE … IN` query.

---

### `include` vs `select`: Load Only What You Use

Eager loading everything over-fetches. Pick the exact fields:

```js
// ❌ Every column of users + every column of every post
await prisma.user.findMany({ include: { posts: true } });

// ✅ Only what the page shows
await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    posts: {
      select: { id: true, title: true },
      where: { published: true },
      orderBy: { createdAt: 'desc' },
      take: 3,
    },
  },
});
```

The same idea exists elsewhere: Sequelize `attributes: ['id', 'name']`,
TypeORM `select: { id: true, name: true }`.

---

### Batching: The DataLoader Pattern (GraphQL)

In GraphQL, each field resolver runs separately, which makes N+1 easy.
**DataLoader** collects every ID requested in the same tick and loads them
in **one** query:

```js
import DataLoader from 'dataloader';

const postsByAuthor = new DataLoader(async (authorIds) => {
  const posts = await prisma.post.findMany({ where: { authorId: { in: authorIds } } });
  return authorIds.map((id) => posts.filter((p) => p.authorId === id));  // same order as the input
});

// resolver: User.posts
posts: (user) => postsByAuthor.load(user.id);   // 100 users → 1 query
```

Create a new DataLoader **per request**, so cached data never leaks
between users.

---

### Configuring Eager Loading by Default (and Why to Avoid It)

```ts
// TypeORM: always load the profile with the user
@OneToOne(() => Profile, { eager: true })
@JoinColumn()
profile: Profile;
```

`eager: true` feels convenient, but every `find` now joins the relation,
even on code paths that don't need it. Prefer eager loading **per query**
(`relations`, `include`), where you can see it.

---

### How to Spot the Problem

- **Log queries** in development:
  `new PrismaClient({ log: ['query'] })`, Sequelize `logging: console.log`,
  TypeORM `logging: true`.
- Look for the **same query repeated** with different IDs.
- Count queries per request in APM tools (Datadog, New Relic, Sentry
  performance).
- Add a test that fails if an endpoint runs more than N queries.

---

### Interview-Ready Summary

- **Eager** = load relations up front (`include`, `relations`): a fixed,
  small number of queries.
- **Lazy** = load on first access (`getPosts()`, `Promise<Post[]>`): hidden
  queries, and **N+1** inside loops.
- **Prisma has no lazy loading**. Relations are absent unless you
  `include` them; the fluent API provides explicit loading.
- Use **`select`** so eager loading doesn't over-fetch. In GraphQL, batch
  with **DataLoader**, one instance per request.
- **Log queries** to catch N+1 early, and avoid global `eager: true`.
