## How an ODM Works

### The Big Picture

Every Mongoose call goes through the same pipeline, from a method call to
BSON on disk and back.

```
 Your code                Post.find({ author: '66f…', published: 'true' }).limit(10)
    │
    ▼
 1. Schema & model   ──►  Which collection? Which paths and types?
    │
    ▼
 2. Query building   ──►  Query object: filter, projection, options (chainable)
    │
    ▼
 3. Casting          ──►  '66f…' → ObjectId, 'true' → true
    │
    ▼
 4. Middleware       ──►  pre('find') hooks
    │
    ▼
 5. Driver command   ──►  db.posts.find({ author: ObjectId(…), published: true }, { limit: 10 })
    │                     (BSON over the wire via the mongodb driver + connection pool)
    ▼
 6. Hydration        ──►  BSON → Mongoose documents (getters, virtuals, methods)
    │
    ▼
 7. post middleware  ──►  post('find') hooks
    │
    ▼
 Your code receives   [PostDocument, …]
```

---

### Step 1: Schema → Model

A **schema** describes the shape. A **model** is a schema compiled
against a collection, and it's the class you query with.

```js
const { Schema, model } = require('mongoose');

const postSchema = new Schema(
  {
    title:     { type: String, required: true },
    published: { type: Boolean, default: false },
    author:    { type: Schema.Types.ObjectId, ref: 'User', index: true },
    tags:      [String],
  },
  { timestamps: true }
);

const Post = model('Post', postSchema);   // collection name: "posts" (lower-cased, pluralized)
```

From the schema, Mongoose knows each **path** (`title`, `author`,
`tags`), its **SchemaType** (String, ObjectId, Array), validators,
defaults, getters/setters and indexes.

See [Schemas](../06-odm-concepts/01_schemas.md) ·
[Models](../06-odm-concepts/02_models.md)

---

### Step 2: Query Building (Lazy and Chainable)

`Post.find()` returns a **Query** object. Nothing is sent until you
`await` it (or call `.exec()`).

```js
const q = Post.find({ published: true })   // Query, not results
  .where('tags').in(['mongodb'])
  .select('title author createdAt')
  .sort({ createdAt: -1 })
  .limit(10);

const posts = await q;   // ← the command is sent here
```

```
Query
├── model:      Post  → collection "posts"
├── op:         find
├── filter:     { published: true, tags: { $in: ['mongodb'] } }
├── projection: { title: 1, author: 1, createdAt: 1 }
└── options:    { sort: { createdAt: -1 }, limit: 10 }
```

---

### Step 3: Casting

Mongoose converts filter values and update values to the schema types
**before** sending them.

| You pass | Path type | Sent to MongoDB |
|---|---|---|
| `'66f1c0…'` | `ObjectId` | `ObjectId("66f1c0…")` |
| `'true'` | `Boolean` | `true` |
| `'42'` | `Number` | `42` |
| `'2026-09-25'` | `Date` | `ISODate("2026-09-25T00:00:00Z")` |
| `'abc'` | `ObjectId` | ❌ `CastError` |

**Strict query mode** can also drop filter keys that aren't in the
schema. Both behaviours protect you from typos and some injection
tricks (for example, `{ $ne: null }` sent where a string was expected).

---

### Step 4: Writes: Defaults, Validation, Middleware

For `doc.save()` / `Model.create()`:

```
new Post({...})
   │  apply defaults, cast values
   ▼
pre('validate') → validate() → post('validate')
   │
   ▼
pre('save')  → insertOne / updateOne  → post('save')
```

```js
const post = new Post({ title: 'Hello', author: userId });
post.isNew;         // true
await post.save();  // defaults → validation → pre('save') → insertOne → post('save')
```

---

### Step 5: Talking to MongoDB

1. Mongoose hands the command to the official **`mongodb` driver**.
2. The driver takes a connection from its **pool** (`maxPoolSize`, default 100).
3. The command is encoded as **BSON** and sent over the wire protocol.
4. The server returns BSON documents (in batches, via a **cursor**).

```
Mongoose  →  mongodb driver  →  connection pool  →  TCP (BSON)  →  mongod / replica set
```

