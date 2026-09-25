## Indexes

### What Is an Index

An **index** is a sorted data structure (a B-tree) that lets MongoDB find
documents **without scanning the whole collection**. Without one, a query
is a **COLLSCAN** (read every document). With the right one, it's an
**IXSCAN** (jump straight to the matches).

```
posts.find({ author: X }).sort({ createdAt: -1 })

No index:  read all 1,000,000 posts → filter → sort in memory     (slow)
Index {author: 1, createdAt: -1}:  jump to author X, already sorted (fast)
```

Every collection has a unique index on **`_id`** automatically.

Deep dive: [Indexing](../09-advanced/03_indexing.md)

---

### Declaring Indexes in Mongoose

**On a field:**

```js
const userSchema = new Schema({
  email:    { type: String, unique: true },     // unique index
  username: { type: String, index: true },      // regular index
  phone:    { type: String, unique: true, sparse: true },   // unique, but only where it exists
});
```

**On the schema** (compound and special indexes):

```js
postSchema.index({ author: 1, createdAt: -1 });              // compound
postSchema.index({ tags: 1 });                               // multikey (array field)
postSchema.index({ title: 'text', body: 'text' });           // full-text search
postSchema.index({ 'seo.slug': 1 }, { unique: true });       // unique on a nested path
```

`1` = ascending, `-1` = descending. The direction matters only for
compound indexes used in sorts.

---

### Index Types

| Type | Declared as | Use for |
|---|---|---|
| Single field | `{ email: 1 }` | Lookups on one field |
| Compound | `{ author: 1, createdAt: -1 }` | Filter + sort, multi-field queries |
| Multikey | `{ tags: 1 }` (on an array) | "Posts tagged X" |
| Text | `{ title: 'text', body: 'text' }` | Keyword search (`$text`) |
| Geospatial | `{ location: '2dsphere' }` | "Near me" queries |
| Hashed | `{ userId: 'hashed' }` | Hashed sharding |
| Wildcard | `{ 'attributes.$**': 1 }` | Flexible / unknown sub-fields |

### Index Options

| Option | Effect |
|---|---|
| `unique: true` | Reject duplicate values (error `11000`) |
| `sparse: true` | Skip documents missing the field |
| `partialFilterExpression` | Index only documents matching a filter |
| `expireAfterSeconds` / `expires` | **TTL**: auto-delete documents after a time |
| `collation` | Case-insensitive comparisons |
| `name` | Custom index name |

```js
// Unique email among active users only
userSchema.index({ email: 1 }, { unique: true, partialFilterExpression: { deletedAt: null } });

// Sessions deleted 7 days after creation
sessionSchema.index({ createdAt: 1 }, { expireAfterSeconds: 60 * 60 * 24 * 7 });
// or on the field: createdAt: { type: Date, default: Date.now, expires: '7d' }

// Case-insensitive unique usernames
userSchema.index({ username: 1 }, { unique: true, collation: { locale: 'en', strength: 2 } });
```

---

### Compound Indexes and the ESR Rule

Order fields in a compound index by **E**quality → **S**ort → **R**ange:

```js
// Query: published posts of an author, newest first, from this year
Post.find({ author: id, published: true, createdAt: { $gte: jan1 } }).sort({ createdAt: -1 });

// ESR index: equality (author, published) → sort (createdAt) → range (createdAt, same field)
postSchema.index({ author: 1, published: 1, createdAt: -1 });
```

**Prefix rule:** an index on `{ a: 1, b: 1, c: 1 }` also serves queries
on `{ a }` and `{ a, b }`, but **not** `{ b }` or `{ c }` alone.

---

### Checking a Query with `explain()`

```js
const plan = await Post.find({ author: id }).sort({ createdAt: -1 }).explain('executionStats');

plan.queryPlanner.winningPlan;              // look for IXSCAN (good) vs COLLSCAN (bad)
plan.executionStats.totalDocsExamined;      // docs read
plan.executionStats.nReturned;              // docs returned
plan.executionStats.executionTimeMillis;
```

| Sign | Meaning |
|---|---|
| `COLLSCAN` | No usable index; every document was read |
| `IXSCAN` | Index used |
| `SORT` stage (in memory) | The index doesn't cover the sort |
| `totalDocsExamined` ≫ `nReturned` | Index isn't selective enough |
| `totalDocsExamined: 0` | **Covered query**: answered from the index alone |

**Covered query**: project only indexed fields (and exclude `_id` if it isn't
in the index):

```js
await User.find({ email: 'ashu@example.com' }, { email: 1, _id: 0 });
```

---

### Building and Syncing Indexes

By default Mongoose calls `createIndexes()` for every model at startup
(**`autoIndex: true`**). Convenient in development, risky in production
(index builds on big collections use resources).

```js
// Production: turn it off
mongoose.set('autoIndex', false);   // or per schema: { autoIndex: false }

// Then build deliberately (a deploy script or migration step)
await Post.syncIndexes();   // create missing indexes, DROP ones not in the schema
await Post.diffIndexes();   // preview: { toCreate: [...], toDrop: [...] }
await Post.listIndexes();   // what exists now
```

> `syncIndexes()` **drops** indexes that aren't in the schema, including
> ones someone added by hand. Run `diffIndexes()` first.

---

### Unique Index Gotchas

- `unique` only works once the index **exists**. If duplicates are already in
  the collection, the index build fails.
- Duplicate inserts throw `E11000 duplicate key error` (`err.code === 11000`).
  Handle it as a 409 Conflict. See [Validation](05_validation.md).
- Without `sparse` or a partial filter, **only one** document may be missing
  the field (a missing value is treated as `null`).

---

### The Cost of Indexes

| Benefit | Cost |
|---|---|
| Much faster reads, filters and sorts | Every insert/update/delete also updates each index |
| Enables unique constraints and TTL | Uses RAM and disk; indexes should fit in memory |
| Covered queries | Max 64 indexes per collection |

Index the fields you **filter, sort and join on**, and remove indexes
nothing uses:

```js
await mongoose.connection.db.collection('posts').aggregate([{ $indexStats: {} }]).toArray();
```

---

### Blog Index Plan

```js
userSchema.index({ email: 1 }, { unique: true });
postSchema.index({ author: 1, createdAt: -1 });            // an author's posts, newest first
postSchema.index({ published: 1, createdAt: -1 });         // public feed
postSchema.index({ tags: 1, createdAt: -1 });              // tag pages
postSchema.index({ 'seo.slug': 1 }, { unique: true });     // post by URL
postSchema.index({ title: 'text', body: 'text' });         // search
tagSchema.index({ name: 1 }, { unique: true });
```

---

### Interview-Ready Summary

- An index turns a **COLLSCAN** into an **IXSCAN**; `_id` is always indexed.
- Declare with `index: true` / `unique: true` on fields or `schema.index()`
  for compound, text, geospatial and TTL indexes.
- Compound indexes follow **ESR** (Equality → Sort → Range) and the **prefix
  rule**.
- Verify with **`explain('executionStats')`**: look for IXSCAN and compare
  `totalDocsExamined` to `nReturned`.
- Turn **`autoIndex` off in production**; build with `syncIndexes()` after
  checking `diffIndexes()`.
- Indexes speed reads but slow writes and use memory, so index what you query
  and drop what's unused.
