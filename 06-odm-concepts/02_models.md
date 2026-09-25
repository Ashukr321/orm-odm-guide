## Models

### What Is a Model

A **model** is a schema compiled against a collection. It's a class: the
model itself runs queries (`User.find()`), and its instances are
**documents** (`new User({...})`).

```js
const { Schema, model } = require('mongoose');

const userSchema = new Schema({ email: String, name: String });
const User = model('User', userSchema);

await User.create({ email: 'ashu@example.com' });   // model = queries
const user = new User({ email: 'riya@example.com' }); // instance = document
await user.save();
```

```
Schema  ── model('User', schema) ──►  Model (class)  ──►  "users" collection
 shape                                 static API          documents
```

---

### Model Names and Collection Names

Mongoose lower-cases and **pluralizes** the model name to get the
collection name.

| Model name | Collection |
|---|---|
| `User` | `users` |
| `Person` | `people` |
| `BlogPost` | `blogposts` |
| `Category` | `categories` |

Set it explicitly to avoid surprises:

```js
const Post = model('Post', postSchema, 'blog_posts');   // 3rd argument
// or in the schema: new Schema({...}, { collection: 'blog_posts' })
```

See [Collections](04_collections.md).

---

### The Model API

**Static (query) methods on the model:**

| Category | Methods |
|---|---|
| Create | `create`, `insertMany`, `new Model()` + `save()` |
| Read | `find`, `findById`, `findOne`, `countDocuments`, `estimatedDocumentCount`, `exists`, `distinct` |
| Update | `updateOne`, `updateMany`, `findByIdAndUpdate`, `findOneAndUpdate`, `replaceOne` |
| Delete | `deleteOne`, `deleteMany`, `findByIdAndDelete`, `findOneAndDelete` |
| Bulk / advanced | `bulkWrite`, `aggregate`, `watch` (change streams) |
| Indexes | `syncIndexes`, `createIndexes`, `listIndexes`, `diffIndexes` |

```js
const post = await Post.create({ title: 'Hello', author: userId });
const drafts = await Post.find({ published: false }).sort('-createdAt').limit(20);
const exists = await User.exists({ email: 'ashu@example.com' });   // { _id } or null
await Post.updateOne({ _id: post._id }, { $set: { published: true } });
await Post.deleteMany({ author: userId });
```

Hands-on: [Mongoose CRUD](../07-odm-examples/mongoose/03_crud.md)

---

### Adding Behavior: Statics, Methods, Query Helpers, Virtuals

Define them on the **schema, before** calling `model()`.

```js
// Static: called on the model
userSchema.statics.findByEmail = function (email) {
  return this.findOne({ email: email.toLowerCase() });
};

// Instance method: called on a document
userSchema.methods.comparePassword = function (plain) {
  return bcrypt.compare(plain, this.password);
};

// Query helper: chainable on queries
postSchema.query.byAuthor = function (authorId) {
  return this.where({ author: authorId });
};

// Virtual: computed property
userSchema.virtual('isAdmin').get(function () {
  return this.role === 'admin';
});

const User = model('User', userSchema);
const Post = model('Post', postSchema);

const user = await User.findByEmail('ASHU@example.com').select('+password');
await user.comparePassword('secret');
const posts = await Post.find().byAuthor(user._id).sort('-createdAt');
```

| Kind | Defined with | Called on | `this` is |
|---|---|---|---|
| Static | `schema.statics.x` | `User.x()` | The model |
| Method | `schema.methods.x` | `user.x()` | The document |
| Query helper | `schema.query.x` | `User.find().x()` | The query |
| Virtual | `schema.virtual('x')` | `user.x` | The document |

---

### Class Syntax with `loadClass`

```js
class UserClass {
  get displayName() { return this.name ?? this.email.split('@')[0]; }   // virtual
  isAdmin() { return this.role === 'admin'; }                           // method
  static findByEmail(email) { return this.findOne({ email }); }         // static
}

userSchema.loadClass(UserClass);
const User = model('User', userSchema);
```

---

### Discriminators: Several Models, One Collection

Store related types with different fields in the **same collection**,
told apart by a key (`kind` here).

```js
const eventSchema = new Schema({ at: { type: Date, default: Date.now } }, { discriminatorKey: 'kind' });
const Event = model('Event', eventSchema);

const Click = Event.discriminator('Click', new Schema({ url: String }));
const Purchase = Event.discriminator('Purchase', new Schema({ amount: Number, orderId: String }));

await Click.create({ url: '/pricing' });          // { kind: 'Click', url: '/pricing', at }
await Purchase.create({ amount: 499, orderId: 'A1' });

await Event.find();      // all events
await Purchase.find();   // automatically adds { kind: 'Purchase' }
```

This is the document-database version of **single-table inheritance**.

---

### Models and Connections

`mongoose.model()` registers the model on the **default connection**. For
several databases, create models on each connection:

```js
const main = mongoose.createConnection(process.env.MAIN_URI);
const analytics = mongoose.createConnection(process.env.ANALYTICS_URI);

const User = main.model('User', userSchema);
const PageView = analytics.model('PageView', pageViewSchema);
```

**Hot reload (Next.js, nodemon):** re-running `model('User', …)` throws
`OverwriteModelError`. Reuse the existing model:

```js
const User = mongoose.models.User || model('User', userSchema);
```

---

### Dropping to the Driver

Every model exposes the raw driver collection:

```js
await User.collection.insertOne({ email: 'raw@example.com' });  // no schema, no hooks, no casting
const stats = await User.db.db.command({ collStats: 'users' });
```

Use it for special commands or bulk work. Remember that it **skips**
validation, defaults, casting and middleware.

---

### TypeScript Models

```ts
import { Schema, model, Model } from 'mongoose';

interface IUser { email: string; name?: string; role: 'user' | 'admin' }
interface IUserMethods { isAdmin(): boolean }
type UserModel = Model<IUser, {}, IUserMethods>;

const userSchema = new Schema<IUser, UserModel, IUserMethods>({
  email: { type: String, required: true },
  name:  String,
  role:  { type: String, enum: ['user', 'admin'], default: 'user' },
});
userSchema.method('isAdmin', function isAdmin() { return this.role === 'admin'; });

export const User = model<IUser, UserModel>('User', userSchema);
const u = await User.findOne();
u?.isAdmin();   // typed
```

For class-first definitions, see **Typegoose**.

---

### Interview-Ready Summary

- A **model** is a schema compiled against a collection. The model runs queries;
  its instances are documents.
- Collection names default to the **lower-cased plural** of the model name;
  set `collection` explicitly if needed.
- Add **statics** (model), **methods** (document), **query helpers** (query) and
  **virtuals** on the schema **before** calling `model()`.
- **Discriminators** store several related models in one collection with a
  discriminator key.
- Use `connection.model()` for multiple databases, `mongoose.models.X ||` for hot
  reload, and `Model.collection` for raw driver access (no schema).
