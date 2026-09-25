## Prisma vs Sequelize

### At a Glance

| | Prisma | Sequelize |
|---|---|---|
| First released | 2019 (Prisma 2) | 2011 |
| Style | **Schema-first**, generated client | **Code-first**, Active Record |
| Models defined in | `schema.prisma` | JavaScript/TS classes (`Model.init`) |
| Returns | **Plain typed objects** | Model **instances** (`save()`, getters, dirty tracking) |
| TypeScript | Excellent, generated from the schema | Works, but types are largely hand-maintained |
| Migrations | **Generated** from schema diffs (`migrate dev`) | **Hand-written** with `sequelize-cli` (or `sync()`) |
| Relations | `include` / `select`, nested writes | `include`, association mixins (`getPosts`, `addTag`) |
| Hooks | Client extensions (query hooks) | Rich lifecycle hooks (`beforeCreate`, …) |
| Validation | Types only; use zod for rules | Built-in validators on attributes |
| Databases | PostgreSQL, MySQL, MariaDB, SQLite, SQL Server, CockroachDB (+ MongoDB) | PostgreSQL, MySQL, MariaDB, SQLite, SQL Server, (Db2, Snowflake) |
| Tooling | Prisma Studio, format/validate, VS Code extension | CLI |

Hands-on: [Prisma](../04-orm-examples/prisma/01_setup.md) ·
[Sequelize](../04-orm-examples/sequelize/01_setup.md)

---

### Philosophy

- **Prisma**: "the schema is the source of truth." You describe the data
  model once; Prisma generates the typed client **and** the migrations.
  Objects are plain data with no behavior.
- **Sequelize**: "models are classes." Each model defines its attributes,
  validations, hooks and associations in code, and instances know how to
  save themselves (**Active Record**).

---

### Defining the Same Model

```prisma
// Prisma
model Post {
  id        Int      @id @default(autoincrement())
  title     String   @db.VarChar(200)
  published Boolean  @default(false)
  authorId  Int      @map("author_id")
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  tags      Tag[]
  createdAt DateTime @default(now()) @map("created_at")
  @@index([authorId])
  @@map("posts")
}
```

```js
// Sequelize
class Post extends Model {
  static associate(models) {
    Post.belongsTo(models.User, { foreignKey: 'authorId', as: 'author', onDelete: 'CASCADE' });
    Post.belongsToMany(models.Tag, { through: 'post_tags', foreignKey: 'postId', as: 'tags' });
  }
}
Post.init({
  title:     { type: DataTypes.STRING(200), allowNull: false, validate: { len: [3, 200] } },
  published: { type: DataTypes.BOOLEAN, defaultValue: false },
  authorId:  { type: DataTypes.INTEGER, allowNull: false },
}, { sequelize, tableName: 'posts', underscored: true, indexes: [{ fields: ['author_id'] }] });
// + a separate, hand-written migration that creates the same table
```

Prisma: **one** definition. Sequelize: a model **and** a migration you keep
in sync by hand.

---

### Queries Side by Side

**Filter, sort, paginate:**

```js
// Prisma
await prisma.post.findMany({
  where: { published: true, title: { contains: 'orm', mode: 'insensitive' } },
  orderBy: { createdAt: 'desc' }, skip: 20, take: 10,
});

// Sequelize
await Post.findAll({
  where: { published: true, title: { [Op.iLike]: '%orm%' } },
  order: [['createdAt', 'DESC']], offset: 20, limit: 10,
});
```

**Relations:**

```js
// Prisma
await prisma.user.findUnique({ where: { id }, include: { posts: { where: { published: true } } } });

// Sequelize
await User.findByPk(id, { include: [{ association: 'posts', where: { published: true }, required: false }] });
```

**Nested create:**

```js
// Prisma
await prisma.user.create({ data: { email, posts: { create: [{ title: 'Hi' }] } } });

// Sequelize
await User.create({ email, posts: [{ title: 'Hi' }] }, { include: ['posts'] });
```

**Update:**

```js
// Prisma: explicit
await prisma.post.update({ where: { id }, data: { title: 'New' } });

// Sequelize: instance dirty-checking (or static update)
const post = await Post.findByPk(id);
post.title = 'New';
await post.save();
```

