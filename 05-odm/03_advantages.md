## Advantages of ODMs

### At a Glance

| Advantage | One-line explanation |
|---|---|
| Schema & structure | One definition of each document's shape |
| Validation | Bad data is rejected before it reaches the database |
| Casting | Values are converted to the declared types |
| Defaults & timestamps | Common fields filled in automatically |
| Relationships | `populate()` resolves references for you |
| Middleware | Hooks for hashing, slugs, auditing, cascades |
| Rich model API | Methods, statics, virtuals, query helpers |
| Flexible schema evolution | Add fields without migrations |
| Productivity & consistency | Less boilerplate, one style across the team |
| TypeScript support | Typed models (inferred types, Typegoose) |

---

### 1. Schema and Structure

```js
const postSchema = new Schema({
  title:     { type: String, required: true, trim: true, maxlength: 200 },
  slug:      { type: String, unique: true },
  body:      String,
  tags:      [String],
  published: { type: Boolean, default: false },
  author:    { type: Schema.Types.ObjectId, ref: 'User', required: true },
}, { timestamps: true });
```

Anyone on the team can open one file and see exactly what a post looks
like. **Strict mode** (on by default) silently drops fields that aren't in
the schema, so typos never reach the database.

---

### 2. Validation

```js
const userSchema = new Schema({
  email: { type: String, required: [true, 'Email is required'], match: /^\S+@\S+\.\S+$/ },
  age:   { type: Number, min: [13, 'Must be at least 13'] },
  role:  { type: String, enum: ['user', 'admin'] },
  username: {
    type: String,
    validate: {
      validator: (v) => /^[a-z0-9_]{3,20}$/.test(v),
      message: (p) => `${p.value} is not a valid username`,
    },
  },
});

try {
  await User.create({ email: 'nope', age: 10 });
} catch (err) {
  err.name;                 // 'ValidationError'
  err.errors.email.message; // 'Path `email` is invalid (nope).'
  err.errors.age.message;   // 'Must be at least 13'
}
```

---

### 3. Automatic Casting

```js
await Post.find({ author: req.params.userId });   // string → ObjectId
await User.create({ age: '27' });                 // '27' → 27
await Event.create({ at: '2026-09-25' });         // string → Date
```

With the raw driver you'd write `new ObjectId(id)`, `Number(x)` and
`new Date(s)` everywhere, and forget one of them.

---

### 4. Defaults and Timestamps

```js
new Schema({
  status:  { type: String, default: 'draft' },
  views:   { type: Number, default: 0 },
  slug:    { type: String, default: () => crypto.randomUUID() },
}, { timestamps: true });   // createdAt + updatedAt managed for you
```

---

### 5. Relationships with `populate()`

```js
const post = await Post.findById(id)
  .populate('author', 'name email')
  .populate({ path: 'comments.author', select: 'name' });
```

One line replaces "collect IDs → `$in` query → map results back". See
[Population](../06-odm-concepts/07_population.md).

---

### 6. Middleware (Hooks)

```js
userSchema.pre('save', async function () {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
});

postSchema.pre('save', function () {
  if (this.isModified('title')) this.slug = slugify(this.title);
});

userSchema.post('findOneAndDelete', async function (doc) {
  if (doc) await Post.deleteMany({ author: doc._id });   // cascade
});
```

Logic that must *always* run lives in one place. See
[Middleware](../06-odm-concepts/06_middleware.md).

---

### 7. A Rich Model API

```js
// Instance method
userSchema.methods.comparePassword = function (plain) {
  return bcrypt.compare(plain, this.password);
};

// Static
userSchema.statics.findByEmail = function (email) {
  return this.findOne({ email: email.toLowerCase() });
};

// Virtual (computed, not stored)
userSchema.virtual('displayName').get(function () {
  return this.name ?? this.email.split('@')[0];
});

// Query helper
postSchema.query.published = function () {
  return this.where({ published: true });
};

const user = await User.findByEmail('ASHU@example.com');
await user.comparePassword('secret');
const posts = await Post.find().published().sort('-createdAt');
```

---

### 8. Flexible Schema Evolution

Adding a field needs **no migration**. Old documents just don't have it
yet, and defaults apply when they're loaded.

```js
// v2 of the schema
bio: { type: String, default: '' },
```

For bigger shape changes, write a one-off backfill (`updateMany`) or read
both shapes during a transition. That's still much lighter than
`ALTER TABLE` on a huge table.

---

### 9. Productivity and Consistency

| Raw driver | Mongoose |
|---|---|
| `db.collection('users').findOne({ _id: new ObjectId(id) })` | `User.findById(id)` |
| Manual `$in` lookups | `.populate('author')` |
| Validate in every route | Validate in the schema |
| `updatedAt: new Date()` on every update | `timestamps: true` |

---

### 10. TypeScript Support

Mongoose infers document types from the schema:

```ts
import { Schema, model, InferSchemaType } from 'mongoose';

const userSchema = new Schema({ email: { type: String, required: true }, age: Number });
type User = InferSchemaType<typeof userSchema>;   // { email: string; age?: number | null }

const UserModel = model('User', userSchema);
const u = await UserModel.findOne();
u?.email;   // string
```

Or define models as classes with **Typegoose**.

---

### Interview-Ready Summary

- An ODM gives a schemaless database an **app-level schema**: structure,
  strict mode, **validation**, **casting**, **defaults** and **timestamps**.
- **`populate()`** resolves references; **middleware** centralizes logic
  that must always run.
- **Methods, statics, virtuals and query helpers** make models expressive.
- Schemas evolve **without migrations**, and TypeScript types can be
  inferred from the schema.
- These benefits have costs. See [Disadvantages](04_disadvantages.md).
