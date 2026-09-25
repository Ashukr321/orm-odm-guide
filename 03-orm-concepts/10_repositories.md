## Repositories

### What Is the Repository Pattern

A **repository** is an object that acts like an in-memory **collection of
entities**: `findById`, `findByEmail`, `save`, `delete`. It hides *how*
the data is stored behind methods named after **what the business needs**.

```
Route / controller  →  Service (business rules)  →  Repository  →  ORM  →  Database
                                                    "usersRepo.findActiveByEmail()"
```

The pattern comes from Martin Fowler's *Patterns of Enterprise Application
Architecture*: *"mediates between the domain and data mapping layers using
a collection-like interface."*

---

### Why Use Repositories

| Benefit | What it means in practice |
|---|---|
| **One place for queries** | "Active users" is defined once, not copied into 12 routes |
| **Business-named methods** | `findOverdueInvoices()` reads better than a 15-line `findMany` |
| **Testability** | Services can be tested with an in-memory fake, with no database |
| **Swap or upgrade the ORM** | ORM calls live in one layer, not all over the codebase |
| **Security defaults** | Always exclude soft-deleted rows; never select the password hash |

---

### A Repository With Prisma

Prisma has no repository class, so you write a small one around the
client:

```ts
// users.repository.ts
import { Prisma, PrismaClient } from '@prisma/client';

const publicUser = { id: true, email: true, name: true, createdAt: true } satisfies Prisma.UserSelect;

export class UsersRepository {
  constructor(private db: PrismaClient | Prisma.TransactionClient) {}

  findById(id: number) {
    return this.db.user.findUnique({ where: { id }, select: publicUser });
  }

  findByEmail(email: string) {
    return this.db.user.findUnique({ where: { email: email.toLowerCase() } });
  }
  // …continued below
}
```

Rules like "hide soft-deleted users" and "never select the password hash"
live in the repository, so no route can forget them:

```ts
export class UsersRepository {
  // …
  listActive({ take = 20, cursor }: { take?: number; cursor?: number }) {
    return this.db.user.findMany({
      where: { deletedAt: null },
      select: publicUser,
      orderBy: { id: 'desc' },
      take,
      ...(cursor && { cursor: { id: cursor }, skip: 1 }),
    });
  }

  create(data: { email: string; name: string; passwordHash: string }) {
    return this.db.user.create({ data: { ...data, email: data.email.toLowerCase() }, select: publicUser });
  }
}
```

Accepting `PrismaClient | Prisma.TransactionClient` lets the **same
repository run inside a transaction**.

---

### TypeORM: Built-In and Custom Repositories

```ts
// Built-in: a repository per entity, with generic methods
const userRepo = dataSource.getRepository(User);
await userRepo.findOneBy({ email });
await userRepo.save(user);

// Custom methods (TypeORM 0.3+)
export const UserRepository = dataSource.getRepository(User).extend({
  findActiveByEmail(email: string) {
    return this.findOne({ where: { email, isActive: true } });
  },
  countSignupsSince(date: Date) {
    return this.createQueryBuilder('u').where('u.createdAt >= :date', { date }).getCount();
  },
});
```

MikroORM works the same way: `em.getRepository(User)`, or a custom class
that extends `EntityRepository<User>`.

---

### Repositories + Services + Transactions

Services hold the **business rules**. Repositories hold the **queries**.
Transactions wrap several repositories:

```ts
// orders.service.ts
export async function placeOrder(userId: number, items: CartItem[]) {
  return prisma.$transaction(async (tx) => {
    const orders = new OrdersRepository(tx);        // same tx for both repositories
    const products = new ProductsRepository(tx);

    for (const item of items) {
      const ok = await products.reserveStock(item.productId, item.qty);
      if (!ok) throw new OutOfStockError(item.productId);   // rolls back everything
    }
    return orders.create(userId, items);
  });
}
```

Repositories never start transactions themselves. The **service decides
the transaction boundary**.

---

### Testing With an In-Memory Fake

Depend on an **interface**, not on the ORM:

```ts
export interface UsersRepo {
  findByEmail(email: string): Promise<User | null>;
  create(data: NewUser): Promise<User>;
}

// Production: new UsersRepository(prisma)   Tests: the fake below
export class InMemoryUsersRepo implements UsersRepo {
  private rows: User[] = [];
  async findByEmail(email: string) { return this.rows.find((u) => u.email === email) ?? null; }
  async create(data: NewUser) { const u = { id: this.rows.length + 1, ...data } as User; this.rows.push(u); return u; }
}

// signup.test.ts
const repo = new InMemoryUsersRepo();
await signUp(repo, { email: 'a@x.com', password: 'secret123' });
await expect(signUp(repo, { email: 'a@x.com', password: 'x' })).rejects.toThrow('Email taken');
```

> Fakes test **business logic** quickly. Also keep a few **integration
> tests against a real database**, because a fake can't check constraints,
> SQL or transactions.

---

### Anti-Patterns

| Anti-pattern | Why it hurts | Better |
|---|---|---|
| **Generic repository** (`Repository<T>` with `find(filter: any)`) | Leaks ORM query syntax upward; adds nothing | Specific, business-named methods |
| **One method per call site** (`findUsersForPageX`) | Hundreds of near-duplicates | Methods with a few options |
| **Business rules in repositories** | Data access and rules get tangled | Rules in services or entities |
| **Repositories starting transactions** | Can't combine them into one unit | The service owns the transaction |
| **Returning ORM query objects** | Callers depend on the ORM again | Return plain data or entities |

---

### When You Don't Need Repositories

- **Small apps and scripts**: `prisma.user.findMany()` in a route is fine.
- **Prisma is already a typed data-access layer.** Wrapping every call
  one-to-one adds files and no value.
- Add a repository when a query is **reused**, **complex**, or has
  **rules** (soft delete, tenant filter, safe `select`), or when you need
  **fast tests** of business logic.

---

### Interview-Ready Summary

- A repository is a **collection-like interface** for entities, with
  methods named after business needs (Fowler, PoEAA).
- Benefits: **queries in one place**, readable services, **testability
  with fakes**, and security defaults.
- With **Prisma**, write small classes around the client and accept a
  `TransactionClient`. **TypeORM / MikroORM** have built-in repositories you
  can extend.
- **Services own business rules and transaction boundaries.**
  Repositories only access data.
- Avoid **generic repositories** and one-method-per-call-site. In small
  apps, you may not need repositories at all.
