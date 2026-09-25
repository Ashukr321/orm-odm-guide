## Mongoose Schema

### The Blog Models

Three models, three collections. Comments are **embedded** in posts
(always read with the post, bounded in size); authors and tags are
**referenced** (shared and queried on their own).

```
users  { _id, email, name, password, role, bio, createdAt, updatedAt }
posts  { _id, title, slug, body, author → users, tags → [tags], published,
         views, comments: [{ _id, body, author → users, createdAt }], createdAt, updatedAt }
tags   { _id, name, postCount }
```

Concepts: [Schemas](../../06-odm-concepts/01_schemas.md) ·
[Collections](../../06-odm-concepts/04_collections.md)

---

### User

**src/models/user.js**

```js
import mongoose, { Schema, model } from 'mongoose';
import bcrypt from 'bcrypt';

const userSchema = new Schema(
  {
    email: {
      type: String,
      required: [true, 'Email is required'],
      unique: true,
      lowercase: true,
      trim: true,
      match: [/^\S+@\S+\.\S+$/, 'Invalid email'],
    },
    name:     { type: String, trim: true, maxLength: 80 },
    password: { type: String, required: true, minLength: 8, select: false },   // never returned by default
    role:     { type: String, enum: ['user', 'admin'], default: 'user' },
    bio:      { type: String, maxLength: 500, default: '' },
  },
  {
    timestamps: true,
    toJSON: {
      virtuals: true,
      transform: (_doc, ret) => { delete ret.password; delete ret.__v; return ret; },
    },
  }
);

// Hash the password whenever it changes (create or save)
userSchema.pre('save', async function () {
  if (this.isModified('password')) this.password = await bcrypt.hash(this.password, 10);
});

userSchema.methods.comparePassword = function (plain) {
  return bcrypt.compare(plain, this.password);
};

userSchema.statics.findByEmail = function (email) {
  return this.findOne({ email: email.toLowerCase().trim() });
};

userSchema.virtual('displayName').get(function () {
  return this.name || this.email.split('@')[0];
});

// Reverse relation: user.posts without storing post ids on the user
userSchema.virtual('posts', { ref: 'Post', localField: '_id', foreignField: 'author' });

export const User = mongoose.models.User || model('User', userSchema);
```

```bash
npm install bcrypt
```

---

### Post (with Embedded Comments)

**src/models/post.js**

```js
import mongoose, { Schema, model } from 'mongoose';

const commentSchema = new Schema(
  {
    body:   { type: String, required: true, trim: true, maxLength: 1000 },
    author: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  },
  { timestamps: true }
);

const postSchema = new Schema(
  {
    title:     { type: String, required: true, trim: true, minLength: 3, maxLength: 200 },
    slug:      { type: String, unique: true, lowercase: true },
    body:      { type: String, default: '' },
    author:    { type: Schema.Types.ObjectId, ref: 'User', required: true },
    tags:      [{ type: Schema.Types.ObjectId, ref: 'Tag' }],
    published: { type: Boolean, default: false },
    publishedAt: Date,
    views:     { type: Number, default: 0, min: 0 },
    comments: {
      type: [commentSchema],
      validate: { validator: (a) => a.length <= 500, message: 'Too many comments on one post' },
    },
  },
  { timestamps: true, toJSON: { virtuals: true } }
);

// Indexes that match the app's queries
postSchema.index({ author: 1, createdAt: -1 });      // an author's posts, newest first
postSchema.index({ published: 1, createdAt: -1 });   // public feed
postSchema.index({ tags: 1, createdAt: -1 });        // tag pages
postSchema.index({ title: 'text', body: 'text' });   // search

// Slug from title, publishedAt when first published
postSchema.pre('validate', function () {
  if (this.isModified('title') && !this.slug) {
    this.slug = `${this.title.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '')}-${this._id.toString().slice(-6)}`;
  }
  if (this.published && !this.publishedAt) this.publishedAt = new Date();
});

postSchema.virtual('commentCount').get(function () {
  return this.comments?.length ?? 0;
});

postSchema.query.published = function () {
  return this.where({ published: true });
};

export const Post = mongoose.models.Post || model('Post', postSchema);
```

