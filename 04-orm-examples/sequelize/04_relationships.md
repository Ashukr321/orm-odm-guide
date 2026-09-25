## Sequelize Relationships

### The Four Association Methods

| Method | Meaning | FK lives on |
|---|---|---|
| `A.hasOne(B)` | A has one B | **B** |
| `A.belongsTo(B)` | A belongs to one B | **A** |
| `A.hasMany(B)` | A has many B | **B** |
| `A.belongsToMany(B, { through })` | Many-to-many | The **join table** |

Define **both sides** of every relation so you can query from either end:

| Relation | Pair |
|---|---|
| 1:1 | `hasOne` + `belongsTo` |
| 1:N | `hasMany` + `belongsTo` |
| M:N | `belongsToMany` + `belongsToMany` |

Concepts: [Relationships](../../03-orm-concepts/03_relationships.md)

---

### Declaring the Blog Associations

Inside each model's `static associate(models)` (see [Models](02_models.md)):

```js
// 1:1  User ↔ Profile (profiles.user_id)
User.hasOne(models.Profile, { foreignKey: 'userId', as: 'profile', onDelete: 'CASCADE' });
Profile.belongsTo(models.User, { foreignKey: 'userId', as: 'user' });

// 1:N  User ↔ Post (posts.author_id)
User.hasMany(models.Post, { foreignKey: 'authorId', as: 'posts', onDelete: 'CASCADE' });
Post.belongsTo(models.User, { foreignKey: 'authorId', as: 'author' });

// M:N  Post ↔ Tag (post_tags.post_id, post_tags.tag_id)
Post.belongsToMany(models.Tag, { through: 'post_tags', foreignKey: 'postId', otherKey: 'tagId', as: 'tags' });
Tag.belongsToMany(models.Post, { through: 'post_tags', foreignKey: 'tagId', otherKey: 'postId', as: 'posts' });
```

**Always set `foreignKey` and `as` explicitly**, and use the same
`foreignKey` on both sides. Otherwise Sequelize guesses names (`UserId`) and
you can end up with two FK columns.

---

### Association Options

| Option | Purpose |
|---|---|
| `foreignKey` | FK attribute name (`'authorId'`) or full definition |
| `as` | Alias; also names the mixin methods (`getPosts`) |
| `sourceKey` / `targetKey` | Reference a non-PK column |
| `onDelete` | `CASCADE`, `SET NULL`, `RESTRICT`, `NO ACTION` |
| `onUpdate` | Same options, for PK changes |
| `through` | Join table name or model (M:N) |
| `otherKey` | FK to the other model in the join table (M:N) |
| `constraints: false` | Don't create a DB FK constraint |

---

### The Join Table Migration

```js
// src/db/migrations/20260925130000-create-post-tags.js
module.exports = {
  async up(qi, Sequelize) {
    await qi.createTable('post_tags', {
      post_id: {
        type: Sequelize.INTEGER, allowNull: false, primaryKey: true,
        references: { model: 'posts', key: 'id' }, onDelete: 'CASCADE',
      },
      tag_id: {
        type: Sequelize.INTEGER, allowNull: false, primaryKey: true,
        references: { model: 'tags', key: 'id' }, onDelete: 'CASCADE',
      },
      created_at: { type: Sequelize.DATE, allowNull: false, defaultValue: Sequelize.fn('now') },
      updated_at: { type: Sequelize.DATE, allowNull: false, defaultValue: Sequelize.fn('now') },
    });
    await qi.addIndex('post_tags', ['tag_id']);
  },
  down: (qi) => qi.dropTable('post_tags'),
};
```

**Join table with its own data**: use a model as `through`:

```js
class PostTag extends Model {}
PostTag.init(
  { addedBy: DataTypes.INTEGER },
  { sequelize, modelName: 'PostTag', tableName: 'post_tags', underscored: true }
);

Post.belongsToMany(Tag, { through: PostTag, foreignKey: 'postId', otherKey: 'tagId', as: 'tags' });
Tag.belongsToMany(Post, { through: PostTag, foreignKey: 'tagId', otherKey: 'postId', as: 'posts' });

await post.addTag(tag, { through: { addedBy: currentUserId } });
```

---

### Mixin Methods

Every association adds methods to instances. The names come from `as`.

| Association | Methods on the instance |
|---|---|
| `hasOne` / `belongsTo` (`as: 'profile'`) | `getProfile`, `setProfile`, `createProfile` |
| `hasMany` (`as: 'posts'`) | `getPosts`, `countPosts`, `hasPost(s)`, `addPost(s)`, `removePost(s)`, `setPosts`, `createPost` |
| `belongsToMany` (`as: 'tags'`) | `getTags`, `countTags`, `hasTag(s)`, `addTag(s)`, `removeTag(s)`, `setTags`, `createTag` |

