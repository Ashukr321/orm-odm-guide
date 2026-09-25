## Prisma CRUD

All examples use the blog schema from [Schema](02_schema.md) and the shared
client from [Setup](01_setup.md):

```ts
import { prisma } from './db.js';
```

### Method Cheat Sheet

| Operation | Methods |
|---|---|
| **Create** | `create`, `createMany`, `createManyAndReturn` |
| **Read** | `findUnique`, `findUniqueOrThrow`, `findFirst`, `findFirstOrThrow`, `findMany`, `count` |
| **Update** | `update`, `updateMany`, `updateManyAndReturn`, `upsert` |
| **Delete** | `delete`, `deleteMany` |

---

### Create

**One record:**

```ts
const user = await prisma.user.create({
  data: { email: 'ashu@example.com', name: 'Ashu' },
});
// INSERT INTO "users" ("email","name",…) VALUES ($1,$2,…) RETURNING …
```

**Return only some fields:**

```ts
const user = await prisma.user.create({
  data: { email: 'riya@example.com' },
  select: { id: true, email: true },
});
// { id: 2, email: 'riya@example.com' }
```

**Many records (one INSERT):**

```ts
const { count } = await prisma.user.createMany({
  data: [
    { email: 'a@example.com' },
    { email: 'b@example.com' },
    { email: 'a@example.com' },   // duplicate
  ],
  skipDuplicates: true,           // ON CONFLICT DO NOTHING
});
// count: 2
```

`createManyAndReturn` does the same but returns the created rows
(PostgreSQL, CockroachDB, SQLite).

---

### Read

**By a unique field** (`@id` or `@unique`):

```ts
const user = await prisma.user.findUnique({ where: { email: 'ashu@example.com' } });
// User | null

const user = await prisma.user.findUniqueOrThrow({ where: { id: 1 } });
// throws P2025 if not found
```

**First match of any filter:**

```ts
const latestDraft = await prisma.post.findFirst({
  where: { published: false },
  orderBy: { createdAt: 'desc' },
});
```

**Many, with filters, sorting and pagination:**

```ts
const posts = await prisma.post.findMany({
  where: {
    published: true,
    title: { contains: 'prisma', mode: 'insensitive' },
    createdAt: { gte: new Date('2026-01-01') },
  },
  orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
  skip: 20,     // page 3 …
  take: 10,     // … of 10
});
```

**Common filter operators:**

| Operator | Example |
|---|---|
| `equals`, `not` | `{ role: { not: 'ADMIN' } }` |
| `in`, `notIn` | `{ id: { in: [1, 2, 3] } }` |
| `lt`, `lte`, `gt`, `gte` | `{ views: { gte: 100 } }` |
| `contains`, `startsWith`, `endsWith` | `{ email: { endsWith: '@example.com' } }` |
| `mode: 'insensitive'` | Case-insensitive string match (Postgres, MongoDB) |
| `AND`, `OR`, `NOT` | `{ OR: [{ name: null }, { name: '' }] }` |
| `null` | `{ name: null }` → `IS NULL` |

**Choose columns** with `select`, or hide some with `omit`:

```ts
await prisma.user.findMany({ select: { id: true, email: true } });
await prisma.user.findMany({ omit: { balance: true } });
```

**Count:**

```ts
const total = await prisma.post.count({ where: { published: true } });
```

**Paginated list endpoint:**

```ts
async function listPosts(page = 1, pageSize = 10) {
  const where = { published: true };
  const [items, total] = await prisma.$transaction([
    prisma.post.findMany({
      where,
      orderBy: { createdAt: 'desc' },
      skip: (page - 1) * pageSize,
      take: pageSize,
      select: { id: true, title: true, createdAt: true },
    }),
    prisma.post.count({ where }),
  ]);
  return { items, total, page, pages: Math.ceil(total / pageSize) };
}
```

