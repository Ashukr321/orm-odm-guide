## Population

### What Is Population

**Population** replaces `ObjectId` references in a document with the
actual referenced documents. It's how Mongoose loads related data across
collections.

```js
const postSchema = new Schema({
  title:  String,
  author: { type: Schema.Types.ObjectId, ref: 'User' },   // stores only the id
});

const post = await Post.findById(id).populate('author');
post.author.name;   // 'Ashu', a full User document instead of an ObjectId
```

```
Before populate                          After populate
{ title: 'Hello',                        { title: 'Hello',
  author: ObjectId('66a…') }      →        author: { _id: '66a…', name: 'Ashu', email: … } }
```

---

### How It Works (It's Not a Join)

```
1. posts.find({ … })                                   → posts with author ids
2. collect distinct ids                                → [66a…, 66b…]
3. users.find({ _id: { $in: [66a…, 66b…] } })          → one extra query per populated path
4. match users to posts in Node.js
```

- One extra query **per path**, not per document, so it's not N+1.
- It still costs a round trip each time, and the matching happens in your app.
- For a single server-side query, use `$lookup`. See [Aggregation](08_aggregation.md).

---

### Basic Usage

```js
// One path
await Post.find().populate('author');

// Only some fields
await Post.find().populate('author', 'name email');       // or '-password'

// Several paths
await Post.findById(id).populate('author').populate('tags');
await Post.findById(id).populate(['author', 'tags']);

// Arrays of references
const postSchema = new Schema({ tags: [{ type: Schema.Types.ObjectId, ref: 'Tag' }] });
await Post.find().populate('tags', 'name');   // tags becomes an array of Tag documents
```

---

### Populate Options

```js
await User.findById(id).populate({
  path: 'posts',                       // (a virtual, see below)
  select: 'title createdAt',
  match: { published: true },          // filter the populated docs
  options: { sort: { createdAt: -1 } },
  perDocumentLimit: 5,                 // at most 5 posts per user
});
```

| Option | Effect |
|---|---|
| `path` | Which field to populate |
| `select` | Fields of the referenced documents |
| `match` | Filter populated documents (non-matches become `null` / are removed from arrays) |
| `options.sort` | Sort the populated documents |
| `perDocumentLimit` | Limit per parent document |
| `options.limit` | Limit for the **whole** populate query (usually not what you want) |
| `populate` | Nested populate |
| `model` | Override the model to use |

---

### Nested Population

```js
// Post → comments[].author (users)
await Post.findById(id).populate({ path: 'comments.author', select: 'name' });

// Post → author → company (two levels of references)
await Post.find().populate({
  path: 'author',
  select: 'name company',
  populate: { path: 'company', select: 'name' },
});
```

Each level adds another query. Deep nesting is a sign the data might be
better **embedded** or copied (extended reference).

---

### Virtual Populate (Reverse Relations)

Store the reference only on the **child** (`post.author`), but still load
`user.posts` without keeping an array of ids on the user.

```js
userSchema.virtual('posts', {
  ref: 'Post',
  localField: '_id',
  foreignField: 'author',
});

userSchema.virtual('postCount', {
  ref: 'Post',
  localField: '_id',
  foreignField: 'author',
  count: true,                // only the number
});

const user = await User.findById(id)
  .populate({ path: 'posts', select: 'title', options: { sort: { createdAt: -1 } } })
  .populate('postCount');

user.posts;       // Post[]
user.postCount;   // 12
```

Include virtuals in JSON with `toJSON: { virtuals: true }`.

| | Array of ids on the parent | Virtual populate |
|---|---|---|
| Storage | `user.posts: [ObjectId, …]` grows forever | Nothing stored on the user |
| Consistency | Must update both sides | Single source of truth (`post.author`) |
| Best for | Small, bounded lists | One-to-many that can grow |

---

### Dynamic References (`refPath`)

When a field can point to documents in different collections:

```js
const commentSchema = new Schema({
  body: String,
  on:      { type: Schema.Types.ObjectId, required: true, refPath: 'onModel' },
  onModel: { type: String, required: true, enum: ['Post', 'Video'] },
});

await Comment.find().populate('on');   // loads a Post or a Video per comment
```

---

### Populating an Existing Document

```js
const post = await Post.findById(id);
await post.populate('author');           // populates in place
post.populated('author');                // the original ObjectId (truthy if populated)
post.depopulate('author');               // back to the ObjectId
```

---

### Missing References

If the referenced document was deleted, the populated value is **`null`**
(or it's removed from an array). Nothing enforces referential integrity,
so handle it:

```js
const posts = await Post.find().populate('author', 'name');
posts.forEach((p) => console.log(p.author?.name ?? '[deleted user]'));
```

Clean up references with middleware (cascade deletes) or periodic jobs.

---

### Populate with `.lean()`

```js
const posts = await Post.find().populate('author', 'name').lean();
// plain objects, author is a plain object too: fastest option for API responses
```

---

### `populate()` vs `$lookup` vs Embedding

| | `populate()` | `$lookup` (aggregation) | Embedding |
|---|---|---|---|
| Queries | 1 + one per path | 1 | 1 |
| Where the "join" happens | Node.js | MongoDB server | No join needed |
| Filtering/sorting parents by child fields | ❌ | ✅ | ✅ |
| Mongoose casting, virtuals, documents | ✅ | ❌ (plain objects) | ✅ |
| Best for | Everyday reads | Reports, filters on related data | Data read together |

```js
// Need "posts whose author is an admin"? populate + match can't filter parents; use $lookup:
await Post.aggregate([
  { $lookup: { from: 'users', localField: 'author', foreignField: '_id', as: 'author' } },
  { $unwind: '$author' },
  { $match: { 'author.role': 'admin' } },
]);
```

Hands-on: [Mongoose Population](../07-odm-examples/mongoose/05_population.md)

---

### Interview-Ready Summary

- **Population** swaps `ObjectId` refs for documents via an extra
  `find({ _id: { $in } })` **per path**, merged in Node.js. It's not a join.
- Options: `select`, `match`, `options.sort`, **`perDocumentLimit`**, nested
  `populate`.
- **Virtual populate** loads reverse relations (`user.posts`) without storing
  id arrays; `count: true` returns just the number.
- `refPath` handles polymorphic references; deleted refs become **`null`**.
- Use **`$lookup`** to filter parents by related data, and **embedding** when
  data is always read together.
