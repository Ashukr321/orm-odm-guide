## Sequelize Models

### Two Ways to Define a Model

**`sequelize.define`:**

```js
const User = sequelize.define('User', {
  email: { type: DataTypes.STRING, allowNull: false, unique: true },
}, { tableName: 'users' });
```

**`class extends Model` + `init`** (what the CLI generates; easier to add
instance and static methods):

```js
const { Model } = require('sequelize');

class User extends Model {}
User.init({
  email: { type: DataTypes.STRING, allowNull: false, unique: true },
}, { sequelize, modelName: 'User', tableName: 'users' });
```

Both produce the same model. This guide uses the class style.

---

### The Blog Models

**src/models/user.js**

```js
const { Model } = require('sequelize');

module.exports = (sequelize, DataTypes) => {
  class User extends Model {
    static associate(models) {
      User.hasOne(models.Profile, { foreignKey: 'userId', as: 'profile', onDelete: 'CASCADE' });
      User.hasMany(models.Post,   { foreignKey: 'authorId', as: 'posts', onDelete: 'CASCADE' });
    }

    // Instance method
    isAdmin() {
      return this.role === 'ADMIN';
    }

    // Hide sensitive fields when sent as JSON
    toJSON() {
      const { passwordHash, ...rest } = this.get();
      return rest;
    }
  }

  User.init(
    {
      id: { type: DataTypes.INTEGER, autoIncrement: true, primaryKey: true },
      email: {
        type: DataTypes.STRING(255),
        allowNull: false,
        unique: true,
        validate: { isEmail: true },
        set(value) {
          this.setDataValue('email', value.trim().toLowerCase());
        },
      },
      name: { type: DataTypes.STRING },
      passwordHash: { type: DataTypes.STRING },
      role: { type: DataTypes.ENUM('USER', 'ADMIN'), allowNull: false, defaultValue: 'USER' },
      balance: { type: DataTypes.INTEGER, allowNull: false, defaultValue: 0, validate: { min: 0 } },
      displayName: {
        type: DataTypes.VIRTUAL,                    // not stored in the DB
        get() {
          return this.name ?? this.email?.split('@')[0];
        },
      },
    },
    {
      sequelize,
      modelName: 'User',
      tableName: 'users',
      underscored: true,                            // created_at, updated_at
      defaultScope: { attributes: { exclude: ['passwordHash'] } },
      scopes: {
        admins: { where: { role: 'ADMIN' } },
        withPassword: { attributes: { include: ['passwordHash'] } },
      },
    }
  );

  return User;
};
```

**src/models/profile.js**

```js
const { Model } = require('sequelize');

module.exports = (sequelize, DataTypes) => {
  class Profile extends Model {
    static associate(models) {
      Profile.belongsTo(models.User, { foreignKey: 'userId', as: 'user' });
    }
  }
  Profile.init(
    {
      bio: DataTypes.TEXT,
      userId: { type: DataTypes.INTEGER, allowNull: false, unique: true },
    },
    { sequelize, modelName: 'Profile', tableName: 'profiles', underscored: true }
  );
  return Profile;
};
```

**src/models/post.js**

```js
const { Model } = require('sequelize');

module.exports = (sequelize, DataTypes) => {
  class Post extends Model {
    static associate(models) {
      Post.belongsTo(models.User, { foreignKey: 'authorId', as: 'author' });
      Post.belongsToMany(models.Tag, {
        through: 'post_tags', foreignKey: 'postId', otherKey: 'tagId', as: 'tags',
      });
    }
  }
  Post.init(
    {
      title: { type: DataTypes.STRING(200), allowNull: false, validate: { len: [3, 200] } },
      content: DataTypes.TEXT,
      published: { type: DataTypes.BOOLEAN, allowNull: false, defaultValue: false },
      views: { type: DataTypes.INTEGER, allowNull: false, defaultValue: 0 },
      authorId: { type: DataTypes.INTEGER, allowNull: false },
    },
    {
      sequelize,
      modelName: 'Post',
      tableName: 'posts',
      underscored: true,
      paranoid: true,                               // soft delete via deleted_at
      indexes: [{ fields: ['author_id'] }, { fields: ['published', 'created_at'] }],
      scopes: { published: { where: { published: true } } },
    }
  );
  return Post;
};
```

**src/models/tag.js**

```js
const { Model } = require('sequelize');

module.exports = (sequelize, DataTypes) => {
  class Tag extends Model {
    static associate(models) {
      Tag.belongsToMany(models.Post, {
        through: 'post_tags', foreignKey: 'tagId', otherKey: 'postId', as: 'posts',
      });
    }
  }
  Tag.init(
    { name: { type: DataTypes.STRING, allowNull: false, unique: true } },
    { sequelize, modelName: 'Tag', tableName: 'tags', underscored: true, timestamps: false }
  );
  return Tag;
};
```

---

### Data Types

| Sequelize | PostgreSQL | Notes |
|---|---|---|
| `STRING` / `STRING(n)` | `VARCHAR(255)` / `VARCHAR(n)` | |
| `TEXT` | `TEXT` | Long text |
| `INTEGER` / `BIGINT` | `INTEGER` / `BIGINT` | `BIGINT` comes back as a **string** by default |
| `FLOAT` / `DOUBLE` | `REAL` / `DOUBLE PRECISION` | |
| `DECIMAL(10, 2)` | `DECIMAL(10,2)` | Money; returned as a string |
| `BOOLEAN` | `BOOLEAN` | |
| `DATE` / `DATEONLY` | `TIMESTAMPTZ` / `DATE` | |
| `UUID` | `UUID` | `defaultValue: DataTypes.UUIDV4` |
| `ENUM('A','B')` | Native enum | |
| `JSON` / `JSONB` | `JSON` / `JSONB` | `JSONB` is Postgres only |
| `ARRAY(DataTypes.STRING)` | `VARCHAR[]` | Postgres only |
| `VIRTUAL` | none | Computed, not stored |

