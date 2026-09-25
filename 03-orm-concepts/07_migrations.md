## Migrations

### What Is a Migration

A **migration** is a versioned file that changes the database schema from
one state to the next: create a table, add a column, add an index. The
files are committed to git and applied **in order**, so every environment
(your laptop, CI, staging, production) ends up with the same schema.

```
migrations/
  20260901_create_users/migration.sql
  20260910_add_posts/migration.sql
  20260925_add_user_phone/migration.sql     ← newest
```

The database keeps a **migrations table** (`_prisma_migrations`,
`SequelizeMeta`, `migrations`) recording which files have already run, so
each one runs **exactly once**.

---

### Why Not Just Edit the Database by Hand?

| By hand | With migrations |
|---|---|
| "Did anyone add that column on staging?" | Every change is in git, with an author and a review |
| Each environment drifts | Every environment follows the same ordered history |
| New developers copy a dump from someone | `migrate` builds the schema from zero |
| Rolling back means guessing | The previous state is a known version |
| Deploys break because the schema doesn't match the code | Migrations run as a deploy step, before the new code |

---

### Prisma Migrate

```bash
# 1. Edit schema.prisma (e.g. add `phone String?` to User), then:
npx prisma migrate dev --name add_user_phone
#    → generates prisma/migrations/<timestamp>_add_user_phone/migration.sql
#    → applies it to your dev database and regenerates the client

# 2. In CI / production: apply pending migrations, never generate
npx prisma migrate deploy
```

```sql
-- prisma/migrations/20260925101500_add_user_phone/migration.sql
ALTER TABLE "users" ADD COLUMN "phone" TEXT;
```

| Command | Where | What it does |
|---|---|---|
| `migrate dev` | Local only | Diff the schema, create and apply a migration, may reset the dev DB |
| `migrate deploy` | CI / production | Apply pending migrations only; never resets |
| `migrate status` | Anywhere | Show applied and pending migrations |
| `db push` | Prototyping only | Sync the schema **without** migration files |

---

### Sequelize and TypeORM Migrations

```js
// Sequelize CLI: npx sequelize-cli migration:generate --name add-user-phone
module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.addColumn('users', 'phone', { type: Sequelize.STRING });
    await queryInterface.addIndex('users', ['phone']);
  },
  async down(queryInterface) {
    await queryInterface.removeIndex('users', ['phone']);
    await queryInterface.removeColumn('users', 'phone');
  },
};
// npx sequelize-cli db:migrate   /   db:migrate:undo
```

```ts
// TypeORM: npx typeorm migration:generate src/migrations/AddUserPhone -d src/data-source.ts
export class AddUserPhone1727258100000 implements MigrationInterface {
  async up(q: QueryRunner) { await q.query(`ALTER TABLE "users" ADD "phone" varchar`); }
  async down(q: QueryRunner) { await q.query(`ALTER TABLE "users" DROP COLUMN "phone"`); }
}
// npx typeorm migration:run -d src/data-source.ts
```

`up` applies the change. `down` reverses it (Prisma generates only
forward migrations).

---

### The Golden Rules

1. **Never edit a migration that has already run** on a shared
   environment. Write a new one.
2. **Commit migrations with the code that needs them**, in the same pull
   request.
3. **Review the generated SQL.** A column rename can be generated as
   "drop + add", which destroys the data.
4. **Run migrations as a deploy step** (`migrate deploy`), not on app
   startup from several instances at once.
5. **Never use schema sync in production**: TypeORM `synchronize: true`,
   Sequelize `sync({ alter: true })`, Prisma `db push`.
6. **Test migrations** against a copy of production-sized data before
   running them on production.

---

### Zero-Downtime Changes: Expand and Contract

During a deploy, old and new code run **at the same time**. A breaking
schema change (rename, drop, `NOT NULL`) must be split into safe steps:

```
Rename users.name → users.full_name without downtime:

1. Expand     ADD COLUMN full_name              (nullable; old code ignores it)
2. Dual-write deploy code that writes BOTH name and full_name
3. Backfill   UPDATE users SET full_name = name WHERE full_name IS NULL   (in batches)
4. Switch     deploy code that reads full_name
5. Contract   DROP COLUMN name                  (after every instance runs the new code)
```

Each step is its own migration and its own deploy.

---

### Locks: When a Migration Can Take Production Down

Some DDL statements lock the table while they run, blocking reads or
writes:

| Operation (PostgreSQL) | Risk | Safer way |
|---|---|---|
| `CREATE INDEX` | Blocks writes while building | `CREATE INDEX CONCURRENTLY` (can't run inside a transaction) |
| `ADD COLUMN … DEFAULT x` | Fast since PostgreSQL 11 | ✅ fine |
| `ADD COLUMN … NOT NULL` without a default on a full table | Fails | Add nullable → backfill → `SET NOT NULL` |
| Change a column type | Rewrites the table | Add a new column and migrate the data |
| `ADD FOREIGN KEY` | Scans the table under lock | `ADD … NOT VALID`, then `VALIDATE CONSTRAINT` |

```sql
-- Prisma: hand-edit the migration file for a concurrent index
-- (generate with `migrate dev --create-only`, then edit before applying)
CREATE INDEX CONCURRENTLY "orders_user_id_idx" ON "orders"("user_id");
```

Set a `lock_timeout` so a migration fails fast instead of queueing behind
a long query.

---

### Schema Migrations vs Data Migrations vs Seeds

| | Changes | Example | Runs |
|---|---|---|---|
| **Schema migration** | Structure | Add a column, create an index | Once, in order |
| **Data migration** | Existing rows | Backfill `full_name`, fix bad statuses | Once; in batches for big tables |
| **Seed** | Starting data | Admin user, countries, demo data | Dev / test, or on first setup |

```bash
npx prisma db seed          # runs the seed script set in package.json / prisma config
```

Keep large data backfills **out of** schema migrations: run them in
batches so they don't hold locks or time out.

---

### Rolling Back

- **Down migrations** (Sequelize, TypeORM) can undo schema changes, but
  they **can't restore dropped data**.
- In production, most teams **roll forward**: they fix the problem with a
  new migration.
- Before any destructive migration (drop, type change): **take a backup**
  and confirm the restore works.

---

### Interview-Ready Summary

- A migration is a **versioned, ordered schema change** in git. A
  migrations table makes sure each one runs **exactly once**.
- Use `migrate dev` locally and `migrate deploy` in CI / production.
  **Never schema-sync in production.**
- **Never edit applied migrations**, and always review the generated SQL.
- Make breaking changes with **expand → backfill → contract**, one deploy
  per step.
- Watch out for **locks**: `CREATE INDEX CONCURRENTLY`, `NOT VALID`
  constraints, `lock_timeout`.
- Keep **schema changes, data backfills and seeds** separate. Roll forward
  in production.
