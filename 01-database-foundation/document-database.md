## Document Database

### What Is a Document Database

A **document database** stores data as self-contained **documents**. These
are JSON-like objects that can hold nested objects and arrays. It does not
spread data across rows in separate tables.

```
Relational:  users row + addresses rows + orders rows  → JOIN at read time
Document:    one user document with addresses/orders nested inside → read once
```

Documents are grouped into **collections**. Documents in the same collection
don't have to share the same fields. The schema is *flexible*, not *absent*.

Popular document databases:
- **MongoDB**: the most widely used. It stores **BSON** (binary JSON, with
  extra types like `Date`, `ObjectId`, `Decimal128`).
- **Firestore** (Firebase), **Couchbase**, **CouchDB**
- **Amazon DocumentDB** and **Azure Cosmos DB**, which offer
  MongoDB-compatible APIs

---

### Terminology: Relational vs Document

| Relational (SQL) | Document (MongoDB) |
|---|---|
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |
| Primary key | `_id` (auto-generated `ObjectId` by default) |
| JOIN | Embedding, or `$lookup` in aggregation |
| Foreign key | Reference (store the other document's `_id`) |
| Schema (DDL) | Optional validation (`$jsonSchema`) or an ODM like Mongoose |

---

### What a Document Looks Like

```js
{
  _id: ObjectId("66f2a1c4e4b0a1b2c3d4e5f6"),
  name: "Ashutosh",
  email: "ashu@example.com",
  addresses: [                                   // array of sub-documents
    { type: "home", city: "Patna",     pin: "800001" },
    { type: "work", city: "Bengaluru", pin: "560001" }
  ],
  preferences: { theme: "dark", newsletter: true }, // nested object
  createdAt: ISODate("2026-09-24T10:00:00Z")
}
```

The whole user, including addresses and preferences, comes back in **one
read** with no joins. That is the main benefit of a document database.

> `ObjectId` is 12 bytes, and its first 4 bytes are a creation timestamp.
> So `_id` values sort roughly by insert time, and you can extract the
> creation time from them.

---

### Data Modeling: Embed vs Reference

This is **the** key design decision in a document database. In SQL you
normalize by default. In a document DB you model around **how the data is
read**.

#### Embed when…
- The child data belongs to the parent and is read together with it
  (addresses in a user, line items in an order).
- The child list is **bounded** (a user has 1–5 addresses, not 1M).
- The child is rarely queried on its own.

#### Reference when…
- The related data is **shared** by many documents (a product referenced by
  many orders).
- The list grows **without bound** (comments on a viral post, logs, events).
- The related data changes often, and copying it everywhere would mean
  updating many documents.

```js
// Embed: order line items (bounded, read with the order, a snapshot of the price)
{
  _id: ObjectId("..."),
  userId: ObjectId("..."),            // reference: the user is shared and changes on its own
  items: [
    { productId: ObjectId("..."), name: "Keyboard", priceMinor: 249900, qty: 1 }
  ],
  status: "paid"
}

// Reference: comments live in their own collection (unbounded)
// comments: { _id, postId: ObjectId("..."), authorId, text, createdAt }
```

> **Hard limit:** a single document can't exceed **16 MB**. An array that
> keeps growing (the *unbounded array* anti-pattern) will eventually hit it.
> Long before that, it makes every read and update slower.

---

### Node.js: Working With Documents (native `mongodb` driver)

```js
const { MongoClient, ObjectId } = require('mongodb');

const client = new MongoClient(process.env.MONGO_URI);
const users = client.db('app_db').collection('users');

// Create
const { insertedId } = await users.insertOne({
  name: 'Ashutosh',
  email: 'ashu@example.com',
  addresses: [{ type: 'home', city: 'Patna' }],
  createdAt: new Date(),
});

// Read: query nested fields with dot notation
const inPatna = await users.find({ 'addresses.city': 'Patna' }).toArray();

// Update: operators change only the fields you name, not the whole document
await users.updateOne(
  { _id: insertedId },
  {
    $set: { 'preferences.theme': 'dark' },
    $push: { addresses: { type: 'work', city: 'Bengaluru' } },
  }
);

// Delete
await users.deleteOne({ _id: new ObjectId(id) });
```

Common update operators: `$set`, `$unset`, `$inc`, `$push`, `$pull`,
`$addToSet`. Always use them. Replacing the whole document can overwrite
changes other requests made at the same time.

#### Aggregation: the "SQL GROUP BY / JOIN" of MongoDB

```js
// Revenue per user for paid orders, with the user's name joined in
const report = await client.db('app_db').collection('orders').aggregate([
  { $match: { status: 'paid' } },
  { $unwind: '$items' },
  { $group: {
      _id: '$userId',
      revenueMinor: { $sum: { $multiply: ['$items.priceMinor', '$items.qty'] } },
  } },
  { $lookup: { from: 'users', localField: '_id', foreignField: '_id', as: 'user' } },
  { $sort: { revenueMinor: -1 } },
  { $limit: 10 },
]).toArray();
```

`$lookup` is a left outer join. It works, but if you need it on every hot
read, your data probably should have been embedded.

---

### Indexes

Without an index, MongoDB scans every document in the collection (a
`COLLSCAN`). The index types you will use most:

```js
await users.createIndex({ email: 1 }, { unique: true });          // single field + unique
await orders.createIndex({ userId: 1, createdAt: -1 });           // compound
await users.createIndex({ 'addresses.city': 1 });                 // multikey (indexes array elements)
await posts.createIndex({ title: 'text', body: 'text' });         // text search
await sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 }); // TTL: auto-delete
await orders.createIndex(                                         // partial: index only some docs
  { userId: 1 },
  { partialFilterExpression: { status: 'pending' } }
);
```

- **Compound index order: the ESR rule.** Put **E**quality fields first,
  then **S**ort fields, then **R**ange fields.
- Check a query with `.explain('executionStats')`. You want `IXSCAN`, not
  `COLLSCAN`.

---

### Schema Flexibility: Feature and Risk

Flexible schema means:
- ✅ You can add a field without a migration. Old documents just don't
  have it yet.
- ✅ You can store records of different shapes in one collection (products
  with different attributes).
- ❌ Nothing stops `emial` (typo), `age: "twenty"`, or a missing required
  field unless *you* add validation.

A senior developer adds the schema back where it matters:

```js
// Database-level validation: enforced for every client, not just your app
await db.createCollection('users', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['email', 'createdAt'],
      properties: {
        email:     { bsonType: 'string', pattern: '^.+@.+$' },
        createdAt: { bsonType: 'date' },
      },
    },
  },
});
```

Or use an **ODM** like Mongoose to define schemas, validation and middleware
in application code. See [ODM](../05-odm/01_what-is-odm.md).

---

### Consistency and Transactions

- **Single-document writes are atomic.** Updating several fields and arrays
  in one document is all-or-nothing. That is why good embedding removes the
  need for most transactions.
- **Multi-document ACID transactions** exist (replica sets since MongoDB
  4.0, sharded clusters since 4.2). They cost more than single-document
  writes, so use them for true cross-document invariants like money moving
  between accounts.

```js
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    const debit = await accounts.updateOne(
      { _id: from, balance: { $gte: amt } },     // only debit if funds are enough
      { $inc: { balance: -amt } },
      { session }
    );
    if (debit.modifiedCount === 0) throw new Error('Insufficient funds'); // aborts the transaction
    await accounts.updateOne({ _id: to }, { $inc: { balance: amt } }, { session });
  });
} finally {
  await session.endSession();
}
```

- **Write concern** controls how many replicas must confirm a write. The
  default since MongoDB 5.0 is `w: "majority"`, which is safe.
- **Read concern / read preference** controls whether reads can come from
  secondaries. Secondaries can be slightly behind the primary.

---

### Scaling

| Mechanism | What it does | Solves |
|---|---|---|
| **Replica set** | 1 primary + N secondaries holding copies of the data; automatic failover | High availability, read scaling |
| **Sharding** | Splits a collection across machines by a **shard key** | Data or write volume too big for one machine |

The shard key is very hard to change later. A bad one (for example, a
monotonically increasing `createdAt`) sends every new write to a single
shard. Pick a key with high cardinality that your queries filter on.

---

### When to Use a Document Database

**Good fit**
- Data is naturally hierarchical and read as a unit: product catalogs,
  user profiles, CMS content, settings.
- Records in the same collection have different shapes (e-commerce products
  with different attributes).
- Fast iteration where the shape changes often.
- High write volume with simple access patterns: event logging, IoT.

**Poor fit**
- Heavily relational data with many-to-many joins in every query.
- Many invariants across entities (double-entry accounting, inventory
  reservation). Doable with transactions, but SQL makes it easier.
- Heavy ad-hoc reporting and analytics across entities.

See [SQL vs NoSQL](sql-vs-nosql.md) for the full comparison.

---

### Common Mistakes

| Mistake | Fix |
|---|---|
| Modeling collections like SQL tables, then `$lookup` everywhere | Model around read patterns; embed what is read together |
| Unbounded arrays inside one document | Move the growing list to its own collection with a reference |
| No validation, so bad data piles up | `$jsonSchema` validator or a Mongoose schema |
| Replacing whole documents on update | Use `$set` / `$inc` / `$push` operators |
| No indexes, leading to `COLLSCAN` in production | Index every hot query path; check with `explain()` |
| New `MongoClient` per request (common in Next.js) | One client per process, cached on `globalThis`. See [Database Drivers](database-drivers.md) |
| Storing money as a floating-point `Number` | Integer minor units (paise/cents) or `Decimal128` |

---

### Interview-Ready Summary

- A document database stores **self-contained JSON-like documents** in
  **collections**. The schema is flexible, not absent.
- The core design question is **embed vs reference**. Embed data that is
  bounded and read together. Reference data that is shared or unbounded.
  Remember the **16 MB** document limit.
- **Single-document writes are atomic**, so good embedding removes most of
  the need for transactions. Multi-document transactions exist but cost
  more.
- Index hot queries (ESR rule for compound indexes). Check with
  `explain()`.
- Add schema back with `$jsonSchema` validation or an ODM like Mongoose.
- Scale reads and availability with **replica sets**, and data or writes
  with **sharding**. The shard key choice is close to permanent.
