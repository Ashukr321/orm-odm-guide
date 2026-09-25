## Prisma Setup

### What You'll Build

A Node.js + TypeScript project connected to **PostgreSQL** through Prisma.
Every Prisma page in this guide uses the same blog domain:

```
User ──1:1── Profile
  │
  └──1:N── Post ──M:N── Tag
```

> Examples target **Prisma 7**. Differences from Prisma 5/6 are called out
> in [Older Prisma Versions](#older-prisma-versions-56).

---

### Prerequisites

| Tool | Version |
|---|---|
| Node.js | 20.19+ (LTS) |
| PostgreSQL | 14+ (local, Docker, or a hosted DB) |
| npm / pnpm | Any recent version |

**Start Postgres with Docker (optional):**

```bash
docker run --name blog-db -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=blog -p 5432:5432 -d postgres:16
```

---

### 1. Create the Project

```bash
mkdir prisma-blog && cd prisma-blog
npm init -y
npm install -D typescript tsx @types/node prisma
npm install @prisma/client @prisma/adapter-pg pg dotenv
npx tsc --init
```

| Package | Role |
|---|---|
| `prisma` (dev) | CLI: `init`, `migrate`, `generate`, `studio` |
| `@prisma/client` | Runtime used by the generated client |
| `@prisma/adapter-pg` + `pg` | Driver adapter: Prisma sends queries through the `pg` driver |
| `dotenv` | Loads `.env` (Prisma 7 no longer loads it automatically) |
| `tsx` | Runs TypeScript files directly |

Set `"type": "module"` in `package.json`. Prisma 7's generated client is ESM.

---

### 2. Initialize Prisma

```bash
npx prisma init --datasource-provider postgresql --output ../src/generated/prisma
```

This creates:

```
prisma-blog/
├── prisma/
│   └── schema.prisma      ← models, generator, datasource
├── prisma.config.ts       ← CLI config: schema path, migrations, DB URL
├── .env                   ← DATABASE_URL
└── package.json
```

---

### 3. Configure the Connection

**.env**

```bash
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/blog?schema=public"
```

Format: `postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=SCHEMA`.
Add `.env` to `.gitignore`.

**prisma.config.ts**

```ts
import 'dotenv/config';
import { defineConfig, env } from 'prisma/config';

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: {
    path: 'prisma/migrations',
    seed: 'tsx prisma/seed.ts',
  },
  datasource: {
    url: env('DATABASE_URL'),
  },
});
```

**prisma/schema.prisma**

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
}
```

The full blog schema is in [Schema](02_schema.md).

---

### 4. Create the Tables (First Migration)

```bash
npx prisma migrate dev --name init
npx prisma generate
```

- `migrate dev` compares the schema with the DB, writes
  `prisma/migrations/<timestamp>_init/migration.sql` and applies it.
- `generate` writes the typed client to `src/generated/prisma`. Run it again
  after **every** schema change. (In Prisma 7, `migrate dev` no longer runs
  it for you.)

Add the generated folder to `.gitignore`:

```
src/generated/
```

---

### 5. Create a Single Prisma Client

Create **one** client for the whole app. Each client has its own
connection pool, so creating one per request exhausts DB connections.

**src/db.ts**

```ts
import 'dotenv/config';
import { PrismaPg } from '@prisma/adapter-pg';
import { PrismaClient } from './generated/prisma/client.js';

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL! });

export const prisma = new PrismaClient({
  adapter,
  log: process.env.NODE_ENV === 'development' ? ['query', 'warn', 'error'] : ['error'],
});
```

**Hot-reload safe version** (Next.js / nodemon reload modules and would
create a new client on every reload):

```ts
const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

export const prisma = globalForPrisma.prisma ?? new PrismaClient({ adapter });

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

---

### 6. First Query

**src/index.ts**

```ts
import { prisma } from './db.js';

async function main() {
  const user = await prisma.user.create({
    data: { email: 'ashu@example.com', name: 'Ashu' },
  });
  console.log('Created:', user);

  const users = await prisma.user.findMany();
  console.log('All users:', users);
}

main()
  .catch((e) => {
    console.error(e);
    process.exitCode = 1;
  })
  .finally(() => prisma.$disconnect());
```

```bash
npx tsx src/index.ts
```

```
prisma:query INSERT INTO "public"."User" ("email","name","createdAt") VALUES ($1,$2,$3) RETURNING ...
Created: { id: 1, email: 'ashu@example.com', name: 'Ashu', createdAt: 2026-09-25T... }
```

---

### 7. Seed Data

**prisma/seed.ts**

```ts
import { prisma } from '../src/db.js';

await prisma.user.upsert({
  where: { email: 'admin@example.com' },
  update: {},
  create: { email: 'admin@example.com', name: 'Admin' },
});

await prisma.$disconnect();
```

```bash
npx prisma db seed
```

`upsert` makes the seed **idempotent**, so running it twice is safe.

---

### 8. Useful Scripts

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "db:migrate": "prisma migrate dev",
    "db:deploy": "prisma migrate deploy",
    "db:generate": "prisma generate",
    "db:seed": "prisma db seed",
    "db:studio": "prisma studio",
    "postinstall": "prisma generate"
  }
}
```

| Command | When |
|---|---|
| `prisma migrate dev` | Local: create + apply a migration |
| `prisma migrate deploy` | CI / production: apply pending migrations only |
| `prisma migrate reset` | Local: drop DB, re-apply all migrations, re-seed |
| `prisma db push` | Prototyping: sync schema without a migration file |
| `prisma db pull` | Existing DB: generate models from tables |
| `prisma studio` | Browse and edit data in a web UI |
| `prisma format` / `validate` | Format / check `schema.prisma` |

---

### Older Prisma Versions (5/6)

| | Prisma 5/6 | Prisma 7 |
|---|---|---|
| Generator | `provider = "prisma-client-js"` | `provider = "prisma-client"` + required `output` |
| DB URL | `url = env("DATABASE_URL")` in `schema.prisma` | `datasource.url` in `prisma.config.ts` |
| Import | `import { PrismaClient } from '@prisma/client'` | From your `output` path |
| Driver | Built-in Rust engine | Driver adapter (`@prisma/adapter-pg`) |
| `.env` | Loaded automatically | Load with `dotenv` |
| `migrate dev` | Also runs `generate` and seed | Run `generate` / `db seed` yourself |

The query API (`findMany`, `create`, `include`, `$transaction`, …) is the
same, so every other page in this guide applies to both.

---

### Common Setup Errors

| Error | Fix |
|---|---|
| `P1001: Can't reach database server` | DB not running, or wrong host/port in `DATABASE_URL` |
| `P1000: Authentication failed` | Wrong user/password |
| `Cannot find module './generated/prisma/client'` | Run `npx prisma generate` |
| Types don't match the schema | Run `npx prisma generate` after editing the schema |
| `Too many connections` | You create `new PrismaClient()` in many places; use one shared client |
| `DATABASE_URL` is undefined | Add `import 'dotenv/config'` |

---

### Interview-Ready Summary

- Install `prisma` (CLI) and `@prisma/client`, plus a **driver adapter**
  (`@prisma/adapter-pg` + `pg`) in Prisma 7.
- `schema.prisma` holds the models; `prisma.config.ts` holds the DB URL and
  migration settings.
- The workflow: **edit schema → `migrate dev` → `generate` → use the typed client**.
- Use **one shared `PrismaClient`**, since each client owns a connection pool.
- Use `migrate deploy` in production and `db push` only for prototyping.

Next: [Schema](02_schema.md)
