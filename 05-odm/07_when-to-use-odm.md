## When to Use an ODM

### The Short Answer

Use an ODM when you've **chosen a document database** (usually MongoDB)
and your app code benefits from a **consistent schema, validation and
relationships**, which is almost always. The bigger question is usually
**"MongoDB or a relational database?"** Once that's settled, an ODM like
Mongoose is the default for app code.

Background: [SQL vs NoSQL](../01-database-foundation/sql-vs-nosql.md)

---

### Good Fit: Use a Document Database + ODM

| Scenario | Why it fits |
|---|---|
| Data is naturally **nested / hierarchical** | Orders with items, posts with comments, CMS content blocks |
| Documents are **read as a whole** | One read returns everything the screen needs |
| **Varying shapes** per record | Product catalogs with category-specific attributes |
| **Fast-changing requirements** | Add fields without migrations |
| **High write volume / horizontal scale** | Sharding is built into MongoDB |
| Event logs, activity feeds, IoT readings | Append-heavy, schema varies over time |
| JavaScript/JSON end to end | Documents map directly to API responses |

```js
// A product catalog where each category has different attributes
const productSchema = new Schema({
  name: { type: String, required: true },
  price: { type: Number, required: true, min: 0 },
  category: { type: String, required: true, index: true },
  attributes: Schema.Types.Mixed,     // { ram: '16GB' } for laptops, { size: 'M' } for shirts
  variants: [{ sku: String, color: String, stock: Number }],
});
```

---

### Poor Fit: Prefer a Relational Database + ORM

| Scenario | Why a document DB struggles |
|---|---|
| Many-to-many relationships everywhere | Lots of references, `populate` and `$lookup` |
| **Strong integrity** requirements (finance, inventory ledgers) | No foreign keys; multi-document transactions are costlier |
| Heavy **ad-hoc reporting** and complex joins | SQL is far more expressive |
| Data is **highly normalized** and shared | Embedding duplicates data; references become joins |
| The team knows SQL and the domain is tabular | You'd be fighting the tool |

---

### ODM vs Raw Driver (Once You've Picked MongoDB)

| Use the **ODM** (Mongoose) when | Use the **driver** directly when |
|---|---|
| Building an app / API with business rules | Writing a tiny script or one-off migration |
| Several developers touch the same collections | Running bulk ETL on millions of documents |
| You want validation, hooks, populate, virtuals | Every millisecond matters (hot path) |
| Data must have a consistent shape | Data is truly schemaless (raw event capture) |

You can mix them: Mongoose exposes `Model.collection` (the raw driver
collection), plus `.lean()`, `bulkWrite()` and `aggregate()` for the
heavy lifting.

---

### Decision Checklist

| Question | Leans document DB + ODM | Leans relational + ORM |
|---|---|---|
| Is data mostly nested and read together? | Yes | No, it's spread across many entities |
| Do record shapes vary a lot? | Yes | Uniform |
| Do you need joins across many entities? | Rarely | Often |
| Are strict integrity/transactions central? | Nice to have | Critical |
| Will you need horizontal write scaling? | Likely | Unlikely |
| Is heavy ad-hoc reporting required? | Occasional | Frequent |
| Does the schema change often? | Yes | Stable |

Mostly left column → MongoDB + Mongoose. Mostly right column → PostgreSQL
+ an ORM. Mixed → PostgreSQL with `JSONB` columns is often a great middle
ground.

---

### Which ODM?

| You want… | Pick |
|---|---|
| The standard, most documented Node.js ODM | **Mongoose** |
| Mongoose with TypeScript classes/decorators | **Typegoose** |
| A schema-first typed client across SQL and MongoDB | **Prisma** (check its MongoDB support for your version) |
| Minimal overhead with DB-level JSON Schema validation | **Papr** |
| Python (async, Pydantic) | **Beanie** or **ODMantic** |
| Java / Spring | **Spring Data MongoDB** |

Hands-on: [Mongoose Setup](../07-odm-examples/mongoose/01_setup.md)

---

### Rules of Thumb for Using an ODM Well

1. **Model around read patterns**: embed what's read together, reference
   what's large, unbounded or shared.
2. **Never let arrays grow without limit** (the 16 MB document cap and slow
   updates). Use the bucket pattern or a separate collection.
3. **Index every field you filter or sort on**; check with `explain()`.
   See [Indexing](../09-advanced/03_indexing.md).
4. **Use `.lean()`** for read-only API responses.
5. **Populate sparingly**, select only needed fields, or use `$lookup`.
6. **Turn on `runValidators`** for update queries, and know which hooks fire
   for which operations.
7. **Add `$jsonSchema` validation** if other services write to the same data.
8. **Use transactions** (replica set required) when several documents must
   change together.
9. **Version your document shapes** (`schemaVersion`) and keep backfill
   scripts in git.

---

### Interview-Ready Summary

- The real decision is **document vs relational database**. Once you pick
  MongoDB, an ODM is the default for app code.
- **Good fit**: nested data read as a whole, varying shapes, fast-changing
  schemas, horizontal scale, event and catalog data.
- **Poor fit**: many-to-many-heavy domains, strict integrity/ledgers, heavy
  ad-hoc reporting. Use PostgreSQL + an ORM (or `JSONB`).
- Use the **driver** for scripts, bulk ETL and hot paths; mix it with Mongoose
  via `.lean()`, `bulkWrite()`, `aggregate()`.
- Use it well: model for reads, bound arrays, index, `lean`, populate
  sparingly, `runValidators`, transactions and schema versioning.
