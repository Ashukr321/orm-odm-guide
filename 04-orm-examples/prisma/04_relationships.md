## Prisma Relationships

### The Three Relation Types

```
User ──1:1── Profile        one user has at most one profile
User ──1:N── Post           one user writes many posts
Post ──M:N── Tag            a post has many tags; a tag is on many posts
```

Concepts: [Relationships](../../03-orm-concepts/03_relationships.md)

---

### One-to-One

```prisma
model User {
  id      Int      @id @default(autoincrement())
  profile Profile?                                // virtual side
}

model Profile {
  id     Int  @id @default(autoincrement())
  bio    String?
  userId Int  @unique                             // @unique makes it 1:1
  user   User @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

```ts
// Create a user and their profile together
await prisma.user.create({
  data: { email: 'ashu@example.com', profile: { create: { bio: 'Backend dev' } } },
});

// Read with the profile
await prisma.user.findUnique({ where: { id: 1 }, include: { profile: true } });

// Create or update the profile from the user side
await prisma.user.update({
  where: { id: 1 },
  data: {
    profile: {
      upsert: { create: { bio: 'Hi' }, update: { bio: 'Hi again' } },
    },
  },
});
```

---

### One-to-Many

```prisma
model User {
  id    Int    @id @default(autoincrement())
  posts Post[]                                    // virtual list
}

model Post {
  id       Int  @id @default(autoincrement())
  authorId Int                                    // FK column
  author   User @relation(fields: [authorId], references: [id], onDelete: Cascade)
  @@index([authorId])
}
```

```ts
// Create a post for an existing user: either set the FK...
await prisma.post.create({ data: { title: 'Hello', authorId: 1 } });

// ...or connect through the relation
await prisma.post.create({
  data: { title: 'Hello', author: { connect: { id: 1 } } },
});

// Create a user with several posts (one transaction)
await prisma.user.create({
  data: {
    email: 'riya@example.com',
    posts: { create: [{ title: 'First' }, { title: 'Second' }] },
  },
});
```

---

### Many-to-Many

#### Implicit (Prisma Manages the Join Table)

```prisma
model Post {
  id   Int   @id @default(autoincrement())
  tags Tag[]
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]
}
// Prisma creates the table "_PostToTag" (A, B) for you
```

```ts
// Attach tags: create them if they don't exist
await prisma.post.update({
  where: { id: 10 },
  data: {
    tags: {
      connectOrCreate: [
        { where: { name: 'orm' },    create: { name: 'orm' } },
        { where: { name: 'prisma' }, create: { name: 'prisma' } },
      ],
    },
  },
});

// Remove one tag, or replace them all
await prisma.post.update({ where: { id: 10 }, data: { tags: { disconnect: { name: 'orm' } } } });
await prisma.post.update({ where: { id: 10 }, data: { tags: { set: [{ name: 'prisma' }] } } });
```

#### Explicit (You Own the Join Table)

Use this when the relation itself has data, like *when* a tag was added or
*who* added it.

```prisma
model Post {
  id   Int       @id @default(autoincrement())
  tags PostTag[]
}

model Tag {
  id    Int       @id @default(autoincrement())
  name  String    @unique
  posts PostTag[]
}

model PostTag {
  postId  Int
  tagId   Int
  addedAt DateTime @default(now())
  post    Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag     Tag      @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
  @@index([tagId])
}
```

```ts
await prisma.postTag.create({ data: { postId: 10, tagId: 3 } });

