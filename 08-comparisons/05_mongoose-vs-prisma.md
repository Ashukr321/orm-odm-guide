## Mongoose vs Prisma

### Two Different Questions

"Mongoose vs Prisma" usually means one of two things:

1. **On MongoDB**: Mongoose, or Prisma's **MongoDB connector**?
2. **For a new project**: Mongoose + MongoDB, or Prisma + a **SQL**
   database?

This page answers both.

> **Version note:** Prisma's MongoDB connector is available in Prisma 6.
> Check the Prisma 7 release notes for its current MongoDB status before
> planning a new project around it.

---

### At a Glance (Both on MongoDB)

| | Mongoose | Prisma (MongoDB connector) |
|---|---|---|
| Type | ODM built for MongoDB | ORM with a MongoDB connector |
| Schema | JS `Schema` objects | `schema.prisma` |
| Types | Inferred from the schema (good) | Generated client (excellent) |
| Embedded documents | Subdocuments (full schema, hooks, `_id`) | **Composite types** (`type Comment { … }`) |
| References | `ref` + `populate()` | `@relation` + `include` |
| Validation | Built-in and custom validators | Types only; use zod |
| Middleware | Rich pre/post hooks | Client extensions |
| Aggregation | `Model.aggregate()`, builder | `aggregateRaw()`, `findRaw()` |
| Change tracking | Documents + `save()` | None; explicit `update` calls |
| Migrations | Not needed | None; `db push` syncs indexes |
| Replica set | Needed only for transactions | **Required** |
| MongoDB feature coverage | Nearly everything | Core CRUD and relations; the rest via raw |

---

### The Same Model

```js
// Mongoose
const postSchema = new Schema({
  title:  { type: String, required: true },
  author: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  tags:   [{ type: Schema.Types.ObjectId, ref: 'Tag' }],
  comments: [{
    body:      { type: String, required: true },
    author:    { type: Schema.Types.ObjectId, ref: 'User' },
    createdAt: { type: Date, default: Date.now },
  }],
}, { timestamps: true });
```

```prisma
// Prisma (MongoDB)
datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

model Post {
  id        String    @id @default(auto()) @map("_id") @db.ObjectId
  title     String
  authorId  String    @db.ObjectId
  author    User      @relation(fields: [authorId], references: [id])
  tagIds    String[]  @db.ObjectId
  tags      Tag[]     @relation(fields: [tagIds], references: [id])
  comments  Comment[]                      // embedded composite type
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

type Comment {
  body      String
  authorId  String   @db.ObjectId
  createdAt DateTime @default(now())
}

model Tag {
  id      String   @id @default(auto()) @map("_id") @db.ObjectId
  name    String   @unique
  postIds String[] @db.ObjectId
  posts   Post[]   @relation(fields: [postIds], references: [id])
}
```

Note that many-to-many in Prisma's MongoDB connector keeps **id arrays on
both sides** (`tagIds` and `postIds`), and Prisma keeps them in sync.

---

### The Same Queries

**Feed with authors:**

```js
// Mongoose
await Post.find({ published: true }).sort({ createdAt: -1 }).limit(10).populate('author', 'name').lean();

// Prisma
await prisma.post.findMany({
  where: { published: true }, orderBy: { createdAt: 'desc' }, take: 10,
  include: { author: { select: { name: true } } },
});
```

**Add an embedded comment:**

```js
// Mongoose: atomic $push
await Post.updateOne({ _id: id }, { $push: { comments: { body, author: userId } } });

// Prisma: push on a composite list
await prisma.post.update({ where: { id }, data: { comments: { push: { body, authorId: userId } } } });
```

**Aggregation:**

```js
// Mongoose
await Post.aggregate([{ $unwind: '$tags' }, { $group: { _id: '$tags', n: { $sum: 1 } } }]);

// Prisma: raw pipeline; results are raw extended JSON ({ $oid: … })
await prisma.post.aggregateRaw({ pipeline: [{ $unwind: '$tagIds' }, { $group: { _id: '$tagIds', n: { $sum: 1 } } }] });
```

**Hooks (hash a password):**

```js
// Mongoose
userSchema.pre('save', async function () { if (this.isModified('password')) this.password = await hash(this.password); });

// Prisma: a query extension
const xprisma = prisma.$extends({
  query: { user: { async create({ args, query }) {
    args.data.password = await hash(args.data.password);
    return query(args);
  } } },
});
```

---

### Where Each Wins (on MongoDB)

| Mongoose wins | Prisma wins |
|---|---|
| Full MongoDB feature coverage (discriminators, change streams, time series, collation…) | **Type safety**: result types follow the query shape |
| Built-in validation and rich middleware | Same API as Prisma on SQL, so a team using both learns one tool |
| Aggregation builder with Mongoose models | Readable schema file, Prisma Studio |
| Subdocuments with their own hooks and `_id`s | Nested writes and relation filters (`some`, `every`) |
| Huge ecosystem (plugins, tutorials, answers) | No hydrated documents; plain objects |
| Works without a replica set (except transactions) | |

---

### The Other Question: Mongoose + MongoDB, or Prisma + PostgreSQL?

For a **new** project, this is usually the real decision, and it's a
**database** decision first (see [SQL vs NoSQL](02_sql-vs-nosql.md)).

| Your data is… | Pick |
|---|---|
| Relational: users, teams, orders, payments, many-to-many | **Prisma + PostgreSQL** |
| Documents read as a whole: CMS pages, catalogs, configs | **Mongoose + MongoDB** |
| Mostly relational with some flexible fields | **Prisma + PostgreSQL** with `Json` fields |
| Heavy on aggregations/reports | **PostgreSQL** (SQL), with raw queries where needed |
| Needing horizontal write scaling from day one | **MongoDB** (sharding) |

---

### Scorecard

| Criterion | Mongoose + MongoDB | Prisma + MongoDB | Prisma + PostgreSQL |
|---|---|---|---|
| Type safety | ●●●○ | ●●●● | ●●●● |
| MongoDB feature coverage | ●●●● | ●●○○ | n/a |
| Data integrity (DB-enforced) | ●○○○ | ●○○○ | ●●●● |
| Schema flexibility | ●●●● | ●●●○ | ●●○○ (`Json` helps) |
| Reporting / analytics | ●●○○ | ●○○○ | ●●●● |
| Migrations needed | None | None | Yes (generated) |
| Ecosystem / community | ●●●● | ●●○○ | ●●●● |

(A rough, opinionated guide for typical web apps, not a benchmark.)

---

### Interview-Ready Summary

- On MongoDB, **Mongoose** is the full-featured, MongoDB-native ODM
  (validation, middleware, subdocuments, aggregation). **Prisma's MongoDB
  connector** offers better types and one API across databases, but covers
  fewer MongoDB features and requires a **replica set**.
- Prisma MongoDB specifics: `@db.ObjectId` ids, **composite types** for
  embedded data, M:N with id arrays on both sides, `aggregateRaw` for
  pipelines, `db push` instead of migrations.
- For a new project, decide **MongoDB vs PostgreSQL** first. Relational data
  → Prisma + PostgreSQL; document-shaped data → Mongoose + MongoDB.
- Check Prisma's current MongoDB support for the version you plan to use.
