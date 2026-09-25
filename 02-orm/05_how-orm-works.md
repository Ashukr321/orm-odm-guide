## How an ORM Works

### The Big Picture

Every ORM call goes through the same pipeline, from a method call in your
code to rows on disk and back.

```
 Your code                prisma.user.findMany({ where: { active: true } })
    │
    ▼
 1. Metadata         ──►  Which table? Which columns? Which relations?
    │
    ▼
 2. Query building   ──►  Abstract query (AST): SELECT from User where active = ?
    │
    ▼
 3. SQL generation   ──►  SELECT "id","email" FROM "User" WHERE "active" = $1
    │                     (dialect-specific: Postgres $1, MySQL ?)
    ▼
 4. Execution        ──►  Pool gives a connection → driver sends SQL + [true]
    │
    ▼
 5. Hydration        ──►  rows → objects, types converted, relations attached
    │
    ▼
 6. Change tracking  ──►  (some ORMs) remember original values for later UPDATEs
    │
    ▼
 Your code receives   [{ id: 1, email: 'a@x.com' }, …]
```

---

### Step 1: Metadata (the Mapping)

Before it can build any query, the ORM needs to know how your models map to
tables. There are three common ways to describe that mapping.

**Schema file (Prisma):**

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  createdAt DateTime @default(now()) @map("created_at")
  posts     Post[]
  @@map("users")
}
```

**Decorators (TypeORM):**

```ts
@Entity('users')
export class User {
  @PrimaryGeneratedColumn() id: number;
  @Column({ unique: true }) email: string;
  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
  @OneToMany(() => Post, (post) => post.author) posts: Post[];
}
```

**Code definitions (Sequelize):**

```js
const User = sequelize.define('User', {
  email: { type: DataTypes.STRING, unique: true },
}, { tableName: 'users', underscored: true });
User.hasMany(Post, { foreignKey: 'authorId' });
```

From any of these, the ORM builds an in-memory **metadata model**:

| Metadata | Example |
|---|---|
| Table name | `User` → `users` |
| Column names & types | `createdAt: Date` → `created_at TIMESTAMP` |
| Primary key | `id`, auto-increment |
| Relations | `User.posts` → `posts.author_id` (one-to-many) |
| Constraints & defaults | `email` unique, `createdAt` default `now()` |

---

### Step 2: Query Building

Your method call is turned into an internal, database-neutral
representation of the query (often an **AST**, an abstract syntax tree).

```js
prisma.user.findMany({
  where: { email: { endsWith: '@example.com' }, active: true },
  orderBy: { createdAt: 'desc' },
  take: 10,
});
```

```
Select
├── from:    users
├── columns: id, email, created_at, …
├── where:   AND(email LIKE ?, active = ?)
├── orderBy: created_at DESC
└── limit:   10
```

Nothing database-specific has happened yet.

---

### Step 3: SQL Generation (Dialects)

A **dialect** turns the neutral query into SQL for a specific database.

| Feature | PostgreSQL | MySQL | SQLite |
|---|---|---|---|
| Placeholder | `$1, $2` | `?` | `?` |
| Identifier quoting | `"users"` | `` `users` `` | `"users"` |
| Return inserted row | `RETURNING *` | Separate `SELECT` | `RETURNING *` (3.35+) |
| Limit | `LIMIT 10` | `LIMIT 10` | `LIMIT 10` |

```sql
-- PostgreSQL
SELECT "id", "email", "created_at" FROM "users"
WHERE "email" LIKE $1 AND "active" = $2
ORDER BY "created_at" DESC LIMIT $3;
-- params: ['%@example.com', true, 10]
```

Values **never** go into the SQL text. They are sent as parameters, which
is why ORMs are safe from SQL injection by default.

---

### Step 4: Execution

1. Borrow a connection from the **connection pool**.
2. Hand the SQL and parameters to the **driver** (`pg`, `mysql2`, …).
3. The driver sends them over the wire protocol and waits for rows.
4. Return the connection to the pool.

```
ORM  →  Pool (e.g. 10 connections)  →  Driver (pg)  →  TCP  →  PostgreSQL
```

If you are inside a transaction, every query uses the **same**
connection until `COMMIT` or `ROLLBACK`.

See [Database Drivers](../01-database-foundation/database-drivers.md) ·
[Connection Pooling](../03-orm-concepts/09_connection-pooling.md)

---

### Step 5: Hydration (Rows → Objects)

The driver returns flat rows. The ORM turns them into the objects your code
expects.

```
Raw rows from a JOIN                      Hydrated result
┌────┬─────────┬─────────┬──────────┐     [
│ id │ email   │ post_id │ title    │       { id: 1, email: 'a@x.com',
├────┼─────────┼─────────┼──────────┤         posts: [
│ 1  │ a@x.com │ 10      │ Hello    │   →       { id: 10, title: 'Hello' },
│ 1  │ a@x.com │ 11      │ ORMs 101 │           { id: 11, title: 'ORMs 101' } ] },
│ 2  │ b@x.com │ NULL    │ NULL     │       { id: 2, email: 'b@x.com', posts: [] }
└────┴─────────┴─────────┴──────────┘     ]
```

Hydration does:

- **Renaming**: `created_at` → `createdAt`.
- **Type conversion**: `TIMESTAMPTZ` → `Date`, `BIGINT` → `bigint`, `NUMERIC` → `Decimal`.
- **De-duplication**: collapse repeated parent rows from a JOIN.
- **Relation assembly**: attach children arrays and parent objects.
- **Instances**: Active Record ORMs wrap each row in a model instance with
  methods like `save()`. Prisma returns plain objects.

---

### Step 6: Change Tracking and Writing Back

How updates work depends on the ORM's pattern.

**Explicit update (Prisma):** you say exactly what changes.

```js
await prisma.user.update({ where: { id: 1 }, data: { name: 'Ashu' } });
// UPDATE "users" SET "name" = $1 WHERE "id" = $2
```

**Dirty checking (Sequelize, TypeORM, Hibernate, MikroORM):** the ORM
remembers the original values and writes only what changed.

```js
const user = await User.findByPk(1);   // snapshot: { name: 'A', email: 'a@x.com' }
user.name = 'Ashu';
user.changed();                        // ['name']
await user.save();
// UPDATE "users" SET "name" = $1, "updated_at" = $2 WHERE "id" = $3
```

**Unit of Work (MikroORM, EF Core, Hibernate):** collect all changes, then
flush them in one transaction.

```ts
const user = await em.findOneOrFail(User, 1);
user.name = 'Ashu';
em.persist(new Post({ title: 'Hi', author: user }));
await em.flush();
// BEGIN; UPDATE users …; INSERT INTO posts …; COMMIT;
```

---

### Loading Relations: Two Strategies

| Strategy | SQL | Used by |
|---|---|---|
| **JOIN** | One query; parent columns repeated on each child row | Sequelize `include`, TypeORM `leftJoinAndSelect` |
| **Separate queries** | One query per relation level: `WHERE author_id IN (…)` | Prisma `include` (default), MikroORM, DataLoader |

```sql
-- Separate-query strategy for users + posts
SELECT * FROM users WHERE active = true;
SELECT * FROM posts WHERE author_id IN (1, 2, 3, …);
```

Both avoid N+1. JOINs use fewer round trips; separate queries avoid huge
duplicated result sets. See
[Eager vs Lazy Loading](../03-orm-concepts/06_eager-vs-lazy-loading.md).

---

### A Full Trace

```js
const prisma = new PrismaClient({ log: ['query'] });