const post = await prisma.post.findUnique({
  where: { id: 10 },
  include: { tags: { include: { tag: true } } },
});
// post.tags → [{ postId, tagId, addedAt, tag: { id, name } }]
```

| | Implicit M:N | Explicit M:N |
|---|---|---|
| Join table | Created and hidden by Prisma | A normal model you define |
| Extra columns | Not possible | Yes (`addedAt`, `addedBy`, …) |
| Queries | `post.tags` directly | `post.tags[].tag` (one extra level) |
| Use when | Pure link | The link has data |

---

### Self-Relation (Followers)

```prisma
model User {
  id        Int    @id @default(autoincrement())
  followers User[] @relation("Follows")
  following User[] @relation("Follows")
}
```

```ts
await prisma.user.update({
  where: { id: 1 },
  data: { following: { connect: { id: 2 } } },   // user 1 follows user 2
});
```

When two relations join the same pair of models, give each a name
(`@relation("Follows")`) so Prisma can tell them apart.

---

### Reading Related Data

**`include`: the whole related record**

```ts
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    profile: true,
    posts: {
      where: { published: true },
      orderBy: { createdAt: 'desc' },
      take: 5,
      include: { tags: true },
    },
  },
});
```

**`select`: exactly the fields you want, at every level**

```ts
const user = await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    email: true,
    posts: { select: { title: true, tags: { select: { name: true } } } },
  },
});
// { email, posts: [{ title, tags: [{ name }] }] }
```

You can't use `include` and `select` at the same level, but you can nest
one inside the other.

**Count relations without loading them:**

```ts
const users = await prisma.user.findMany({
  select: { email: true, _count: { select: { posts: true } } },
});
// [{ email: 'a@x.com', _count: { posts: 12 } }]
```

---

### Filtering by Relations

| Relation | Operators |
|---|---|
| To-many (`posts`, `tags`) | `some`, `every`, `none` |
| To-one (`author`, `profile`) | `is`, `isNot` |

```ts
// Users with at least one published post
await prisma.user.findMany({ where: { posts: { some: { published: true } } } });

// Users with no posts
await prisma.user.findMany({ where: { posts: { none: {} } } });

// Posts whose author is an admin
await prisma.post.findMany({ where: { author: { is: { role: 'ADMIN' } } } });

// Posts tagged "prisma"
await prisma.post.findMany({ where: { tags: { some: { name: 'prisma' } } } });

// Users without a profile
await prisma.user.findMany({ where: { profile: { is: null } } });
```

---

### Nested Write Operations

| Operation | Does |
|---|---|
| `create` / `createMany` | Create new related records |
| `connect` | Link existing records by unique field |
| `connectOrCreate` | Link if it exists, otherwise create |
| `disconnect` | Unlink (sets FK to `null`, or removes the join row) |
| `set` | Replace all links (to-many) |
| `update` / `updateMany` | Update related records |
| `upsert` | Update or create a related record |
| `delete` / `deleteMany` | Delete related records |

```ts
await prisma.user.update({
  where: { id: 1 },
  data: {
    posts: {
      create: { title: 'New post' },
      updateMany: { where: { published: false }, data: { published: true } },
      deleteMany: { createdAt: { lt: new Date('2024-01-01') } },
    },
  },
});
```

All nested writes in one call run in **one transaction**.

---

### Referential Actions

```prisma
author User @relation(fields: [authorId], references: [id], onDelete: Cascade, onUpdate: Cascade)
```

| `onDelete` | When the parent is deleted |
|---|---|
| `Cascade` | Delete the children too |
| `Restrict` / `NoAction` | Block the delete if children exist (default for required relations) |
| `SetNull` | Set the FK to `null` (FK must be optional) |
| `SetDefault` | Set the FK to its default |

---

### Avoiding N+1

```ts
// ❌ 1 + N queries
const users = await prisma.user.findMany();
for (const u of users) {
  const posts = await prisma.post.findMany({ where: { authorId: u.id } });
}

// ✅ 2 queries: users, then posts WHERE author_id IN (…)
const users = await prisma.user.findMany({ include: { posts: true } });
```

Prisma also batches concurrent `findUnique` calls with the same shape
(helpful in GraphQL resolvers). See
[N+1 Problem](../../09-advanced/01_n-plus-one-problem.md).

---

### Interview-Ready Summary

- **1:1**: FK with `@unique`. **1:N**: FK on the "many" side plus a list on
  the "one" side. **M:N**: implicit (hidden `_AToB` table) or explicit
  (your own join model with extra columns).
- The side with `@relation(fields, references)` owns the FK.
- Read with `include` (whole records) or `select` (chosen fields), count
  with `_count`.
- Filter with `some`/`every`/`none` (to-many) and `is`/`isNot` (to-one).
- Nested writes (`create`, `connect`, `connectOrCreate`, `set`, …) are atomic.
- Set `onDelete` deliberately, and use `include` instead of loops to avoid N+1.

Next: [Advanced Queries](05_advanced-queries.md)
