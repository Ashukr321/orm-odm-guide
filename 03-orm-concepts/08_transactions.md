## Transactions

### What a Transaction Is

A **transaction** groups several database operations so they **all
succeed or all fail**. If anything goes wrong halfway, every change is
rolled back.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;        -- or ROLLBACK; nothing is half-done
```

Transactions provide **ACID**: Atomicity, Consistency, Isolation,
Durability. See [Relational Database](../01-database-foundation/relational-database.md#acid-transactions).

**Use one whenever several writes must stay consistent**: an order with its
items, a money transfer, stock reservation with order creation.

---

### Prisma: Two Styles

```js
// 1. Sequential (batch): independent operations, run in order, one transaction
const [order, _] = await prisma.$transaction([
  prisma.order.create({ data: { userId: 1, totalMinor: 49900 } }),
  prisma.product.update({ where: { id: 7 }, data: { stock: { decrement: 1 } } }),
]);

// 2. Interactive: logic between queries, using the tx client
await prisma.$transaction(async (tx) => {
  const from = await tx.account.update({
    where: { id: fromId },
    data: { balance: { decrement: amount } },
  });
  if (from.balance < 0) throw new Error('Insufficient funds');   // throw = ROLLBACK
  await tx.account.update({ where: { id: toId }, data: { balance: { increment: amount } } });
}, {
  isolationLevel: 'Serializable',   // optional
  timeout: 5000,                    // ms (default 5000)
});
```

> Inside an interactive transaction, use **`tx`**, not `prisma`. A query on
> `prisma` runs **outside** the transaction and won't be rolled back.

---

### Sequelize and TypeORM

```js
// Sequelize: managed transaction (auto commit / rollback)
await sequelize.transaction(async (t) => {
  const order = await Order.create({ userId: 1 }, { transaction: t });
  await OrderItem.bulkCreate(items.map((i) => ({ ...i, orderId: order.id })), { transaction: t });
});
// Forgetting { transaction: t } on a query runs it OUTSIDE the transaction
```

```ts
// TypeORM: everything through the transactional entity manager
await dataSource.transaction(async (manager) => {
  const order = await manager.save(Order, { userId: 1 });
  await manager.save(OrderItem, items.map((i) => ({ ...i, order })));
});
```

Every ORM has the same trap: **one query that uses the global client
instead of the transaction's client silently escapes the transaction.**
Sequelize's CLS mode (`Sequelize.useCLS(namespace)`) can pass the
transaction along automatically.

---

### Keep Transactions Short

A transaction holds **locks** and a **pooled connection** until it ends.
Long transactions block other requests and can exhaust the pool.

```js
// ❌ Slow external call inside the transaction
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({ data });
  await stripe.paymentIntents.create({ amount });   // 800 ms network call holding locks
  await tx.order.update({ where: { id: order.id }, data: { status: 'paid' } });
});

// ✅ Keep network calls outside; make each step safe to retry
const order = await prisma.order.create({ data: { ...data, status: 'pending' } });
const intent = await stripe.paymentIntents.create({ amount }, { idempotencyKey: `order-${order.id}` });
await prisma.order.update({ where: { id: order.id }, data: { status: 'paid', paymentId: intent.id } });
```

Rule: **no HTTP calls, emails or queue publishes inside a transaction**.
Use an outbox table if a message must be sent only after a commit.

---

### Concurrency: Optimistic vs Pessimistic Locking

Two requests edit the same row at once. Who wins?

**Optimistic locking**: add a `version` column, and only update if nobody
else changed the row.

```js
const updated = await prisma.product.updateMany({
  where: { id: 7, version: product.version },     // still the version we read?
  data: { stock: newStock, version: { increment: 1 } },
});
if (updated.count === 0) throw new ConflictError('Changed by someone else, please retry');
```

**Pessimistic locking**: lock the row while you work on it (`SELECT … FOR
UPDATE`).

```ts
// TypeORM
await dataSource.transaction(async (m) => {
  const acc = await m.findOne(Account, { where: { id }, lock: { mode: 'pessimistic_write' } });
  acc.balance -= amount;
  await m.save(acc);
});
// Sequelize: Account.findByPk(id, { transaction: t, lock: t.LOCK.UPDATE })
```

| | Optimistic | Pessimistic |
|---|---|---|
| How | `version` check on update | Row lock (`FOR UPDATE`) |
| Best when | Conflicts are rare (profiles, CMS edits) | Conflicts are common (balances, stock, seats) |
| Cost | A retry on conflict | Waiting; risk of deadlocks |

---

### Isolation Levels, Retries and Deadlocks

| Level | Prevents | Notes |
|---|---|---|
| Read Committed | Dirty reads | PostgreSQL default |
| Repeatable Read | + Non-repeatable reads | MySQL InnoDB default |
| Serializable | + Phantoms and write skew | Safest; may **abort** transactions, which you retry |

```js
// Retry on serialization failures / deadlocks (Prisma error code P2034)
async function withRetry(fn, attempts = 3) {
  for (let i = 1; ; i++) {
    try { return await fn(); }
    catch (e) {
      if (e.code !== 'P2034' || i === attempts) throw e;
      await new Promise((r) => setTimeout(r, 50 * 2 ** i));   // exponential backoff
    }
  }
}
await withRetry(() => prisma.$transaction(transfer, { isolationLevel: 'Serializable' }));
```

Avoid deadlocks by **locking rows in a consistent order** (for example,
always the lower account ID first).

---

### Nested Transactions and Savepoints

Some ORMs turn a "transaction inside a transaction" into a **savepoint**:
rolling back the inner part doesn't undo the outer part.

```js
// Sequelize: pass the parent transaction to create a SAVEPOINT
await sequelize.transaction(async (t) => {
  await Order.create(order, { transaction: t });
  try {
    await sequelize.transaction({ transaction: t }, async (inner) => {
      await Coupon.update({ used: true }, { where: { code }, transaction: inner });
    });
  } catch {
    // only the coupon update was rolled back (ROLLBACK TO SAVEPOINT); the order stays
  }
});
```

Support varies by ORM version and database, so check your ORM's docs
before relying on nested transactions.

Prisma has no nested interactive transactions: pass the same `tx` down to
your helper functions instead.

More: [Advanced Transactions](../09-advanced/05_transactions.md) ·
[Concurrency](../09-advanced/06_concurrency.md)

---

### Interview-Ready Summary

- A transaction makes several writes **all-or-nothing** (ACID).
- Prisma has **sequential** (`$transaction([...])`) and **interactive**
  (`$transaction(async tx => …)`) styles. Always use **`tx`** inside.
- In every ORM, a query on the **global client escapes the transaction**.
  Pass the transaction along.
- Keep transactions **short**: no network calls inside. Use idempotency
  keys and an outbox.
- **Optimistic** (version column) for rare conflicts; **pessimistic**
  (`FOR UPDATE`) for hot rows.
- `Serializable` is the safest level but needs **retries**. Lock rows in a
  **consistent order** to avoid deadlocks.
