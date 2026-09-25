## Mongoose Setup

### What You'll Build

A Node.js blog API on **MongoDB** with Mongoose. It's the same domain as
the Prisma and Sequelize pages, modelled the document way:

```
users ──1:N── posts        (post.author → users._id)
                │
                ├── comments[]   embedded in each post
                └── tags[]       references to tags
```

> Examples target **Mongoose 8** with ES modules. Concepts behind each step
> are in [ODM Concepts](../../06-odm-concepts/01_schemas.md).

---

### Prerequisites

| Tool | Version |
|---|---|
| Node.js | 20+ |
| MongoDB | 7+ (Docker, local install, or MongoDB Atlas) |

**Start MongoDB with Docker as a single-node replica set** (a replica set
is required for **transactions**):

```bash
docker run -d --name blog-mongo -p 27017:27017 mongo:8 --replSet rs0
```

```bash
docker exec blog-mongo mongosh --quiet --eval "rs.initiate({ _id: 'rs0', members: [{ _id: 0, host: 'localhost:27017' }] })"
```

**Or use MongoDB Atlas** (free tier): create a cluster, add a database
user, allow your IP, and copy the `mongodb+srv://…` connection string.
Atlas clusters are replica sets already.

---

### 1. Create the Project

```bash
mkdir mongoose-blog && cd mongoose-blog
npm init -y
npm install mongoose dotenv express
npm install -D nodemon
```

**package.json**

```json
{
  "type": "module",
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js",
    "seed": "node src/seed.js",
    "db:indexes": "node src/scripts/sync-indexes.js"
  }
}
```

```
mongoose-blog/
├── .env
├── package.json
└── src/
    ├── db.js              ← connection
    ├── models/
    │   ├── user.js
    │   ├── post.js
    │   └── tag.js
    ├── routes/
    ├── scripts/
    ├── seed.js
    └── server.js
```

---

### 2. Configure the Connection String

**.env**

```bash
# Docker replica set
MONGODB_URI="mongodb://localhost:27017/blog?replicaSet=rs0"

# Atlas
# MONGODB_URI="mongodb+srv://USER:PASSWORD@cluster0.xxxxx.mongodb.net/blog?retryWrites=true&w=majority"
```

| Part | Meaning |
|---|---|
| `mongodb://` / `mongodb+srv://` | Standard / DNS seed-list (Atlas) format |
| `localhost:27017` | Host and port |
| `/blog` | Database name (created on first write) |
| `replicaSet=rs0` | Replica set name (needed for transactions locally) |
| `retryWrites=true&w=majority` | Retry transient write errors; wait for majority acknowledgement |

Add `.env` to `.gitignore`.

---

### 3. Connect Once

**src/db.js**

```js
import 'dotenv/config';
import mongoose from 'mongoose';

mongoose.set('strictQuery', true);                             // drop unknown filter keys
if (process.env.NODE_ENV === 'development') mongoose.set('debug', true);  // log every command

export async function connectDB() {
  await mongoose.connect(process.env.MONGODB_URI, {
    maxPoolSize: 10,                    // connections in the pool (default 100)
    serverSelectionTimeoutMS: 5000,     // fail fast if MongoDB is unreachable
    autoIndex: process.env.NODE_ENV !== 'production',   // build indexes on startup in dev only
  });
  console.log(`✅ MongoDB connected: ${mongoose.connection.name}`);
}

mongoose.connection.on('disconnected', () => console.warn('⚠️  MongoDB disconnected'));
mongoose.connection.on('error', (err) => console.error('MongoDB error:', err.message));

export async function disconnectDB() {
  await mongoose.disconnect();
}
```

**src/server.js**

```js
import express from 'express';
import { connectDB, disconnectDB } from './db.js';

const app = express();
app.use(express.json());
app.get('/health', (_req, res) => res.json({ ok: true }));

await connectDB();                      // connect BEFORE accepting requests
const server = app.listen(3000, () => console.log('API on http://localhost:3000'));

// Graceful shutdown: finish requests, then close the pool
for (const signal of ['SIGINT', 'SIGTERM']) {
  process.on(signal, async () => {
    server.close();
    await disconnectDB();
    process.exit(0);
  });
}
```

```bash
npm run dev
```

```
✅ MongoDB connected: blog
API on http://localhost:3000
```

---

### Connection Details Worth Knowing

**One connection pool per app.** `mongoose.connect()` creates the default
connection and its pool; every model uses it. Don't connect per request.

