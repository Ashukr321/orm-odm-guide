## Sequelize Transactions

### Why Transactions

A transaction makes several writes **all-or-nothing** (ACID). The running
example is the same credit transfer used in the
[Prisma Transactions](../prisma/06_transactions.md) page.

Concepts: [Transactions](../../03-orm-concepts/08_transactions.md)

### Two Styles

| | Managed | Unmanaged |
|---|---|---|
| API | `sequelize.transaction(async (t) => …)` | `const t = await sequelize.transaction()` |
| Commit | Automatic when the callback resolves | You call `t.commit()` |
| Rollback | Automatic when the callback throws | You call `t.rollback()` |
| Recommended | ✅ Yes | Only when you need manual control |

---

### Managed Transactions (Recommended)

```js
const { sequelize, User } = require('./models');

async function transfer(fromId, toId, amount) {
  return sequelize.transaction(async (t) => {
    const sender = await User.findByPk(fromId, { transaction: t, lock: t.LOCK.UPDATE });
    const receiver = await User.findByPk(toId, { transaction: t, lock: t.LOCK.UPDATE });

    if (!sender || !receiver) throw new Error('User not found');
    if (sender.balance < amount) throw new Error('Insufficient balance');

    await sender.decrement('balance', { by: amount, transaction: t });
    await receiver.increment('balance', { by: amount, transaction: t });

    return { from: sender.id, to: receiver.id, amount };
  });
  // resolved → COMMIT, thrown → ROLLBACK (and the error is re-thrown)
}
```

```
START TRANSACTION;
SELECT … FROM "users" WHERE "id" = 1 FOR UPDATE;
SELECT … FROM "users" WHERE "id" = 2 FOR UPDATE;
UPDATE "users" SET "balance" = "balance" - 100 WHERE "id" = 1;
UPDATE "users" SET "balance" = "balance" + 100 WHERE "id" = 2;
COMMIT;
```

> **Pass `{ transaction: t }` to every query.** A query without it runs on
> a different connection, outside the transaction, and won't be rolled back.

To avoid deadlocks when two transfers run in opposite directions, lock
rows in a consistent order (for example, lower `id` first).

---

### Unmanaged Transactions

```js
const t = await sequelize.transaction();
try {
  const user = await User.create({ email: 'ashu@example.com' }, { transaction: t });
  await user.createProfile({ bio: 'Backend dev' }, { transaction: t });
  await t.commit();
} catch (err) {
  await t.rollback();
  throw err;
}
```

Forgetting `commit()`/`rollback()` on some code path leaves the connection
checked out and eventually exhausts the pool.

---

### Automatic Transaction Passing (CLS)

Passing `{ transaction: t }` everywhere is easy to forget. With
**continuation-local storage**, Sequelize attaches the current
transaction to every query automatically.

```bash
npm install cls-hooked
```

```js
// Must run BEFORE creating the Sequelize instance
const cls = require('cls-hooked');
const { Sequelize } = require('sequelize');

Sequelize.useCLS(cls.createNamespace('app-tx'));
const sequelize = new Sequelize(process.env.DATABASE_URL);
```

```js
await sequelize.transaction(async () => {
  const user = await User.create({ email: 'x@example.com' });   // uses the transaction
  await Post.create({ title: 'Hi', authorId: user.id });        // uses the transaction
});
```

Only **managed** transactions use CLS.

---

### Isolation Levels

```js
const { Transaction } = require('sequelize');

await sequelize.transaction(
  { isolationLevel: Transaction.ISOLATION_LEVELS.SERIALIZABLE },
  async (t) => { /* … */ }
);
```

| Level | Prevents | Notes |
|---|---|---|
| `READ_UNCOMMITTED` | Nothing | Postgres treats it as READ COMMITTED |
| `READ_COMMITTED` | Dirty reads | Postgres default |
| `REPEATABLE_READ` | + Non-repeatable reads | MySQL InnoDB default |
| `SERIALIZABLE` | + Phantoms / write skew | May fail with a serialization error; retry |

**Retry on serialization failure / deadlock:**

