## Mongoose CRUD

All examples use the models from [Schema](02_schema.md):

```js
import { User } from './models/user.js';
import { Post } from './models/post.js';
import { Tag } from './models/tag.js';
```

### Method Cheat Sheet

| Operation | Methods |
|---|---|
| **Create** | `Model.create`, `new Model()` + `save()`, `insertMany` |
| **Read** | `find`, `findOne`, `findById`, `countDocuments`, `exists`, `distinct` |
| **Update** | `updateOne`, `updateMany`, `findByIdAndUpdate`, `findOneAndUpdate`, `doc.save()` |
| **Delete** | `deleteOne`, `deleteMany`, `findByIdAndDelete`, `doc.deleteOne()` |
| **Bulk** | `bulkWrite` |

---

### Create

```js
// One document: validation + save hooks run
const user = await User.create({ email: 'riya@example.com', name: 'Riya', password: 'password123' });

// Build, adjust, save
const post = new Post({ title: 'Draft post', author: user._id });
post.tags.push(tagId);
await post.save();

// Many documents, fast (validates, but no save hooks; ordered: false keeps going after an error)
await Tag.insertMany([{ name: 'mongodb' }, { name: 'express' }], { ordered: false });
```

---

### Read

**By id / first match / existence:**

```js
const post = await Post.findById(id);                              // or null
const user = await User.findOne({ email: 'riya@example.com' });
const taken = await User.exists({ email: 'riya@example.com' });   // { _id } or null
```

**Filters, projection, sorting, pagination:**

```js
const posts = await Post.find({
  published: true,
  tags: { $in: [mongoTagId] },
  createdAt: { $gte: new Date('2026-01-01') },
  title: { $regex: 'mongoose', $options: 'i' },
})
  .select('title slug author createdAt')      // projection
  .sort({ createdAt: -1 })
  .skip(20)
  .limit(10)
  .lean();                                     // plain objects for the API response
```

**Common query operators:**

| Operator | Example |
|---|---|
| `$eq`, `$ne` | `{ role: { $ne: 'admin' } }` |
| `$gt`, `$gte`, `$lt`, `$lte` | `{ views: { $gte: 100 } }` |
| `$in`, `$nin` | `{ _id: { $in: ids } }` |
| `$exists` | `{ publishedAt: { $exists: true } }` |
| `$regex` | `{ title: { $regex: '^hello', $options: 'i' } }` |
| `$or`, `$and`, `$nor` | `{ $or: [{ published: true }, { author: me }] }` |
| `$all`, `$size`, `$elemMatch` | `{ tags: { $all: [a, b] } }`, `{ comments: { $size: 0 } }` |
| Dot notation | `{ 'comments.author': userId }` |

**Chainable query builder** (same query, different style):

```js
await Post.find().where('published').equals(true).where('views').gte(100).sort('-createdAt').limit(10);
await Post.find().published().sort('-createdAt');   // query helper from the schema
```

**Count and distinct:**

```js
await Post.countDocuments({ published: true });
await Post.estimatedDocumentCount();         // fast, whole collection, from metadata
await Post.distinct('author', { published: true });
```

**Full-text search** (uses the text index on title/body):

```js
await Post.find({ $text: { $search: 'embedding referencing' } }, { score: { $meta: 'textScore' } })
  .sort({ score: { $meta: 'textScore' } })
  .limit(10);
```

---

### Pagination: Offset vs Cursor

```js
// Offset: simple, but slow on deep pages
async function page(n = 1, size = 10) {
  const filter = { published: true };
  const [items, total] = await Promise.all([
    Post.find(filter).sort({ _id: -1 }).skip((n - 1) * size).limit(size).lean(),
    Post.countDocuments(filter),
  ]);
  return { items, total, pages: Math.ceil(total / size) };
}

// Cursor ("load more"): constant speed, stable under inserts
async function feed(after, size = 10) {
  const filter = { published: true, ...(after && { _id: { $lt: after } }) };
  const items = await Post.find(filter).sort({ _id: -1 }).limit(size + 1).lean();
  const hasMore = items.length > size;
  return { items: items.slice(0, size), next: hasMore ? items[size - 1]._id : null };
}
```

`_id` works as a cursor because ObjectIds increase over time.

---

### Update

**Update operators** (atomic, one round trip):

```js
await Post.updateOne({ _id: id }, { $set: { title: 'New title' } });
await Post.updateOne({ _id: id }, { $inc: { views: 1 } });
await Post.updateOne({ _id: id }, { $addToSet: { tags: tagId } });   // add if missing
await Post.updateOne({ _id: id }, { $pull: { tags: tagId } });       // remove
await User.updateOne({ _id: id }, { $unset: { bio: '' } });
```

| Operator | Does |
|---|---|
| `$set` / `$unset` | Set / remove fields |
| `$inc` | Add to a number |
| `$push` / `$addToSet` | Append / append if not present |
| `$pull` / `$pop` | Remove matching / first-or-last element |
| `$min` / `$max` | Update only if lower / higher |
| `$currentDate` | Set to now |

**Return the updated document:**

```js
const post = await Post.findByIdAndUpdate(
  id,
  { $set: { published: true } },
  { new: true, runValidators: true }        // return the updated doc; validate the update
);
```

