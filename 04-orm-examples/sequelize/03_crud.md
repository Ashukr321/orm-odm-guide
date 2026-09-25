## Sequelize CRUD

All examples use the models from [Models](02_models.md):

```js
const { Op } = require('sequelize');
const { sequelize, User, Post, Tag } = require('./models');
```

### Method Cheat Sheet

| Operation | Static (on the model) | Instance (on a record) |
|---|---|---|
| **Create** | `create`, `bulkCreate`, `findOrCreate` | `build` + `save` |
| **Read** | `findByPk`, `findOne`, `findAll`, `findAndCountAll`, `count` | `reload` |
| **Update** | `update`, `increment`, `upsert` | `set` / `update` / `save`, `increment` |
| **Delete** | `destroy`, `truncate` | `destroy`, `restore` (paranoid) |

---

### Create

```js
const user = await User.create({ email: 'Ashu@Example.com ', name: 'Ashu' });
// setter lower-cases and trims email; validators run; INSERT … RETURNING *
user.id;          // 1
user.toJSON();    // plain object
```

**Build, then save** (lets you change fields before inserting):

```js
const user = User.build({ email: 'riya@example.com' });
user.name = 'Riya';
await user.save();
```

**Allow only certain fields** (protect against mass assignment):

```js
await User.create(req.body, { fields: ['email', 'name'] });   // role/balance ignored
```

**Many rows:**

```js
await User.bulkCreate(
  [{ email: 'a@example.com' }, { email: 'b@example.com' }],
  { validate: true, ignoreDuplicates: true }
);
```

`bulkCreate` skips per-row validation unless `validate: true`.

**Find or create:**

```js
const [tag, created] = await Tag.findOrCreate({
  where: { name: 'orm' },
  defaults: { name: 'orm' },
});
```

---

### Read

**By primary key / first match:**

```js
const user = await User.findByPk(1);                                   // or null
const user = await User.findOne({ where: { email: 'ashu@example.com' } });
```

**Many, with filters, sorting and pagination:**

```js
const posts = await Post.findAll({
  where: {
    published: true,
    title: { [Op.iLike]: '%sequelize%' },                  // Postgres case-insensitive
    createdAt: { [Op.gte]: new Date('2026-01-01') },
  },
  order: [['createdAt', 'DESC'], ['id', 'DESC']],
  limit: 10,
  offset: 20,
});
```

**Operators (`Op`):**

| Operator | Example | SQL |
|---|---|---|
| `Op.eq`, `Op.ne` | `{ role: { [Op.ne]: 'ADMIN' } }` | `role != 'ADMIN'` |
| `Op.gt`, `Op.gte`, `Op.lt`, `Op.lte` | `{ views: { [Op.gte]: 100 } }` | `views >= 100` |
| `Op.in`, `Op.notIn` | `{ id: [1, 2, 3] }` (shorthand) | `id IN (1,2,3)` |
| `Op.like`, `Op.iLike` | `{ email: { [Op.like]: '%@x.com' } }` | `LIKE` / `ILIKE` |
| `Op.startsWith`, `Op.endsWith`, `Op.substring` | `{ email: { [Op.endsWith]: '@x.com' } }` | `LIKE '%@x.com'` |
| `Op.between` | `{ views: { [Op.between]: [10, 50] } }` | `BETWEEN 10 AND 50` |
| `Op.is` | `{ name: null }` (shorthand) | `IS NULL` |
| `Op.or`, `Op.and`, `Op.not` | `{ [Op.or]: [{ a: 1 }, { b: 2 }] }` | `(a = 1 OR b = 2)` |

**Choose columns:**

```js
await User.findAll({ attributes: ['id', 'email'] });
await User.findAll({ attributes: { exclude: ['balance'] } });
await User.findAll({
  attributes: ['id', ['name', 'fullName']],                  // alias
});
```

**Count / paginate:**

```js
const total = await Post.count({ where: { published: true } });

const { rows, count } = await Post.findAndCountAll({
  where: { published: true },
  order: [['createdAt', 'DESC']],
  limit: 10,
  offset: 0,
});
```

**Plain objects instead of instances** (faster, but no instance methods):

```js
const users = await User.findAll({ raw: true });
```

