## Concurrency

Two requests touching the same row at the same moment is normal in
production. Without a plan, one of them silently overwrites the other.

Background: [Transactions (Advanced)](05_transactions.md) ·
[Transactions](../03-orm-concepts/08_transactions.md)

---

### The Race: Read-Modify-Write

```js
// Two requests withdraw 30 from a balance of 100 at the same time
const user = await prisma.user.findUnique({ where: { id } });        // both read 100
await prisma.user.update({ where: { id }, data: { balance: user.balance - 30 } });  // both write 70
// Expected 40, got 70: one withdrawal was lost
```

```
Request A: read 100 ──────────── write 70
Request B:      read 100 ──────────── write 70
```

A transaction alone doesn't fix this under Read Committed: both
transactions read 100 and both commit. Use one of the four fixes below.

---

### Fix 1: Atomic Updates

Let the database do the arithmetic in one statement.

```js
// Prisma
await prisma.user.update({ where: { id }, data: { balance: { decrement: 30 } } });

// Sequelize
await User.decrement('balance', { by: 30, where: { id } });

// Mongoose
await User.updateOne({ _id: id }, { $inc: { balance: -30 } });
```

```sql
UPDATE users SET balance = balance - 30 WHERE id = $1;
```

The row lock taken by `UPDATE` serializes the two writers. This is the
simplest fix and works whenever the new value depends only on the old value.

---

### Fix 2: Conditional Updates

Put the business rule in the `WHERE` clause and check how many rows changed.

```js
// Prisma: updateMany returns { count }
const { count } = await prisma.user.updateMany({
  where: { id, balance: { gte: 30 } },
  data: { balance: { decrement: 30 } },
});
if (count === 0) throw new Error('Insufficient funds');

// Mongoose: findOneAndUpdate returns null if the filter didn't match
const user = await User.findOneAndUpdate(
  { _id: id, balance: { $gte: 30 } },
  { $inc: { balance: -30 } },
  { new: true },
);
```

Same pattern for inventory (`stock >= qty`), seat booking
(`seat_holder IS NULL`) and state machines (`status = 'pending'`).

---

### Fix 3: Optimistic Locking

Add a `version` column. Every update checks the version it read and
increments it. If someone else updated first, zero rows match and you retry
or report a conflict.

```js
// Prisma: manual version check
const post = await prisma.post.findUnique({ where: { id } });
const { count } = await prisma.post.updateMany({
  where: { id, version: post.version },
  data: { title: newTitle, version: { increment: 1 } },
});
if (count === 0) throw new ConflictError('Post was changed by someone else');

// Sequelize: built in
Post.init({ /* ... */ }, { sequelize, version: true });   // throws OptimisticLockError on conflict

// Mongoose: built in
const postSchema = new Schema({ /* ... */ }, { optimisticConcurrency: true });   // save() throws VersionError
```

Best for **low contention** and long "read, edit in a form, save" flows,
where holding a lock for minutes isn't possible. HTTP can expose it with
`ETag` / `If-Match`.

---

### Fix 4: Pessimistic Locking (`SELECT ... FOR UPDATE`)

#### Locking the Row

Lock the row when you read it, so other writers wait until you commit.

```js
// Prisma: raw SQL inside an interactive transaction
await prisma.$transaction(async (tx) => {
  const [acct] = await tx.$queryRaw`SELECT id, balance FROM accounts WHERE id = ${id} FOR UPDATE`;
  if (acct.balance < amount) throw new Error('Insufficient funds');
  await tx.account.update({ where: { id }, data: { balance: acct.balance - amount } });
});

// Sequelize
await sequelize.transaction(async (t) => {
  const acct = await Account.findByPk(id, { lock: t.LOCK.UPDATE, transaction: t });
  // ... checks and update with { transaction: t }
});
```

#### Lock Variants

| Clause | Behaviour |
|---|---|
| `FOR UPDATE` | Wait for the lock |
| `FOR UPDATE NOWAIT` | Fail immediately if locked |
| `FOR UPDATE SKIP LOCKED` | Skip locked rows (job queues) |
| `FOR SHARE` | Others can read-lock too, but not write |

Best for **high contention** and complex checks that span several rows.

---

### Job Queues with `SKIP LOCKED`

Many workers can pull jobs from one table without picking the same row.

```sql
UPDATE jobs SET status = 'running', locked_at = now()
 WHERE id = (
   SELECT id FROM jobs
    WHERE status = 'queued'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED
 )
RETURNING *;
```

This is how tools like `pg-boss` and Graphile Worker use PostgreSQL as a
reliable queue.

---

### Uniqueness Races: Let the Constraint Decide

```js
// Race: two sign-ups with the same email both pass the check
if (!(await prisma.user.findUnique({ where: { email } }))) {
  await prisma.user.create({ data: { email } });
}
```

Two requests can both see "no user" before either inserts. A **unique
constraint** is the only reliable check.

```js
try {
  await prisma.user.create({ data: { email } });
} catch (e) {
  if (e.code === 'P2002') throw new ConflictError('Email already registered');   // unique violation
  throw e;
}

// Or an upsert when "create or update" is the intent
await prisma.user.upsert({ where: { email }, create: { email, name }, update: { name } });
```

Sequelize throws `UniqueConstraintError`; MongoDB returns error code
`11000`.

---

### Locks Outside Rows

```sql
-- PostgreSQL advisory lock: one cron run at a time, released at commit
SELECT pg_try_advisory_xact_lock(hashtext('nightly-report'));
```

- **Advisory locks** coordinate work that isn't tied to a row (cron jobs,
  migrations, per-tenant imports).
- **Redis locks** (`SET key value NX PX 30000`) work across databases, but
  they expire. Always pair them with a database-level guard such as a
  conditional update or a fencing token.

---

### Choosing a Strategy

| Situation | Use |
|---|---|
| Counter, balance delta, stock decrement | Atomic update |
| Rule on the same row (`balance >= x`, `status = 'pending'`) | Conditional update |
| User edits a record in a form; conflicts are rare | Optimistic locking (`version`) |
| Many writers on a hot row; checks span several rows | `SELECT ... FOR UPDATE` |
| Worker queue | `FOR UPDATE SKIP LOCKED` |
| "Must be unique" | Unique constraint + handle the error |
| Invariant across many rows (write skew) | Serializable isolation + retry |

---

### Interview-Ready Summary

- **Read-modify-write** in application code loses updates, even inside a
  Read Committed transaction.
- Prefer **atomic** (`increment`, `$inc`) and **conditional** updates; check
  the affected row count.
- **Optimistic locking** (`version` column) for low contention;
  **pessimistic** `SELECT ... FOR UPDATE` for high contention.
- `SKIP LOCKED` turns a table into a safe multi-worker queue.
- Enforce uniqueness with a **unique constraint**, never check-then-insert.
- Advisory and Redis locks coordinate jobs; back them with a database-level
  guard.
