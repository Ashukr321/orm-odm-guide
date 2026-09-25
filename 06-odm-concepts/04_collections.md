## Collections

### What Is a Collection

A **collection** is a group of documents in a MongoDB database, like a
**table** in SQL but without a fixed schema. Each Mongoose model maps to
one collection.

```
Database "blog"
├── users      ← User model
├── posts      ← Post model (comments embedded inside each post)
└── tags       ← Tag model
```

| SQL | MongoDB |
|---|---|
| Database | Database |
| Table | **Collection** |
| Row | Document |
| Column | Field |
| Schema (DDL) | Optional (app schema in Mongoose, or `$jsonSchema`) |

Collections are created **lazily**: the first insert (or index build)
creates them. Mongoose also creates them on startup by default
(`autoCreate`).

---

### Naming

```js
model('User', userSchema);                       // → "users"
model('Person', personSchema);                   // → "people" (pluralized)
model('Post', postSchema, 'blog_posts');         // → "blog_posts" (explicit)
new Schema({ /* … */ }, { collection: 'audit_log' });
```

Tips:

- Use **lower-case, plural** names (`users`, `order_items`).
- Set `collection` explicitly when sharing a database with other services,
  so Mongoose's pluralization never surprises you.
- Avoid `$` and names starting with `system.`.

---

### One Collection or Several? (Modelling)

The key design question is **what goes in its own collection and what gets
embedded**.

| Put it in its **own collection** when | **Embed** it when |
|---|---|
| It's queried on its own (all users, all tags) | It's always read with the parent (address, line items) |
| It can grow without limit (all orders of a user) | It's small and bounded (last 5 comments, 3 addresses) |
| Many parents share it (tags, categories) | It belongs to one parent only |
| It's updated independently and often | It rarely changes on its own |

```js
// Blog design
users:  { _id, email, name, role }
posts:  { _id, title, author: ObjectId(users), tags: ['mongodb'], comments: [ {...}, … ] }
tags:   { _id, name, postCount }
```

If comments could reach tens of thousands per post, move them to their own
`comments` collection with a `post` reference.

---

### Common Design Patterns

| Pattern | Idea | Example |
|---|---|---|
| **Embedded** | Child data inside the parent | `order.items` |
| **Referenced** | Store `ObjectId`s, load with `populate` / `$lookup` | `post.author` |
| **Subset** | Embed the most-used part, reference the rest | Last 10 comments in the post, all in `comments` |
| **Extended reference** | Copy a few fields from the referenced doc | `post.author = { _id, name }` |
| **Bucket** | Group many small records into one doc per time window | IoT readings per sensor per hour |
| **Computed** | Store pre-calculated values | `tag.postCount`, `post.commentCount` |
| **Polymorphic** | Different shapes in one collection | Discriminators (`kind`: Click or Purchase) |

**Extended reference** trades duplication for fewer lookups:

```js
author: {
  _id:  { type: Schema.Types.ObjectId, ref: 'User' },
  name: String,     // copied; update it when the user renames (rare)
},
```

---

### Special Collection Types

**Capped collections**: fixed size, insertion order, oldest documents
overwritten. Good for recent logs.

```js
new Schema({ message: String, level: String }, { capped: { size: 10 * 1024 * 1024, max: 10000 } });
```

**Time series collections**: optimized storage for measurements over time.

```js
const readingSchema = new Schema(
  { sensorId: String, at: Date, temperature: Number },
  { timeseries: { timeField: 'at', metaField: 'sensorId', granularity: 'minutes' } }
);
```

**TTL (auto-expiring documents)**: not a collection type, but a TTL index
that deletes documents after a time.

```js
const sessionSchema = new Schema({ token: String, createdAt: { type: Date, default: Date.now, expires: '7d' } });
```

See [Indexes](09_indexes.md).

---

### Collection-Level Validation

MongoDB can enforce rules itself, for every writer, not just Mongoose:

```js
await mongoose.connection.db.createCollection('users', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['email'],
      properties: {
        email: { bsonType: 'string', pattern: '^\\S+@\\S+$' },
        role:  { enum: ['user', 'admin'] },
      },
    },
  },
  validationLevel: 'moderate',   // skip existing invalid docs on update
  validationAction: 'error',     // or 'warn'
});
```

See [Validation](05_validation.md).

---

### Working with Collections Directly

```js
const db = mongoose.connection.db;

await db.listCollections().toArray();       // all collections
await Post.collection.countDocuments();     // raw driver collection behind a model
await Post.createCollection();              // create now (with schema options like capped/timeseries)
await db.collection('old_logs').drop();     // drop a collection (irreversible)
```

Changing collection options (like converting to capped) usually means
creating a new collection and copying the data.

---

### Limits to Keep in Mind

| Limit | Value | Design impact |
|---|---|---|
| Max document size | **16 MB** | Don't embed unbounded arrays |
| Max nesting depth | 100 levels | Keep documents reasonably flat |
| Indexes per collection | 64 | Index deliberately |
| Namespace (`db.collection`) length | 255 bytes (recent versions) | Short names |

---

### Interview-Ready Summary

- A **collection** is MongoDB's table: a group of documents without a
  required schema, created lazily. Each Mongoose model maps to one.
- Names default to the **lower-case plural** of the model name; set
  `collection` explicitly when it matters.
- Core design choice: **own collection** (queried alone, unbounded, shared)
  vs **embed** (read together, bounded, owned).
- Patterns: embedded, referenced, subset, extended reference, bucket,
  computed, polymorphic.
- Special types: **capped**, **time series**, plus **TTL** indexes. Add
  `$jsonSchema` validation when other writers exist. Watch the **16 MB** limit.
