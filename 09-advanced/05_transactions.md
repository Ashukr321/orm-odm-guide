## Transactions (Advanced)

The basics (what a transaction is, how each ORM opens one) are in
[Transactions](../03-orm-concepts/08_transactions.md). This page covers what
goes wrong under load: isolation anomalies, retries, deadlocks, and
transactions that span more than one system.

---

### ACID in One Table

| Letter | Guarantee | What provides it |
|---|---|---|
| **A**tomicity | All statements commit, or none do | Write-ahead log, rollback |
| **C**onsistency | Constraints hold before and after | `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK` |
| **I**solation | Concurrent transactions don't see each other's partial work | Isolation level, MVCC, locks |
| **D**urability | A committed change survives a crash | WAL flushed to disk (`fsync`) |

Isolation is the one you tune, and the one that causes most surprises.

---

### Concurrency Anomalies

| Anomaly | What happens |
|---|---|
| Dirty read | T2 reads a row T1 changed but hasn't committed; T1 rolls back |
| Non-repeatable read | T1 reads a row twice and gets different values because T2 committed in between |
| Phantom read | T1 runs the same `WHERE` twice and gets new rows T2 inserted |
| Lost update | T1 and T2 both read `balance = 100`, both write back a new value; one write is lost |
| Write skew | T1 and T2 each check a condition, then write different rows; together they break the rule |

```
Write skew: "at least one doctor must stay on call"
T1: SELECT count(*) FROM doctors WHERE on_call   → 2
T2: SELECT count(*) FROM doctors WHERE on_call   → 2
T1: UPDATE doctors SET on_call = false WHERE id = 1
T2: UPDATE doctors SET on_call = false WHERE id = 2
Both commit → nobody is on call
```

---

### Isolation Levels

| Level | Dirty read | Non-repeatable | Phantom | Lost update / write skew |
|---|---|---|---|---|
| Read Uncommitted | Possible* | Possible | Possible | Possible |
| Read Committed | No | Possible | Possible | Possible |
| Repeatable Read | No | No | Possible** | Lost update blocked in PG; write skew possible |
| Serializable | No | No | No | No |

\* PostgreSQL treats Read Uncommitted as Read Committed.
\*\* PostgreSQL's Repeatable Read is snapshot isolation, so phantoms don't
appear there either.

| Database | Default |
|---|---|
| PostgreSQL | Read Committed |
| MySQL InnoDB | Repeatable Read |
| SQL Server | Read Committed |
| MongoDB transactions | Snapshot (with `readConcern: 'snapshot'`) |

Stricter levels are safer but abort more transactions under contention.

---

### Setting the Isolation Level

```js
// Prisma
import { Prisma } from '@prisma/client';

await prisma.$transaction(
  async (tx) => { /* ... */ },
  { isolationLevel: Prisma.TransactionIsolationLevel.Serializable, maxWait: 5000, timeout: 10000 },
);

// Sequelize
import { Transaction } from 'sequelize';

await sequelize.transaction(
  { isolationLevel: Transaction.ISOLATION_LEVELS.SERIALIZABLE },
  async (t) => { /* pass { transaction: t } to every query */ },
);

// Mongoose / MongoDB
await mongoose.connection.transaction(
  async (session) => { /* pass { session } to every operation */ },
  { readConcern: { level: 'snapshot' }, writeConcern: { w: 'majority' } },
);
```

Use Serializable for the few operations with real invariants (money,
inventory, scheduling), not as a global default.

---

### Retrying Serialization Failures

Under Repeatable Read or Serializable, the database aborts one of two
conflicting transactions. The application must **retry the whole
transaction**.

#### The Errors to Catch

| Database | Error |
|---|---|
| PostgreSQL | `40001 serialization_failure`, `40P01 deadlock_detected` |
| MySQL | `1213 ER_LOCK_DEADLOCK`, `1205 ER_LOCK_WAIT_TIMEOUT` |
| Prisma | `P2034` (write conflict or deadlock) |
| MongoDB | Error label `TransientTransactionError` |