**Command buffering.** Mongoose queues model calls until the connection
is open. If it never opens, you get:

```
MongooseError: Operation `users.findOne()` buffering timed out after 10000ms
```

That almost always means "not connected". Check the URI, and make sure
you `await connectDB()` before serving traffic.

**Serverless / hot reload** (Next.js, Vercel functions): cache the
connection on `globalThis` so each invocation doesn't open a new pool.

```js
const cached = (globalThis._mongoose ??= { conn: null, promise: null });

export async function connectDB() {
  if (cached.conn) return cached.conn;
  cached.promise ??= mongoose.connect(process.env.MONGODB_URI, { maxPoolSize: 5 });
  cached.conn = await cached.promise;
  return cached.conn;
}
```

**Several databases:** use `mongoose.createConnection(uri)` and register
models on that connection. See [Models](../../06-odm-concepts/02_models.md).

---

### 4. Seed Some Data

**src/seed.js** (the models come from [Schema](02_schema.md))

```js
import { connectDB, disconnectDB } from './db.js';
import { User } from './models/user.js';
import { Post } from './models/post.js';
import { Tag } from './models/tag.js';

await connectDB();
await Promise.all([User.deleteMany(), Post.deleteMany(), Tag.deleteMany()]);

const [ashu, riya] = await User.create([
  { email: 'ashu@example.com', name: 'Ashu', password: 'password123', role: 'admin' },
  { email: 'riya@example.com', name: 'Riya', password: 'password123' },
]);
const [mongo, node] = await Tag.create([{ name: 'mongodb' }, { name: 'node' }]);

await Post.create([
  { title: 'Hello Mongoose', body: 'First post', author: ashu._id, tags: [mongo._id], published: true,
    comments: [{ body: 'Nice!', author: riya._id }] },
  { title: 'Embedding vs referencing', body: 'Draft', author: riya._id, tags: [mongo._id, node._id] },
]);

console.log('🌱 Seeded');
await disconnectDB();
```

```bash
npm run seed
```

`User.create([...])` runs validation and `save` hooks (like password
hashing) for each document. `insertMany` would be faster but skips `save`
hooks.

---

### 5. Build Indexes in Production

With `autoIndex` off in production, build indexes as a deploy step:

**src/scripts/sync-indexes.js**

```js
import { connectDB, disconnectDB } from '../db.js';
import { User } from '../models/user.js';
import { Post } from '../models/post.js';
import { Tag } from '../models/tag.js';

await connectDB();
for (const model of [User, Post, Tag]) {
  console.log(model.modelName, await model.diffIndexes());
  await model.syncIndexes();
}
await disconnectDB();
```

See [Indexes](../../06-odm-concepts/09_indexes.md).

---

### Inspecting Data

| Tool | Use |
|---|---|
| `mongosh` | Shell: `docker exec -it blog-mongo mongosh blog`, then `db.posts.find()` |
| MongoDB Compass | GUI: browse documents, indexes, explain plans |
| Atlas Data Explorer | Web UI for Atlas clusters |
| `mongoose.set('debug', true)` | Print every command your app sends |

---

### Common Setup Errors

| Error | Fix |
|---|---|
| `MongooseServerSelectionError: connect ECONNREFUSED` | MongoDB isn't running, or wrong host/port |
| `buffering timed out after 10000ms` | Not connected yet; `await connectDB()` first; check the URI |
| `bad auth : authentication failed` | Wrong user/password; URL-encode special characters in the password |
| Atlas `querySrv ENOTFOUND` / timeout | Wrong cluster host, DNS issues, or your IP isn't allow-listed |
| `Transaction numbers are only allowed on a replica set member or mongos` | Start MongoDB as a replica set (see Prerequisites) |
| `OverwriteModelError: Cannot overwrite model once compiled` | Hot reload re-registering models; use `mongoose.models.X || model(...)` |

---

### Interview-Ready Summary

- Install `mongoose` (it bundles the official `mongodb` driver) and put the
  connection string in `MONGODB_URI`.
- Call **`mongoose.connect()` once** at startup, before serving requests;
  it creates one pooled default connection (`maxPoolSize`).
- Set `serverSelectionTimeoutMS` to fail fast, `strictQuery`, `debug` in
  development, and `autoIndex` only outside production.
- "Buffering timed out" means **not connected**; transactions need a
  **replica set** (Atlas or a single-node `--replSet` locally).
- Cache the connection on `globalThis` in serverless/hot-reload
  environments, and close it on shutdown.

Next: [Schema](02_schema.md)