---

### Update

**Instance update** (only changed fields are written, thanks to dirty
checking):

```js
const user = await User.findByPk(1);
user.name = 'Ashutosh';
user.changed();                 // ['name']
await user.save();
// UPDATE "users" SET "name"=$1,"updated_at"=$2 WHERE "id" = $3

// or in one call
await user.update({ name: 'Ashutosh' });
```

**Static update** (no fetch; returns affected count):

```js
const [affected] = await Post.update(
  { published: true },
  { where: { authorId: 1, published: false } }
);
```

**Atomic increment** (no read-modify-write race):

```js
await Post.increment('views', { by: 1, where: { id: 10 } });
// UPDATE "posts" SET "views" = "views" + 1 WHERE "id" = 10
```

**Upsert:**

```js
const [tag, created] = await Tag.upsert({ name: 'orm' });   // ON CONFLICT … DO UPDATE
```

---

### Delete

```js
const post = await Post.findByPk(10);
await post.destroy();                          // paranoid → UPDATE posts SET deleted_at = now()

await Post.destroy({ where: { published: false } });

// Paranoid models
await Post.findAll({ paranoid: false });       // include soft-deleted rows
await post.restore();                          // undo soft delete
await post.destroy({ force: true });           // real DELETE
```

`Model.destroy()` with no `where` throws. Use `truncate: true` only if you
really mean it.

---

### Handling Errors

| Error class | When | Typical HTTP |
|---|---|---|
| `ValidationError` | A `validate` rule or `allowNull` failed | 400 |
| `UniqueConstraintError` | Unique index violated | 409 |
| `ForeignKeyConstraintError` | FK violated | 400 / 409 |
| `DatabaseError` | Any other SQL error | 500 |

```js
const { ValidationError, UniqueConstraintError } = require('sequelize');

try {
  await User.create({ email: 'not-an-email' });
} catch (err) {
  if (err instanceof UniqueConstraintError) return res.status(409).json({ error: 'Email taken' });
  if (err instanceof ValidationError) {
    return res.status(400).json({ errors: err.errors.map((e) => ({ field: e.path, message: e.message })) });
  }
  throw err;
}
```

`UniqueConstraintError` extends `ValidationError`, so check it **first**.

---

### Putting It Together: an Express Router

```js
const { Router } = require('express');
const { User } = require('./models');

const users = Router();

users.get('/', async (req, res) => {
  const page = Number(req.query.page) || 1;
  const { rows, count } = await User.findAndCountAll({
    attributes: ['id', 'email', 'name'],
    order: [['id', 'ASC']],
    limit: 20,
    offset: (page - 1) * 20,
  });
  res.json({ items: rows, total: count });
});

users.get('/:id', async (req, res) => {
  const user = await User.findByPk(req.params.id);
  user ? res.json(user) : res.sendStatus(404);
});

users.post('/', async (req, res) => {
  const user = await User.create(req.body, { fields: ['email', 'name'] });
  res.status(201).json(user);
});

users.patch('/:id', async (req, res) => {
  const user = await User.findByPk(req.params.id);
  if (!user) return res.sendStatus(404);
  await user.update(req.body, { fields: ['name'] });
  res.json(user);
});

users.delete('/:id', async (req, res) => {
  const n = await User.destroy({ where: { id: req.params.id } });
  res.sendStatus(n ? 204 : 404);
});

module.exports = users;
```

---

### Interview-Ready Summary

- **Create**: `create` (validates, runs hooks), `build` + `save`, `bulkCreate`
  (`validate: true`), `findOrCreate`. Use `fields` to block mass assignment.
- **Read**: `findByPk`, `findOne`, `findAll` with `where` + `Op`, `attributes`,
  `order`, `limit`/`offset`; `findAndCountAll` for pagination; `raw: true`
  for plain objects.
- **Update**: instance `save`/`update` (dirty checking writes changed fields
  only), static `update`, atomic `increment`, `upsert`.
- **Delete**: `destroy`; `paranoid` models soft-delete, `restore()` undoes it,
  `force: true` hard-deletes.
- Catch `UniqueConstraintError` before `ValidationError`.

Next: [Relationships](04_relationships.md)
