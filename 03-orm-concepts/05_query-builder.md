## Query Builder

### What Is a Query Builder

A **query builder** lets you build SQL with **chained function calls**
instead of writing SQL strings. It sits between raw SQL and a full ORM:

```
Raw SQL strings  →  Query builder  →  ORM models
(full control)      (SQL shape,        (objects,
                     but safe and       relations,
                     composable)        less SQL)
```

```js
// Knex
const users = await knex('users')
  .select('id', 'name', 'email')
  .where('role', 'admin')
  .andWhere('created_at', '>', since)
  .orderBy('created_at', 'desc')
  .limit(20);
// → select "id", "name", "email" from "users"
//   where "role" = ? and "created_at" > ? order by "created_at" desc limit ?
```

Values are always sent as **parameters** (`?` / `$1`), so they can't cause
SQL injection.

---

### Two Kinds of Query Builders

| Kind | Examples | Used for |
|---|---|---|
| **Standalone** | Knex, Kysely | Your main database layer, without an ORM |
| **Inside an ORM** | TypeORM `createQueryBuilder`, MikroORM `qb`, Sequelize `Op` | Queries too complex for the ORM's `find` API |

```ts
// Kysely: fully type-safe; column names and result types are checked
const rows = await db
  .selectFrom('users')
  .innerJoin('orders', 'orders.user_id', 'users.id')
  .select(['users.name', db.fn.count('orders.id').as('order_count')])
  .where('orders.status', '=', 'paid')
  .groupBy('users.name')
  .having(db.fn.count('orders.id'), '>', 5)
  .execute();
// rows: { name: string; order_count: string | number | bigint }[]
```

---

### The ORM's Built-In Query Builder (TypeORM)

Use it when `find()` isn't enough: aggregations, subqueries, complex
joins.

```ts
const stats = await dataSource.getRepository(Order)
  .createQueryBuilder('o')
  .select('o.status', 'status')
  .addSelect('COUNT(*)', 'count')
  .addSelect('SUM(o.total_minor)', 'revenue')
  .where('o.created_at >= :from', { from })
  .groupBy('o.status')
  .getRawMany();                 // raw rows, not entities

// Subquery: users whose last order is older than 90 days
const inactive = await dataSource.getRepository(User)
  .createQueryBuilder('u')
  .where((qb) => {
    const sub = qb.subQuery()
      .select('MAX(o.created_at)').from(Order, 'o')
      .where('o.user_id = u.id').getQuery();
    return `${sub} < NOW() - INTERVAL '90 days'`;
  })
  .getMany();
```

`getMany()` returns entities. `getRawMany()` returns plain rows for
aggregates.

---

### Building Dynamic Queries

The real strength of a query builder: add conditions **only when they are
needed**, such as search filters coming from the request.

```js
function searchProducts({ q, minPrice, maxPrice, category, sort = 'newest' }) {
  let query = knex('products').select('id', 'name', 'price_minor');

  if (q)        query = query.whereILike('name', `%${q}%`);
  if (minPrice) query = query.where('price_minor', '>=', minPrice);
  if (maxPrice) query = query.where('price_minor', '<=', maxPrice);
  if (category) query = query.where('category_id', category);

  const sorts = { newest: ['created_at', 'desc'], cheapest: ['price_minor', 'asc'] };
  const [column, dir] = sorts[sort] ?? sorts.newest;     // allow-list, never raw user input
  return query.orderBy(column, dir).limit(50);
}
```

> Values are parameterized automatically, but **identifiers are not
> values**. Column and table names from the user must come from an
> allow-list, as in `sorts` above.

---

### Pagination: Offset vs Cursor

```js
// Offset: simple, but slow on deep pages (the DB still scans the skipped rows)
knex('posts').orderBy('id', 'desc').limit(20).offset(20 * (page - 1));

// Cursor / keyset: fast at any depth, stable when new rows arrive
knex('posts').where('id', '<', lastSeenId).orderBy('id', 'desc').limit(20);
```

| | Offset | Cursor (keyset) |
|---|---|---|
| Jump to page 57 | ✅ | ❌ (next / previous only) |
| Speed on deep pages | Gets slower | Constant (uses the index) |
| New rows while paging | Items shift and repeat | Stable |
| Best for | Admin tables | Infinite scroll, feeds, APIs |

Prisma supports both: `skip` / `take`, and `cursor: { id } + skip: 1 + take`.

---

### Raw Fragments, Safely

Every query builder lets you drop in raw SQL. Always pass values as
**bindings**:

```js
// Knex: ? placeholders with a bindings array
knex('users').whereRaw('LOWER(email) = LOWER(?)', [email]);

// Kysely: the sql tagged template parameterizes ${…}
sql`LOWER(email) = LOWER(${email})`;

// ❌ Never interpolate values into a raw string
knex('users').whereRaw(`email = '${email}'`);     // SQL injection
```

---

### Query Builder vs ORM: When to Use Which

| Use a query builder when… | Use the ORM API when… |
|---|---|
| Reports, aggregations, `GROUP BY`, window functions | Regular CRUD |
| Complex filters built at runtime | Loading relations as nested objects |
| You want SQL-shaped code with type safety (Kysely) | You want the shortest, most readable code |
| Performance-critical paths | Nested writes in one transaction |

Many teams combine them: an ORM for most things, and its query builder (or
Kysely) for the hard 5%.

---

### Interview-Ready Summary

- A query builder builds SQL from **chained functions**: safer than
  strings, closer to SQL than an ORM.
- **Standalone** (Knex, Kysely) or **built into the ORM** (TypeORM
  `createQueryBuilder`).
- Its strength is **dynamic queries**: add filters only when present.
  Values are parameterized, but **identifiers need an allow-list**.
- **Offset** pagination is simple but slows down on deep pages. **Cursor**
  pagination is fast and stable.
- Raw fragments must use **bindings** (`?` placeholders, or Kysely's `sql`
  template tag), never string interpolation.
