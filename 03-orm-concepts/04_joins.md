## Joins

### What a Join Is

A **join** combines rows from two or more tables using a related column,
usually a foreign key. It is how a relational database answers questions
like "every order *with* its customer's name".

```sql
SELECT o.id, o.status, u.name
FROM orders o
JOIN users u ON u.id = o.user_id;
```

ORMs hide the join behind relation loading (`include`, `with`,
`relations`), but it is still a join (or an extra query) underneath. Knowing
which one you get is what separates fast code from slow code.

---

### Join Types

| Join | Returns | ORM equivalent |
|---|---|---|
| `INNER JOIN` | Only rows that match on both sides | Sequelize `required: true`, TypeORM `innerJoin` |
| `LEFT JOIN` | Every left row, with matches or `NULL` | Sequelize `include` (default), TypeORM `leftJoin` |
| `RIGHT JOIN` | Every right row (swap the tables instead) | Rarely used |
| `FULL OUTER JOIN` | Every row from both sides | Raw SQL only |
| `CROSS JOIN` | Every combination (a Cartesian product) | Raw SQL only; usually a bug |

```sql
-- LEFT JOIN: every user, even users without orders
SELECT u.name, COUNT(o.id) AS orders
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
```

---

### How ORMs Load Relations: JOIN vs Separate Queries

There are two strategies, and ORMs use different defaults:

| Strategy | SQL sent | Good at | Weak at |
|---|---|---|---|
| **JOIN** | One query with `JOIN`s | One round trip | Duplicated parent rows; row explosion with several 1:N includes |
| **Separate queries** | Parent query, then `WHERE id IN (…)` per relation | No duplication; simple SQL | One round trip per relation level |

```js
// Prisma: pick the strategy per query (may need the "relationJoins" preview flag)
await prisma.user.findMany({
  relationLoadStrategy: 'join',     // or 'query' = separate queries
  include: { posts: true },
});
```

- **Prisma** has historically used separate queries. Newer versions can
  also use database-level joins (`relationLoadStrategy: 'join'`).
- **Sequelize** and **TypeORM** `find` with relations use `LEFT JOIN`s by
  default.

---

### Joins in Sequelize

```js
// LEFT OUTER JOIN (default): every user, with posts if any
await User.findAll({ include: [{ model: Post, as: 'posts' }] });

// INNER JOIN: only users who HAVE a published post
await User.findAll({
  include: [{ model: Post, as: 'posts', where: { published: true }, required: true }],
});

// Avoid row explosion with two 1:N includes: load one of them separately
await User.findAll({
  include: [
    { model: Post, as: 'posts' },
    { model: Comment, as: 'comments', separate: true },   // separate query
  ],
});
```

> Adding a `where` inside an `include` makes Sequelize switch to an
> `INNER JOIN` unless you set `required: false`.

---

### Joins in TypeORM (Query Builder) and Drizzle

```ts
// TypeORM
const users = await dataSource.getRepository(User)
  .createQueryBuilder('u')
  .leftJoinAndSelect('u.posts', 'p', 'p.published = :pub', { pub: true })
  .innerJoin('u.profile', 'pr')
  .where('u.createdAt > :since', { since })
  .orderBy('u.createdAt', 'DESC')
  .getMany();
```

```ts
// Drizzle: SQL-shaped joins, flat rows back
const rows = await db
  .select({ userName: users.name, postTitle: posts.title })
  .from(users)
  .leftJoin(posts, eq(posts.authorId, users.id))
  .where(eq(posts.published, true));
```

`leftJoinAndSelect` also **hydrates** the joined rows into `user.posts`.
`leftJoin` alone only uses the join for filtering.

---

### Filtering Through a Relation Without Loading It

Often you want to *filter* by a related table, not *load* it:

```js
// Prisma: users with at least one paid order (EXISTS subquery)
await prisma.user.findMany({ where: { orders: { some: { status: 'paid' } } } });

// every / none
await prisma.user.findMany({ where: { orders: { none: {} } } });     // users with no orders
```

```sql
-- What that means in SQL
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'paid');
```

`EXISTS` stops at the first match and never duplicates user rows. That
makes it faster and cleaner than joining and then de-duplicating.

---

### The Two Classic Join Pitfalls

**1. Row explosion (Cartesian product)**
A user with 50 posts and 100 comments, joined to both in one query, returns
50 × 100 = **5,000 rows** for **one** user.
Fix: load one relation in a separate query (`separate: true`, or
`relationLoadStrategy: 'query'`).

**2. `LIMIT` with joined 1:N rows**
`LIMIT 10` on a joined result limits **rows**, not users. You may get
3 users with their posts instead of 10.
Fix: ORMs wrap the parent query in a subquery (Sequelize `subQuery`,
TypeORM `take` / `skip` instead of `limit` / `offset`). Check the generated
SQL.

```ts
// TypeORM: use take/skip with joins, not limit/offset
qb.leftJoinAndSelect('u.posts', 'p').take(10).skip(0);
```

---

### Performance Checklist

- Index **every foreign key** column you join on.
- Select only the columns you need (`select`, `attributes`).
- Don't join relations you don't use.
- Watch out for more than one 1:N join in the same query.
- Log the SQL and run `EXPLAIN ANALYZE` on slow queries.

See [Query Optimization](../09-advanced/02_query-optimization.md) and
[Indexing](../09-advanced/03_indexing.md).

---

### Interview-Ready Summary

- A join combines tables through related columns. `INNER` returns only
  matches; `LEFT` keeps every left row.
- ORMs load relations with either **JOINs** (one round trip, possible
  duplication) or **separate queries** (`WHERE id IN (…)`). Know your ORM's
  default.
- To **filter by a relation without loading it**, use `some` / `every` /
  `none` (`EXISTS` subqueries).
- Know the two pitfalls: **row explosion** from several 1:N joins, and
  **`LIMIT` counting joined rows**.
- Index foreign keys, and check the generated SQL.
