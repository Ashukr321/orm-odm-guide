## Mongoose Population

This page builds the blog's read endpoints with `populate()`. How it
works under the hood is covered in
[Population](../../06-odm-concepts/07_population.md).

```js
import { User } from './models/user.js';
import { Post } from './models/post.js';
import { Tag } from './models/tag.js';

mongoose.set('debug', true);   // watch the extra queries populate sends
```

---

### 1. Feed: Posts with Author and Tags

```js
async function getFeed({ limit = 20 } = {}) {
  return Post.find({ published: true })
    .select('title slug author tags createdAt')
    .sort({ createdAt: -1 })
    .limit(limit)
    .populate('author', 'name')        // only the fields the card shows
    .populate('tags', 'name')
    .lean();
}
```

```
Mongoose: posts.find({ published: true }, { sort: { createdAt: -1 }, limit: 20, projection: {…} })
Mongoose: users.find({ _id: { '$in': [ …up to 20 ids ] } }, { projection: { name: 1 } })
Mongoose: tags.find({ _id: { '$in': [ … ] } }, { projection: { name: 1 } })
```

**3 queries** for 20 posts, however many there are. Always `select` on
populated paths so you don't ship emails or password hashes to the client.

```json
[
  {
    "_id": "66f1…18",
    "title": "Hello Mongoose",
    "author": { "_id": "66f1…01", "name": "Ashu" },
    "tags": [{ "_id": "66f1…05", "name": "mongodb" }],
    "createdAt": "2026-09-25T10:00:00.000Z"
  }
]
```

---

### 2. Post Page: Nested Population of Comment Authors

```js
async function getPost(slug) {
  return Post.findOne({ slug, published: true })
    .populate('author', 'name bio')
    .populate('tags', 'name')
    .populate({ path: 'comments.author', select: 'name' })   // path inside the embedded array
    .orFail()
    .lean();
}
```

Four queries: the post, users (author), tags, and users (comment authors).
Mongoose collects all comment-author ids into **one** `$in` query.

---

### 3. Profile: Virtual Populate with Counts

The `User` schema defines a virtual `posts` (see [Schema](02_schema.md)).
Add a count virtual next to it:

```js
userSchema.virtual('postCount', {
  ref: 'Post',
  localField: '_id',
  foreignField: 'author',
  count: true,
  match: { published: true },
});
```

```js
async function getProfile(userId) {
  return User.findById(userId)
    .select('name bio createdAt')
    .populate({
      path: 'posts',
      match: { published: true },
      select: 'title slug createdAt',
      options: { sort: { createdAt: -1 }, limit: 5 },
    })
    .populate('postCount')
    .orFail();
}

const profile = await getProfile(id);
profile.toJSON();   // { name, bio, posts: [...5 newest], postCount: 42, … }  (toJSON has virtuals: true)
```

`options.limit` is fine here because there's only **one** parent. For many
parents, use `perDocumentLimit`.

---

### 4. Many Parents, Limited Children: `perDocumentLimit`

```js
// Each author with their 3 latest posts
const authors = await User.find({ role: 'admin' })
  .select('name')
  .populate({
    path: 'posts',
    select: 'title',
    options: { sort: { createdAt: -1 } },
    perDocumentLimit: 3,
  });
```

| Option | Limits |
|---|---|
| `options.limit: 3` | 3 posts **in total** across all authors (usually a bug) |
| `perDocumentLimit: 3` | 3 posts **per author** (one query per author) |

---

### 5. Filtering Populated Data (and Its Limits)

```js
// Tags only if they're "featured"; others become null / are removed
await Post.find().populate({ path: 'tags', match: { featured: true }, select: 'name' });
```

`match` filters the **populated** documents, not the parents. To get "posts
whose author is an admin", query the child's field or use `$lookup`:

```js
// Two steps with populate-style thinking
const adminIds = await User.find({ role: 'admin' }).distinct('_id');
const posts = await Post.find({ author: { $in: adminIds }, published: true }).populate('author', 'name');

// One query with $lookup: see Aggregation
```

---

### 6. Populating After the Fact

```js
const post = await Post.create({ title: 'New', author: req.user.id, tags });
await post.populate([{ path: 'author', select: 'name' }, { path: 'tags', select: 'name' }]);
res.status(201).json(post);     // respond with names, not bare ids
```

---

### 7. Polymorphic References: Notifications with `refPath`

```js
const notificationSchema = new Schema(
  {
    user:        { type: Schema.Types.ObjectId, ref: 'User', required: true, index: true },
    type:        { type: String, enum: ['comment', 'follow'], required: true },
    target:      { type: Schema.Types.ObjectId, required: true, refPath: 'targetModel' },
    targetModel: { type: String, required: true, enum: ['Post', 'User'] },
    read:        { type: Boolean, default: false },
  },
  { timestamps: true }
);
export const Notification = model('Notification', notificationSchema);

await Notification.create({ user: authorId, type: 'comment', target: postId, targetModel: 'Post' });
await Notification.create({ user: them, type: 'follow', target: me, targetModel: 'User' });

const inbox = await Notification.find({ user: me, read: false })
  .sort({ createdAt: -1 })
  .populate('target', 'title name')   // Post → title, User → name
  .lean();
```

---

### 8. Handling Deleted References

```js
const posts = await Post.find().populate('author', 'name').lean();
posts.map((p) => ({ ...p, author: p.author ?? { name: '[deleted]' } }));
```

A populated ref whose document is gone comes back as `null` (and missing
items are dropped from arrays). There are no foreign keys, so clean up
references when you delete (see [Relationships](04_relationships.md)).

---

### 9. The N+1 Trap

```js
// ❌ 1 + N queries: 21 queries for 20 posts
const posts = await Post.find().limit(20);
for (const p of posts) {
  p.authorName = (await User.findById(p.author)).name;
}

// ✅ 2 queries
const posts = await Post.find().limit(20).populate('author', 'name');
```

The same happens with populate **inside a loop**, or GraphQL resolvers
that populate per item. Batch with one populate (or DataLoader). See
[N+1 Problem](../../09-advanced/01_n-plus-one-problem.md).

---

### When to Stop Populating

| Symptom | Better option |
|---|---|
| You populate the same small fields on every read | **Extended reference**: copy them into the document |
| You need to filter/sort parents by child fields | **`$lookup`** in an aggregation ([Aggregation](06_aggregation.md)) |
| Populated data is always read with the parent and bounded | **Embed** it |
| Deep chains (`a.b.c.d`) | Rethink the model; each level is another query |

---

### Interview-Ready Summary

- Feed: `populate('author', 'name')` + `populate('tags', 'name')` + `.lean()`
  gives **1 + one query per path**, not per document.
- Nested paths (`comments.author`) are batched into one `$in` query.
- Profiles use **virtual populate** (`posts`, `postCount` with `count: true`);
  use **`perDocumentLimit`** for per-parent limits across many parents.
- `match` filters populated docs, not parents; use `$in` on ids or `$lookup`
  instead.
- `refPath` handles polymorphic targets; deleted refs become `null`.
  Always `select` populated fields and never populate in loops.

Next: [Aggregation](06_aggregation.md)
