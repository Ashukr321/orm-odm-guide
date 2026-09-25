## Schemas

### What Is a Schema

A **schema** describes the shape of the documents in a collection: which
fields exist, their types, defaults, validation rules, indexes, virtuals
and hooks. MongoDB itself doesn't require one. The schema lives in your
**application** (Mongoose), and every write goes through it.

```js
const { Schema } = require('mongoose');

const userSchema = new Schema(
  {
    email: { type: String, required: true, unique: true, lowercase: true, trim: true },
    name:  { type: String, trim: true },
    age:   { type: Number, min: 13 },
    role:  { type: String, enum: ['user', 'admin'], default: 'user' },
  },
  { timestamps: true }
);
```

A schema is only a **definition**. To query, compile it into a **model**.
See [Models](02_models.md).

> Examples target **Mongoose 8** and use the blog domain from this guide:
> `User`, `Post` (with embedded comments) and `Tag`.

---

### Schema Types

| SchemaType | Stored as | Notes |
|---|---|---|
| `String` | string | `lowercase`, `uppercase`, `trim`, `match`, `enum`, `minLength`, `maxLength` |
| `Number` | double (or int) | `min`, `max`, `enum` |
| `Boolean` | bool | Casts `'true'`, `1`, `'yes'` |
| `Date` | date | `min`, `max`; casts ISO strings and timestamps |
| `Schema.Types.ObjectId` | ObjectId | References; use with `ref` |
| `Schema.Types.Decimal128` | decimal128 | Money and exact decimals |
| `BigInt` | long | 64-bit integers |
| `Buffer` | binary | Files, hashes |
| `Map` | object | Arbitrary keys with typed values |
| `Schema.Types.UUID` | UUID (binary) | |
| `Schema.Types.Mixed` | anything | No casting or validation; use sparingly |
| `[Type]` | array | `[String]`, `[subSchema]`, `[{ type: ObjectId, ref: 'Tag' }]` |

```js
const productSchema = new Schema({
  price:  { type: Schema.Types.Decimal128, required: true },
  specs:  { type: Map, of: String },            // { ram: '16GB', cpu: 'M3' }
  extra:  Schema.Types.Mixed,
  photos: [String],
});
```

---

### Field Options

| Option | Purpose | Example |
|---|---|---|
| `type` | SchemaType | `type: String` |
| `required` | Must be present | `required: [true, 'Email is required']` |
| `default` | Value or function | `default: Date.now`, `default: () => []` |
| `unique` | Creates a **unique index** (not a validator) | `unique: true` |
| `index` | Creates an index | `index: true` |
| `select` | Exclude from queries by default | `select: false` (passwords) |
| `immutable` | Can't change after creation | `immutable: true` |
| `get` / `set` | Transform on read / write | `set: (v) => v.trim()` |
| `alias` | Alternative property name | `alias: 'fullName'` |
| `validate` | Custom validator | See [Validation](05_validation.md) |

```js
password:  { type: String, required: true, minLength: 8, select: false },
createdBy: { type: Schema.Types.ObjectId, ref: 'User', immutable: true },
```

---

### Nested Objects, Subdocuments and Arrays

```js
const commentSchema = new Schema(
  {
    body:   { type: String, required: true, maxLength: 1000 },
    author: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  },
  { timestamps: true }                           // each comment gets its own _id and timestamps
);

const postSchema = new Schema({
  title:    { type: String, required: true, trim: true },
  author:   { type: Schema.Types.ObjectId, ref: 'User', required: true, index: true },
  tags:     [{ type: String, lowercase: true }],   // array of primitives
  seo: {                                           // nested object (plain path, no _id)
    slug:        { type: String, unique: true },
    description: String,
  },
  comments: [commentSchema],                       // array of subdocuments
  published: { type: Boolean, default: false },
}, { timestamps: true });
```

| | Nested object (`seo: { … }`) | Subdocument (`[commentSchema]`) |
|---|---|---|
| Has its own `_id` | No | Yes (by default) |
| Own validators / hooks / timestamps | Via parent paths | Yes, its own schema |
| Good for | A group of related fields | Repeating items (comments, line items) |

> Keep embedded arrays **bounded**. A document can't exceed **16 MB**, and huge
> arrays make every update slower.

---

### Schema Options

```js
new Schema({ /* fields */ }, {
  timestamps: true,            // createdAt / updatedAt
  collection: 'blog_posts',    // exact collection name
  strict: true,                // drop fields not in the schema (default)
  versionKey: '__v',           // array version counter; false to disable
  optimisticConcurrency: true, // version check on every save
  toJSON:   { virtuals: true },
  toObject: { virtuals: true },
  minimize: true,              // don't store empty objects (default)
  autoIndex: true,             // build indexes on startup (turn off in production)
});
```

| `strict` value | Unknown field on save |
|---|---|
| `true` (default) | Silently dropped |
| `'throw'` | Throws an error |
| `false` | Saved as-is |

---

### Virtuals

Computed properties that are **not stored** in MongoDB.

```js
userSchema.virtual('displayName').get(function () {
  return this.name ?? this.email.split('@')[0];
});

postSchema.virtual('commentCount').get(function () {
  return this.comments.length;
});
```

Virtuals are not in `JSON.stringify` output unless `toJSON: { virtuals: true }`
is set, and they don't exist on `.lean()` results. Virtual **populate** is
covered in [Population](07_population.md).

---

### Methods, Statics and Query Helpers

```js
userSchema.methods.isAdmin = function () { return this.role === 'admin'; };
userSchema.statics.findByEmail = function (email) { return this.findOne({ email: email.toLowerCase() }); };
postSchema.query.published = function () { return this.where({ published: true }); };
```

These are defined on the schema but used on the model. See [Models](02_models.md).
Use **regular functions**, not arrow functions, so `this` works.

---

### Reusing Schemas: Plugins and `add`

```js
// A plugin: reusable schema behavior
function softDelete(schema) {
  schema.add({ deletedAt: { type: Date, default: null } });
  schema.query.notDeleted = function () { return this.where({ deletedAt: null }); };
  schema.methods.softDelete = function () { this.deletedAt = new Date(); return this.save(); };
}

postSchema.plugin(softDelete);
userSchema.plugin(softDelete);
```

---

### TypeScript: Inferring Types from the Schema

```ts
import { Schema, model, InferSchemaType } from 'mongoose';

const tagSchema = new Schema({ name: { type: String, required: true, unique: true } });
type Tag = InferSchemaType<typeof tagSchema>;   // { name: string }

export const TagModel = model('Tag', tagSchema);
```

---

### Interview-Ready Summary

- A **schema** defines document shape, types, defaults, validation, indexes,
  virtuals, methods and hooks. It's enforced by **Mongoose**, not MongoDB.
- Core SchemaTypes: String, Number, Boolean, Date, ObjectId, Decimal128, Map,
  Mixed, arrays. Use **Decimal128** for money and avoid Mixed where possible.
- Field options: `required`, `default`, `unique` (an index, not a validator),
  `select: false`, `immutable`, getters/setters.
- **Subdocuments** have their own schema and `_id`; nested objects are just
  grouped paths. Keep arrays bounded (16 MB cap).
- Schema options: `timestamps`, `strict`, `collection`, `versionKey`,
  `toJSON`. Reuse behavior with **plugins**.