**Transactions:**

```js
// Prisma
await prisma.$transaction(async (tx) => { /* use tx */ });

// Sequelize: pass { transaction: t } to every query (or enable CLS)
await sequelize.transaction(async (t) => { /* … { transaction: t } */ });
```

**Raw SQL:**

```js
await prisma.$queryRaw`SELECT * FROM posts WHERE id = ${id}`;
await sequelize.query('SELECT * FROM posts WHERE id = $id', { bind: { id }, type: QueryTypes.SELECT });
```

---

### Type Safety

```ts
// Prisma: result type follows the query shape
const u = await prisma.user.findUnique({ where: { id: 1 }, select: { email: true } });
u?.email;   // string
u?.name;    // ❌ compile error: not selected

// Sequelize: you declare the attribute types yourself
class User extends Model<InferAttributes<User>, InferCreationAttributes<User>> {
  declare id: CreationOptional<number>;
  declare email: string;
}
const u = await User.findByPk(1, { attributes: ['email'] });
u?.name;    // compiles fine, but undefined at runtime
```

This is Prisma's biggest advantage in TypeScript codebases.

---

### Migrations

| | Prisma | Sequelize |
|---|---|---|
| How | Edit the schema → `migrate dev` generates SQL | Write `up`/`down` JS with `queryInterface` |
| Rollback | No auto `down`; roll forward, or write a fix migration | `db:migrate:undo` runs `down` |
| Drift detection | Yes (shadow database) | No |
| Risk | Renames become drop + add unless you edit the SQL | Model and migration can drift apart |

---

### Performance Notes

- Both generate reasonable SQL for everyday queries. Check the logs for hot
  paths either way.
- **Prisma** loads relations with separate queries by default (no huge JOIN
  result sets) and returns plain objects, so hydration is cheap.
- **Sequelize** uses JOINs for `include`; multiple `hasMany` includes can
  create large cartesian results (use `separate: true`). Model instances are
  heavier; use `raw: true` for read-only lists.
- In serverless, bundle size and cold starts matter. Prisma 7's move to
  TypeScript driver adapters targets exactly this.

---

### Strengths and Weaknesses

| | Prisma | Sequelize |
|---|---|---|
| **Strengths** | Best-in-class types, generated migrations, readable schema, great DX and docs, Studio | Very mature, flexible, rich hooks and validators, familiar Active Record, broad dialect support |
| **Weaknesses** | Less control over generated SQL, schema DSL to learn, no built-in validators, some advanced SQL needs raw queries | Weaker TypeScript story, hand-written migrations, model/migration drift, slower evolution in recent years |

---

### When to Pick Which

| Pick **Prisma** when | Pick **Sequelize** when |
|---|---|
| It's a new TypeScript project | You maintain an existing Sequelize codebase |
| You want types from DB to API | You want Active Record with hooks and validators on models |
| You'd rather not write migrations by hand | You need a dialect Prisma doesn't support |
| The team values DX and fast onboarding | The team is JS-first and knows Sequelize well |

Also consider **Drizzle** (SQL-like, very light, excellent types) and
**TypeORM** or **MikroORM** (decorator entities, Data Mapper, Unit of Work).

---

### Migrating from Sequelize to Prisma

1. `npx prisma db pull`: generate `schema.prisma` from the existing database.
2. Clean up names with `@map` / `@@map`, and add missing relations.
3. Baseline migrations: `prisma migrate diff` + `migrate resolve --applied`.
4. Replace data access module by module, keeping Sequelize for the rest.
5. Move hooks and validators into services, zod schemas or client extensions.
6. Remove Sequelize when nothing imports it.

---

### Interview-Ready Summary

- **Prisma** is schema-first: one `schema.prisma` generates a typed client and
  migrations, and queries return plain objects.
- **Sequelize** is code-first **Active Record**: model classes with validators,
  hooks, mixins and instance `save()`, plus hand-written migrations.
- Prisma wins on **type safety, migrations and DX**. Sequelize wins on
  **maturity, hooks/validators and dialect breadth**.
- New TypeScript project → Prisma (or Drizzle). Existing Sequelize app → keep
  it, or migrate incrementally with `prisma db pull`.
