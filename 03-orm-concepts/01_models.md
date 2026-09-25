## Models

### What Is a Model

A **model** is the ORM's definition of one kind of data: its **fields**,
their **types**, its **constraints** and its **relations**. The ORM uses the
model to create tables, build queries and type-check your results.

| Model part | Maps to | Example |
|---|---|---|
| Model name | Table | `User` → `users` |
| Field | Column | `email String` → `email TEXT` |
| Field type | Column type | `DateTime` → `TIMESTAMPTZ` / `DATETIME` |
| Attribute / option | Constraint or default | `@unique` → `UNIQUE` |
| Relation field | Foreign key | `posts Post[]` ↔ `posts.author_id` |

> The model is the **single source of truth**. Types, queries and
> migrations are all generated from it, so it only needs to change in one
> place.

---

### Defining a Model in Prisma

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  role      Role     @default(USER)
  bio       String?                        // ? = optional (nullable)
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt      @map("updated_at")
  posts     Post[]

  @@map("users")                           // table name
  @@index([createdAt])
}

enum Role {
  USER
  ADMIN
}
```

- `@id`, `@unique`, `@default()` become constraints and defaults in SQL.
- `String?` means the column is nullable. Without `?` it is `NOT NULL`.
- `@map` / `@@map` keep `camelCase` in code and `snake_case` in the
  database.
- `@updatedAt` is set by Prisma automatically on every update.

---

### Defining a Model in Sequelize and TypeORM

```js
// Sequelize
const User = sequelize.define('User', {
  email: { type: DataTypes.STRING, allowNull: false, unique: true,
           validate: { isEmail: true } },
  name:  { type: DataTypes.STRING, allowNull: false },
  role:  { type: DataTypes.ENUM('USER', 'ADMIN'), defaultValue: 'USER' },
}, {
  tableName: 'users',
  underscored: true,        // createdAt → created_at
  timestamps: true,         // adds createdAt / updatedAt
});
```

```ts
// TypeORM
@Entity('users')
export class User {
  @PrimaryGeneratedColumn() id: number;
  @Column({ unique: true }) email: string;
  @Column() name: string;
  @Column({ type: 'enum', enum: ['USER', 'ADMIN'], default: 'USER' }) role: string;
  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
  @OneToMany(() => Post, (post) => post.author) posts: Post[];
}
```

Three syntaxes, same model: **fields, types, constraints, defaults,
relations**.

---

### Field Types: Code ↔ Database

| Prisma | PostgreSQL | MySQL | JS / TS value |
|---|---|---|---|
| `String` | `TEXT` | `VARCHAR(191)` | `string` |
| `Int` | `INTEGER` | `INT` | `number` |
| `BigInt` | `BIGINT` | `BIGINT` | `bigint` |
| `Decimal` | `DECIMAL(65,30)` | `DECIMAL(65,30)` | `Decimal` (decimal.js) |
| `Boolean` | `BOOLEAN` | `TINYINT(1)` | `boolean` |
| `DateTime` | `TIMESTAMP(3)` | `DATETIME(3)` | `Date` |
| `Json` | `JSONB` | `JSON` | `object` |

Override the default with **native type attributes** when it matters:

```prisma
price     Decimal  @db.Decimal(12, 2)     // money: exact, never Float
createdAt DateTime @db.Timestamptz(3)     // keep the time zone
code      String   @db.VarChar(20)
```

> `Float` is a binary floating-point number: `0.1 + 0.2 !== 0.3`. For
> money, use `Decimal` or integer minor units (`Int` paise/cents).

---

### Validation: Database vs Application

| Rule | Where it belongs | Example |
|---|---|---|
| Required, unique, foreign key, check | **Database** (via the model) | `@unique`, `NOT NULL`, `@relation` |
| Format and business rules | **Application** | Valid email, password strength, age ≥ 18 |

- Database constraints protect data from **every** client and from race
  conditions.
- Application validation gives **friendly error messages** before the
  query runs.

```js
// Sequelize model-level validation (runs before the INSERT)
email: { type: DataTypes.STRING, validate: { isEmail: true } }

// Prisma has no built-in validators: validate input with a schema library
const UserInput = z.object({ email: z.string().email(), name: z.string().min(1) });
await prisma.user.create({ data: UserInput.parse(req.body) });
```

---

### Model ≠ API Response

A model describes **how data is stored**. An API response describes **what
a client may see**. Don't return models directly.

```js
// ❌ Leaks the password hash and internal fields
res.json(await prisma.user.findUnique({ where: { id } }));

// ✅ Select only what the client needs
res.json(await prisma.user.findUnique({
  where: { id },
  select: { id: true, name: true, email: true },
}));
```

Map models to **DTOs** (data transfer objects) or use `select` at the
boundary. See [Entities](02_entities.md).

---

### Common Mistakes

| Mistake | Fix |
|---|---|
| `Float` for money | `Decimal` or integer minor units |
| Everything optional (`String?`) "to be safe" | Required by default; optional only when the business allows missing data |
| No `@unique` on natural keys (email, slug) | Add the constraint; app checks alone fail under concurrency |
| `DateTime` without time zone on PostgreSQL | `@db.Timestamptz` |
| Returning models straight to the client | `select` or map to a DTO |
| Changing the model without a migration | Every model change goes through a migration. See [Migrations](07_migrations.md) |

---

### Interview-Ready Summary

- A **model** defines one kind of data: fields, types, constraints,
  defaults and relations. It maps to a **table**.
- It is the **single source of truth** for types, queries and migrations.
- Know the **type mapping**, and choose exact types on purpose: `Decimal`
  for money, `Timestamptz` for time.
- Put **integrity rules in the database** and **friendly validation in the
  app**.
- **Models are not API responses.** Use `select` or DTOs to control what
  leaves the server.