See [Database Drivers](../01-database-foundation/database-drivers.md).

---

### Step 6: Hydration (BSON → Documents)

Raw results become **Mongoose documents**:

```
Raw BSON from the server                    Mongoose document
{ _id: ObjectId("66f…"),                    post._id          → ObjectId
  title: "Hello",                  →        post.id           → "66f…"   (virtual string id)
  published: false,                         post.title        → "Hello"
  author: ObjectId("66a…"),                 post.author       → ObjectId (or a User after populate)
  createdAt: ISODate(...) }                 post.save(), post.isModified(), post.toJSON() …
```

Hydration adds getters/setters, virtuals, instance methods and **change
tracking**. That's the cost `.lean()` skips:

```js
const posts = await Post.find().lean();   // plain objects: no methods, no virtuals, much faster
```

---

### Step 7: Change Tracking and Saving Back

Mongoose documents remember what changed and send only a minimal update.

```js
const post = await Post.findById(id);
post.title = 'Updated title';
post.tags.push('odm');

post.modifiedPaths();   // ['title', 'tags']
await post.save();
// db.posts.updateOne({ _id: … }, { $set: { title: 'Updated title', tags: [...], updatedAt: … } })
```

Or skip loading the document and update in one command:

```js
await Post.updateOne({ _id: id }, { $set: { title: 'Updated' }, $push: { tags: 'odm' } });
```

| | `find` + change + `save()` | `updateOne` / `findOneAndUpdate` |
|---|---|---|
| Round trips | 2 | 1 |
| `save` middleware & full validation | ✅ | ❌ (query middleware; `runValidators` opt-in) |
| Race conditions | Possible (read-modify-write) | Atomic operators (`$inc`, `$push`) |

---

### How `populate()` Works

```js
const posts = await Post.find({ published: true }).limit(10).populate('author', 'name');
```

```
1. db.posts.find({ published: true }, { limit: 10 })
2. collect author ids  → [ObjectId(a), ObjectId(b)]
3. db.users.find({ _id: { $in: [a, b] } }, { projection: { name: 1 } })
4. replace each post.author id with the matching user object (in Node.js)
```

That's **one extra query per populated path**, not a server-side join.
For a single-query join, use `$lookup` in an aggregation. See
[Population](../06-odm-concepts/07_population.md) ·
[Aggregation](../06-odm-concepts/08_aggregation.md).

---

### Indexes

Indexes declared in the schema are created by Mongoose on startup
(`autoIndex`, handy in development) or explicitly:

```js
postSchema.index({ author: 1, createdAt: -1 });
postSchema.index({ title: 'text', body: 'text' });

await Post.syncIndexes();   // create missing, drop removed (run in deploy scripts)
```

In production, set `autoIndex: false` and build indexes deliberately.
See [Indexes](../06-odm-concepts/09_indexes.md).

---

### A Full Trace

```js
mongoose.set('debug', true);

await Post.find({ author: '66a1f2c3d4e5f60718293a4b', published: 'true' })
  .select('title')
  .limit(2)
  .populate('author', 'name');
```

```
Mongoose: posts.find({ author: ObjectId("66a1f2c3d4e5f60718293a4b"), published: true },
                     { limit: 2, projection: { title: 1, author: 1 } })
Mongoose: users.find({ _id: { '$in': [ ObjectId("66a1f2c3d4e5f60718293a4b") ] } },
                     { projection: { name: 1 } })
```

Note that Mongoose added `author` to the projection automatically so it
could populate it.

---

### Interview-Ready Summary

- A Mongoose call flows through **schema/model → query building → casting
  → middleware → driver command (BSON, pool) → hydration → post middleware**.
- Queries are **lazy and chainable**; nothing runs until `await` / `exec()`.
- **Casting** turns strings into `ObjectId`, numbers, booleans and dates, and
  throws `CastError` for invalid values.
- Writes go **defaults → validation → `pre('save')` → insert/update →
  `post('save')`**, and `save()` sends only the **modified paths**.
- **Hydration** adds methods, virtuals and change tracking; `.lean()` skips it.
- **`populate()`** = one extra `$in` query per path, merged in Node.js.
