## When to Use an ORM

### The Short Answer

Use an ORM when most of your work is **CRUD and business logic on
related entities**. Use **raw SQL or a query builder** when most of your
work is **complex reporting, bulk data processing, or squeezing out
every millisecond**. Most real apps use both.

---

### Good Fit: Use an ORM

| Scenario | Why an ORM fits |
|---|---|
| CRUD-heavy web apps and REST/GraphQL APIs | One-line reads/writes, pagination, filtering |
| Rich domain with many related entities | Relations, nested writes, cascades |
| TypeScript projects | Types flow from the schema to the API response |
| Fast-moving products / MVPs | Schema changes are a model edit + migration |
| Teams with mixed SQL experience | Safe, consistent API; parameterized by default |
| Need versioned schema changes | Built-in migrations in git |
| Multiple databases (SQLite in tests, Postgres in prod) | Dialect abstraction |

```js
// Typical ORM-shaped work
const order = await prisma.order.create({
  data: {
    userId,
    items: { create: cart.map((i) => ({ productId: i.id, qty: i.qty })) },
  },
  include: { items: { include: { product: true } } },
});
```

---

### Poor Fit: Prefer Raw SQL or a Query Builder

| Scenario | Why an ORM struggles | Better tool |
|---|---|---|
| Analytics, dashboards, reports | Window functions, CTEs, complex `GROUP BY` | Raw SQL, views, a BI tool |
| Bulk ETL / data migrations | Millions of rows; hydration overhead | Raw SQL, `COPY`, streaming cursors |
| Extreme hot paths | Every allocation and round trip matters | Driver + hand-tuned SQL |
| Heavy use of DB-specific features | `JSONB` ops, full-text search, PostGIS, stored procedures | Raw SQL / query builder |
| Existing messy legacy schema | Composite keys, no FKs, odd naming | Query builder (Knex, Kysely) |
| Tiny scripts or one-off tools | ORM setup costs more than it saves | Driver directly |
| Document-shaped data | Nested, schemaless records | A document DB + ODM (see [What Is ODM](../05-odm/01_what-is-odm.md)) |

```sql
-- Report-shaped work: write the SQL
WITH monthly AS (
  SELECT date_trunc('month', created_at) AS month, SUM(total) AS revenue
  FROM orders
  GROUP BY 1
)
SELECT month, revenue,
       revenue - LAG(revenue) OVER (ORDER BY month) AS change
FROM monthly
ORDER BY month;
```

---

### The Hybrid Approach (What Most Teams Do)

```
┌────────────────────────────────────────────────────────────┐
│ ORM                                                         │
│  • CRUD, auth, user/order/product flows                     │
│  • Relations, nested writes, transactions                   │
│  • Migrations                                               │
├────────────────────────────────────────────────────────────┤
│ Raw SQL (via the ORM's escape hatch)                        │
│  • Reports, aggregates, window functions                    │
│  • Bulk updates/inserts                                     │
│  • Hot paths found by profiling                             │
└────────────────────────────────────────────────────────────┘
```

```js
// Same connection pool, same transaction support, typed result
const top = await prisma.$queryRaw`
  SELECT author_id, COUNT(*)::int AS posts
  FROM posts
  WHERE created_at > ${since}
  GROUP BY author_id
  ORDER BY posts DESC
  LIMIT 10`;
```

Start with the ORM, measure, and move a query to raw SQL only when
profiling shows it matters.

---

### Decision Checklist

Answer these about your project:

| Question | Leans ORM | Leans raw SQL / builder |
|---|---|---|
| Is most work CRUD on related entities? | Yes | No, mostly reports/ETL |
| Is the schema changing often? | Yes | Stable |
| Is the team comfortable with SQL? | Mixed | Strong |
| Do you rely on DB-specific features? | Rarely | Heavily |
| Is type safety from DB to API important? | Yes | Less so |
| Is raw throughput the top priority? | No | Yes |
| Greenfield or legacy schema? | Greenfield | Legacy / unusual |

Mostly left column → ORM. Mostly right column → query builder or raw SQL.
Mixed → ORM with raw SQL for the exceptions.

---

### Which Tool on the Spectrum?

| You want… | Pick |
|---|---|
| Best DX, schema-first, generated types | **Prisma** |
| Classic Active Record in JS | **Sequelize** |
| Decorator-based entities, Active Record or Data Mapper | **TypeORM** |
| Closest to SQL but fully typed, lightweight | **Drizzle** |
| Unit of Work + Identity Map (Hibernate-style) | **MikroORM** |
| Typed SQL builder, no models | **Kysely** / **Knex** |
| Total control | **Driver** (`pg`, `mysql2`) |

Comparisons: [Prisma vs Sequelize](../08-comparisons/04_prisma-vs-sequelize.md) ·
[Raw SQL vs ORM](../08-comparisons/03_raw-sql-vs-orm.md) ·
[ORM vs ODM](../08-comparisons/01_orm-vs-odm.md)

---

### Rules of Thumb for Using an ORM Well

1. **Log the SQL** in development and read it.
2. **Eager-load deliberately** (`include`) to avoid N+1; don't include
   everything.
3. **Select only what you need** and never return `passwordHash` by accident.
4. **Paginate** every list endpoint.
5. **Wrap multi-step writes in a transaction.**
6. **Review every generated migration** before applying it.
7. **Use raw SQL for reports and bulk work**, always parameterized.
8. **Add indexes** for the columns you filter and sort on. The ORM won't
   do it for you. See [Indexing](../09-advanced/03_indexing.md).

---

### Interview-Ready Summary

- **Use an ORM** for CRUD-heavy apps, rich related domains, TypeScript
  projects, fast-changing schemas and mixed-skill teams.
- **Avoid (or bypass) it** for analytics, bulk ETL, extreme hot paths,
  heavy DB-specific features, messy legacy schemas and tiny scripts.
- The practical answer is **hybrid**: ORM by default, raw parameterized SQL
  for the queries that profiling shows need it.
- Using an ORM well means logging SQL, eager-loading deliberately, selecting
  only needed fields, paginating, using transactions, reviewing migrations
  and indexing.