await prisma.user.findMany({
  where: { active: true },
  include: { posts: { where: { published: true } } },
  take: 2,
});
```

```
prisma:query SELECT "id","email","active" FROM "users" WHERE "active" = $1 LIMIT $2 OFFSET $3
prisma:query SELECT "id","title","published","author_id" FROM "posts"
             WHERE "published" = $1 AND "author_id" IN ($2,$3)
```

```js
[
  { id: 1, email: 'a@x.com', active: true, posts: [{ id: 10, title: 'Hello', … }] },
  { id: 2, email: 'b@x.com', active: true, posts: [] },
]
```

---

### Migrations: Keeping Tables in Sync with Models

Separate from the query pipeline, most ORMs also manage the schema:

```
Models (schema.prisma)  ─┐
                         ├─►  diff  ─►  migration.sql  ─►  apply  ─►  database
Current DB / history    ─┘
```

See [Migrations](../03-orm-concepts/07_migrations.md).

---

### Interview-Ready Summary

- An ORM call flows through: **metadata → query building → SQL generation
  (dialect) → execution (pool + driver) → hydration → change tracking**.
- Metadata comes from a **schema file** (Prisma), **decorators** (TypeORM)
  or **code definitions** (Sequelize).
- Values are always sent as **bound parameters**, never inlined.
- **Hydration** renames columns, converts types, de-duplicates JOIN rows and
  assembles relations.
- Writes happen by **explicit update** (Prisma), **dirty checking**
  (Sequelize/TypeORM) or **Unit of Work flush** (MikroORM/Hibernate/EF Core).
- Relations load by **JOIN** or **separate `IN (…)` queries**; both avoid N+1.
