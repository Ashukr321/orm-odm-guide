## Middleware

### What Is Middleware

**Middleware** (also called **hooks**) are functions that run automatically
**before (`pre`)** or **after (`post`)** a Mongoose operation. Use them for
logic that must *always* happen: hashing passwords, generating slugs,
auditing, cascading deletes, soft-delete filters.

```js
userSchema.pre('save', async function () {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
});

userSchema.post('save', function (doc) {
  console.log(`User ${doc._id} saved`);
});
```

> Register middleware on the schema **before** calling `model()`. Hooks
> added afterwards are ignored.

---

### The Four Kinds of Middleware

| Kind | `this` refers to | Operations |
|---|---|---|
| **Document** | The document | `validate`, `save`, `updateOne`*, `deleteOne`* , `init` |
| **Query** | The query | `find`, `findOne`, `countDocuments`, `findOneAndUpdate`, `findOneAndDelete`, `updateOne`, `updateMany`, `deleteOne`, `deleteMany`, `replaceOne`, `distinct` |
| **Aggregate** | The aggregation object | `aggregate` |
| **Model** | The model | `insertMany`, `bulkWrite` |

\* `updateOne` and `deleteOne` are **query** middleware by default. To hook
the document methods (`doc.deleteOne()`), opt in:

```js
schema.pre('deleteOne', { document: true, query: false }, async function () {
  // this = the document being deleted
});
```

---

### Which Hook Fires for Which Call?

| You call | Hooks that run |
|---|---|
| `doc.save()`, `Model.create()` | `validate`, `save` (document) |
| `Model.find()` / `findOne()` / `findById()` | `find` / `findOne` (query) |
| `Model.findByIdAndUpdate()` | `findOneAndUpdate` (query) |
| `Model.updateOne()` | `updateOne` (query) |
| `Model.updateMany()` | `updateMany` (query) |
| `Model.findByIdAndDelete()` | `findOneAndDelete` (query) |
| `doc.deleteOne()` | `deleteOne` (document, if opted in) |
| `Model.deleteMany()` | `deleteMany` (query) |
| `Model.insertMany()` | `insertMany` (model); **not** `save` |
| `Model.aggregate()` | `aggregate` |
| `Model.bulkWrite()`, `Model.collection.*` | Only `bulkWrite` model hooks; raw driver: **none** |

The classic bug: a `pre('save')` password-hashing hook **doesn't run** when
the password is changed with `findByIdAndUpdate`.

---

### Document Middleware Examples

**Generate a slug:**

```js
postSchema.pre('save', function () {
  if (this.isModified('title')) {
    this.seo.slug = this.title.toLowerCase().trim().replace(/[^a-z0-9]+/g, '-');
  }
});
```

**Run before validation** (to fill fields validators depend on):

```js
postSchema.pre('validate', function () {
  if (this.published && !this.publishedAt) this.publishedAt = new Date();
});
```

**Cascade delete a user's posts:**

```js
userSchema.pre('deleteOne', { document: true, query: false }, async function () {
  await model('Post').deleteMany({ author: this._id });
});
```

---

### Query Middleware Examples

**Soft delete: hide deleted documents from every `find*` query:**

```js
postSchema.pre(/^find/, function () {
  if (this.getOptions().withDeleted) return;
  this.where({ deletedAt: null });
});

await Post.find();                                   // excludes deleted
await Post.find().setOptions({ withDeleted: true }); // includes them
```

**Always hash passwords, even in update queries:**

```js
userSchema.pre('findOneAndUpdate', async function () {
  const update = this.getUpdate();
  const pwd = update.password ?? update.$set?.password;
  if (pwd) this.setUpdate({ ...update, $set: { ...update.$set, password: await bcrypt.hash(pwd, 10) } });
});
```

**Timing / slow-query logging:**

```js
schema.pre(/^find/, function () { this._start = Date.now(); });
schema.post(/^find/, function () {
  const ms = Date.now() - this._start;
  if (ms > 200) console.warn(`Slow ${this.op} on ${this.model.modelName}: ${ms}ms`, this.getFilter());
});
```

---

### Aggregate Middleware

Aggregation pipelines don't go through query middleware. Hook them
separately:

```js
postSchema.pre('aggregate', function () {
  this.pipeline().unshift({ $match: { deletedAt: null } });   // soft delete for aggregates too
});
```

---

### `post` Middleware and Error Handling

`post` hooks receive the result:

```js
postSchema.post('findOneAndDelete', async function (doc) {
  if (doc) await Tag.updateMany({ name: { $in: doc.tags } }, { $inc: { postCount: -1 } });
});
```

**Error-handling middleware** (three parameters, `error` first) can
translate errors:

```js
userSchema.post('save', function (error, doc, next) {
  if (error.code === 11000) next(new Error('Email already registered'));
  else next(error);
});
```

**Abort an operation** from a `pre` hook by throwing:

```js
postSchema.pre('save', function () {
  if (this.tags.length > 10) throw new Error('Too many tags');
});
```

---

### Order and `this`

- Hooks run in the **order they were registered** (plugins included).
- Use **regular functions**, not arrow functions. Arrow functions don't get
  `this`.
- Hooks that return a promise (`async function`) are awaited. You don't need
  the old `next()` callback style.

```js
schema.pre('save', () => { this.x = 1; });        // ❌ `this` is not the document
schema.pre('save', function () { this.x = 1; });  // ✅
```

---

### Middleware vs Service Code

| Put it in middleware | Put it in a service function |
|---|---|
| Must happen on **every** write (hashing, slugs, timestamps) | Only happens in one use case (send welcome email) |
| Small and fast | Slow or external (HTTP, email, queues) |
| Pure data concerns | Business workflows with several steps |

Avoid network calls in hooks: they slow every save and are hard to test.
Emit an event or write to an outbox instead.

---

### Interview-Ready Summary

- Middleware = `pre` / `post` hooks on Mongoose operations. Register them
  **before** `model()`.
- Four kinds: **document** (`this` = doc), **query** (`this` = query),
  **aggregate** and **model** (`insertMany`, `bulkWrite`).
- `save` hooks don't run for update queries, `insertMany` or the raw driver.
  Know which call triggers which hook.
- Common uses: password hashing, slugs, soft-delete filters (`pre(/^find/)`),
  cascades, audit and timing logs.
- Throw in `pre` to abort; use error-handling `post` hooks to translate errors;
  use regular functions for `this`; keep slow side effects out of hooks.
