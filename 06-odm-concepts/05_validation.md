## Validation

### Where Validation Happens

```
Request body ──► API validation (zod/joi)  ──► Mongoose schema validation ──► MongoDB
                  shape of the request           shape & rules of the data      unique indexes,
                  (400 Bad Request)              (ValidationError)              $jsonSchema (optional)
```

Mongoose validation runs **in your app, before** anything is sent to
MongoDB. It's defined in the schema and runs on `save()` / `create()`
(and on updates if you opt in).

---

### Built-in Validators

| Type | Validators |
|---|---|
| All types | `required` |
| `String` | `enum`, `match`, `minLength`, `maxLength` |
| `Number` | `min`, `max`, `enum` |
| `Date` | `min`, `max` |

```js
const userSchema = new Schema({
  email: {
    type: String,
    required: [true, 'Email is required'],
    match: [/^\S+@\S+\.\S+$/, 'Invalid email'],
    lowercase: true,
    trim: true,
  },
  username: { type: String, minLength: 3, maxLength: 20 },
  age:      { type: Number, min: [13, 'Must be at least 13, got {VALUE}'] },
  role:     { type: String, enum: { values: ['user', 'admin'], message: '{VALUE} is not a valid role' } },
});
```

Message templates: `{VALUE}`, `{PATH}`, `{MIN}`, `{MAX}` and similar.

**Conditional required:**

```js
publishedAt: {
  type: Date,
  required: function () { return this.published; },   // required only when published
},
```

---

### Custom Validators

```js
const postSchema = new Schema({
  tags: {
    type: [String],
    validate: {
      validator: (arr) => arr.length <= 10,
      message: 'A post can have at most 10 tags',
    },
  },
  slug: {
    type: String,
    validate: {
      validator: (v) => /^[a-z0-9-]+$/.test(v),
      message: (props) => `${props.value} is not a valid slug`,
    },
  },
});
```

**Async validator** (return a promise):

```js
username: {
  type: String,
  validate: {
    validator: async function (v) {
      const taken = await this.constructor.exists({ username: v, _id: { $ne: this._id } });
      return !taken;
    },
    message: 'Username already taken',
  },
},
```

Async uniqueness checks are **racy**: two requests can pass at the same
time. Always back them with a **unique index**.

---

### `unique` Is NOT a Validator

`unique: true` creates a **unique index** in MongoDB. Duplicates are
rejected by the **database** with error code `11000`, not by Mongoose
validation.

```js
try {
  await User.create({ email: 'ashu@example.com' });
} catch (err) {
  if (err.code === 11000) {
    // err.keyValue → { email: 'ashu@example.com' }
    return res.status(409).json({ error: 'Email already registered' });
  }
  throw err;
}
```

The index must actually exist (built by `autoIndex` or `syncIndexes()`)
for this to work.

---

### Validation Errors

```js
try {
  await User.create({ email: 'nope', age: 5, role: 'root' });
} catch (err) {
  err.name;                      // 'ValidationError'
  Object.keys(err.errors);       // ['email', 'age', 'role']
  err.errors.age.message;        // 'Must be at least 13, got 5'
  err.errors.age.kind;           // 'min'
}
```

**Cast errors** happen when a value can't be converted to the path's type:

```js
await Post.findById('not-an-id');          // CastError: Cast to ObjectId failed
await User.create({ age: 'twenty' });      // ValidationError containing a CastError for 'age'
```

**Express error handler:**

```js
app.use((err, req, res, next) => {
  if (err.name === 'ValidationError') {
    return res.status(400).json({
      errors: Object.values(err.errors).map((e) => ({ field: e.path, message: e.message })),
    });
  }
  if (err.name === 'CastError') return res.status(400).json({ error: `Invalid ${err.path}` });
  if (err.code === 11000) return res.status(409).json({ error: 'Duplicate', fields: err.keyValue });
  next(err);
});
```

---

### Validation on Updates

By default, `updateOne`, `updateMany` and `findOneAndUpdate` **skip
validators**.

```js
await User.updateOne({ _id: id }, { age: 5 });                          // ❌ saved!
await User.updateOne({ _id: id }, { age: 5 }, { runValidators: true }); // ✅ ValidationError

mongoose.set('runValidators', true);   // turn it on globally
```

Update validators have limits:

- They only check paths **in the update**. `required` is checked only when
  you `$unset` a field.
- Inside a custom validator, `this` is the **query**, not the document
  (use `this.getUpdate()` if needed).

For full validation, load the document, change it and call `save()`.

---

### Running Validation Yourself

```js
const user = new User({ email: 'nope' });
await user.validate();          // throws ValidationError
user.validateSync();            // sync version: returns the error or undefined

user.invalidate('email', 'Blocked domain');   // mark a path invalid manually
```

**Skip validation** (rarely a good idea):

```js
await user.save({ validateBeforeSave: false });
```

---

### Validation Order

```
new User(data)
   │  cast values (CastError if impossible)
   │  apply defaults and setters (lowercase, trim)
   ▼
pre('validate')  →  run validators  →  post('validate')
   ▼
pre('save') → write to MongoDB → unique index check (E11000)
```

Setters like `lowercase` and `trim` run **before** validators, so `match`
sees the cleaned value.

---

### Database-Level Validation (`$jsonSchema`)

Mongoose validation doesn't protect against the Mongo shell, scripts or
other services. For shared data, add rules in MongoDB too:

```js
await mongoose.connection.db.command({
  collMod: 'users',
  validator: { $jsonSchema: { bsonType: 'object', required: ['email'],
    properties: { email: { bsonType: 'string' } } } },
});
```

See [Collections](04_collections.md).

---

### Interview-Ready Summary

- Mongoose validation runs **in the app before writes**, on `save()` / `create()`.
- Built-ins: `required`, `enum`, `match`, `minLength`/`maxLength`, `min`/`max`,
  plus **custom** (sync or async) validators.
- **`unique` is an index, not a validator**: handle error code **11000**.
- Errors: **`ValidationError`** (with `err.errors[path]`) and **`CastError`**;
  map them to 400/409 in one error handler.
- Update queries **skip validators** unless `runValidators: true`, and even then
  only check updated paths.
- Layer it: API validation (zod) → Mongoose schema → unique indexes / `$jsonSchema`.