```js
async function withRetry(fn, attempts = 3) {
  for (let i = 1; ; i++) {
    try {
      return await fn();
    } catch (err) {
      const code = err.parent?.code;           // Postgres: 40001 serialization, 40P01 deadlock
      if (!['40001', '40P01'].includes(code) || i >= attempts) throw err;
    }
  }
}

await withRetry(() =>
  sequelize.transaction({ isolationLevel: Transaction.ISOLATION_LEVELS.SERIALIZABLE }, async (t) => {
    /* … */
  })
);
```

---

### Row Locking (Pessimistic)

```js
await sequelize.transaction(async (t) => {
  const post = await Post.findByPk(10, { transaction: t, lock: t.LOCK.UPDATE });
  // other transactions trying to lock row 10 now wait
  post.views += 1;
  await post.save({ transaction: t });
});
```

| Lock | SQL |
|---|---|
| `t.LOCK.UPDATE` | `FOR UPDATE` |
| `t.LOCK.SHARE` | `FOR SHARE` |
| `t.LOCK.KEY_SHARE` | `FOR KEY SHARE` (Postgres) |
| `t.LOCK.NO_KEY_UPDATE` | `FOR NO KEY UPDATE` (Postgres) |
| `skipLocked: true` | `SKIP LOCKED` (job queues) |

**Job queue pattern with `SKIP LOCKED`:**

```js
await sequelize.transaction(async (t) => {
  const job = await Job.findOne({
    where: { status: 'pending' },
    order: [['id', 'ASC']],
    lock: t.LOCK.UPDATE,
    skipLocked: true,                   // workers never pick the same job
    transaction: t,
  });
  if (!job) return;
  await job.update({ status: 'running' }, { transaction: t });
});
```

---

### Optimistic Locking

Turn it on per model with `version: true`:

```js
Post.init({ /* … */ }, { sequelize, modelName: 'Post', version: true });
// adds a "version" column (add it in a migration too)
```

```js
const { OptimisticLockError } = require('sequelize');

const post = await Post.findByPk(10);    // version = 3
post.title = 'Updated';
try {
  await post.save();
  // UPDATE posts SET title = …, version = 4 WHERE id = 10 AND version = 3
} catch (err) {
  if (err instanceof OptimisticLockError) {
    // someone else saved first → reload and ask the user to retry
  }
}
```

See [Concurrency](../../09-advanced/06_concurrency.md).

---

### Run Code Only After Commit

Send emails, publish events or clear caches **after** the data is
committed, never inside the transaction:

```js
await sequelize.transaction(async (t) => {
  const post = await Post.create({ title, authorId }, { transaction: t });
  t.afterCommit(() => notifyFollowers(post.id));
});
```

`afterCommit` callbacks don't run if the transaction rolls back.

---

### Pitfalls

| Pitfall | Why it hurts | Fix |
|---|---|---|
| Missing `{ transaction: t }` on a query | Runs outside the transaction | Pass it everywhere, or use CLS |
| Unmanaged transaction without `rollback` in `catch` | Leaked connection, pool exhaustion | Prefer managed transactions |
| HTTP calls / slow work inside | Holds locks and a connection | Use `t.afterCommit` |
| `user.balance -= x; save()` without a lock | Lost updates under concurrency | `increment`/`decrement`, `lock`, or `version: true` |
| Locking rows in random order | Deadlocks | Lock in a consistent order; retry `40P01` |
| Long transactions | Lock contention | Keep them short |

---

### Interview-Ready Summary

- **Managed** `sequelize.transaction(async (t) => …)` commits on resolve and
  rolls back on throw; **unmanaged** needs explicit `commit()`/`rollback()`.
- Pass **`{ transaction: t }`** to every query, or enable **CLS**
  (`Sequelize.useCLS`) to do it automatically.
- Set `isolationLevel`; retry serialization failures (`40001`) and deadlocks
  (`40P01`).
- **Pessimistic**: `lock: t.LOCK.UPDATE` (+ `skipLocked` for queues).
  **Optimistic**: `version: true` → `OptimisticLockError`.
- Use `t.afterCommit()` for side effects; keep transactions short.

Compare: [Prisma Transactions](../prisma/06_transactions.md)
