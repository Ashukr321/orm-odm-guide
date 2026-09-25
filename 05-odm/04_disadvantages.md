## Disadvantages of ODMs

### At a Glance

| Disadvantage | One-line explanation |
|---|---|
| App-level schema only | Other writers can bypass it |
| `populate()` isn't a join | Extra queries; N+1 when misused |
| Performance overhead | Hydration, casting, change tracking |
| Hidden queries | You don't see what's sent to MongoDB |
| Middleware surprises | Hooks don't run for every operation |
| Validation gaps on updates | Update queries skip validators by default |
| Encourages relational habits | Over-referencing instead of good document design |
| Learning curve & lock-in | Mongoose-specific concepts and API |
| No schema history | No migrations means no record of shape changes |

---

### 1. The Schema Lives Only in the App

Mongoose validates in Node.js. The database itself still accepts
anything.

```js
// A script, another service, or the Mongo shell:
db.users.insertOne({ emial: 'oops', age: 'old' });   // ✅ accepted by MongoDB
```

**Fix:** for data several systems write to, add **MongoDB JSON Schema
validation** on the collection too:

```js
await db.createCollection('users', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['email'],
      properties: { email: { bsonType: 'string' }, age: { bsonType: 'number', minimum: 13 } },
    },
  },
});
```

---

### 2. `populate()` Is Not a Join

`populate` runs a **second query** (`find({ _id: { $in: [...] } })`) per
populated path and merges the results in Node.js.

```js
// ❌ N+1: one extra query per post
const posts = await Post.find();
for (const p of posts) {
  p.authorDoc = await User.findById(p.author);
}

// ✅ 2 queries total
const posts = await Post.find().populate('author', 'name');

// ✅ 1 query: a real server-side join
const posts = await Post.aggregate([
  { $lookup: { from: 'users', localField: 'author', foreignField: '_id', as: 'author' } },
  { $unwind: '$author' },
]);
```

Deep, multi-level populates can still fan out into many queries. If you
populate on every read, the data probably should be **embedded**.

See [N+1 Problem](../09-advanced/01_n-plus-one-problem.md).

---

### 3. Performance Overhead

Every result becomes a full Mongoose document with getters, setters,
change tracking and methods.

```js
// Hydrated documents: heavier, slower for big reads
const posts = await Post.find({ published: true });

// Plain JS objects: much lighter for read-only API responses
const posts = await Post.find({ published: true }).lean();
```

| Operation | Faster alternative |
|---|---|
| Large read-only lists | `.lean()` |
| Many inserts | `insertMany()` or `bulkWrite()` |
| Mass updates | `updateMany()` instead of load → change → `save()` in a loop |
| Streaming huge results | `.cursor()` |

---

### 4. Hidden Queries

One `save()` or `populate()` can send several commands. Turn on debug
logging in development:

```js
mongoose.set('debug', true);
// Mongoose: posts.find({ published: true }, { sort: { createdAt: -1 }, limit: 10 })
// Mongoose: users.find({ _id: { '$in': [ ObjectId("…"), … ] } })
```

Use `.explain('executionStats')` to check that a query uses an index.

---

### 5. Middleware Surprises

Hooks are tied to **specific operations**. They don't run for everything.

| You call | `pre('save')` runs? | `pre('findOneAndUpdate')` runs? |
|---|---|---|
| `doc.save()` / `Model.create()` | ✅ | ❌ |
| `Model.findByIdAndUpdate()` | ❌ | ✅ |
| `Model.updateOne()` / `updateMany()` | ❌ | ❌ (needs `pre('updateOne')` / `pre('updateMany')`) |
| `Model.insertMany()` | ❌ | ❌ (has its own `insertMany` hook) |
| `Model.bulkWrite()` / raw driver | ❌ | ❌ |

A password-hashing `pre('save')` hook does **nothing** when someone
updates the password with `findByIdAndUpdate`.

---

### 6. Validators Don't Run on Updates by Default

```js
// ❌ Saves age: -5, because validators are skipped on update queries
await User.updateOne({ _id: id }, { age: -5 });

// ✅ Opt in
await User.updateOne({ _id: id }, { age: -5 }, { runValidators: true });

// ✅ Or globally
mongoose.set('runValidators', true);
```

Even with `runValidators`, update validators have limits (for example,
`this` isn't the document).

---

### 7. It Encourages Relational Habits

Because `ref` + `populate` is so easy, teams often model MongoDB like SQL:
a collection per entity, references everywhere, populate on every read.
You end up with **joins without a join engine** and none of the benefits of
documents.

> Design documents around **how the data is read**. Embed what is read
> together; reference what is large, unbounded or shared.

---

### 8. Learning Curve and Lock-in

Mongoose has its own concepts: schema types, `strict`, `lean`, virtuals,
discriminators, query vs document middleware, `this` in hooks, population
options. Switching ODMs (or dropping to the driver) means rewriting the
data layer.

---

### 9. No Schema History

"No migrations" also means **no record** of how the shape changed. Old
documents quietly keep old shapes.

**Fix:**

- Add a `schemaVersion` field and upgrade documents lazily on read, or
- Run versioned backfill scripts (tools like `migrate-mongo` help).

```js
await User.updateMany({ schemaVersion: { $exists: false } }, { $set: { schemaVersion: 2, bio: '' } });
```

---

### Mitigation Cheat Sheet

| Problem | Mitigation |
|---|---|
| Other writers bypass the schema | MongoDB `$jsonSchema` validation |
| Slow populate / N+1 | Embed, `populate` once, or `$lookup` |
| Heavy documents | `.lean()`, projections (`select`) |
| Hidden queries | `mongoose.set('debug', true)`, `explain()` |
| Hooks not firing | Know which operation triggers which hook |
| Invalid updates | `runValidators: true` |
| Shape drift over time | `schemaVersion` + backfill scripts |

---

### Interview-Ready Summary

- The ODM schema is **app-level only**; add MongoDB `$jsonSchema`
  validation when other systems write to the same data.
- **`populate()` is extra queries**, not a join. Watch for N+1, embed
  when data is read together, or use `$lookup`.
- Hydrated documents are heavy; use **`.lean()`**, projections and bulk ops.
- **Hooks and validators don't run for every operation** (update queries
  skip `save` hooks and validators by default).
- Easy references tempt teams into **relational modelling**, and schema-less
  evolution leaves **no history** unless you add versioning.
