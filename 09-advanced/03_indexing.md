## Indexing

An index is a sorted copy of some columns that points back to the full rows.
It turns "scan every row" into "jump to the right place", at the cost of
extra storage and slower writes.

Background: [Indexes in MongoDB](../06-odm-concepts/09_indexes.md) ·
[Prisma Schema](../04-orm-examples/prisma/02_schema.md)

---

### How a B-tree Index Works

```
Index on posts(author_id)                 Table: posts (heap, unordered)
┌──────────────────────────┐
│ author_id │ row pointer  │              row 1: id=1  author_id=7  ...
│     3     │ → row 5      │              row 2: id=2  author_id=12 ...
│     7     │ → row 1      │              row 3: id=3  author_id=7  ...
│     7     │ → row 3      │              row 4: id=4  author_id=42 ...
│    12     │ → row 2      │              row 5: id=5  author_id=3  ...
│    42     │ → row 4      │
└──────────────────────────┘
WHERE author_id = 7  → binary search in the index → 2 row fetches
```

A lookup costs about **O(log n)** instead of **O(n)**. A B-tree (the default
in PostgreSQL, MySQL and MongoDB) handles:

- Equality: `=`, `IN`
- Ranges: `<`, `>`, `BETWEEN`, prefix `LIKE 'abc%'`
- Sorting: `ORDER BY` in index order (or exactly reversed)
- `MIN` / `MAX` on the indexed column

---

### Composite Indexes and Column Order

An index on `(a, b, c)` is sorted by `a`, then `b`, then `c`. It can serve
any query that uses a **leftmost prefix** of those columns.

| Query on index `(author_id, published, created_at)` | Uses the index? |
|---|---|
| `WHERE author_id = 7` | Yes |
| `WHERE author_id = 7 AND published = true` | Yes |
| `WHERE author_id = 7 AND published = true ORDER BY created_at DESC` | Yes, including the sort |
| `WHERE published = true` | No: skips the first column |
| `WHERE author_id = 7 ORDER BY created_at` | Filter yes; sort needs extra work |

Order columns by the **ESR rule**: **E**quality filters first, then
**S**ort columns, then **R**ange filters.

```sql
-- Query: WHERE author_id = $1 AND created_at > $2 ORDER BY views DESC
CREATE INDEX posts_author_views_created_idx ON posts (author_id, views DESC, created_at);
--                                                    equality   sort          range
```

One well-ordered composite index often replaces several single-column ones.

---

### Covering, Partial and Expression Indexes

```sql
-- Covering: extra columns stored in the index → Index Only Scan
CREATE INDEX posts_author_created_idx ON posts (author_id, created_at DESC) INCLUDE (title);

-- Partial: index only the rows you actually query (smaller, faster)
CREATE INDEX posts_published_created_idx ON posts (created_at DESC) WHERE published = true;

-- Expression: index the result of a function
CREATE UNIQUE INDEX users_email_lower_idx ON users (LOWER(email));
```

| Type | Use it when |
|---|---|
| Covering (`INCLUDE`) | A hot query reads a few extra columns you can store in the index |
| Partial (`WHERE`) | Queries always filter on the same condition, like `deleted_at IS NULL` |
| Expression | Queries filter on `LOWER(col)`, `date_trunc(...)`, or a JSON path |

Partial and expression indexes aren't expressible in every ORM schema
language. Add them with raw SQL in a migration.

---

### Index Types Beyond B-tree (PostgreSQL)

| Type | Good for | Example |
|---|---|---|
| B-tree | Equality, ranges, sorting (default) | `email`, `created_at` |
| Hash | Equality only | Rarely better than B-tree |
| GIN | "Contains" queries over many values per row | `JSONB @>`, arrays, full-text `tsvector`, `pg_trgm` |
| GiST | Geometry, ranges, nearest-neighbour | PostGIS, `tstzrange` overlaps |
| BRIN | Huge, naturally ordered tables | Append-only logs by timestamp |

MySQL InnoDB offers B-tree, `FULLTEXT` and spatial indexes. MongoDB adds
multikey, text, geospatial, hashed and wildcard indexes.

