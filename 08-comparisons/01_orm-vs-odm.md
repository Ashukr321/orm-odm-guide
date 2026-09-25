## ORM vs ODM

### The Short Answer

- An **ORM** (Prisma, Sequelize, TypeORM, Drizzle) maps objects to **rows in
  relational tables** and generates **SQL**.
- An **ODM** (Mongoose, Typegoose) maps objects to **documents in a document
  database** and builds **MongoDB commands**.

You rarely choose between them directly. You choose a **database**, and
the database decides whether you need an ORM or an ODM. This page compares
how the two feel **in daily work**, feature by feature.

Concepts first: [What Is ORM](../02-orm/01_what-is-orm.md) ·
[What Is ODM](../05-odm/01_what-is-odm.md) ·
[ODM vs ORM (concepts)](../05-odm/06_odm-vs-orm.md)

---

### At a Glance

| | ORM | ODM |
|---|---|---|
| Database | PostgreSQL, MySQL, SQLite, SQL Server | MongoDB (mainly) |
| Maps to | Tables, rows, columns | Collections, documents, fields |
| Generates | SQL | Driver commands, aggregation pipelines |
| Who enforces the schema | **The database** (DDL) plus the ORM | **The ODM (app)**; MongoDB is flexible by default |
| Schema changes | **Migrations** | Usually none; backfills when needed |
| Relations | Foreign keys + JOINs | Embedding, or refs + `populate()` / `$lookup` |
| Integrity | FK, unique, check constraints in the DB | Unique indexes; the rest is app code |
| Nested data | More tables | Embedded objects and arrays |
| Transactions | Everywhere, cheap, common | Supported (replica sets), used less |
| Impedance mismatch | Large | Small; documents already look like objects |

---

### Feature by Feature, Side by Side

The same blog, `Post` with an author and comments, in Prisma (ORM) and
Mongoose (ODM).

#### 1. Defining the Model

```prisma
// ORM: Prisma + PostgreSQL (comments are their own table)
model Post {
  id        Int       @id @default(autoincrement())
  title     String    @db.VarChar(200)
  authorId  Int
  author    User      @relation(fields: [authorId], references: [id])
  comments  Comment[]
}
model Comment {
  id     Int    @id @default(autoincrement())
  body   String
  postId Int
  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
}
```

```js
// ODM: Mongoose + MongoDB (comments embedded in the post)
const postSchema = new Schema({
  title:  { type: String, required: true, maxLength: 200 },
  author: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  comments: [{ body: { type: String, required: true }, author: { type: Schema.Types.ObjectId, ref: 'User' } }],
});
```

#### 2. Validation

| | ORM (Prisma) | ODM (Mongoose) |
|---|---|---|
| Types / NOT NULL | Enforced by the **database** | Enforced by **Mongoose** |
| Max length | `@db.VarChar(200)` (DB) | `maxLength: 200` (app) |
| Business rules (email format, ranges) | App layer (zod) or DB `CHECK` | Built-in and custom validators |
| Someone writes with raw SQL / shell | Still protected by DB constraints | **Not protected** unless `$jsonSchema` is set |

#### 3. Reading Related Data

```js
// ORM: 2–3 queries or a JOIN, decided by the ORM
await prisma.post.findUnique({ where: { id }, include: { author: true, comments: true } });

// ODM: comments arrive with the post; author via one extra query
await Post.findById(id).populate('author', 'name');
```

#### 4. Adding a Child

```js
// ORM: INSERT into another table
await prisma.comment.create({ data: { postId: id, body: 'Nice!' } });

// ODM: atomic $push into the same document
await Post.updateOne({ _id: id }, { $push: { comments: { body: 'Nice!', author: userId } } });
```

#### 5. Multi-Record Consistency

```js
// ORM: transactions are the everyday tool
await prisma.$transaction(async (tx) => {
  await tx.order.create({ data: { userId, total } });
  await tx.user.update({ where: { id: userId }, data: { balance: { decrement: total } } });
});

// ODM: prefer a single-document atomic update; use a transaction when you must
await mongoose.connection.transaction(async (session) => {
  await Order.create([{ user: userId, total }], { session });
  await User.updateOne({ _id: userId }, { $inc: { balance: -total } }, { session });
});
```

#### 6. Changing the Schema

```bash
# ORM: every change is a migration
npx prisma migrate dev --name add_post_subtitle
```

```js
// ODM: add the field with a default. Old documents keep working.
subtitle: { type: String, default: '' },
```

