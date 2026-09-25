## Query Optimization

Most slow endpoints come from a small number of bad queries. Measure first,
read the plan, change one thing, then measure again.

Background: [Query Builder](../03-orm-concepts/05_query-builder.md) ·
[Raw SQL vs ORM](../08-comparisons/03_raw-sql-vs-orm.md)

---

### The Workflow

```
1. Measure     → find the slow query (APM, slow query log, pg_stat_statements)
2. Get the SQL → turn on ORM query logging and copy the real statement
3. EXPLAIN     → see how the database actually runs it
4. Fix one thing → index, rewrite, select fewer columns, paginate
5. Re-measure  → same data, same parameters, compare timings
```

Always optimize the **SQL the ORM generates**, not the ORM call you think it
generates.

```sql
-- PostgreSQL: top queries by total time (needs the pg_stat_statements extension)
SELECT calls, round(total_exec_time) AS total_ms, round(mean_exec_time, 2) AS mean_ms, query
  FROM pg_stat_statements
 ORDER BY total_exec_time DESC
 LIMIT 10;
```

---

### Reading `EXPLAIN ANALYZE`

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title FROM posts
 WHERE author_id = 42 AND published = true
 ORDER BY created_at DESC
 LIMIT 20;
```

```
Limit  (cost=0.43..8.62 rows=20) (actual time=0.041..0.090 rows=20 loops=1)
  ->  Index Scan using posts_author_id_created_at_idx on posts
        (actual time=0.040..0.086 rows=20 loops=1)
        Index Cond: (author_id = 42)
        Filter: published
Planning Time: 0.2 ms
Execution Time: 0.1 ms
```

---

### Plan Nodes Cheat Sheet

| Node | Meaning | Usually |
|---|---|---|
| Seq Scan | Reads the whole table | Fine for small tables; a red flag on big ones |
| Index Scan | Walks an index, then fetches rows | Good for selective filters |
| Index Only Scan | Answers from the index alone | Best case |
| Bitmap Heap Scan | Collects matches from an index, then reads pages | Medium selectivity |
| Nested Loop | For each outer row, look up inner rows | Good when the outer side is small |
| Hash Join | Builds a hash table from one side | Good for large, unsorted joins |
| Sort | Sorts in memory or on disk | Watch for `external merge` (disk) |

Compare **estimated rows** to **actual rows**. A big gap means stale
statistics; run `ANALYZE posts;`. MongoDB's equivalent is
`.explain('executionStats')`: look for `COLLSCAN` vs `IXSCAN` and compare
`totalDocsExamined` to `nReturned`.

---

### Fetch Only What You Use

```js
// Over-fetching: every column of every post, plus every column of the author
const posts = await prisma.post.findMany({ include: { author: true } });

// Just what the list page renders
const posts = await prisma.post.findMany({
  select: { id: true, title: true, createdAt: true, author: { select: { name: true } } },
});

// Sequelize
await Post.findAll({ attributes: ['id', 'title'], include: [{ model: User, as: 'author', attributes: ['name'] }] });

// Mongoose: projection + lean() skips building full documents
await Post.find({}, 'title createdAt').populate('author', 'name').lean();
```

Selecting fewer columns moves less data over the network, and it lets the
database use an **index-only scan** when every selected column is in the
index. Large `TEXT`/`JSONB` columns are the most expensive to over-fetch.

---

### Pagination: Offset vs Cursor

#### In Prisma

```js
// Offset: simple, but the database still reads and discards `skip` rows
await prisma.post.findMany({ orderBy: { id: 'desc' }, skip: 10000, take: 20 });

// Cursor (keyset): jumps straight to the position using the index
await prisma.post.findMany({
  orderBy: { id: 'desc' },
  cursor: { id: lastSeenId },
  skip: 1,          // skip the cursor row itself
  take: 20,
});
```

#### In SQL, and the Trade-offs

```sql
-- Keyset in SQL with a non-unique sort column: add a tiebreaker
SELECT id, title, created_at FROM posts
 WHERE (created_at, id) < ($1, $2)
 ORDER BY created_at DESC, id DESC
 LIMIT 20;
