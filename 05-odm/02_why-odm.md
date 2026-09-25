## Why ODM

### The Short Answer

MongoDB lets you store **anything**. Your application, however, expects a
`User` to always have an `email`, a numeric `age` and a `createdAt`. An ODM
puts that contract **in the app**, so every write is checked, cast and
shaped the same way.

> People use an ODM to get **structure and safety** on top of a flexible
> database, without giving up the flexibility.

---

### Life Without an ODM

With only the `mongodb` driver, the first few queries are pleasant. Then
the data starts to drift.

```js
// Developer A
await db.collection('users').insertOne({ email: 'a@x.com', createdAt: new Date() });

// Developer B, six months later
await db.collection('users').insertOne({ Email: 'b@x.com', created: Date.now(), age: '31' });

// Now every read has to defend against both shapes
const users = await db.collection('users').find().toArray();
users.forEach((u) => {
  const email = u.email ?? u.Email;
  const created = u.createdAt ?? new Date(u.created);
  const age = u.age !== undefined ? Number(u.age) : null;
});
```

| Pain | What happens in a real codebase |
|---|---|
| **Shape drift** | The same collection holds several document shapes |
| **No validation** | Invalid emails, negative prices, missing required fields |
| **Type mix-ups** | `'31'` vs `31`, strings vs `ObjectId`, strings vs `Date` |
| **Manual "joins"** | Collect IDs → `$in` query → stitch results, every time |
| **Duplicated logic** | Hashing passwords, setting timestamps, slugs in many places |
| **ObjectId handling** | `new ObjectId(id)` everywhere; bad ids throw deep in the code |

---

### Life With an ODM

```js
const userSchema = new Schema(
  {
    email: { type: String, required: true, unique: true, lowercase: true, trim: true,
             match: /^\S+@\S+\.\S+$/ },
    name:  { type: String, trim: true },
    age:   { type: Number, min: 13 },
    role:  { type: String, enum: ['user', 'admin'], default: 'user' },
  },
  { timestamps: true }                           // createdAt / updatedAt
);

userSchema.pre('save', async function () {
  if (this.isModified('password')) this.password = await hash(this.password);
});

const User = model('User', userSchema);

await User.create({ email: ' B@X.com ', age: '31', Email: 'ignored' });
// stored: { email: 'b@x.com', age: 31, role: 'user', createdAt, updatedAt }
// 'Email' is dropped (strict mode); invalid data throws a ValidationError
```

Every document in `users` now has **one shape**, enforced in one place.

---

### The Seven Reasons Teams Choose an ODM

| # | Reason | What you get |
|---|---|---|
| 1 | **Schema in the app** | One definition of each document's shape |
| 2 | **Validation** | `required`, `min`/`max`, `enum`, `match`, custom validators |
| 3 | **Casting** | Strings to numbers, dates and `ObjectId`s automatically |
| 4 | **Relationships** | `ref` + `populate()` instead of hand-written `$in` lookups |
| 5 | **Middleware** | `pre`/`post` hooks for hashing, slugs, auditing, cascades |
| 6 | **Model API** | Instance methods, statics, virtuals, query helpers |
| 7 | **Productivity** | Less boilerplate; readable, consistent data access |

Details and trade-offs: [Advantages](03_advantages.md) ·
[Disadvantages](04_disadvantages.md)

---

### "But MongoDB Is Schemaless. Isn't a Schema the Opposite?"

No. Schemaless means the **database** doesn't force one shape. It doesn't
mean your data *has* no shape. You still get the flexibility:

- **Change the schema without migrations**: add a field with a default and
  old documents keep working.
- **Mixed / flexible fields where you want them**: `Schema.Types.Mixed` or
  `strict: false` for truly dynamic data.
- **Different document shapes on purpose**: discriminators (`Admin` and
  `Customer` in one `users` collection).

```js
const eventSchema = new Schema({
  type: { type: String, required: true },
  payload: Schema.Types.Mixed,          // anything goes here, deliberately
});
```

---

### Why Not Just Use the Driver?

Sometimes you should: tiny scripts, heavy bulk jobs, or services where
every millisecond counts. An ODM is not "the driver is bad". It is
"most of our app code benefits from a schema".

```
A typical MongoDB app
┌──────────────────────────────────────────────┬─────────┐
│  ~90%: CRUD, validation, populate, hooks     │  ~10%   │
│  → ODM (Mongoose)                            │ → driver│
│                                              │ / .lean │
└──────────────────────────────────────────────┴─────────┘
                           bulk writes, analytics pipelines, hot paths
```

Mongoose exposes the driver when you need it: `Model.collection`,
`Model.bulkWrite()`, `Model.aggregate()` and `.lean()`.

---

### Interview-Ready Summary

- MongoDB is schemaless, so without an app-level schema, collections drift
  into many shapes with invalid and mistyped data.
- An ODM adds **schemas, validation, casting, references + populate,
  middleware and a model API**, all defined in one place.
- It keeps MongoDB's flexibility: no migrations for new fields, `Mixed`
  types and discriminators where you want variety.
- Use the ODM for everyday app code and the driver (or `.lean()`,
  `bulkWrite`, `aggregate`) for bulk and hot paths.