#### 7. Hooks and Custom Logic

| | ORM | ODM |
|---|---|---|
| Lifecycle hooks | Sequelize/TypeORM hooks; Prisma client extensions | Rich `pre`/`post` document, query, aggregate and model middleware |
| Computed fields | Extensions / getters | Virtuals |
| Custom model methods | Extensions (Prisma), class methods (Sequelize) | `methods`, `statics`, query helpers |

#### 8. Escape Hatches

| | ORM | ODM |
|---|---|---|
| Raw access | `prisma.$queryRaw`, `sequelize.query` | `Model.collection`, `mongoose.connection.db` |
| Complex analytics | SQL (CTEs, window functions) | Aggregation pipeline |
| Plain objects for speed | Prisma returns plain objects; Sequelize `raw: true` | `.lean()` |

---

### Tool Matrix (Node.js)

| Tool | Kind | Style | Type safety | Migrations |
|---|---|---|---|---|
| **Prisma** | ORM (+ MongoDB connector) | Schema-first, generated client | Excellent | Built in |
| **Sequelize** | ORM | Active Record | Fair (TS added later) | CLI, hand-written |
| **TypeORM** | ORM (+ limited MongoDB) | Decorators, AR or Data Mapper | Good | Generated |
| **Drizzle** | ORM / SQL-like builder | Code-first, SQL-shaped | Excellent | drizzle-kit |
| **MikroORM** | ORM (+ MongoDB) | Data Mapper, Unit of Work | Good | Generated |
| **Mongoose** | ODM | Schema + model classes | Good (inferred types) | None needed |
| **Typegoose** | ODM (on Mongoose) | TS classes + decorators | Good | None needed |

---

### Scenario Verdicts

| Scenario | Pick | Why |
|---|---|---|
| E-commerce orders, payments, inventory | **ORM + PostgreSQL** | Integrity, transactions, reporting |
| CMS with varied content blocks | **ODM + MongoDB** | Nested, varying shapes read as a whole |
| SaaS with users, teams, roles, billing | **ORM** | Many relations, many-to-many, constraints |
| Product catalog with category-specific attributes | **ODM**, or ORM + `JSONB` | Flexible attributes per item |
| Activity feed / event log at high volume | **ODM** (or a specialized store) | Append-heavy, flexible payloads, sharding |
| Analytics dashboards over business data | **ORM + SQL** (raw for reports) | SQL is the best analytics language |
| Rapid prototype, schema changes daily | Either: **ODM** or **Prisma `db push`** | Both avoid migration friction early |

---

### Moving Between Them

**SQL → MongoDB** (ORM → ODM) pitfalls:

- Don't copy tables 1:1 into collections. Redesign around **read patterns**
  and embed.
- Constraints you relied on (FKs, `CHECK`) must move into app code or
  `$jsonSchema`.
- Reports written in SQL become aggregation pipelines.

**MongoDB → SQL** (ODM → ORM) pitfalls:

- Embedded arrays become child tables; `Mixed` fields become `JSONB` or new
  tables.
- Documents with drifted shapes need cleaning before strict columns will
  accept them.
- `ObjectId`s need a mapping to new primary keys (or keep them as text/UUID).

---

### Interview Questions

**"ORM or ODM for a new project?"**
Decide the database first, based on data shape, integrity needs and query
patterns. Relational data with transactions and reporting → PostgreSQL +
ORM. Nested, flexible, document-shaped data → MongoDB + ODM.

**"Why doesn't an ODM need migrations?"**
MongoDB doesn't enforce a schema, so old documents stay valid. The ODM
applies defaults on read, and bigger reshapes are handled by backfill
scripts.

**"Is `populate()` the same as a JOIN?"**
No. It's an extra `$in` query per path, merged in Node.js. `$lookup` is the
server-side join.

**"Which is safer for data integrity?"**
The ORM, because constraints live in the database and apply to every
writer.

---

### Interview-Ready Summary

- ORM ↔ relational (SQL, DB-enforced schema, migrations, FKs + JOINs);
  ODM ↔ documents (driver commands, app-enforced schema, embedding + refs).
- Daily differences: validation lives in the **DB vs the app**; children are
  **rows vs embedded arrays**; consistency is **transactions vs
  single-document atomicity**; schema changes are **migrations vs defaults
  and backfills**.
- Pick the **database** by data shape and integrity needs; the ORM/ODM
  follows.
- When switching, redesign the model rather than copying it.