For large tables use cursor pagination instead. See
[Advanced Queries](05_advanced-queries.md#cursor-pagination).

---

### Update

**One record (by unique field):**

```ts
const user = await prisma.user.update({
  where: { id: 1 },
  data: { name: 'Ashutosh' },
});
// UPDATE "users" SET "name" = $1, "updated_at" = $2 WHERE "id" = $3 RETURNING …
// throws P2025 if id 1 doesn't exist
```

**Atomic number operations** (no read-modify-write race):

```ts
await prisma.post.update({
  where: { id: 10 },
  data: { views: { increment: 1 } },   // also decrement, multiply, divide, set
});
// UPDATE "posts" SET "views" = "views" + $1 WHERE …
```

**Many records:**

```ts
const { count } = await prisma.post.updateMany({
  where: { authorId: 1, published: false },
  data: { published: true },
});
```

**Upsert** (update if found, otherwise create):

```ts
const tag = await prisma.tag.upsert({
  where: { name: 'orm' },
  update: {},
  create: { name: 'orm' },
});
```

---

### Delete

```ts
await prisma.post.delete({ where: { id: 10 } });          // throws P2025 if missing

const { count } = await prisma.post.deleteMany({
  where: { published: false, createdAt: { lt: new Date('2025-01-01') } },
});

await prisma.post.deleteMany();   // ⚠️ deletes every row
```

With `onDelete: Cascade` in the schema, deleting a user also deletes their
profile and posts at the database level.

---

### Handling Errors

Prisma throws `PrismaClientKnownRequestError` with a stable `code`.

| Code | Meaning | Typical HTTP |
|---|---|---|
| `P2002` | Unique constraint failed | 409 Conflict |
| `P2003` | Foreign key constraint failed | 400 / 409 |
| `P2025` | Record not found (update/delete/…OrThrow) | 404 |

```ts
import { Prisma } from './generated/prisma/client.js';

try {
  await prisma.user.create({ data: { email: 'ashu@example.com' } });
} catch (e) {
  if (e instanceof Prisma.PrismaClientKnownRequestError) {
    if (e.code === 'P2002') throw new ConflictError('Email already registered');
    if (e.code === 'P2025') throw new NotFoundError('User not found');
  }
  throw e;
}
```

---

### Putting It Together: an Express Router

```ts
import { Router } from 'express';
import { prisma } from './db.js';

export const users = Router();

users.get('/', async (_req, res) => {
  res.json(await prisma.user.findMany({ select: { id: true, email: true, name: true } }));
});

users.get('/:id', async (req, res) => {
  const user = await prisma.user.findUnique({ where: { id: Number(req.params.id) } });
  user ? res.json(user) : res.sendStatus(404);
});

users.post('/', async (req, res) => {
  const { email, name } = req.body;               // validate input first (e.g. zod)
  res.status(201).json(await prisma.user.create({ data: { email, name } }));
});

users.patch('/:id', async (req, res) => {
  const user = await prisma.user.update({
    where: { id: Number(req.params.id) },
    data: { name: req.body.name },
  });
  res.json(user);
});

users.delete('/:id', async (req, res) => {
  await prisma.user.delete({ where: { id: Number(req.params.id) } });
  res.sendStatus(204);
});
```

Map `P2002`/`P2025` to 409/404 in a shared error-handling middleware.

---

### Interview-Ready Summary

- **Create**: `create`, `createMany` (`skipDuplicates`), `createManyAndReturn`.
- **Read**: `findUnique` (unique fields only), `findFirst`, `findMany` with
  `where`, `orderBy`, `skip`/`take`, `select`/`omit`; `count`.
- **Update**: `update` (throws if missing), `updateMany`, `upsert`, atomic
  `increment`/`decrement`.
- **Delete**: `delete`, `deleteMany` (no `where` deletes everything).
- Handle `P2002` (unique), `P2003` (FK) and `P2025` (not found).
- Always `select` only the fields an API needs.

Next: [Relationships](04_relationships.md)
