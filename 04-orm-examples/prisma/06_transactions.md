## Prisma Transactions

### Why Transactions

A transaction makes several writes **all-or-nothing** (ACID). Classic
example: transferring credits between two users. If the debit succeeds
and the credit fails, money disappears.

Concepts: [Transactions](../../03-orm-concepts/08_transactions.md)

### Four Ways to Get Atomicity in Prisma

| Approach | Use when |
|---|---|
| **Nested writes** | Creating/updating a record and its relations together |
| **Bulk operations** (`updateMany`, `deleteMany`, `createMany`) | Many rows, one statement |
| **Batch** `$transaction([...])` | Independent queries that must all succeed |
| **Interactive** `$transaction(async (tx) => …)` | Later steps depend on earlier results or need logic |

---

### 1. Nested Writes Are Already Atomic

```ts
await prisma.user.create({
  data: {
    email: 'ashu@example.com',
    profile: { create: { bio: 'Backend dev' } },
    posts:   { create: [{ title: 'Hello' }, { title: 'ORMs 101' }] },
  },
});
// BEGIN; INSERT users; INSERT profiles; INSERT posts …; COMMIT
```

If any insert fails, nothing is saved. You don't need `$transaction` here.

---

### 2. Batch Transactions

Pass an **array** of queries. They run in order, in one transaction, and
the results come back as an array.

```ts
const [post, count] = await prisma.$transaction([
  prisma.post.create({ data: { title: 'New', authorId: 1 } }),
  prisma.post.count({ where: { authorId: 1 } }),
]);
```

Limit: a query can't use the result of an earlier query in the batch.
For that you need an interactive transaction.

---

### 3. Interactive Transactions

Pass an **async function**. Prisma gives you `tx`, a client bound to a
single connection with an open transaction. Return a value to commit;
throw an error to roll back.

**Credit transfer:**

```ts
async function transfer(fromId: number, toId: number, amount: number) {
  return prisma.$transaction(async (tx) => {
    // 1. Debit the sender (atomic decrement)
    const sender = await tx.user.update({
      where: { id: fromId },
      data: { balance: { decrement: amount } },
    });

    // 2. Check the business rule; throwing rolls everything back
    if (sender.balance < 0) {
      throw new Error(`Insufficient balance for user ${fromId}`);
    }

    // 3. Credit the receiver
    return tx.user.update({
      where: { id: toId },
      data: { balance: { increment: amount } },
    });
  });
}
```

```
BEGIN
UPDATE "users" SET "balance" = "balance" - $1 WHERE "id" = $2 RETURNING …
UPDATE "users" SET "balance" = "balance" + $1 WHERE "id" = $2 RETURNING …
COMMIT            ← or ROLLBACK if anything threw
```

> Inside the callback, always use **`tx`**, never `prisma`. A query on
> `prisma` runs on a different connection, **outside** the transaction.

---

### Options: Isolation Level and Timeouts

```ts
import { Prisma } from './generated/prisma/client.js';

await prisma.$transaction(
  async (tx) => { /* … */ },
  {
    isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
    maxWait: 5000,   // ms to wait for a connection from the pool (default 2000)
    timeout: 10000,  // ms before the transaction is rolled back (default 5000)
  },
);
```

| Isolation level | Prevents | Notes |
|---|---|---|
| `ReadUncommitted` | Nothing | Postgres treats it as ReadCommitted |
| `ReadCommitted` | Dirty reads | **Postgres default** |
| `RepeatableRead` | + Non-repeatable reads | MySQL InnoDB default |
| `Serializable` | + Phantoms / write skew | Safest; may fail with `P2034` and need a retry |

---

### Retrying Serialization Failures

At `Serializable` (and sometimes `RepeatableRead`), the database may abort
a transaction because of a conflict. Prisma raises `P2034`. The fix is to
retry.

```ts
async function withRetry<T>(fn: () => Promise<T>, attempts = 3): Promise<T> {
  for (let i = 1; ; i++) {
    try {
      return await fn();
    } catch (e) {
      const retryable = e instanceof Prisma.PrismaClientKnownRequestError && e.code === 'P2034';
      if (!retryable || i >= attempts) throw e;
    }
  }
}

await withRetry(() =>
  prisma.$transaction(async (tx) => { /* … */ }, { isolationLevel: 'Serializable' }),
);
```

---

### Optimistic Concurrency Control

Stop two users from overwriting each other's edits without holding locks.
Add a `version Int @default(0)` field to the model.

```ts
async function updateTitle(id: number, title: string, expectedVersion: number) {
  const { count } = await prisma.post.updateMany({
    where: { id, version: expectedVersion },          // only if nobody changed it
    data: { title, version: { increment: 1 } },
  });
  if (count === 0) throw new ConflictError('Post was modified by someone else; reload');
}
```

`updateMany` is used because it returns a **count** instead of throwing, so
`count === 0` tells you the version didn't match.

### Pessimistic Locking (`SELECT … FOR UPDATE`)

Prisma has no lock option, so use raw SQL inside the transaction:

```ts
await prisma.$transaction(async (tx) => {
  const [row] = await tx.$queryRaw<{ balance: number }[]>`
    SELECT balance FROM users WHERE id = ${fromId} FOR UPDATE`;   // lock the row
  if (row.balance < amount) throw new Error('Insufficient balance');
  await tx.user.update({ where: { id: fromId }, data: { balance: { decrement: amount } } });
  await tx.user.update({ where: { id: toId },   data: { balance: { increment: amount } } });
});
```

See [Concurrency](../../09-advanced/06_concurrency.md).

---

### Passing the Transaction Through Your Services

Accept an optional client so a function works both inside and outside a
transaction:

```ts
import type { Prisma, PrismaClient } from './generated/prisma/client.js';

type Db = PrismaClient | Prisma.TransactionClient;

export function createPost(db: Db, authorId: number, title: string) {
  return db.post.create({ data: { authorId, title } });
}

// Standalone
await createPost(prisma, 1, 'Hello');

// As part of a bigger transaction: charge 10 credits per post
await prisma.$transaction(async (tx) => {
  await createPost(tx, 1, 'Hello');
  await tx.user.update({ where: { id: 1 }, data: { balance: { decrement: 10 } } });
});
```

---

### Pitfalls

| Pitfall | Why it hurts | Fix |
|---|---|---|
| Using `prisma` instead of `tx` inside the callback | Query runs outside the transaction | Always use `tx` |
| HTTP calls, emails or slow work inside a transaction | Holds a connection and locks; hits `timeout` | Do it after commit (or use an outbox table) |
| Read-then-write (`balance = balance - x` in JS) | Race conditions | Atomic `increment`/`decrement`, or lock the row |
| Very long transactions | Pool exhaustion, lock contention | Keep them short; batch large jobs |
| Not handling `P2034` | Random failures under load | Retry serializable transactions |
| Nesting `$transaction` calls | Not supported as savepoints | Pass `tx` down instead |

---

### Interview-Ready Summary

- Nested writes and bulk operations are **already atomic**.
- **Batch** `$transaction([q1, q2])`: independent queries, all or nothing.
- **Interactive** `$transaction(async (tx) => …)`: dependent steps with logic.
  Throw to roll back, and always use `tx` inside.
- Options: `isolationLevel`, `maxWait`, `timeout`. Retry `P2034` at `Serializable`.
- Concurrency: atomic `increment`, **optimistic** version checks with
  `updateMany`, or **pessimistic** `SELECT … FOR UPDATE` via `$queryRaw`.
- Keep transactions short; never do network I/O inside one.

Compare: [Sequelize Transactions](../sequelize/05_transactions.md)
