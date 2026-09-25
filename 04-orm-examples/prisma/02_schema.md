## Prisma Schema

### The Full Blog Schema

`prisma/schema.prisma` is the **single source of truth** for your models,
the generated TypeScript types and the database migrations.

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

enum Role {
  USER
  ADMIN
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique @db.VarChar(255)
  name      String?
  role      Role     @default(USER)
  balance   Int      @default(0)          // used in the Transactions page
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  profile   Profile?
  posts     Post[]

  @@map("users")
}

model Profile {
  id     Int     @id @default(autoincrement())
  bio    String?
  userId Int     @unique @map("user_id")
  user   User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("profiles")
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String   @db.VarChar(200)
  content   String?
  published Boolean  @default(false)
  views     Int      @default(0)
  authorId  Int      @map("author_id")
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  tags      Tag[]
  createdAt DateTime @default(now()) @map("created_at")

  @@index([authorId])
  @@index([published, createdAt])
  @@map("posts")
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]

  @@map("tags")
}
```

---

### Anatomy of the Schema

| Block | Purpose |
|---|---|
| `generator` | What to generate (the TypeScript client) and where |
| `datasource` | Which database type (`postgresql`, `mysql`, `sqlite`, `sqlserver`, `cockroachdb`, `mongodb`) |
| `model` | A table (or collection) and its columns and relations |
| `enum` | A fixed set of values; a native enum type in Postgres |

A model field has **name, type, optional modifier, attributes**:

```
email     String?    @unique @db.VarChar(255)
│         │     │    └── attributes
│         │     └── modifier: ? optional, [] list
│         └── type
└── field name
```

---

### Scalar Types

| Prisma type | PostgreSQL (default) | TypeScript |
|---|---|---|
| `String` | `TEXT` | `string` |
| `Int` | `INTEGER` | `number` |
| `BigInt` | `BIGINT` | `bigint` |
| `Float` | `DOUBLE PRECISION` | `number` |
| `Decimal` | `DECIMAL(65,30)` | `Prisma.Decimal` |
| `Boolean` | `BOOLEAN` | `boolean` |
| `DateTime` | `TIMESTAMP(3)` | `Date` |
| `Json` | `JSONB` | `JsonValue` |
| `Bytes` | `BYTEA` | `Uint8Array` |

**Native type attributes** pick the exact DB type:

```prisma
email    String   @db.VarChar(255)
price    Decimal  @db.Decimal(10, 2)   // use Decimal for money, never Float
bornOn   DateTime @db.Date
id       String   @id @default(uuid()) @db.Uuid
```

---

### Field Attributes

| Attribute | Meaning | Example |
|---|---|---|
| `@id` | Primary key | `id Int @id` |
| `@default(...)` | Default value | `autoincrement()`, `now()`, `uuid()`, `cuid()`, `false` |
| `@unique` | Unique constraint | `email String @unique` |
| `@updatedAt` | Set to current time on every update | `updatedAt DateTime @updatedAt` |
| `@map("col")` | Column name in the DB | `createdAt … @map("created_at")` |
| `@relation(...)` | Foreign key details | `fields: [authorId], references: [id]` |
| `@db.X` | Native DB type | `@db.VarChar(255)` |
| `@ignore` | Exclude field from the client | Legacy columns |

### Block Attributes

| Attribute | Meaning | Example |
|---|---|---|
| `@@id([a, b])` | Composite primary key | Join tables |
| `@@unique([a, b])` | Composite unique | `@@unique([userId, slug])` |
| `@@index([a, b])` | Index | `@@index([published, createdAt])` |
| `@@map("table")` | Table name in the DB | `@@map("users")` |

> Prisma does **not** create indexes on foreign keys automatically in
> Postgres. Add `@@index([authorId])` yourself.

---

### Naming Conventions

| Layer | Convention | Example |
|---|---|---|
| Model | PascalCase, singular | `User`, `BlogPost` |
| Field | camelCase | `createdAt` |
| Table | snake_case, plural (via `@@map`) | `users`, `blog_posts` |
| Column | snake_case (via `@map`) | `created_at` |

This keeps the TypeScript API idiomatic and the database idiomatic.

---

### Relations (Preview)

```prisma
// 1:1    User.profile  ↔  Profile.user   (Profile.userId is @unique)
// 1:N    User.posts    ↔  Post.author    (Post.authorId is the FK)
// M:N    Post.tags     ↔  Tag.posts      (Prisma creates the join table "_PostToTag")
```

- The side with `@relation(fields: …, references: …)` holds the **foreign key**.
- The other side (`profile Profile?`, `posts Post[]`) is a **virtual** field;
  it doesn't exist as a column.
- `onDelete: Cascade` deletes profiles/posts when their user is deleted.

Full guide: [Relationships](04_relationships.md)

---

### From Schema to Database: the Generated SQL

```bash
npx prisma migrate dev --name blog_schema
```

```sql
-- prisma/migrations/20260925120000_blog_schema/migration.sql (excerpt)
CREATE TYPE "Role" AS ENUM ('USER', 'ADMIN');

CREATE TABLE "users" (
    "id" SERIAL NOT NULL,
    "email" VARCHAR(255) NOT NULL,
    "name" TEXT,
    "role" "Role" NOT NULL DEFAULT 'USER',
    "balance" INTEGER NOT NULL DEFAULT 0,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,
    CONSTRAINT "users_pkey" PRIMARY KEY ("id")
);

CREATE UNIQUE INDEX "users_email_key" ON "users"("email");
CREATE INDEX "posts_author_id_idx" ON "posts"("author_id");

ALTER TABLE "posts" ADD CONSTRAINT "posts_author_id_fkey"
  FOREIGN KEY ("author_id") REFERENCES "users"("id") ON DELETE CASCADE ON UPDATE CASCADE;
```

Note: `@updatedAt` is set by Prisma, not by a DB trigger, so raw SQL
updates won't change it.

---

### Schema Change Workflow

```
edit schema.prisma → prisma migrate dev --name xyz → review migration.sql → prisma generate → commit
                                                                                 │
                                   CI / production: prisma migrate deploy ◄──────┘
```

**Renames need care.** Renaming a field makes Prisma generate
`DROP COLUMN` + `ADD COLUMN`, which loses data. Create the migration with
`--create-only`, edit the SQL to `ALTER TABLE … RENAME COLUMN …`, then apply:

```bash
npx prisma migrate dev --create-only --name rename_name_to_full_name
# edit migration.sql
npx prisma migrate dev
```

**Existing database?** Generate models from it:

```bash
npx prisma db pull
```

---

### Using the Generated Types

```ts
import type { User, Post, Prisma } from './generated/prisma/client.js';

// Model type
function greet(user: User) { return `Hi ${user.name ?? user.email}`; }

// Input types
const data: Prisma.UserCreateInput = { email: 'a@x.com', name: 'A' };

// The type of a query result that includes relations
type UserWithPosts = Prisma.UserGetPayload<{ include: { posts: true } }>;
```

---

### Interview-Ready Summary

- `schema.prisma` has **generator**, **datasource**, **models** and **enums**;
  it drives both the typed client and migrations.
- Fields: `name Type[?|[]] @attributes`. Key attributes: `@id`, `@default`,
  `@unique`, `@updatedAt`, `@map`, `@relation`, `@db.*`; block attributes
  `@@index`, `@@unique`, `@@id`, `@@map`.
- Use `@map`/`@@map` for camelCase code with snake_case tables.
- The FK side has `@relation(fields, references)`; add `@@index` on FKs.
- Review migration SQL, especially for **renames**, which default to drop + add.

Next: [CRUD](03_crud.md)
