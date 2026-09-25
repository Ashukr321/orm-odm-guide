## Free Platforms & Databases for Projects

Free tiers you can use to build and host the ORM/ODM examples in this repo
without paying anything. Free tiers change often: check the provider's
pricing page before you rely on one.

---

### Start Local: Docker Is Always Free

Every example in this guide runs on your machine first. No account, no
limits, no sleeping databases.

```bash
# PostgreSQL (Prisma / Sequelize examples)
docker run -d --name blog-pg -p 5432:5432 -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=blog postgres:17

# MySQL
docker run -d --name blog-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=mysql -e MYSQL_DATABASE=blog mysql:8

# MongoDB as a single-node replica set (needed for transactions)
docker run -d --name blog-mongo -p 27017:27017 mongo:8 --replSet rs0
docker exec blog-mongo mongosh --quiet --eval "rs.initiate({ _id: 'rs0', members: [{ _id: 0, host: 'localhost:27017' }] })"

# Redis (Caching page)
docker run -d --name blog-redis -p 6379:6379 redis:7
```

Move to a hosted free tier when you want to deploy or share the project.

---

### Free Hosting / Deployment Platforms

| Platform | Good for | Free tier notes | Link |
|---|---|---|---|
| Vercel | Next.js, frontends, serverless API routes | Hobby plan, non-commercial use | [vercel.com](https://vercel.com) |
| Netlify | Static sites, serverless functions | Free plan with monthly usage limits | [netlify.com](https://netlify.com) |
| Render | Node.js APIs, cron jobs, background workers | Free web services sleep after ~15 min idle | [render.com](https://render.com) |
| Koyeb | Dockerized or Git-deployed APIs | One small free service | [koyeb.com](https://www.koyeb.com) |
| Cloudflare Workers / Pages | Edge APIs, static sites | Generous daily request limits | [workers.cloudflare.com](https://workers.cloudflare.com) |
| Deno Deploy | TypeScript/JavaScript at the edge | Free plan for hobby projects | [deno.com/deploy](https://deno.com/deploy) |
| GitHub Pages | Static sites, docs, these slide decks | Free for public repos | [pages.github.com](https://pages.github.com) |

**Serverless and edge platforms** open many short-lived connections. Use the
database's **pooled** connection string (Neon, Supabase) or a driver
adapter; see [Connection Pooling](../03-orm-concepts/09_connection-pooling.md).

---

### Free SQL Databases

| Provider | Database | Works with | Free tier notes | Link |
|---|---|---|---|---|
| Neon | Serverless PostgreSQL | Prisma, Sequelize | Scales to zero, database branching | [neon.tech](https://neon.tech) |
| Supabase | PostgreSQL + auth + storage | Prisma, Sequelize | Free projects pause after a week of inactivity | [supabase.com](https://supabase.com) |
| Prisma Postgres | Managed PostgreSQL | Prisma (any Postgres client) | Free plan with operation limits | [prisma.io/postgres](https://www.prisma.io/postgres) |
| Aiven | PostgreSQL, MySQL | Prisma, Sequelize | Free plan: one small single-node service | [aiven.io](https://aiven.io) |
| TiDB Cloud | MySQL-compatible, serverless | Prisma, Sequelize | Free monthly usage allowance | [tidbcloud.com](https://www.pingcap.com/tidb-cloud/) |
| CockroachDB Cloud | PostgreSQL-compatible, distributed | Prisma (`cockroachdb` provider) | Free monthly usage allowance | [cockroachlabs.com](https://www.cockroachlabs.com) |
| Turso | SQLite (libSQL), edge replicas | Prisma (driver adapter), Drizzle | Free plan with many small databases | [turso.tech](https://turso.tech) |

For the Prisma and Sequelize examples, **Neon** or **Supabase** is the
closest match to the local PostgreSQL setup.

---

### Free NoSQL / Document Databases

| Provider | Database | Works with | Free tier notes | Link |
|---|---|---|---|---|
| MongoDB Atlas | MongoDB | Mongoose, Prisma (MongoDB connector) | Free M0 cluster (512 MB), replica set, so transactions work | [mongodb.com/atlas](https://www.mongodb.com/atlas) |
| Upstash | Serverless Redis | ioredis, `@upstash/redis` | Free daily command allowance | [upstash.com](https://upstash.com) |
| Redis Cloud | Redis | ioredis | Small free database (~30 MB) | [redis.io/cloud](https://redis.io/cloud/) |
| Firebase Firestore | Document database | Firebase SDK only | Spark plan; not usable with Mongoose | [firebase.google.com](https://firebase.google.com) |

**MongoDB Atlas M0** covers every Mongoose example, including transactions
and aggregation. Use **Upstash** or **Redis Cloud** for the
[Caching](../09-advanced/04_caching.md) examples.

---

### Which One for Each Part of This Guide

| Section | Local | Hosted free option |
|---|---|---|
| 04 ORM Examples (Prisma, Sequelize) | Docker PostgreSQL / MySQL | Neon, Supabase, Aiven |
| 07 ODM Examples (Mongoose) | Docker MongoDB replica set | MongoDB Atlas M0 |
| 09 Advanced: Caching | Docker Redis | Upstash, Redis Cloud |
| 09 Advanced: Read replicas | Two Docker containers | Neon read replicas |
| Deploying an example API | — | Render, Koyeb, Vercel |
| Hosting the slide decks | — | GitHub Pages, Netlify, Cloudflare Pages |

---

### Connecting the Examples to a Hosted Database

#### Environment and Prisma

```bash
# .env
DATABASE_URL="postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/blog?sslmode=require"
DIRECT_URL="postgresql://user:pass@ep-xxx.region.aws.neon.tech/blog?sslmode=require"
MONGODB_URI="mongodb+srv://user:pass@cluster0.xxxxx.mongodb.net/blog?retryWrites=true&w=majority"
```

```ts
// prisma.config.ts: the CLI (migrations) uses the direct, unpooled URL
import 'dotenv/config';
import { defineConfig, env } from 'prisma/config';

export default defineConfig({
  schema: 'prisma/schema.prisma',
  datasource: { url: env('DIRECT_URL') },
});

// src/db.ts: the app uses the pooled URL through the adapter
const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL! });
export const prisma = new PrismaClient({ adapter });
```

#### Sequelize, Mongoose and Checklist

```js
// Sequelize: hosted Postgres requires SSL
const sequelize = new Sequelize(process.env.DATABASE_URL, {
  dialect: 'postgres',
  dialectOptions: { ssl: { require: true } },
});

// Mongoose: same code as local, just a different URI
await mongoose.connect(process.env.MONGODB_URI);
```

- Never commit `.env`; add it to `.gitignore`.
- Atlas: add your IP (or `0.0.0.0/0` for a hosted app) under **Network
  Access**.
- Run migrations against the **direct** URL, and the app against the
  **pooled** one.

---

### No Longer Free (Avoid Old Tutorials)

| Service | What changed |
|---|---|
| Cyclic | Shut down in 2024 |
| Glitch | Stopped hosting projects in 2025 |
| ElephantSQL | Shut down in January 2025 |
| Fauna | Shut down in 2025 |
| PlanetScale | Removed its free Hobby plan in 2024 |
| Fly.io | No free allowance for new accounts; pay-as-you-go |
| Railway | Trial credit only, then paid plans |
| Heroku | Free dynos and databases removed in 2022 |
| Upstash Kafka | Discontinued; Upstash Redis is still free |

If a tutorial tells you to sign up for one of these, swap in an option from
the tables above.

---

### Free-Tier Gotchas

| Gotcha | What it means for you |
|---|---|
| Sleep / scale to zero | First request after idle is slow (cold start) |
| Paused projects | Supabase pauses inactive free projects; wake them from the dashboard |
| Expiring databases | Some hosts delete free databases after a trial period; keep backups |
| Connection limits | Small plans allow few connections; use pooling |
| Region | Put the app and the database in the same region |
| No backups | Free tiers rarely include point-in-time recovery; `pg_dump` / `mongodump` yourself |

Use these for practicing the ORM (Prisma/Sequelize + SQL) and ODM
(Mongoose + MongoDB) examples in this guide without any cost.
