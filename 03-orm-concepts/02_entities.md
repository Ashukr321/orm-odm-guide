## Entities

### What Is an Entity

An **entity** is an object with a **unique identity** that stays the same
over time, even when its data changes. A user who changes their name and
email is still the same user, because their `id` hasn't changed.

In ORM terms, an entity is usually **one class mapped to one table**, where
each instance is **one row**, identified by its primary key.

> **Model vs entity:** the words are often used interchangeably. "Model"
> usually means the schema definition (Prisma, Sequelize). "Entity" usually
> means a class whose instances are tracked objects (TypeORM, MikroORM,
> Hibernate, EF Core).

---

### Entity vs Value Object vs DTO

| | Entity | Value object | DTO |
|---|---|---|---|
| Identity | Has an ID | No ID; defined by its values | No ID of its own |
| Equality | Same ID = same entity | Same values = equal | Not compared |
| Mutable? | Yes, over time | Usually immutable (replace it) | Plain data |
| Example | `User`, `Order` | `Address`, `Money(500, 'INR')` | `UserResponse { id, name }` |
| Stored as | Its own table | Columns inside the owner's table | Not stored; sent over the API |

```ts
const a = new Money(500, 'INR');
const b = new Money(500, 'INR');
a.equals(b);        // true: value objects compare by value

userA.id === userB.id;   // entities compare by identity
```

---

### Defining an Entity (TypeORM)

```ts
@Entity('orders')
export class Order {
  @PrimaryGeneratedColumn('uuid') id: string;

  @ManyToOne(() => User, (user) => user.orders, { nullable: false })
  @JoinColumn({ name: 'user_id' })
  user: User;

  @Column({ type: 'varchar', default: 'pending' }) status: string;

  @Column(() => Address) shippingAddress: Address;   // embedded value object

  @OneToMany(() => OrderItem, (item) => item.order, { cascade: true })
  items: OrderItem[];

  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
  @VersionColumn() version: number;                  // optimistic locking
}

export class Address {                               // no @Entity: no table of its own
  @Column() street: string;
  @Column() city: string;
  @Column() pin: string;
}
```

The embedded `Address` becomes columns in `orders`: `shippingAddressStreet`,
`shippingAddressCity`, `shippingAddressPin` (or `shipping_address_street`, …
with a snake-case naming strategy).

---

### Entity Lifecycle States

ORMs with a Unit of Work (TypeORM, MikroORM, Hibernate, EF Core) track what
state each entity is in:

| State | Meaning | How it gets there |
|---|---|---|
| **New / transient** | Exists only in memory; no row yet | `new User()` / `repo.create()` |
| **Managed** | Tracked by the ORM; changes will be saved | Loaded from the DB, or `persist()` |
| **Detached** | Has a row, but the ORM stopped tracking it | The session / entity manager closed |
| **Removed** | Scheduled for deletion | `em.remove(user)`; deleted on flush |

```ts
// MikroORM
const user = em.create(User, { email: 'ashu@example.com' });  // new
await em.persist(user).flush();                               // managed, INSERT
user.name = 'Ashutosh';                                       // change tracked
await em.flush();                                             // UPDATE only the changed column
em.remove(user);                                              // removed
await em.flush();                                             // DELETE
```

---

### Entity Identity and Equality

- The **primary key is the identity**. Never change it, and never reuse it.
- Prefer **surrogate keys** (`id` as an auto-increment or UUID) over
  natural keys (email, phone), because natural keys change.
- Inside one session, an **identity map** returns the same object for the
  same row, so `===` works:

```ts
const a = await em.findOne(User, 1);
const b = await em.findOne(User, 1);   // no second query
a === b;                               // true, inside the same entity manager
```

- Across sessions or requests, compare entities **by ID**, not by object
  reference.

---

### Rich vs Anemic Entities

**Anemic entity:** only fields. All the rules live in services.

**Rich entity:** fields *plus* the behaviour that protects its own rules.

```ts
export class Order {
  status: 'pending' | 'paid' | 'cancelled' = 'pending';

  cancel() {
    if (this.status === 'paid') throw new Error('Paid orders must be refunded, not cancelled');
    this.status = 'cancelled';
  }
}
```

Rich entities keep business rules in one place, next to the data. They
work best with Data Mapper ORMs (MikroORM, TypeORM repositories), where
entities are plain classes.

---

### Entities in Prisma

Prisma has **no entity classes**. Queries return **plain, typed objects**,
with no change tracking, no `save()` and no identity map.

```ts
import type { User } from '@prisma/client';

const user: User = await prisma.user.findUniqueOrThrow({ where: { id: 1 } });
user.name = 'Ashutosh';                // does nothing to the database
await prisma.user.update({ where: { id: 1 }, data: { name: 'Ashutosh' } });   // explicit write
```

If you want rich domain entities with Prisma, map the plain objects into
your own classes inside a repository. See [Repositories](10_repositories.md).

---

### Interview-Ready Summary

- An **entity** has an **identity** (its primary key) that stays the same
  while its data changes.
- **Entity vs value object vs DTO**: entities compare by ID; value objects
  compare by value and are embedded; DTOs are what you send over the API.
- Unit of Work ORMs track the **lifecycle**: new → managed → detached /
  removed, and flush only the changes.
- Use **surrogate keys**, and never change or reuse an identity.
- **Prisma** returns plain objects with no tracking. TypeORM, MikroORM and
  Hibernate give you tracked entity classes.