---

### Attribute Options

| Option | Purpose |
|---|---|
| `type` | Data type |
| `allowNull` | `false` → `NOT NULL` |
| `defaultValue` | Default (`DataTypes.NOW`, `DataTypes.UUIDV4`, literal) |
| `primaryKey`, `autoIncrement` | Primary key |
| `unique` | `true` or a named composite: `unique: 'user_slug'` |
| `field` | Column name if different (`field: 'full_name'`) |
| `references` | FK target (`{ model: 'users', key: 'id' }`) |
| `validate` | Validation rules |
| `get` / `set` | Getter / setter |

### Model Options

| Option | Effect |
|---|---|
| `tableName` | Exact table name (otherwise pluralized model name) |
| `underscored: true` | snake_case columns for timestamps and FKs |
| `timestamps: false` | No `createdAt`/`updatedAt` |
| `paranoid: true` | `destroy()` sets `deletedAt` instead of deleting |
| `indexes` | Index definitions |
| `defaultScope` / `scopes` | Reusable query presets |
| `hooks` | Lifecycle callbacks |
| `version: true` | Optimistic locking (see [Transactions](05_transactions.md)) |

---

### Validation

Validators run in JavaScript **before** the SQL is sent:

```js
email: {
  type: DataTypes.STRING,
  validate: {
    isEmail: { msg: 'Must be a valid email' },
    notEmpty: true,
  },
},
age: {
  type: DataTypes.INTEGER,
  validate: {
    min: 13,
    isAdult(value) {                     // custom validator
      if (value < 18 && this.role === 'ADMIN') throw new Error('Admins must be 18+');
    },
  },
},
```

Common built-ins: `isEmail`, `isUrl`, `isUUID`, `notEmpty`, `len: [min, max]`,
`min`, `max`, `isIn: [['a', 'b']]`, `is: /regex/`.

> `allowNull: false` and `unique: true` are **database constraints**.
> `validate` rules are application-level only.

---

### Hooks

```js
const bcrypt = require('bcrypt');

User.addHook('beforeSave', async (user) => {
  if (user.changed('passwordHash')) {
    user.passwordHash = await bcrypt.hash(user.passwordHash, 10);
  }
});
```

| Hook | Runs |
|---|---|
| `beforeValidate` / `afterValidate` | Around validation |
| `beforeCreate` / `afterCreate` | Around `create` / `save` of a new record |
| `beforeUpdate` / `afterUpdate` | Around `save` of an existing record |
| `beforeSave` / `afterSave` | Both create and update |
| `beforeDestroy` / `afterDestroy` | Around `destroy` |
| `beforeBulkCreate`, `beforeBulkUpdate`, … | Static bulk methods |

Static `Model.update()` / `Model.destroy()` don't run per-instance hooks
unless you pass `individualHooks: true`.

---

### Scopes

```js
await User.findAll();                            // defaultScope: passwordHash excluded
await User.scope('admins').findAll();            // WHERE role = 'ADMIN'
await User.scope('withPassword').findOne({ where: { email } });  // for login
await User.unscoped().findAll();                 // ignore defaultScope
await Post.scope('published').count();
```

---

### The Matching Migration

Models describe tables for your code; **migrations create them**. Keep them in
sync.

**src/db/migrations/20260925120000-create-users.js**

```js
'use strict';

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable('users', {
      id:            { type: Sequelize.INTEGER, autoIncrement: true, primaryKey: true },
      email:         { type: Sequelize.STRING(255), allowNull: false, unique: true },
      name:          { type: Sequelize.STRING },
      password_hash: { type: Sequelize.STRING },
      role:          { type: Sequelize.ENUM('USER', 'ADMIN'), allowNull: false, defaultValue: 'USER' },
      balance:       { type: Sequelize.INTEGER, allowNull: false, defaultValue: 0 },
      created_at:    { type: Sequelize.DATE, allowNull: false, defaultValue: Sequelize.fn('now') },
      updated_at:    { type: Sequelize.DATE, allowNull: false, defaultValue: Sequelize.fn('now') },
    });
  },

  async down(queryInterface) {
    await queryInterface.dropTable('users');
    await queryInterface.sequelize.query('DROP TYPE IF EXISTS "enum_users_role";');
  },
};
```

**Adding a column later:**

```js
module.exports = {
  up:   (qi, Sequelize) => qi.addColumn('users', 'avatar_url', { type: Sequelize.STRING }),
  down: (qi) => qi.removeColumn('users', 'avatar_url'),
};
```

Migrations use **column names** (`created_at`); models use **attribute names**
(`createdAt`) plus `underscored: true`.

---

### Interview-Ready Summary

- Define models with `sequelize.define` or `class extends Model` + `init`.
- Attributes: `type`, `allowNull`, `defaultValue`, `unique`, `field`,
  `validate`, `get`/`set`; `VIRTUAL` for computed fields.
- Options: `tableName`, `underscored`, `timestamps`, `paranoid` (soft delete),
  `indexes`, `scopes`/`defaultScope`, `hooks`.
- `validate` is app-level; `allowNull`/`unique` are DB constraints.
- Models don't create tables in production. **Migrations** do, and must be
  kept in sync by hand.

Next: [CRUD](03_crud.md)
