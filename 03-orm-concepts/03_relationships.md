## Relationships

### The Three Relationship Types

| Type | Example | Where the foreign key lives |
|---|---|---|
| **One-to-one (1:1)** | User ↔ Profile | On one side, with `UNIQUE` (`profiles.user_id UNIQUE`) |
| **One-to-many (1:N)** | User → Posts | On the "many" side (`posts.author_id`) |
| **Many-to-many (M:N)** | Posts ↔ Tags | In a **join table** (`post_tags.post_id`, `post_tags.tag_id`) |

The side that holds the foreign key is called the **owning side**. The
other side is the **inverse side**: it has a relation field in code, but no
column in its table.

---

### Relationships in Prisma

```prisma
model User {
  id      Int      @id @default(autoincrement())
  profile Profile?                      // 1:1, inverse side
  posts   Post[]                        // 1:N, inverse side
}

model Profile {
  id     Int  @id @default(autoincrement())
  bio    String
  user   User @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId Int  @unique                   // @unique makes it 1:1
}

model Post {
  id       Int   @id @default(autoincrement())
  author   User  @relation(fields: [authorId], references: [id])
  authorId Int                          // FK: the owning side
  tags     Tag[]                        // M:N, implicit join table
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]
}
```

Prisma creates the M:N join table (`_PostToTag`) for you. That is called an
**implicit** many-to-many.

---

### Explicit Join Tables (M:N With Extra Data)

When the relationship itself has data (who added the tag, when, a
quantity, a role), model the join table explicitly:

```prisma
model PostTag {
  post    Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId  Int
  tag     Tag      @relation(fields: [tagId], references: [id], onDelete: Cascade)
  tagId   Int
  addedAt DateTime @default(now())
  addedBy Int

  @@id([postId, tagId])                 // composite primary key
}
```

> Rule of thumb: start explicit when you *might* need extra columns later.
> Converting an implicit join table into an explicit one means a data
> migration.

---

### Relationships in Sequelize and TypeORM

```js
// Sequelize: declare both sides
User.hasOne(Profile, { foreignKey: 'userId' });
Profile.belongsTo(User, { foreignKey: 'userId' });

User.hasMany(Post, { foreignKey: 'authorId', as: 'posts' });
Post.belongsTo(User, { foreignKey: 'authorId', as: 'author' });

Post.belongsToMany(Tag, { through: 'post_tags' });
Tag.belongsToMany(Post, { through: 'post_tags' });
```

```ts
// TypeORM
@Entity() export class Post {
  @ManyToOne(() => User, (user) => user.posts, { onDelete: 'CASCADE' })
  author: User;                                   // owning side: author_id column

  @ManyToMany(() => Tag, (tag) => tag.posts)
  @JoinTable({ name: 'post_tags' })               // owning side of the M:N
  tags: Tag[];
}

@Entity() export class User {
  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];                                  // inverse side: no column
}
```

In Sequelize, `belongsTo` is the side with the foreign key. `hasOne` and
`hasMany` describe the inverse side.

---

### Self-Relations

A table that references **itself**: employees and managers, comment
replies, category trees, followers.

```prisma
model Employee {
  id        Int        @id @default(autoincrement())
  name      String
  managerId Int?
  manager   Employee?  @relation("Management", fields: [managerId], references: [id])
  reports   Employee[] @relation("Management")
}

model User {
  id         Int    @id @default(autoincrement())
  followers  User[] @relation("Follows")
  following  User[] @relation("Follows")          // M:N self-relation
}
```

Relation names (`"Management"`, `"Follows"`) tell Prisma which two fields
belong together.

---

### Referential Actions: What Happens on Delete

| Action | When the parent row is deleted | Use for |
|---|---|---|
| `Restrict` / `NoAction` | The delete fails while children exist | Most relations (users with orders) |
| `Cascade` | The children are deleted too | Children that mean nothing alone (profile, order items) |
| `SetNull` | The child's FK becomes `NULL` | Optional links (post's editor left the company) |

```prisma
user User @relation(fields: [userId], references: [id], onDelete: Cascade)
```

> Choose per relation. `Cascade` everywhere turns one accidental delete
> into a mass deletion.

---

### Querying and Writing Relations

```js
// Read with relations
const post = await prisma.post.findUnique({
  where: { id: 1 },
  include: { author: true, tags: true },
});

// Filter by a relation: users who have at least one published post
await prisma.user.findMany({ where: { posts: { some: { published: true } } } });

// Nested write: create a post, connect an existing author, create or connect tags
await prisma.post.create({
  data: {
    title: 'Relations 101',
    author: { connect: { id: 1 } },
    tags: {
      connectOrCreate: [{ where: { name: 'orm' }, create: { name: 'orm' } }],
    },
  },
});
```

Nested writes run in **one transaction**, so either everything is created
or nothing is.

---

### Common Mistakes

| Mistake | Fix |
|---|---|
| Forgetting `@unique` on a 1:1 foreign key | Without it, the database allows 1:N |
| Implicit M:N when the link needs data | An explicit join table model |
| `Cascade` on every relation | Decide per relation; default to `Restrict` |
| No index on the FK column (PostgreSQL) | Index it; PostgreSQL doesn't do it automatically |
| Declaring only one side in Sequelize | Declare both (`hasMany` + `belongsTo`) to query from both sides |
| Polymorphic "`commentable_type` + `commentable_id`" | No real FK is possible; prefer separate FK columns or separate tables |

---

### Interview-Ready Summary

- Three types: **1:1** (FK + `UNIQUE`), **1:N** (FK on the many side),
  **M:N** (join table).
- The **owning side** holds the FK; the **inverse side** is a virtual field
  in code.
- Use an **explicit join table** when the relationship has its own data.
- **Self-relations** model trees, hierarchies and follower graphs.
- Pick **referential actions** (`Restrict`, `Cascade`, `SetNull`) per
  relation, on purpose.
- Nested writes (`connect`, `create`, `connectOrCreate`) run in one
  transaction.
