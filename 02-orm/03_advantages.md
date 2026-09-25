## Advantages of ORMs

### At a Glance

| Advantage | One-line explanation |
|---|---|
| Productivity | Less code for every CRUD operation |
| Type safety | Wrong column or relation names fail at compile time |
| Security | Queries are parameterized by default |
| Relations | Load and write related data without hand-written JOINs |
| Migrations | Schema changes are versioned and repeatable |
| Maintainability | One place (the model) describes each table |
| Portability | Swap databases with minimal code change |
| Built-in features | Validation, hooks, transactions, pooling, soft deletes |
| Testability | Mockable repositories and easy test databases |

---

### 1. Productivity: Less Code

**Raw driver:**

```js
const { rows } = await pool.query(
  `INSERT INTO users (email, name) VALUES ($1, $2)
   RETURNING id, email, name, created_at`,
  [email, name]
);
const user = { ...rows[0], createdAt: rows[0].created_at };
```

**ORM:**

```js
const user = await prisma.user.create({ data: { email, name } });
```

Updates, pagination, counting and upserts are just as short:

```js
await prisma.user.update({ where: { id }, data: { name } });
await prisma.user.findMany({ skip: 20, take: 10, orderBy: { id: 'asc' } });
await prisma.user.count({ where: { active: true } });
await prisma.user.upsert({ where: { email }, create: { email }, update: {} });
```

---

### 2. Type Safety

With a schema-driven ORM, the database shape flows into your types.

```ts
const user = await prisma.user.findUnique({
  where: { id: 1 },
  select: { email: true },
});

user.email;   // ✅ string
user.name;    // ❌ compile error: 'name' was not selected
await prisma.user.findMany({ where: { emial: 'x' } }); // ❌ compile error: typo
```

Rename a column in the schema and the compiler lists every line that has
to change. With raw SQL strings, you find out in production.

---

### 3. Security: Parameterized by Default

ORMs send values **separately** from the SQL text, so user input can never
become SQL.

```js
// User input: "x' OR '1'='1"
await prisma.user.findFirst({ where: { email: input } });
// SQL:    SELECT … FROM "User" WHERE "email" = $1
// Params: ["x' OR '1'='1"]   ← treated as a plain string, not SQL
```

You have to go out of your way (`$queryRawUnsafe`, `sequelize.literal`) to
create an injection hole.

---

### 4. Relations Without the Plumbing

**Reading a graph:**

```js
const post = await prisma.post.findUnique({
  where: { id },
  include: { author: true, comments: { include: { author: true } } },
});
```

**Nested writes, in one transaction:**

```js
await prisma.user.create({
  data: {
    email: 'ashu@example.com',
    posts: {
      create: [{ title: 'Hello' }, { title: 'ORMs 101' }],
    },
  },
});
```

Without an ORM, this is a transaction, two INSERTs, passing the new user's
id into the second INSERT, and error handling around all of it.

See [Relationships](../03-orm-concepts/03_relationships.md).

---

### 5. Migrations: Schema History in Git

```
prisma/migrations/
├── 20260101120000_init/migration.sql
├── 20260115093000_add_posts/migration.sql
└── 20260202140000_add_user_full_name/migration.sql
```

- Every environment (local, CI, staging, prod) gets the **same** schema.
- Schema changes are **code-reviewed** like any other change.
- A new developer runs one command to get a working database.

See [Migrations](../03-orm-concepts/07_migrations.md).

---

### 6. Maintainability: One Source of Truth

The model is the single place that describes a table: columns, types,
defaults, relations and constraints.

```js
// Sequelize
const User = sequelize.define('User', {
  email: { type: DataTypes.STRING, allowNull: false, unique: true,
           validate: { isEmail: true } },
  name:  { type: DataTypes.STRING },
});
User.hasMany(Post, { foreignKey: 'authorId' });
```

Everyone reads the same model instead of reverse-engineering scattered SQL.

---

### 7. Database Portability

Most ORM code runs unchanged on several databases:

```prisma
datasource db {
  provider = "postgresql"   // or "mysql", "sqlite", "sqlserver"
  url      = env("DATABASE_URL")
}
```

This matters most for **tests** (fast SQLite or a throwaway Postgres
container) and early-stage projects that haven't settled on a database.
True portability ends when you use database-specific features such as
Postgres `JSONB` operators.

---

### 8. Built-in Features You Would Otherwise Write

| Feature | Example |
|---|---|
| Transactions | `prisma.$transaction(async (tx) => { … })` |
| Connection pooling | Managed for you; see [Connection Pooling](../03-orm-concepts/09_connection-pooling.md) |
| Validation | Sequelize `validate: { isEmail: true }` |
| Lifecycle hooks | `beforeCreate`, `afterUpdate` (Sequelize, TypeORM) |
| Timestamps | `createdAt` / `updatedAt` filled in automatically |
| Soft deletes | Sequelize `paranoid: true`, TypeORM `@DeleteDateColumn` |
| Logging | Print every generated SQL statement during development |

---

### 9. Testability

- Data access sits behind a model or repository that is easy to **mock**.
- The same code runs against a **test database**, reset by migrations.
- Data Mapper ORMs keep domain objects free of DB code, so business logic
  can be unit-tested with no database at all.

See [Repositories](../03-orm-concepts/10_repositories.md).

---

### Interview-Ready Summary

- **Productivity**: one-line CRUD, no mapping code.
- **Type safety**: schema-driven types catch column and relation typos at
  compile time.
- **Security**: parameterized queries by default block SQL injection.
- **Relations and nested writes** come without hand-written JOINs.
- **Migrations** keep schema history in git and every environment in sync.
- **Portability, built-in features and testability** round it out.
- These benefits come with costs. See [Disadvantages](04_disadvantages.md).