#### A Retry Wrapper

```js
async function withRetry(fn, attempts = 3) {
  for (let i = 1; ; i++) {
    try {
      return await prisma.$transaction(fn, {
        isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
      });
    } catch (e) {
      if (e.code !== 'P2034' || i === attempts) throw e;
      await new Promise((r) => setTimeout(r, 20 * 2 ** i + Math.random() * 20));   // backoff + jitter
    }
  }
}
```

The transaction function must be **safe to re-run**: no emails, payments or
HTTP calls inside it. MongoDB's `session.withTransaction()` already retries
transient errors.

---

### Deadlocks

```
T1: UPDATE accounts SET ... WHERE id = 1;   -- locks row 1
T2: UPDATE accounts SET ... WHERE id = 2;   -- locks row 2
T1: UPDATE accounts SET ... WHERE id = 2;   -- waits for T2
T2: UPDATE accounts SET ... WHERE id = 1;   -- waits for T1 → deadlock, one is aborted
```

- **Lock in a consistent order**: sort IDs before updating several rows.
- Keep transactions **short**, so locks are held briefly.
- Index the columns in `WHERE`, so updates lock only the rows they need
  (MySQL locks every row it scans).
- Treat deadlocks as retryable errors, like serialization failures.

```js
const [first, second] = [fromId, toId].sort((a, b) => a - b);
await tx.$queryRaw`SELECT id FROM accounts WHERE id IN (${first}, ${second}) ORDER BY id FOR UPDATE`;
```

---

### Long Transactions Hurt Everyone

| Problem | Why |
|---|---|
| Lock waits | Other writers queue behind your row and table locks |
| Pool exhaustion | The connection stays checked out for the whole transaction |
| MVCC bloat | PostgreSQL can't vacuum rows that an open transaction might still see |
| Timeouts | Prisma interactive transactions default to a 5 s `timeout`; MongoDB aborts after 60 s |

```js
// Bad: an HTTP call while holding locks
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({ data });
  await stripe.paymentIntents.create({ amount: order.total, currency: 'usd' });   // slow, can't roll back
});

// Better: call out first (with an idempotency key), then write quickly
const intent = await stripe.paymentIntents.create(
  { amount, currency: 'usd' },
  { idempotencyKey: orderKey },
);
await prisma.order.create({ data: { ...data, paymentIntentId: intent.id } });
```

---

### Across Services: Outbox and Sagas

A database transaction can't include Kafka, a payment API or another
service's database. Two-phase commit exists but is rarely worth it.

**Transactional outbox:** write the event to a table in the same
transaction, and a separate worker publishes it.

```js
await prisma.$transaction([
  prisma.order.create({ data: order }),
  prisma.outbox.create({ data: { topic: 'order.created', payload: order } }),
]);
// Worker: SELECT ... FROM outbox WHERE sent_at IS NULL FOR UPDATE SKIP LOCKED → publish → mark sent
```

**Saga:** a chain of local transactions, each with a **compensating
action** (refund, release stock) if a later step fails.

Consumers must be **idempotent**, because the outbox guarantees
*at-least-once* delivery.

---

### Interview-Ready Summary

- Isolation anomalies: **dirty read, non-repeatable read, phantom, lost
  update, write skew**.
- Defaults: **PostgreSQL Read Committed**, **MySQL Repeatable Read**. Only
  **Serializable** prevents write skew.
- Stricter isolation means **aborts**: retry the whole transaction on
  `40001`, deadlocks or Prisma `P2034`, with backoff.
- Prevent deadlocks with **consistent lock ordering**, short transactions and
  good indexes.
- Never make network calls inside a transaction.
- Across services, use the **transactional outbox** and **sagas** with
  idempotent consumers instead of distributed transactions.