```js
const user = await User.findByPk(1);

await user.createProfile({ bio: 'Backend dev' });
const post = await user.createPost({ title: 'Hello Sequelize' });   // sets authorId

const published = await user.getPosts({ where: { published: true } });
const total = await user.countPosts();

const [orm, node] = await Promise.all([
  Tag.findOrCreate({ where: { name: 'orm' } }).then(([t]) => t),
  Tag.findOrCreate({ where: { name: 'node' } }).then(([t]) => t),
]);
await post.addTags([orm, node]);   // INSERT INTO post_tags …
await post.removeTag(node);
await post.setTags([orm]);         // replace all
```

> Each mixin call is a **separate query**. Calling `getPosts()` in a loop is
> the N+1 problem. Use `include` instead.

---

### Eager Loading with `include`

```js
const user = await User.findByPk(1, {
  include: [
    { association: 'profile' },
    {
      association: 'posts',
      where: { published: true },
      required: false,                    // keep users with no published posts (LEFT JOIN)
      attributes: ['id', 'title'],
      include: [{ association: 'tags', attributes: ['name'], through: { attributes: [] } }],
    },
  ],
  order: [[{ model: Post, as: 'posts' }, 'createdAt', 'DESC']],
});

user.profile.bio;
user.posts[0].tags[0].name;
```

| Option in `include` | Effect |
|---|---|
| `association` / `model` + `as` | Which relation |
| `attributes` | Columns of the related model |
| `where` | Filter related rows (turns `required` on by default) |
| `required: true` | `INNER JOIN`: only parents that have a match |
| `required: false` | `LEFT OUTER JOIN`: keep parents without matches |
| `through: { attributes: [] }` | Hide join-table columns (M:N) |
| `separate: true` | Load a `hasMany` with its own query instead of a JOIN |
| `include` | Nest deeper |

**Filter parents by a child** (users who wrote a post tagged "orm"):

```js
await User.findAll({
  include: [{
    association: 'posts',
    required: true,
    attributes: [],
    include: [{ association: 'tags', where: { name: 'orm' }, attributes: [], through: { attributes: [] } }],
  }],
  distinct: true,
});
```

**Filter on a nested column in the top-level `where`:**

```js
await Post.findAll({
  where: { '$author.role$': 'ADMIN' },
  include: [{ association: 'author', attributes: [] }],
});
```

---

### Pagination + `hasMany` Includes

With JOINs, one user with 50 posts becomes 50 rows, so `limit: 10`
limits **rows**, not users, and counts come out wrong. Fixes:

```js
// Option 1: separate query for the hasMany
await User.findAll({
  limit: 10,
  include: [{ association: 'posts', separate: true, order: [['createdAt', 'DESC']] }],
});
// SELECT … FROM users LIMIT 10;
// SELECT … FROM posts WHERE author_id IN (…);

// Option 2: distinct count with findAndCountAll
await User.findAndCountAll({ include: ['posts'], distinct: true, limit: 10 });
```

---

### Counting Related Rows

```js
const { fn, col } = require('sequelize');

const users = await User.findAll({
  attributes: ['id', 'email', [fn('COUNT', col('posts.id')), 'postCount']],
  include: [{ association: 'posts', attributes: [] }],
  group: ['User.id'],
});
users[0].get('postCount');
```

---

### Creating Parents and Children Together

```js
await User.create(
  {
    email: 'riya@example.com',
    profile: { bio: 'Frontend dev' },
    posts: [{ title: 'First' }, { title: 'Second' }],
  },
  { include: ['profile', 'posts'] }
);
```

Wrap this in a transaction if all rows must succeed or fail together (see
[Transactions](05_transactions.md)).

---

### Avoiding N+1

```js
// ❌ 1 + N queries
const users = await User.findAll();
for (const u of users) {
  u.posts = await u.getPosts();
}

// ✅ one JOIN query (or 2 with separate: true)
const users = await User.findAll({ include: ['posts'] });
```

See [N+1 Problem](../../09-advanced/01_n-plus-one-problem.md).

---

### Interview-Ready Summary

- `hasOne`/`hasMany` put the FK on the **target**; `belongsTo` puts it on the
  **source**; `belongsToMany` uses a **join table** (`through`).
- Define both sides, with the same explicit `foreignKey` and an `as` alias.
- Associations add **mixins** (`getPosts`, `addTag`, `setTags`,
  `createProfile`); each is a separate query.
- Eager-load with `include`: `required` picks INNER vs LEFT JOIN, `where`
  filters children, `through: { attributes: [] }` hides join columns.
- For `limit` with `hasMany`, use `separate: true` or `distinct: true`.
- Use `include` instead of loops to avoid N+1.

Next: [Transactions](05_transactions.md)
