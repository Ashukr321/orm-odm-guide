## Documents

### Two Meanings of "Document"

| | MongoDB document | Mongoose document |
|---|---|---|
| What | A BSON object stored in a collection | A model **instance** wrapping that data |
| Has | Fields and values | Fields **plus** methods, getters/setters, virtuals, change tracking |
| You get it from | The driver, `.lean()`, `aggregate()` | `find()`, `findById()`, `new Model()` |

```js
// MongoDB document (BSON, as stored)
{ _id: ObjectId("66f1…"), title: "Hello", published: false, __v: 0 }

// Mongoose document
const post = await Post.findById(id);
post instanceof Post;   // true
post.save; post.isModified; post.toJSON;   // all available
```

---

### `_id`, `id` and `__v`

```js
post._id;   // ObjectId("66f1c0e2…"), the primary key, auto-generated
post.id;    // "66f1c0e2…", a string virtual getter
post.__v;   // 0, the version key (used for array updates)
```

- Every document has a unique `_id`. The default is an `ObjectId`, which
  contains a timestamp: `post._id.getTimestamp()`.
- Compare ids with `.equals()` or strings, **never** `===`:

```js
post.author.equals(user._id);            // ✅
String(post.author) === String(user._id); // ✅
post.author === user._id;                 // ❌ always false (different objects)
```

---

### Creating Documents

```js
// 1. new + save: lets you change things before saving
const post = new Post({ title: 'Hello', author: userId });
post.tags.push('mongodb');
post.isNew;         // true
await post.save();  // defaults → validation → pre('save') → insertOne → post('save')
post.isNew;         // false

// 2. create: new + save in one call (runs the same validation and hooks)
const post2 = await Post.create({ title: 'Second', author: userId });

// 3. insertMany: fast bulk insert (validates; runs insertMany hooks, not save hooks)
await Post.insertMany([{ title: 'A', author: userId }, { title: 'B', author: userId }]);
```

---

### Reading and Changing Fields

```js
const post = await Post.findById(id);

post.title;                      // getter
post.get('seo.slug');            // path access
post.title = 'New title';        // setter (casts and applies `set` transforms)
post.set({ published: true, 'seo.description': 'Intro' });
```

---

### Change Tracking

Mongoose remembers what changed and sends only those paths on `save()`.

```js
const post = await Post.findById(id);
post.title = 'Updated';
post.tags.push('odm');

post.isModified('title');   // true
post.modifiedPaths();       // ['title', 'tags']
await post.save();
// posts.updateOne({ _id }, { $set: { title: 'Updated', tags: [...], updatedAt: … } })
```

**Mixed and Date mutations aren't detected automatically.** Tell Mongoose:

```js
post.extra.views = 10;          // Mixed field changed in place
post.markModified('extra');
post.publishedAt.setMonth(3);   // Date mutated in place
post.markModified('publishedAt');
await post.save();
```

---

### Subdocuments

Embedded documents (like comments) are documents too, with their own `_id`,
validation and hooks.

```js
const post = await Post.findById(id);

// Add
post.comments.push({ body: 'Great post!', author: userId });
const added = post.comments.at(-1);

// Find by _id
const comment = post.comments.id(commentId);
comment.body = 'Edited';

// Remove
post.comments.id(commentId).deleteOne();   // or post.comments.pull(commentId)

await post.save();   // saves the parent (and all subdocuments) in one write
comment.parent();    // → post
```

Or update the array **atomically** without loading the post:

```js
await Post.updateOne({ _id: id }, { $push: { comments: { body: 'Nice', author: userId } } });
await Post.updateOne({ _id: id, 'comments._id': commentId }, { $set: { 'comments.$.body': 'Edited' } });
await Post.updateOne({ _id: id }, { $pull: { comments: { _id: commentId } } });
```

---

### Converting Documents: `toObject`, `toJSON`, `lean`

```js
const post = await Post.findById(id);
post.toObject();        // plain JS object
post.toJSON();          // what res.json(post) / JSON.stringify use

// Hide fields and include virtuals everywhere
postSchema.set('toJSON', {
  virtuals: true,
  transform: (_doc, ret) => { delete ret.__v; return ret; },
});
```

**`.lean()`** skips creating Mongoose documents entirely:

```js
const posts = await Post.find({ published: true }).lean();
// plain objects: faster and lighter, but no save(), virtuals, getters or change tracking
```

| | Mongoose document | `.lean()` object |
|---|---|---|
| Speed / memory | Slower, heavier | Much faster, lighter |
| `save()`, methods | ✅ | ❌ |
| Virtuals, getters | ✅ | ❌ (unless you add plugins) |
| Use for | Load → modify → save | Read-only API responses |

`Model.hydrate(plainObj)` turns a plain object (for example, from `aggregate`)
back into a document.

---

### Deleting Documents

```js
await post.deleteOne();                    // document method (runs document deleteOne hooks if configured)
await Post.findByIdAndDelete(id);          // returns the deleted document
await Post.deleteMany({ published: false }); // query: no document hooks
```

---

### The Version Key and Concurrent Edits

`__v` protects **array** updates. If two requests load the same post and both
modify `comments` by position, the second `save()` throws a `VersionError`.

```js
// Stronger: check the version on every save, not just array changes
new Schema({ /* … */ }, { optimisticConcurrency: true });
```

For simple counters, skip read-modify-write entirely:

```js
await Post.updateOne({ _id: id }, { $inc: { views: 1 } });   // atomic
```

---

### Interview-Ready Summary

- A **Mongoose document** is a model instance: data plus methods, virtuals,
  getters/setters and **change tracking**. A MongoDB document is just BSON.
- `_id` is the primary key (ObjectId), `id` is its string form, and `__v` is the
  version key. Compare ids with `.equals()`, never `===`.
- `save()` sends only **modified paths**; use `markModified()` for Mixed and
  in-place Date changes.
- **Subdocuments**: `push`, `.id()`, `.deleteOne()` + `save()`, or atomic
  `$push` / `$set` with `$` / `$pull`.
- Use **`.lean()`** for read-only responses; `toJSON` transforms control API
  output; `optimisticConcurrency` or atomic operators handle concurrent edits.
