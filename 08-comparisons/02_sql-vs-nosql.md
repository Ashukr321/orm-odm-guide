## SQL vs NoSQL: Choosing for a Real System

### How This Page Differs from the Foundation Page

The fundamentals (the four NoSQL families, ACID vs BASE, CAP, scaling,
myths) are in [SQL vs NoSQL](../01-database-foundation/sql-vs-nosql.md).
This page is about **making the decision** for an actual product: scenario
by scenario, with the same feature built both ways and the operational
trade-offs that only show up in production.

---

### The Decision in One Table

| Question | Points to SQL | Points to NoSQL (document) |
|---|---|---|
| Is data highly related (many-to-many everywhere)? | ✅ | |
| Must multi-record writes be correct every time (money, stock)? | ✅ | |
| Will people ask new, ad-hoc questions of the data? | ✅ | |
| Is each record mostly read and written as one unit? | | ✅ |
| Do record shapes vary a lot, or change weekly? | | ✅ |
| Do you need horizontal write scaling beyond one big server? | | ✅ |
| Is the team stronger in SQL or in MongoDB? | Either way, it matters | |

Default when unsure: **PostgreSQL**. It handles relational data well,
`JSONB` covers document-shaped parts, and it scales further than most
products ever need.

---

### Scenario Verdicts

| System | Verdict | Reasoning |
|---|---|---|
| **Banking / payments / ledgers** | SQL | Strict ACID, constraints, auditing, reporting |
| **E-commerce** (orders, inventory) | SQL core, maybe document catalog | Orders need transactions; catalogs vary per category |
| **CMS / blogs** | Document | Pages are nested blocks read as a whole |
| **Chat / messaging** | Wide-column or document | Huge append volume, partitioned by conversation |
| **IoT / metrics** | Time-series (Timescale, MongoDB time series, InfluxDB) | Time-bucketed writes, range reads |
| **Social graph / recommendations** | Graph (plus SQL for accounts) | Many-hop traversals |
| **Sessions, cache, rate limits** | Key-value (Redis) | Lookups by key, TTLs, sub-millisecond |
| **Analytics warehouse** | Columnar SQL (BigQuery, ClickHouse, Snowflake) | Aggregations over billions of rows |
| **SaaS app** (users, teams, roles, billing) | SQL | Relations and constraints everywhere |

Most real systems use **several** of these (polyglot persistence): for
example, PostgreSQL for the core, Redis for caching, and a search engine
for full-text.

---

### The Same Feature Built Both Ways: Checkout

**Requirement:** create an order with line items, decrease stock for each
product, and never oversell.

**SQL (PostgreSQL + Prisma):** normalized tables plus one transaction.

```js
await prisma.$transaction(async (tx) => {
  for (const item of cart) {
    const { count } = await tx.product.updateMany({
      where: { id: item.productId, stock: { gte: item.qty } },   // only if enough stock
      data: { stock: { decrement: item.qty } },
    });
    if (count === 0) throw new Error(`Out of stock: ${item.productId}`);
  }
  await tx.order.create({
    data: { userId, items: { create: cart.map((i) => ({ productId: i.productId, qty: i.qty, price: i.price })) } },
  });
});
```

**Document (MongoDB + Mongoose):** the order embeds its items, so creating it
is one atomic document write. Stock lives in other documents, so it still
needs a transaction.

```js
await mongoose.connection.transaction(async (session) => {
  for (const item of cart) {
    const res = await Product.updateOne(
      { _id: item.productId, stock: { $gte: item.qty } },
      { $inc: { stock: -item.qty } },
      { session }
    );
    if (res.modifiedCount === 0) throw new Error(`Out of stock: ${item.productId}`);
  }
  await Order.create([{ user: userId, items: cart }], { session });   // items embedded
});
```

| | SQL | Document |
|---|---|---|
| Order + items | 2 tables, JOIN to read | 1 document, 1 read |
| Stock safety | Conditional update in a transaction | Conditional update in a transaction |
| "Revenue per product last month" | One `GROUP BY` | `$unwind` + `$group` pipeline |
| Changing the item shape later | Migration | Add fields; old orders keep old shape |

