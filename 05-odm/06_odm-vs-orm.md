## ODM vs ORM

### One-Line Difference

- **ORM**: maps objects ↔ **rows in tables** of a **relational** database, by
  generating **SQL**.
- **ODM**: maps objects ↔ **documents in collections** of a **document**
  database, by building **driver commands** (`find`, `insertOne`,
  `aggregate`).

```
ORM:  Object  ⇄  Prisma / Sequelize  ⇄  SQL        ⇄  pg / mysql2  ⇄  PostgreSQL / MySQL
ODM:  Object  ⇄  Mongoose            ⇄  BSON cmds  ⇄  mongodb      ⇄  MongoDB
```

---

### Side-by-Side Comparison

| | ORM | ODM |
|---|---|---|
| Database | Relational (PostgreSQL, MySQL, SQLite) | Document (MongoDB, Couchbase, Firestore) |
| Unit of storage | Row in a table | Document (JSON/BSON) in a collection |
| Query language generated | SQL | Driver commands / aggregation pipelines |
| Schema enforced by | **Database** (DDL) + ORM | **ODM (app)**; DB is schemaless by default |
| Schema changes | Migrations (`ALTER TABLE`) | Usually none; add fields with defaults |
| Nested data | Separate tables + JOINs | **Embedded** objects and arrays |
| Relationships | Foreign keys, JOINs | Embedding or `ObjectId` refs + `populate` / `$lookup` |
| Referential integrity | FK constraints in the DB | Not enforced; handled in app code |
| Transactions | Core feature, everywhere | Supported (replica sets), used less often |
| Impedance mismatch | Large (objects vs tables) | Small (documents already look like objects) |
| Examples (Node) | Prisma, Sequelize, TypeORM, Drizzle | Mongoose, Typegoose, Papr |

---

### The Same Model in Both

**ORM (Prisma + PostgreSQL)**: comments live in their own table.

```prisma
model Post {
  id        Int       @id @default(autoincrement())
  title     String
  authorId  Int
  author    User      @relation(fields: [authorId], references: [id])
  comments  Comment[]
}

model Comment {
  id     Int    @id @default(autoincrement())
  body   String
  postId Int
  post   Post   @relation(fields: [postId], references: [id])
}
```

**ODM (Mongoose + MongoDB)**: comments are embedded in the post.

```js
const postSchema = new Schema({
  title:  { type: String, required: true },
  author: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  comments: [{
    body:      { type: String, required: true },
    author:    { type: Schema.Types.ObjectId, ref: 'User' },
    createdAt: { type: Date, default: Date.now },
  }],
});
```

---

### The Same Queries in Both

**Read a post with its author and comments**

```js
// ORM: 3 tables, joined or batched by the ORM
await prisma.post.findUnique({
  where: { id: 1 },
  include: { author: true, comments: true },
});

// ODM: comments come with the post; author via populate
await Post.findById(id).populate('author', 'name');
```

**Add a comment**

```js
// ORM: INSERT into the comments table
await prisma.comment.create({ data: { postId: 1, body: 'Nice!' } });

// ODM: push into the embedded array (single atomic update)
await Post.updateOne({ _id: id }, { $push: { comments: { body: 'Nice!', author: userId } } });
```

**Count posts per author**

```js
// ORM: GROUP BY
await prisma.post.groupBy({ by: ['authorId'], _count: { _all: true } });

// ODM: aggregation pipeline
await Post.aggregate([{ $group: { _id: '$author', posts: { $sum: 1 } } }]);
```

---

### Concept Mapping

| Relational / ORM | Document / ODM |
|---|---|
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |
| Primary key (`id`) | `_id` (usually `ObjectId`) |
| Foreign key | `ObjectId` reference |
| JOIN | Embedding, `populate()`, `$lookup` |
| `GROUP BY`, window functions | Aggregation pipeline (`$group`, `$setWindowFields`) |
| Migration | Backfill script / `schemaVersion` |
| Model / entity | Schema + model |
| `include` (Prisma) | `populate()` (Mongoose) |

---

### Where They're the Same

Both give you:

- A **model API** instead of raw queries.
- **Validation**, **defaults** and **hooks/middleware**.
- **Relations** you can load eagerly (`include` / `populate`).
- The same traps: **N+1 queries**, **over-fetching**, hidden queries.
- An **escape hatch**: raw SQL for ORMs, the driver / aggregation for ODMs.

---

### Where They Differ Most

| Topic | ORM world | ODM world |
|---|---|---|
| **Data modelling** | Normalize first, then join | Model around **read patterns**; embed what's read together |
| **Integrity** | The DB guarantees FKs, uniqueness, types | The app guarantees most rules (plus unique indexes and `$jsonSchema`) |
| **Schema evolution** | Every change is a migration | Most changes need nothing; old docs keep old shapes |
| **Scaling** | Mostly vertical; sharding is hard | Horizontal sharding is built in |
| **Complex reporting** | SQL is excellent | Aggregation pipelines are powerful but more verbose |

---

### Hybrid Tools

Some tools span both worlds:

- **Prisma** has a MongoDB connector alongside its SQL connectors.
- **TypeORM** and **MikroORM** support MongoDB in addition to SQL.
- PostgreSQL **`JSONB`** lets an ORM-backed app store document-style data.

Their MongoDB support is usually less complete than a dedicated ODM like
Mongoose, and it varies by version.

Full comparison: [ORM vs ODM](../08-comparisons/01_orm-vs-odm.md) ·
[Mongoose vs Prisma](../08-comparisons/05_mongoose-vs-prisma.md)

---

### Interview-Ready Summary

- **ORM** = objects ↔ relational rows via **SQL**; **ODM** = objects ↔
  documents via **driver commands**.
- ORM schemas are enforced by the **database** and changed with
  **migrations**; ODM schemas are enforced by the **app**, and most changes
  need no migration.
- Relations: ORMs use **FKs + JOINs**; ODMs use **embedding** or **refs +
  `populate` / `$lookup`**, with no DB-enforced referential integrity.
- The impedance mismatch is much **smaller** for ODMs, since documents already
  look like objects.
- Both share model APIs, validation, hooks, eager loading and the same
  pitfalls (N+1, over-fetching).
