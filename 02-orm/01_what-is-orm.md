## What Is an ORM

### Definition

An **ORM (Object-Relational Mapper)** is a library that maps **tables in a
relational database** to **classes/objects in your code**. You work with
objects and methods (`prisma.user.findMany()`), and the ORM writes the SQL,
runs it through a database driver, and turns the rows back into objects.

```
Your code (objects)  ⇄  ORM  ⇄  SQL  ⇄  Driver (pg, mysql2)  ⇄  Database (tables)
```

| In your code | In the database |
|---|---|
| Class / model (`User`) | Table (`users`) |
| Object / instance (`user`) | Row |
| Property (`user.email`) | Column (`email`) |
| Reference (`user.posts`) | Foreign key (`posts.author_id → users.id`) |
| Method call (`User.create()`) | SQL statement (`INSERT INTO users …`) |

> An ORM doesn't replace the database or SQL. It **generates** SQL for you
> and does the translation work in both directions.

---

### Without an ORM vs With an ORM

**Task:** get the 10 newest users with an `@example.com` email, together
with their posts.

**Without an ORM: raw SQL + manual mapping (`pg`)**

```js
const { rows: users } = await pool.query(
  `SELECT id, email, name, created_at
     FROM users
    WHERE email LIKE $1
    ORDER BY created_at DESC
    LIMIT 10`,
  ['%@example.com']
);

const { rows: posts } = await pool.query(
  `SELECT id, title, published, author_id FROM posts WHERE author_id = ANY($1)`,
  [users.map((u) => u.id)]
);

// Manual mapping: snake_case → camelCase, and attach children to parents
const result = users.map((u) => ({
  id: u.id,
  email: u.email,
  name: u.name,
  createdAt: u.created_at,
  posts: posts
    .filter((p) => p.author_id === u.id)
    .map((p) => ({ id: p.id, title: p.title, published: p.published })),
}));
```

**With an ORM (Prisma)**

```js
const result = await prisma.user.findMany({
  where: { email: { endsWith: '@example.com' } },
  include: { posts: true },
  orderBy: { createdAt: 'desc' },
  take: 10,
});
// result: fully typed User[] objects, each with posts: Post[]
```

The ORM version is shorter, but more importantly it is **type-checked**,
**parameterized** (safe from SQL injection), and **consistent** across the
whole codebase. Under the hood, Prisma runs a very similar pair of queries.

---

### The Problem ORMs Solve: Object-Relational Impedance Mismatch

Objects and tables think about data differently. That gap is called the
**object-relational impedance mismatch**, and every ORM exists to bridge it.

| Mismatch | Objects (code) | Tables (database) | How ORMs bridge it |
|---|---|---|---|
| **Granularity** | Small nested objects (`user.address.city`) | Flat columns (`address_city`) | Embedded / composite types |
| **Inheritance** | `Admin extends User` | No inheritance | Single-table or table-per-class strategies |
| **Identity** | `a === b` (same object in memory) | Same primary key | Identity map: one object per row per session |
| **Associations** | Object references, in both directions | Foreign keys, one direction; M:N needs a join table | Relations (`@relation`, `hasMany`, `belongsToMany`) |
| **Navigation** | Walk the graph: `user.posts[0].comments` | Set-based JOINs | Eager/lazy loading (and the N+1 trap) |
| **Types & naming** | `Date`, `number`, `camelCase` | `TIMESTAMPTZ`, `BIGINT`, `snake_case` | Type mapping, `@map` / `field` column names |

Without an ORM, you write this translation code yourself, by hand, for
every query.

---

### How an ORM Works (the Short Version)

```
1. Define models   →  model User { id Int @id … }       (schema / classes)
2. Call the API    →  prisma.user.findMany({ … })
3. Build SQL       →  SELECT … FROM "users" WHERE … LIMIT $1
4. Run via driver  →  pooled connection, parameters sent separately
5. Hydrate         →  rows → typed objects, relations attached
6. Track changes   →  (some ORMs) diff objects, flush UPDATEs in a transaction
```

Most ORMs also provide **migrations**: they compare your models with the
database and generate the `ALTER TABLE` scripts. See
[Migrations](../03-orm-concepts/07_migrations.md).

Deep dive: [How ORM Works](05_how-orm-works.md) ·
[ORM Architecture](06_orm-architecture.md)

---

### Two Classic ORM Patterns

These pattern names come from Martin Fowler's *Patterns of Enterprise
Application Architecture* (2002).

**Active Record**: the model object saves itself.

```js
// Sequelize
const user = await User.create({ email: 'ashu@example.com', name: 'Ashu' });
user.name = 'Ashutosh';
await user.save();          // the object knows how to persist itself
```

**Data Mapper**: plain objects; a separate mapper or repository persists them.

```ts
// TypeORM (repository style)
const user = userRepo.create({ email: 'ashu@example.com', name: 'Ashu' });
await userRepo.save(user);  // the repository persists the object
```

| | Active Record | Data Mapper |
|---|---|---|
| Who saves? | The object itself (`user.save()`) | A repository / entity manager |
| Coupling | Model and DB access are mixed together | Domain objects know nothing about the DB |
| Best for | CRUD-heavy apps, fast development | Complex domains, testability |
| Examples | Sequelize, Rails ActiveRecord, Laravel Eloquent | Hibernate, TypeORM repositories, MikroORM, Doctrine, SQLAlchemy |

