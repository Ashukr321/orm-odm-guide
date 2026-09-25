## Why ORM

### The Short Answer

Your application thinks in **objects** (`user.posts`, `order.total`), but a
relational database stores **rows in tables**. Something has to translate
between the two on every read and every write. You can write that
translation by hand for every query, or you can let an ORM do it.

> People use an ORM to stop writing the same data-access code over and over,
> and to make the code that remains **safe, typed and consistent**.

---

### Life Without an ORM

Start with a plain driver (`pg`). The first query is easy. The pain shows up
at query number fifty.

```js
// 1. SQL as strings: typos are only found at runtime
const { rows } = await pool.query(
  'SELECT id, email, created_at FROM users WHERE id = $1',
  [id]
);

// 2. Manual mapping on every query
const user = rows[0] && {
  id: rows[0].id,
  email: rows[0].email,
  createdAt: rows[0].created_at,   // snake_case → camelCase, by hand
};

// 3. Relations mean more queries and more stitching code
const { rows: posts } = await pool.query(
  'SELECT * FROM posts WHERE author_id = $1',
  [id]
);
user.posts = posts;

// 4. Rename a column? grep every SQL string in the codebase and hope.
```

| Pain | What happens in a real codebase |
|---|---|
| **Repetition** | The same `SELECT` + mapping code is copied into many files |
| **No type safety** | `row.emial` is `undefined`, not a compile error |
| **Schema drift** | A column is renamed; the SQL strings that use it break at runtime |
| **Injection risk** | One developer uses string concatenation instead of `$1` and you have a hole |
| **Relation stitching** | Every JOIN or second query needs hand-written grouping code |
| **Schema changes** | `ALTER TABLE` scripts are written, ordered and applied by hand |
| **Inconsistency** | Every developer writes data access in a different style |

---

### Life With an ORM

The same work with Prisma:

```js
const user = await prisma.user.findUnique({
  where: { id },
  include: { posts: true },
});
// user.createdAt, user.posts: typed, mapped, parameterized
```

And a schema change becomes a model edit plus one command:

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  fullName  String?  @map("full_name")   // new column
  createdAt DateTime @default(now()) @map("created_at")
  posts     Post[]
}
```

```bash
npx prisma migrate dev --name add_full_name
```

The compiler now flags every place in the code that uses the model
incorrectly, **before** the code reaches production.

---

### The Seven Reasons Teams Choose an ORM

| # | Reason | What you get |
|---|---|---|
| 1 | **Productivity** | CRUD in one line; no mapping code |
| 2 | **Type safety** | Column and relation names checked at compile time (Prisma, Drizzle, TypeORM) |
| 3 | **Security by default** | Every value is sent as a bound parameter, so SQL injection is avoided unless you opt out |
| 4 | **Relations made easy** | `include` / `with` / `relations` instead of JOIN + grouping code |
| 5 | **Migrations** | Versioned, reviewable schema changes kept in git |
| 6 | **One consistent pattern** | New developers learn one API, not everyone's SQL style |
| 7 | **Portability** | Switch SQLite (tests) ↔ Postgres (prod) with little or no code change |

Details and trade-offs: [Advantages](03_advantages.md) ·
[Disadvantages](04_disadvantages.md)

---

### Why Not Just Write SQL?

You still can, and sometimes you should. An ORM is not "SQL is bad". It is
"most of our queries are boring, so automate the boring ones."

```
A typical app's queries
┌──────────────────────────────────────────────┬─────────┐
│  ~90–95%: CRUD, filters, pagination, joins   │  5–10%  │
│  → ORM                                       │  → raw  │
│                                              │   SQL   │
└──────────────────────────────────────────────┴─────────┘
                                   reports, analytics, hot paths
```

Good ORMs expect this split and give you a safe escape hatch
(`prisma.$queryRaw`, `sequelize.query`, `dataSource.query`).

---

### Who Benefits Most

| Situation | Why an ORM helps |
|---|---|
| Startups / MVPs | Ship features fast; schema changes are cheap |
| Teams with mixed SQL skill | One safe, consistent API for everyone |
| TypeScript codebases | End-to-end types from database to API response |
| Apps with many related entities | Relations and nested writes are built in |
| Long-lived products | Migrations give a history of every schema change |

When an ORM is the **wrong** choice: [When to Use ORM](07_when-to-use-orm.md)

---

### Interview-Ready Summary

- An ORM exists because objects and tables don't match (the **impedance
  mismatch**), and hand-written translation code is repetitive, untyped and
  easy to get wrong.
- The main reasons to use one: **productivity, type safety, parameterized
  queries by default, easy relations, migrations, consistency and
  portability**.
- It's not ORM *versus* SQL. Use the ORM for the ~90% of routine queries and
  raw, parameterized SQL for complex reports and hot paths.