---

### Declaring Indexes in Each ORM

#### Prisma

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  slug      String   @unique
  authorId  Int
  published Boolean  @default(false)
  createdAt DateTime @default(now())
  metadata  Json?

  @@index([authorId, createdAt(sort: Desc)])
  @@index([metadata], type: Gin)          // PostgreSQL only
  @@unique([authorId, slug])
}
```

#### Sequelize and Mongoose

```js
// Sequelize
Post.init({ /* attributes */ }, {
  sequelize,
  indexes: [
    { fields: ['author_id', { name: 'created_at', order: 'DESC' }] },
    { unique: true, fields: ['author_id', 'slug'] },
    { fields: ['created_at'], where: { published: true } },   // partial
  ],
});

// Mongoose
postSchema.index({ author: 1, createdAt: -1 });
postSchema.index({ slug: 1 }, { unique: true });
```

---

### Foreign Keys Need Indexes

PostgreSQL does **not** index foreign key columns automatically. MySQL
InnoDB does.

Without an index on `posts.author_id`:

- `include: { posts: true }` scans the whole `posts` table for each batch.
- `DELETE FROM users WHERE id = 7` scans `posts` to check the constraint or
  cascade.

```sql
-- Find unindexed foreign keys in PostgreSQL (simplified)
SELECT c.conrelid::regclass AS table_name, a.attname AS column_name
  FROM pg_constraint c
  JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY (c.conkey)
 WHERE c.contype = 'f'
   AND NOT EXISTS (
     SELECT 1 FROM pg_index i
      WHERE i.indrelid = c.conrelid AND a.attnum = i.indkey[0]
   );
```

Prisma's `relationMode = "prisma"` (used with PlanetScale/Vitess) creates no
foreign keys at all, so add `@@index` on every relation scalar yourself.

---

### The Cost of Indexes

| Cost | Why |
|---|---|
| Slower writes | Every `INSERT`, `UPDATE` of an indexed column and `DELETE` also updates each index |
| Storage and memory | Indexes compete with table data for the buffer cache |
| Planner choice | More indexes means more plans to consider, and occasionally a worse one |
| Locks during creation | Plain `CREATE INDEX` blocks writes to the table |

```sql
-- Indexes never used since statistics were reset
SELECT relname AS table_name, indexrelname AS index_name, idx_scan
  FROM pg_stat_user_indexes
 WHERE idx_scan = 0
 ORDER BY pg_relation_size(indexrelid) DESC;
```

Index the queries you **actually run**, not every column "just in case".

---

### Adding Indexes in Production

```sql
-- Builds without blocking writes (PostgreSQL). Slower, and can't run inside a transaction.
CREATE INDEX CONCURRENTLY posts_author_created_idx ON posts (author_id, created_at DESC);

-- If it fails, it leaves an INVALID index behind: drop it and retry
DROP INDEX CONCURRENTLY IF EXISTS posts_author_created_idx;
```

- ORM migration tools usually emit plain `CREATE INDEX`. For large tables,
  edit the generated SQL, and check whether your tool wraps migrations in a
  transaction.
- MySQL 8 builds most secondary indexes **online** (`ALGORITHM=INPLACE,
  LOCK=NONE`).
- MongoDB builds indexes without holding an exclusive lock for the whole
  build (4.2+). Still, create them from migrations, not with `autoIndex` in
  production.

---

### Interview-Ready Summary

- A **B-tree index** makes lookups O(log n) and serves equality, ranges,
  prefix `LIKE` and `ORDER BY`.
- Composite indexes follow the **leftmost prefix** rule; order columns by
  **Equality, Sort, Range**.
- Use **covering**, **partial** and **expression** indexes for hot queries;
  **GIN** for JSONB, arrays and full-text search.
- PostgreSQL doesn't index **foreign keys** automatically: add them.
- Indexes slow writes and use memory, so drop unused ones
  (`pg_stat_user_indexes`).
- In production, build with `CREATE INDEX CONCURRENTLY` (PostgreSQL) or an
  online DDL.