The `comments` array is capped by a validator. If comments could grow
without limit, move them to their own `comments` collection (see
[Relationships](04_relationships.md)).

---

### Tag

**src/models/tag.js**

```js
import mongoose, { Schema, model } from 'mongoose';

const tagSchema = new Schema({
  name:      { type: String, required: true, unique: true, lowercase: true, trim: true },
  postCount: { type: Number, default: 0, min: 0 },   // computed pattern: kept up to date on writes
});

export const Tag = mongoose.models.Tag || model('Tag', tagSchema);
```

---

### What Gets Stored

```js
await Post.create({ title: 'Hello Mongoose!', author: ashu._id, tags: [mongo._id], published: true });
```

```js
// db.posts.findOne() in mongosh
{
  _id: ObjectId('66f1c0e2a1b2c3d4e5f60718'),
  title: 'Hello Mongoose!',
  slug: 'hello-mongoose-f60718',
  body: '',
  author: ObjectId('66f1c0e2a1b2c3d4e5f60701'),
  tags: [ ObjectId('66f1c0e2a1b2c3d4e5f60705') ],
  published: true,
  publishedAt: ISODate('2026-09-25T10:00:00.000Z'),
  views: 0,
  comments: [],
  createdAt: ISODate('2026-09-25T10:00:00.000Z'),
  updatedAt: ISODate('2026-09-25T10:00:00.000Z'),
  __v: 0
}
```

- `slug`, `publishedAt`, `views`, `comments` and timestamps come from the
  schema (defaults and hooks).
- Virtuals (`commentCount`) are **not** stored.

---

### Checking the Schema in Action

```js
// Casting: strings become the declared types
await Post.find({ author: '66f1c0e2a1b2c3d4e5f60701', published: 'true' });

// Strict mode: unknown fields are dropped
const p = await Post.create({ title: 'Typo test', author: ashu._id, pubished: true });
p.pubished;   // undefined; never saved

// Validation
await Post.create({ title: 'Hi' }).catch((e) => console.log(Object.keys(e.errors)));
// ['title', 'author']   → title too short, author missing

// select: false
const u = await User.findByEmail('ashu@example.com');
u.password;                                                   // undefined
const withPwd = await User.findByEmail('ashu@example.com').select('+password');
await withPwd.comparePassword('password123');                 // true
```

---

### TypeScript Version (Optional)

```ts
import { Schema, model, InferSchemaType, HydratedDocument } from 'mongoose';

const tagSchema = new Schema({
  name: { type: String, required: true, unique: true, lowercase: true },
  postCount: { type: Number, default: 0 },
});

export type Tag = InferSchemaType<typeof tagSchema>;   // { name: string; postCount: number }
export type TagDoc = HydratedDocument<Tag>;
export const TagModel = model('Tag', tagSchema);
```

---

### Evolving the Schema

Adding a field needs no migration: old documents simply don't have it,
and defaults apply when they're read.

```js
// v2: add a reading-time estimate
readingMinutes: { type: Number, default: 1 },
```

For renames or reshapes, backfill once, then remove the old field:

```js
await Post.updateMany({ summary: { $exists: true } }, [{ $set: { excerpt: '$summary' } }, { $unset: 'summary' }]);
```

(The array form is an **aggregation-pipeline update**, which can reference
other fields with `$field`. Mongoose's strict mode may drop fields that
are no longer in the schema, so run such scripts with the raw collection
or `strict: false` if needed.)

---

### Interview-Ready Summary

- `User`: unique lower-cased email, `select: false` password hashed in a
  `pre('save')` hook, `toJSON` transform, statics, methods and a virtual
  `posts` relation.
- `Post`: embedded, **capped** comment subdocuments; `author`/`tags` as
  **references**; compound, multikey and text **indexes** matching real queries;
  slug/publishedAt set in `pre('validate')`.
- `Tag`: unique name plus a **computed** `postCount`.
- Schemas give **casting**, **strict mode**, **validation** and defaults;
  virtuals aren't stored.
- New fields need no migration; renames need a one-off backfill.

Next: [CRUD](03_crud.md)
