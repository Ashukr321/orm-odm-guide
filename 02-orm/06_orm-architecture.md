## ORM Architecture

### The Layers

Internally, a full-featured ORM is a stack of components. Not every ORM
has every layer, but the shape is the same.

```
┌─────────────────────────────────────────────────────────────┐
│                     Your application                         │
├─────────────────────────────────────────────────────────────┤
│  Public API        Models · Repositories · Query builder     │
├─────────────────────────────────────────────────────────────┤
│  Session / Entity Manager                                    │
│    ├── Identity Map      (one object per row)                │
│    └── Unit of Work      (tracks new / dirty / deleted)      │
├─────────────────────────────────────────────────────────────┤
│  Metadata             Entity mappings, relations, types      │
├─────────────────────────────────────────────────────────────┤
│  Query engine         AST → SQL via Dialect                  │
├─────────────────────────────────────────────────────────────┤
│  Hydrator             Rows → objects, type conversion        │
├─────────────────────────────────────────────────────────────┤
│  Connection layer     Pool · Transactions · Driver adapter   │
├─────────────────────────────────────────────────────────────┤
│  Driver               pg · mysql2 · better-sqlite3 · tedious │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                          Database
     (Migration engine sits beside the stack, using Metadata + Connection)
```

---

### Component by Component

| Component | Responsibility | Examples |
|---|---|---|
| **Public API** | What you call | `prisma.user.findMany`, `User.findAll`, `repo.find` |
| **Metadata** | Knows every entity, column, type and relation | Prisma DMMF, TypeORM `EntityMetadata`, Sequelize model attributes |
| **Query builder / engine** | Builds a query tree and turns it into SQL | TypeORM `SelectQueryBuilder`, Prisma query engine |
| **Dialect** | Database-specific SQL (quoting, placeholders, types) | Sequelize dialects, Knex clients, Prisma providers |
| **Session / Entity Manager** | Scope of work; owns the identity map and unit of work | MikroORM `EntityManager`, Hibernate `Session`, EF `DbContext` |
| **Identity Map** | Ensures one object per row within a session | MikroORM, Hibernate, EF Core |
| **Unit of Work** | Tracks changes; writes them in one transaction on flush | `em.flush()`, `SaveChanges()` |
| **Hydrator** | Converts rows to entities/objects | All ORMs |
| **Connection layer** | Pooling, transactions, retries | Built-in pool or the driver's pool |
| **Migration engine** | Diffs models vs database, generates and runs migrations | `prisma migrate`, `sequelize-cli`, TypeORM migrations |

---

### Architectural Patterns

These are the building blocks from Martin Fowler's *Patterns of
Enterprise Application Architecture*. Real ORMs combine several.

#### Active Record

The model class maps to a table **and** carries the persistence methods.

```js
// Sequelize
const user = await User.findByPk(1);
user.name = 'Ashu';
await user.save();
await user.destroy();
```

```
┌──────────────────────────┐
│ User                     │
│  - id, email, name       │  ← data
│  + save() / destroy()    │  ← persistence
│  + static findAll()      │
└──────────────────────────┘
```

#### Data Mapper

Domain objects are plain classes. A separate **mapper** moves data between
them and the database.

```ts
// TypeORM (Data Mapper style)
const repo = dataSource.getRepository(User);
const user = await repo.findOneBy({ id: 1 });
user.name = 'Ashu';
await repo.save(user);
```

```
┌──────────────┐        ┌──────────────────┐        ┌──────────┐
│ User (plain) │ ◄────► │ UserRepository   │ ◄────► │ Database │
│ id, name     │        │ find/save/remove │        └──────────┘
└──────────────┘        └──────────────────┘
```

#### Repository

A collection-like interface for one aggregate: `find`, `save`, `remove`.
It hides the query details from the service layer.

```ts
class UserRepository {
  constructor(private prisma: PrismaClient) {}
  findByEmail(email: string) {
    return this.prisma.user.findUnique({ where: { email } });
  }
}
```

See [Repositories](../03-orm-concepts/10_repositories.md).

#### Unit of Work

Keeps a list of new, changed and deleted objects, then writes them all in
one transaction.

```ts
// MikroORM
const user = await em.findOneOrFail(User, 1);
user.name = 'Ashu';                  // marked dirty
em.remove(oldPost);                  // marked deleted
em.persist(new Post({ title: 'Hi' })); // marked new
await em.flush();
// BEGIN
// UPDATE users SET name = … WHERE id = 1
// DELETE FROM posts WHERE id = …
// INSERT INTO posts …
// COMMIT
```

#### Identity Map

Inside one session, the same row always returns the same object.

```ts
const a = await em.findOne(User, 1);
const b = await em.findOne(User, 1);   // no second query
a === b;                               // true
```

This prevents two conflicting in-memory copies of the same row.

#### Lazy Load (Proxy)

A relation is a placeholder that queries the database the first time you
touch it.

```ts
// TypeORM lazy relation
@OneToMany(() => Post, (p) => p.author) posts: Promise<Post[]>;

const posts = await user.posts;   // query runs here
```

Convenient, and the main cause of **N+1** queries.

---

### How Popular ORMs Are Built

| ORM | Core pattern | Identity Map | Unit of Work | Query layer |
|---|---|---|---|---|
| **Prisma** | Generated client, plain objects | No | No | Query engine (Rust historically; TypeScript engine in newer versions) |
| **Sequelize** | Active Record | No | No | Built-in query generator + dialects |
| **TypeORM** | Active Record **or** Data Mapper | No | Partial (`save` cascades) | `QueryBuilder` |
| **Drizzle** | SQL-like builder + relational queries | No | No | Thin TS layer over the driver |
| **MikroORM** | Data Mapper | Yes | Yes | Knex-based builder |
| **Hibernate / EF Core** | Data Mapper | Yes | Yes | HQL/JPQL, LINQ |

---

### Prisma's Architecture (Special Case)

Prisma is schema-first and code-generated rather than class-based.

```
schema.prisma
     │  prisma generate
     ▼
Generated client (TypeScript types + query methods)
     │  prisma.user.findMany(…)
     ▼
Query engine  ──►  SQL  ──►  driver / driver adapter  ──►  database
     │
     ▼
Plain JS objects (no instances, no save(), no change tracking)
```

`prisma migrate` reads the same schema to generate migration SQL, so the
schema file is the single source of truth for types, queries and the
database structure.

---

### Where the ORM Fits in an Application

```
Controller / Route       ← HTTP, validation
      │
Service                  ← business rules, transactions
      │
Repository (optional)    ← data-access methods, hides the ORM
      │
ORM client / Entity Mgr  ← Prisma, Sequelize, TypeORM
      │
Driver + Pool            ← pg, mysql2
      │
Database
```

Keep ORM calls out of controllers. Services call repositories (or the ORM
client directly in small apps), which keeps business logic testable and
makes it easier to swap or bypass the ORM.

---

### Interview-Ready Summary

- An ORM is layered: **API → session (identity map + unit of work) →
  metadata → query engine + dialect → hydrator → connection pool → driver**,
  with a **migration engine** alongside.
- Key patterns: **Active Record** (object saves itself), **Data Mapper**
  (a mapper/repository saves plain objects), **Repository**, **Unit of
  Work** (batch changes into one transaction), **Identity Map** (one object
  per row per session), **Lazy Load**.
- Sequelize is Active Record; MikroORM, Hibernate and EF Core are Data
  Mapper with Unit of Work; TypeORM supports both; Prisma is a generated
  client returning plain objects.
- In an app: **controller → service → repository → ORM → driver → DB**.
