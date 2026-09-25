## Sequelize Setup

### What You'll Build

A Node.js project connected to **PostgreSQL** through Sequelize, using the
same blog domain as the Prisma pages so you can compare them directly:

```
User ──1:1── Profile
  │
  └──1:N── Post ──M:N── Tag   (through post_tags)
```

> Examples target **Sequelize v6** (the stable release) with CommonJS.
> Sequelize is an **Active Record** ORM: model instances save themselves.

---

### Prerequisites

| Tool | Version |
|---|---|
| Node.js | 18+ |
| PostgreSQL | 12+ (or MySQL / MariaDB / SQLite / MSSQL) |

```bash
docker run --name blog-db -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=blog -p 5432:5432 -d postgres:16
```

---

### 1. Install

```bash
mkdir sequelize-blog && cd sequelize-blog
npm init -y
npm install sequelize pg pg-hstore dotenv
npm install -D sequelize-cli
```

| Package | Role |
|---|---|
| `sequelize` | The ORM |
| `pg` + `pg-hstore` | PostgreSQL driver (+ hstore support) |
| `sequelize-cli` | Migrations, seeders, model generation |
| `dotenv` | Load `.env` |

**Driver per database:**

| Database | Install |
|---|---|
| PostgreSQL | `pg pg-hstore` |
| MySQL | `mysql2` |
| MariaDB | `mariadb` |
| SQLite | `sqlite3` |
| MSSQL | `tedious` |

---

### 2. Tell the CLI Where Things Live

**.sequelizerc** (project root)

```js
const path = require('path');

module.exports = {
  config:            path.resolve('src/db', 'config.js'),
  'models-path':     path.resolve('src', 'models'),
  'migrations-path': path.resolve('src/db', 'migrations'),
  'seeders-path':    path.resolve('src/db', 'seeders'),
};
```

```bash
npx sequelize-cli init
```

```
sequelize-blog/
├── .sequelizerc
├── .env
└── src/
    ├── db/
    │   ├── config.js        ← connection settings per environment
    │   ├── migrations/
    │   └── seeders/
    └── models/
        └── index.js         ← loads every model and sets up associations
```

---

### 3. Configure the Connection

**.env**

```bash
DATABASE_URL=postgres://postgres:postgres@localhost:5432/blog
```

**src/db/config.js** (use JS rather than the default JSON so it can read env vars)

```js
require('dotenv').config();

const common = {
  use_env_variable: 'DATABASE_URL',
  dialect: 'postgres',
  define: {
    underscored: true,   // createdAt → created_at, authorId → author_id
  },
};

module.exports = {
  development: { ...common, logging: console.log },
  test:        { ...common, logging: false },
  production: {
    ...common,
    logging: false,
    pool: { max: 10, min: 0, acquire: 30000, idle: 10000 },
    dialectOptions: { ssl: { require: true, rejectUnauthorized: false } },
  },
};
```

---

### 4. The Models Loader

`sequelize-cli init` generates `src/models/index.js`. Simplified, it does
this:

```js
const fs = require('fs');
const path = require('path');
const { Sequelize, DataTypes } = require('sequelize');
const env = process.env.NODE_ENV || 'development';
const config = require('../db/config.js')[env];

const sequelize = new Sequelize(process.env[config.use_env_variable], config);
const db = {};

// 1. Load every model file in this folder
fs.readdirSync(__dirname)
  .filter((f) => f !== 'index.js' && f.endsWith('.js'))
  .forEach((file) => {
    const model = require(path.join(__dirname, file))(sequelize, DataTypes);
    db[model.name] = model;
  });

// 2. Let each model declare its associations
Object.values(db).forEach((model) => model.associate?.(db));

db.sequelize = sequelize;
db.Sequelize = Sequelize;
module.exports = db;
```

Everywhere else in the app:

```js
const { sequelize, User, Post } = require('./models');
```

---

### 5. Test the Connection

**src/index.js**

```js
const { sequelize } = require('./models');

(async () => {
  try {
    await sequelize.authenticate();
    console.log('✅ Connected to the database');
  } catch (err) {
    console.error('❌ Unable to connect:', err.message);
  } finally {
    await sequelize.close();
  }
})();
```

```bash
node src/index.js
```

---

### 6. Create a Model and Its Migration

```bash
npx sequelize-cli model:generate --name User --attributes email:string,name:string
```

This creates `src/models/user.js` and
`src/db/migrations/<timestamp>-create-user.js`. Edit both (see
[Models](02_models.md)), then apply:

```bash
npx sequelize-cli db:migrate
```

---

### `sync()` vs Migrations

| | `sequelize.sync()` | Migrations |
|---|---|---|
| How | Creates tables from models at startup | Versioned up/down scripts |
| `sync({ alter: true })` | Tries to alter tables to match models | n/a |
| `sync({ force: true })` | **Drops** and recreates tables | n/a |
| History / rollback | ❌ | ✅ |
| Use for | Tests, quick prototypes | **Everything else, especially production** |

```js
// OK in a test setup file only
await sequelize.sync({ force: true });
```

---

### 7. Connection Pool

```js
new Sequelize(url, {
  pool: {
    max: 10,        // max connections
    min: 0,
    acquire: 30000, // ms to wait for a free connection before throwing
    idle: 10000,    // ms before an idle connection is released
  },
});
```

Create **one** `Sequelize` instance per app. Each instance has its own pool.
See [Connection Pooling](../../03-orm-concepts/09_connection-pooling.md).

---

### 8. Useful Scripts

```json
{
  "scripts": {
    "start": "node src/index.js",
    "db:migrate": "sequelize-cli db:migrate",
    "db:rollback": "sequelize-cli db:migrate:undo",
    "db:seed": "sequelize-cli db:seed:all",
    "db:reset": "sequelize-cli db:migrate:undo:all && sequelize-cli db:migrate && sequelize-cli db:seed:all"
  }
}
```

| Command | Does |
|---|---|
| `db:create` / `db:drop` | Create / drop the database |
| `migration:generate --name x` | Empty migration file |
| `db:migrate` | Apply pending migrations |
| `db:migrate:undo` | Roll back the last migration |
| `db:migrate:status` | Show applied / pending |
| `seed:generate --name x` | Empty seeder file |
| `db:seed:all` | Run all seeders |

---

### Common Setup Errors

| Error | Fix |
|---|---|
| `Please install pg package manually` | `npm install pg pg-hstore` |
| `ECONNREFUSED 127.0.0.1:5432` | DB not running, or wrong host/port |
| `password authentication failed` | Wrong credentials in `DATABASE_URL` |
| `Dialect needs to be explicitly supplied` | Add `dialect: 'postgres'` to the config |
| `relation "users" does not exist` | Run `db:migrate` |
| `SequelizeConnectionAcquireTimeoutError` | Pool exhausted; check for unreleased transactions or raise `pool.max` |

---

### Interview-Ready Summary

- Install `sequelize` + the dialect driver (`pg pg-hstore`) + `sequelize-cli`.
- `.sequelizerc` sets paths; `config.js` holds per-environment settings
  (`use_env_variable`, `dialect`, `pool`, `logging`).
- `models/index.js` loads models and calls each model's `associate()`.
- Use **migrations** in real projects; `sync({ force: true })` drops tables.
- Use **one** `Sequelize` instance (one pool) per app, and `authenticate()`
  to check the connection.

Next: [Models](02_models.md)