**Load, modify, save** (runs full validation and `save` hooks):

```js
const post = await Post.findById(id);
post.title = 'Edited title';
post.published = true;                       // pre('validate') sets publishedAt
await post.save();
```

**Upsert:**

```js
const tag = await Tag.findOneAndUpdate(
  { name: 'mongodb' },
  { $setOnInsert: { name: 'mongodb' } },
  { upsert: true, new: true }
);
```

**Many documents:**

```js
const { matchedCount, modifiedCount } = await Post.updateMany(
  { author: userId, published: false },
  { $set: { published: true } }
);
```

> `findByIdAndUpdate` / `updateOne` **don't run `save` hooks**. Changing a
> password this way would skip hashing. Use `findById` + `save()` for that.

---

### Delete

```js
await Post.deleteOne({ _id: id });
await Post.findByIdAndDelete(id);                       // returns the deleted document
await Post.deleteMany({ published: false, createdAt: { $lt: new Date('2025-01-01') } });

const post = await Post.findById(id);
await post.deleteOne();                                 // document method
```

**Soft delete** instead: add `deletedAt: Date`, set it on "delete", and
filter it out with query middleware (see
[Middleware](../../06-odm-concepts/06_middleware.md)).

---

### Bulk Writes

Mixed operations in one round trip:

```js
await Post.bulkWrite([
  { updateOne: { filter: { _id: a }, update: { $inc: { views: 10 } } } },
  { updateMany: { filter: { author: spammer }, update: { $set: { published: false } } } },
  { deleteOne: { filter: { _id: b } } },
  { insertOne: { document: { title: 'Imported', author: userId } } },
], { ordered: false });
```

---

### Handling Errors

| Error | When | HTTP |
|---|---|---|
| `CastError` | Invalid id or type (`'abc'` as ObjectId) | 400 |
| `ValidationError` | Schema validation failed | 400 |
| `MongoServerError` with `code: 11000` | Duplicate key (unique index) | 409 |
| `DocumentNotFoundError` | `orFail()` found nothing | 404 |

```js
const post = await Post.findById(id).orFail();   // throws DocumentNotFoundError if missing
```

**Central Express error handler:**

```js
export function errorHandler(err, _req, res, _next) {
  if (err.name === 'CastError') return res.status(400).json({ error: `Invalid ${err.path}` });
  if (err.name === 'ValidationError') {
    return res.status(400).json({ errors: Object.values(err.errors).map((e) => ({ field: e.path, message: e.message })) });
  }
  if (err.code === 11000) return res.status(409).json({ error: 'Already exists', fields: err.keyValue });
  if (err.name === 'DocumentNotFoundError') return res.status(404).json({ error: 'Not found' });
  console.error(err);
  res.status(500).json({ error: 'Internal error' });
}
```

---

### Putting It Together: a Posts Router

**src/routes/posts.js**

```js
import { Router } from 'express';
import { isValidObjectId } from 'mongoose';
import { Post } from '../models/post.js';

export const posts = Router();

posts.param('id', (req, res, next, id) => (isValidObjectId(id) ? next() : res.sendStatus(404)));

posts.get('/', async (req, res) => {
  const items = await Post.find().published()
    .select('title slug author createdAt')
    .sort({ _id: -1 }).limit(20).lean();
  res.json(items);
});

posts.get('/:id', async (req, res) => {
  res.json(await Post.findById(req.params.id).orFail().lean());
});

posts.post('/', async (req, res) => {
  const { title, body, tags } = req.body;                 // pick fields; never pass req.body straight in
  const post = await Post.create({ title, body, tags, author: req.user.id });
  res.status(201).json(post);
});

posts.patch('/:id', async (req, res) => {
  const post = await Post.findOne({ _id: req.params.id, author: req.user.id }).orFail();
  Object.assign(post, pick(req.body, ['title', 'body', 'published']));
  res.json(await post.save());
});

posts.delete('/:id', async (req, res) => {
  await Post.deleteOne({ _id: req.params.id, author: req.user.id });
  res.sendStatus(204);
});

const pick = (obj, keys) => Object.fromEntries(keys.filter((k) => k in obj).map((k) => [k, obj[k]]));
```

Express 5 forwards rejected promises from async handlers to the error
handler. On Express 4, wrap handlers or use `express-async-errors`.

> **Security:** never pass raw `req.body` or `req.query` objects into filters.
> `{ email: { $ne: null } }` from a client is a NoSQL injection. Pick and
> validate fields, and keep `strictQuery` on.

---

### Interview-Ready Summary

- **Create** with `create` (validation + hooks) or `insertMany` (fast, no
  `save` hooks).
- **Read** with `find` + operators, `select`, `sort`, `skip`/`limit` (or
  `_id` cursors), and `.lean()` for responses; `$text` for search.
- **Update** with atomic operators (`$set`, `$inc`, `$addToSet`, `$pull`),
  `{ new: true, runValidators: true }`, or load → modify → `save()` when you
  need hooks.
- **Delete** with `deleteOne` / `deleteMany` / `findByIdAndDelete`; batch
  mixed writes with `bulkWrite`.
- Map `CastError`/`ValidationError` → 400, `11000` → 409, `orFail()` → 404.
  Never trust client objects as filters.

Next: [Relationships](04_relationships.md)