Both work. SQL is simpler for the **reporting** side; the document model is
simpler for **reading an order**.

---

### Typical Operation Costs

| Operation | SQL (indexed) | Document (indexed) |
|---|---|---|
| Read one entity with its children | JOIN or 2 queries | **1 document read** (if embedded) |
| Update one field | 1 row update | 1 document update |
| Add a child | INSERT a row | `$push` (atomic) |
| Query across entities (ad hoc) | **Flexible JOINs** | Needs `$lookup` or denormalized data |
| Aggregate over everything | **SQL shines** | Pipeline; fine, but verbose |
| Change the schema on a huge table | Migration (can be slow or locking) | **No migration** for additive changes |
| Scale writes past one machine | Hard (partitioning, Citus, Vitess) | **Built-in sharding** |

---

### Operational Differences (Production Reality)

| Concern | SQL (PostgreSQL / MySQL) | MongoDB |
|---|---|---|
| Managed hosting | RDS, Cloud SQL, Neon, Supabase, PlanetScale | MongoDB Atlas, DocumentDB (compatible-ish) |
| Backups / point-in-time restore | Mature, standard | Mature on Atlas; plan it when self-hosting |
| High availability | Primary + replicas, failover | Replica sets built in |
| Horizontal scaling | Read replicas; sharding is extra work | Sharding built in (choose the shard key carefully) |
| Schema changes in production | Needs migration discipline (`CONCURRENTLY`, expand/contract) | Additive changes are free; reshapes need backfills |
| Tooling & hiring | SQL is universal | Common, but less universal than SQL |
| Analytics / BI tools | Connect directly | Often need a connector or an ETL into a warehouse |

---

### The Middle Ground: PostgreSQL `JSONB`

Keep relational integrity for the core, and put the variable part in a
document column.

```prisma
model Product {
  id         Int      @id @default(autoincrement())
  name       String
  price      Decimal  @db.Decimal(10, 2)
  category   String
  attributes Json     // { "ram": "16GB", "cpu": "M3" } or { "size": "M" }
  @@index([category])
}
```

```js
// Filter inside JSON with Prisma
await prisma.product.findMany({
  where: { category: 'laptop', attributes: { path: ['ram'], equals: '16GB' } },
});
```

Add a GIN index on `attributes` in a migration for fast JSON queries. For
many "we might need NoSQL" cases, this is enough.

---

### Red Flags You Picked the Wrong One

| You chose | Red flag |
|---|---|
| MongoDB | Every read uses `populate` or `$lookup` across 3+ collections |
| MongoDB | You keep writing multi-document transactions for core flows |
| MongoDB | The team re-implements foreign-key checks in app code |
| SQL | Most columns are nullable because each row type uses different ones |
| SQL | You store large JSON blobs you never query relationally |
| SQL | Migrations on huge tables are blocking releases every week |

---

### Interview Questions

**"Design the database for an e-commerce site."**
Relational core (users, orders, order_items, products, payments) in
PostgreSQL for transactions and reporting. `JSONB` or a document store for
the catalog's varying attributes, Redis for carts and sessions, and a search
engine for product search.

**"When would you choose MongoDB over PostgreSQL?"**
When data is naturally document-shaped and read as a unit, shapes vary or
change often, and you expect to need horizontal write scaling, and when
cross-entity joins and strict multi-record integrity are secondary.

**"Can MongoDB do transactions?"**
Yes: multi-document ACID transactions on replica sets and sharded clusters.
But the idiomatic design keeps related data in one document, so most writes
are single-document atomic.

---

### Interview-Ready Summary

- Decide by **data shape, integrity needs, query flexibility and scale**,
  not hype. Default to **PostgreSQL** when unsure.
- Verdicts: SQL for money, orders, SaaS and reporting; documents for
  CMS/catalog-like nested data; key-value for caches; wide-column and
  time-series for massive appends.
- The same feature (checkout) works in both. SQL wins at reporting; documents
  win at reading one aggregate.
- **`JSONB`** in PostgreSQL covers many flexible-schema needs.
- Real systems are **polyglot**; watch the red flags that show you picked the
  wrong fit.
