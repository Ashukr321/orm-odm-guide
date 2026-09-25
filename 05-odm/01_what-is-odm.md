## What Is an ODM

### Definition

An **ODM (Object-Document Mapper)** is a library that maps **documents in a
document database** (like MongoDB) to **objects in your code**. It adds the
things the raw driver doesn't give you: **schemas, validation, defaults,
type casting, relationships (references), middleware and a model API**.

```
Your code (objects)  ⇄  ODM (Mongoose)  ⇄  Driver (mongodb)  ⇄  MongoDB (BSON documents)
```

| In your code | In the database |
|---|---|
| Model (`User`) | Collection (`users`) |
| Document / instance (`user`) | Document (a BSON object) |
| Property (`user.email`) | Field (`email`) |
| Nested object / array (`user.address`, `post.comments`) | Embedded document / array |
| Reference (`post.author`) | `ObjectId` stored in a field (`author: ObjectId("…")`) |
| Method call (`User.create()`) | Driver command (`insertOne`) |

> The ODM is to a document database what an ORM is to a relational database.
> It doesn't write SQL. It builds **driver commands** (`find`, `insertOne`,
> `updateOne`, `aggregate`) and turns BSON documents into objects.

Background: [Document Database](../01-database-foundation/document-database.md) ·
[What Is ORM](../02-orm/01_what-is-orm.md)

---

### Without an ODM vs With an ODM

**Task:** save a user safely and read the 10 newest posts with their
authors.

**Without an ODM: the raw `mongodb` driver**

```js
const { MongoClient, ObjectId } = require('mongodb');
const client = new MongoClient(process.env.MONGODB_URI);
const db = client.db('blog');

// No schema: nothing stops a typo, a missing field or a wrong type
await db.collection('users').insertOne({
  emial: 'ASHU@EXAMPLE.COM',   // typo, not lower-cased, no validation
  age: '27',                   // string instead of number
});

// "Joins" by hand
const posts = await db.collection('posts').find().sort({ createdAt: -1 }).limit(10).toArray();
const authorIds = [...new Set(posts.map((p) => p.author.toString()))].map((id) => new ObjectId(id));
const authors = await db.collection('users').find({ _id: { $in: authorIds } }).toArray();
const byId = new Map(authors.map((a) => [a._id.toString(), a]));
posts.forEach((p) => (p.author = byId.get(p.author.toString())));
```

**With an ODM (Mongoose)**

```js
const userSchema = new Schema({
  email: { type: String, required: true, unique: true, lowercase: true, trim: true },
  age:   { type: Number, min: 13 },
}, { timestamps: true });
const User = model('User', userSchema);

await User.create({ email: 'ASHU@EXAMPLE.COM', age: '27' });
// → email stored as 'ashu@example.com', age cast to 27, typos rejected (strict mode)

const posts = await Post.find()
  .sort({ createdAt: -1 })
  .limit(10)
  .populate('author', 'name email');   // the "join", done for you
```

---

### The Problem ODMs Solve

MongoDB is **schemaless**: any document can have any shape. That's
flexible, but in an application it quickly leads to messy data.

| Problem with raw documents | What the ODM adds |
|---|---|
| Any shape can be saved (typos, missing fields) | **Schema** + strict mode |
| No type safety (`'27'` vs `27`) | **Casting** to the declared type |
| No business rules | **Validation** (`required`, `min`, `match`, custom) |
| Repeated defaults and timestamps | **Defaults**, `timestamps: true` |
| No built-in joins across collections | **References + `populate()`** |
| Logic scattered across the codebase | **Middleware (hooks), methods, statics, virtuals** |
| Raw `ObjectId`/BSON handling | Automatic `ObjectId` casting and conversion |

> The database stays flexible. The **application** gets a schema.

---

### How an ODM Works (the Short Version)

```
1. Define a schema  →  new Schema({ email: { type: String, required: true } })
2. Compile a model  →  const User = model('User', userSchema)   // → "users" collection
3. Call the API     →  User.find({ age: { $gte: 18 } })
4. Cast & validate  →  '18' → 18, run validators, apply defaults
5. Run via driver   →  db.users.find({ age: { $gte: 18 } })
6. Hydrate          →  BSON → Mongoose documents (with methods, virtuals, change tracking)
```

Deep dive: [How ODM Works](05_how-odm-works.md)

---

### Embedding vs Referencing

The biggest modelling decision in a document database. ODMs support both.

**Embedding**: store related data **inside** the parent document.

```js
const postSchema = new Schema({
  title: String,
  comments: [{ body: String, author: String, createdAt: { type: Date, default: Date.now } }],
});
// One read returns the post and all its comments.
```

**Referencing**: store an `ObjectId` and look the other document up.

```js
const postSchema = new Schema({
  title: String,
  author: { type: Schema.Types.ObjectId, ref: 'User' },
});
await Post.findById(id).populate('author');
```

| | Embed | Reference |
|---|---|---|
| Read pattern | Always read together | Read separately or sometimes |
| Size | Small, bounded (address, last 5 comments) | Large or unbounded (all orders) |
| Updates | Child rarely changes independently | Child is updated on its own |
| Example | `user.address`, `order.items` | `post.author`, `order.customer` |

> Rule of thumb: **"data that is accessed together should be stored
> together"**, but never let an array grow without limit (16 MB document cap).

---

### The ODM Landscape

| ODM | Language | Notes |
|---|---|---|
| **Mongoose** | Node.js | The standard MongoDB ODM: schemas, validation, middleware, populate |
| **Typegoose** | TypeScript | Define Mongoose models with TS classes and decorators |
| **Prisma (MongoDB connector)** | Node.js / TS | Schema-first typed client; check your Prisma version's MongoDB support |
| **Papr** | Node.js / TS | Lightweight; uses MongoDB's built-in JSON Schema validation |
| **Beanie / ODMantic** | Python | Async ODMs built on Pydantic |
| **MongoEngine** | Python | Classic synchronous ODM |
| **Spring Data MongoDB / Morphia** | Java | Annotation-based mapping |
| **Doctrine MongoDB ODM** | PHP | Data Mapper style |
| **Mongoid** | Ruby | ActiveRecord-like API for MongoDB |

Hands-on: [Mongoose Setup](../07-odm-examples/mongoose/01_setup.md)

---

### What an ODM Does *Not* Do for You

- **It doesn't design your documents.** Embed vs reference is still your call.
- **It doesn't make `populate()` a real join.** `populate` runs **extra
  queries**; misuse causes N+1-style slowness.
- **It doesn't create the right indexes** for your queries. You declare them.
- **Schema rules live in the app.** Another service writing to the same
  collection with the raw driver skips them (unless you also use DB-level
  JSON Schema validation).
- **It doesn't replace knowing MongoDB**: query operators, aggregation
  pipelines, `explain()`.

---

### Interview-Ready Summary

- An **ODM** maps collections ↔ models, documents ↔ objects, fields ↔
  properties, and `ObjectId` refs ↔ related objects.
- It adds **schemas, casting, validation, defaults, middleware, virtuals and
  `populate()`** on top of the schemaless driver.
- Key modelling choice: **embed** (read together, bounded) vs **reference**
  (large, unbounded or independently updated).
- **Mongoose** is the standard Node.js ODM; Typegoose adds TS classes.
- Schema rules are app-level, `populate` is extra queries, and indexes are
  your job.