```

| | Offset | Cursor / keyset |
|---|---|---|
| Deep pages | Slower the further you go | Constant time |
| Jump to page 57 | Yes | No, only next/previous |
| Rows inserted meanwhile | Duplicates or skipped rows | Stable |
| Needs | Nothing | An index on the sort columns |

---

### Let the Database Do the Work

```js
// Slow: loads every post into Node to sum the views
const posts = await prisma.post.findMany({ where: { authorId } });
const totalViews = posts.reduce((s, p) => s + p.views, 0);

// Fast: one row back
const { _sum } = await prisma.post.aggregate({ where: { authorId }, _sum: { views: true } });
```

Filter, sort, count, group and paginate **in the query**. Moving rows to the
application just to filter them costs network, memory and CPU.

---

### Write Queries the Index Can Use

| Pattern | Problem | Fix |
|---|---|---|
| `WHERE LOWER(email) = $1` | Function on the column skips a plain index | Expression index on `LOWER(email)`, or `citext` |
| `WHERE title LIKE '%sql%'` | Leading wildcard can't use a B-tree | Full-text search or a trigram (`pg_trgm`) GIN index |
| `WHERE created_at::date = $1` | Cast on the column | Range: `created_at >= $1 AND created_at < $1 + 1 day` |
| `WHERE a = 1 OR b = 2` | Often a Seq Scan | Separate indexes (bitmap OR), or `UNION ALL` |
| `WHERE id IN (...10,000 ids)` | Huge parameter lists | Chunk, or join against a temp table / `ANY($1)` array |
| `ORDER BY RANDOM()` | Sorts the whole table | `TABLESAMPLE`, or pick random IDs |

Prisma's `mode: 'insensitive'` compiles to `ILIKE` on PostgreSQL, which has
the same indexing caveats as `LOWER()`.

---

### Bulk Operations

```js
// 1,000 INSERT statements
for (const row of rows) await prisma.tag.create({ data: row });

// One multi-row INSERT
await prisma.tag.createMany({ data: rows, skipDuplicates: true });

// Sequelize / Mongoose equivalents
await Tag.bulkCreate(rows, { ignoreDuplicates: true });
await TagModel.insertMany(rows, { ordered: false });

// Update many rows in one statement
await prisma.post.updateMany({ where: { authorId }, data: { published: false } });
```

For very large imports, write in batches of 500–5,000 rows so a single
statement doesn't hold locks or memory for too long.

---

### Counting Is Not Free

```js
// Exact count scans the matching rows on PostgreSQL (MVCC has no cached row count)
const total = await prisma.post.count({ where: { published: true } });
```

- On big tables, show **"Next page"** instead of "Page 1 of 4,812".
- Use an estimate for dashboards:
  `SELECT reltuples::bigint FROM pg_class WHERE relname = 'posts';`
- Keep a counter column (e.g. `users.post_count`) updated in the same
  transaction when exact counts are hot.

---

### Optimization Checklist

| Check | Tool |
|---|---|
| Is this query actually slow, and how often does it run? | `pg_stat_statements`, APM |
| What SQL does the ORM send? | ORM query logging |
| Seq Scan on a large table? | `EXPLAIN ANALYZE` |
| Estimated vs actual rows far apart? | `ANALYZE table` |
| Selecting columns nobody reads? | `select` / `attributes` / projection |
| Deep offset pagination? | Cursor pagination |
| Filtering or summing in JavaScript? | `where`, `aggregate`, `groupBy` |
| Loop of single-row writes? | `createMany`, `updateMany`, `bulkCreate` |
| N+1 pattern? | [N+1 Problem](01_n-plus-one-problem.md) |

---

### Interview-Ready Summary

- **Measure → get the SQL → EXPLAIN → fix one thing → re-measure.** Optimize
  the SQL the ORM generates.
- In `EXPLAIN ANALYZE`, look for **Seq Scans** on large tables, big
  **estimate vs actual** gaps and **disk sorts**.
- **Select only needed columns**; use `lean()` in Mongoose and `raw: true` in
  Sequelize for read-only data.
- Prefer **cursor (keyset) pagination** for deep or infinite lists.
- Push **filtering, aggregation and bulk writes** into the database.
- Avoid functions on indexed columns and leading-wildcard `LIKE`; exact
  `COUNT(*)` on big tables is expensive.