Two related patterns show up in full ORMs:
- **Unit of Work**: track every changed object, then write all changes in
  one transaction (`em.flush()` in MikroORM, `SaveChanges()` in EF Core).
- **Identity Map**: load each row only once per session, so the same row is
  always the same object.

**Prisma** fits neither pattern exactly. It generates a typed client that
returns **plain objects**, with no model instances and no `save()`.

---

### The ORM Landscape

**Node.js / TypeScript**

| ORM | Style | Notes |
|---|---|---|
| **Prisma** | Schema-first, generated client | `schema.prisma` → typed client; great DX and migrations |
| **Sequelize** | Active Record | The veteran; mature, JS-first |
| **TypeORM** | Decorators; Active Record *or* Data Mapper | Class-based entities |
| **Drizzle** | SQL-like, TypeScript-first | Thin layer close to SQL, very light |
| **MikroORM** | Data Mapper + Unit of Work + Identity Map | Closest to Hibernate in Node |

**Other ecosystems:** Hibernate / JPA (Java), Entity Framework Core (.NET),
Django ORM and SQLAlchemy (Python), ActiveRecord (Ruby on Rails), Eloquent
(PHP / Laravel), GORM (Go).

**The same query in four Node ORMs** (10 newest `@example.com` users, with
their posts):

```js
// Prisma
await prisma.user.findMany({
  where: { email: { endsWith: '@example.com' } },
  include: { posts: true },
  orderBy: { createdAt: 'desc' },
  take: 10,
});

// Sequelize
await User.findAll({
  where: { email: { [Op.endsWith]: '@example.com' } },
  include: [{ model: Post }],
  order: [['createdAt', 'DESC']],
  limit: 10,
});

// TypeORM
await userRepo.find({
  where: { email: Like('%@example.com') },
  relations: { posts: true },
  order: { createdAt: 'DESC' },
  take: 10,
});

// Drizzle (relational queries)
await db.query.users.findMany({
  where: (users, { like }) => like(users.email, '%@example.com'),
  with: { posts: true },
  orderBy: (users, { desc }) => [desc(users.createdAt)],
  limit: 10,
});
```

Hands-on guides: [Prisma](../04-orm-examples/prisma/01_setup.md) ·
[Sequelize](../04-orm-examples/sequelize/01_setup.md)

---

### ORM vs Query Builder vs Raw SQL

```
More control, more code                                   Less code, more abstraction
◄──────────────────────────────────────────────────────────────────────────────────►
Raw driver (pg, mysql2)   →   Query builder (Knex, Kysely)   →   ORM (Prisma, TypeORM)
```

| | Raw SQL (driver) | Query builder | ORM |
|---|---|---|---|
| You write | SQL strings | SQL built with chained functions | Model methods |
| Result | Plain rows | Plain rows (typed with Kysely) | Typed objects with relations |
| Relations | Manual JOINs + mapping | Manual JOINs | `include` / `with` / `relations` |
| Migrations | Separate tool | Usually built in | Built in |
| Performance control | Full | High | Lower; check the generated SQL |
| Best for | Hot paths, complex reports | SQL lovers who want safety | Most app CRUD and business logic |

Most real apps use an ORM for ~95% of queries and **drop to raw SQL** for
the rest, using the ORM's escape hatch:

```js
// Safe: a tagged template, so values are sent as parameters
const rows = await prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}`;

// Dangerous: never pass user input to the *Unsafe* variants
await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE email = '${email}'`); // ❌ SQL injection
```

Full comparison: [Raw SQL vs ORM](../08-comparisons/03_raw-sql-vs-orm.md)

---

### What an ORM Does *Not* Do for You

- **It doesn't replace knowing SQL.** You still need to read the generated
  SQL, understand JOINs and indexes, and debug slow queries.
- **It doesn't design your schema.** Keys, constraints and normalization are
  still your job. See [How a Senior Developer Designs a Database](../12-db-design/01_senior-dev-thought-process.md).
- **It doesn't prevent slow queries.** Lazy loading in a loop causes the
  **N+1 problem**, and `include` everything over-fetches. See
  [N+1 Problem](../09-advanced/01_n-plus-one-problem.md).
- **It doesn't make the database portable for free.** Database-specific
  features (Postgres `JSONB`, partial indexes) often need raw SQL.

---

### Interview-Ready Summary

- An **ORM** maps tables ↔ classes, rows ↔ objects and columns ↔
  properties. It **generates SQL**, runs it through a **driver**, and
  **hydrates** the results into typed objects.
- It exists to bridge the **object-relational impedance mismatch**:
  granularity, inheritance, identity, associations, navigation, and
  types/naming.
- There are two classic patterns. **Active Record**: the object saves
  itself (Sequelize, Rails). **Data Mapper**: a repository saves plain
  objects (Hibernate, MikroORM, TypeORM repositories). Prisma returns plain
  objects from a generated client.
- The spectrum runs **raw driver → query builder → ORM**, trading control
  for productivity. Use the ORM by default and raw SQL (safely
  parameterized) for hot paths.
- An ORM **doesn't replace SQL knowledge or schema design**. Always check
  the generated SQL, and watch for N+1 queries.
