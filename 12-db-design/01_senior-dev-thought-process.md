## How a Senior Developer Designs a Database

A junior developer reads the requirement and starts writing tables.
A senior developer first asks questions, then finds the rules that must never break, then works out how the data will be read. Only after that do they write tables.

> **Core idea:** Data lasts longer than code. You can rewrite an API in a week, but a bad schema with 50M rows stays with you for years. So spend time on the design at the start.

---

### The Mental Model

A senior developer runs every requirement through these steps:

| # | Step | Question they ask |
|---|---|---|
| 0 | Clarify | "What exactly does the business need, and what is it *not* asking for?" |
| 1 | Nouns & verbs | "What are the things, and what happens to them?" |
| 2 | Invariants | "What must **never** be true, even with bugs or two users clicking at once?" |
| 3 | Relationships | "One-to-one, one-to-many, or many-to-many? Can it change over time?" |
| 4 | Access patterns | "What are the top 5 queries, and how often does each run?" |
| 5 | Pick the store | "Relational, document, or both, and why?" |
| 6 | Normalize, then denormalize | "Where does each fact live? Is there a *measured* reason to copy it?" |
| 7 | Keys & types | "What identifies a row? Which exact type fits money, time and status?" |
| 8 | Constraints | "Which rules can the database enforce so the app can't break them?" |
| 9 | Indexes | "Which index serves each of the queries from step 4?" |
| 10 | History & deletes | "Do we need to know what it *was*? What happens when something is removed?" |
| 11 | Growth & change | "How big is this in 1–3 years? How do we change it without downtime?" |

The rest of this page goes through each step, then applies all of them to one real requirement.

---

### Step 0 — Clarify Before You Design

Requirements are always incomplete. A senior developer turns vague sentences into concrete questions:

- **Who** uses it? (customers, admins, internal staff, other services)
- **Scale**: how many users and records? Is traffic mostly reads or mostly writes? Are there peak hours?
- **Money involved?** If yes, you need exact decimals, an audit trail and idempotency.
- **Multi-tenant?** Does each company or clinic need its data kept separate?
- **Time zones / regions?** Is it one country or global?
- **Lifecycle**: can records be edited, cancelled or deleted? Must we keep history for legal reasons?
- **Out of scope**: what are we deliberately *not* building in v1?

The last question matters most. Every "maybe later" you design for today is complexity you pay for now.

---

### Step 1 — Nouns Become Entities, Verbs Become Relationships or Events

Underline the nouns and verbs in the requirement:

> "A **patient** *books* an **appointment** with a **doctor** at a **clinic**."

- Nouns → candidate tables: `patients`, `appointments`, `doctors`, `clinics`
- Verbs → relationships or events: *books* creates an `appointment` row, which links a patient to a doctor's time.

Then check each noun:
- Is it really its own thing, or just an attribute? (`specialty` could be a column, or its own table if it has its own data)
- Is it the same concept as another noun? (a doctor is also a *user* who logs in)

---

### Step 2 — Find the Invariants (the Rules That Must Never Break)

This is what separates senior from junior. An **invariant** is a rule that must always hold, even when there are bugs, retries, or two requests at the same millisecond.

Examples:
- A slot can't be booked twice.
- An account balance can't go negative.
- An order total equals the sum of its line items at the time of purchase.
- An email belongs to only one user.

**Senior rule:** if the database *can* enforce the invariant, the database *should* enforce it (`UNIQUE`, `FOREIGN KEY`, `CHECK`, `NOT NULL`, exclusion constraints). App-level checks like "SELECT first, then INSERT" **fail under concurrency**, because two requests can both pass the SELECT.

---

### Step 3 — Relationships and Cardinality

For each pair of entities, answer three questions:

1. **How many on each side?** 1:1, 1:N, M:N
2. **Is it required?** Can an appointment exist without a patient? (No, so `NOT NULL`)
3. **Does the relationship have its own data?** If yes, you need a join table *with columns*.
   Example: a doctor works at many clinics, and the **fee is different per clinic**. The fee belongs to the relationship, not to the doctor.

Also ask: **"Can this change over time, and do we care about the old value?"** If a doctor's fee changes, what should old appointments show? (They should show the fee that was charged. See Step 6.)

---

### Step 4 — Design for Access Patterns, Not Just Structure

Write down the real queries *before* writing tables:

| Query | Frequency | Latency need |
|---|---|---|
| Show free slots for doctor X on date D | Very high | < 100 ms |
| Book a slot | High (peaks at 9 am) | Must be correct under concurrency |
| Patient's upcoming appointments | High | < 100 ms |
| Doctor's schedule for today | Medium | < 200 ms |
| Monthly revenue per clinic | Low (admin) | Seconds are fine |

This table decides your indexes, what you denormalize, and whether analytics need a separate store.

---

### Step 5 — Choose the Store (Per Domain, Not Per Company)

| Pick relational (Postgres/MySQL) when… | Pick document (MongoDB) when… |
|---|---|
| Data is highly related and you join it | Data is read as one self-contained unit |
| Strong invariants across rows (money, bookings, inventory) | The shape changes often or differs per record |
| You need multi-row transactions a lot | Most writes touch a single document |
| Reporting / ad-hoc queries matter | You rarely need to query across documents |

Default to **Postgres** unless you have a concrete reason not to. It handles relational data, JSON columns, full-text search and strong constraints in one place. See [SQL vs NoSQL](../08-comparisons/02_sql-vs-nosql.md).

---

### Step 6 — Normalize First, Denormalize With a Reason

- **Normalize first**: every fact lives in exactly one place. An update then changes one row, not many.
- **Denormalize only with a written reason**, one of these:
  - **Snapshot / historical truth**: an order stores `unit_price` at purchase time, because the product price will change later. *This is not duplication. It is a different fact: "the price we charged".*
  - **Measured hot read path**: a counter like `post.like_count` avoids counting 1M rows on every page view.
  - **Cross-service boundary**: another service owns the source data.

If you can't name the reason, don't denormalize.

---

### Step 7 — Keys and Data Types

**Primary keys**
- Use a **surrogate key** (`BIGINT IDENTITY` or `UUID`) as the PK. Natural keys (email, phone, PAN) change, and PKs should never change.
- Put a `UNIQUE` constraint on the natural key to enforce the business rule.
- `BIGINT` is small and fast, but IDs are guessable. `UUID` (v7 is time-ordered) is safe to expose in public URLs and can be generated anywhere.

**Types that seniors get right**

| Data | ❌ Junior | ✅ Senior |
|---|---|---|
| Money | `FLOAT` | `BIGINT` minor units (paise/cents) + `currency CHAR(3)`, or `NUMERIC(12,2)` |
| Timestamps | `TIMESTAMP` / string | `TIMESTAMPTZ`, stored in UTC; the display time zone lives elsewhere |
| Status | free text | `TEXT` + `CHECK (status IN (...))`, or an enum |
| Boolean flags piling up | `is_paid, is_shipped, is_cancelled` | One `status` column (a state machine) |
| Phone / ZIP | `INT` | `TEXT` (leading zeros, `+91`) |

---

### Step 8 — Push Rules Into Constraints

```sql
NOT NULL          -- this field is required
UNIQUE            -- no duplicates (email, one booking per slot)
FOREIGN KEY       -- no orphans; choose ON DELETE RESTRICT / CASCADE / SET NULL on purpose
CHECK             -- value rules (amount >= 0, ends_at > starts_at)
PARTIAL UNIQUE    -- unique only for "active" rows (Postgres)
EXCLUDE           -- no overlapping time ranges (Postgres)
```

The app validates to give users **friendly errors**. The database validates to guarantee **correctness**. You need both.

---

### Step 9 — Indexes Come From Queries

Go back to the access-pattern table and give each query an index:
- Put the columns used in `WHERE` with `=` first, then range or `ORDER BY` columns: `(doctor_id, starts_at)`.
- Every FK column you filter or join on usually needs an index. Postgres does **not** create these automatically.
- Don't index everything. Each index slows down writes and uses memory.

See [Indexing](../09-advanced/03_indexing.md).

---

### Step 10 — History, Audit, and Deletes

Ask: **"If someone asks *what happened* six months from now, can we answer?"**

- **Audit / status history**: an append-only events table (`appointment_events`) records who changed what, and when.
- **Soft delete vs hard delete**:
  - Something other rows reference (doctor, product)? **Deactivate it** (`is_active = false`) and keep FKs `RESTRICT`.
  - Personal data under GDPR/DPDP? You may *have* to hard-delete or anonymize it.
  - Avoid a blanket `deleted_at` on every table. Every query then needs `WHERE deleted_at IS NULL`, and someday someone will forget it.

---

### Step 11 — Think About Growth and Change

**Back-of-envelope math** (always do it; it takes two minutes):

> 1,000 doctors × 32 slots/day × 365 days ≈ **11.7M slot rows/year**.
> Postgres handles that easily with the right indexes. Partitioning is *not* needed yet. Revisit at around 100M rows.

**Changing the schema later**
- Every change goes through migrations. See [Migrations](../03-orm-concepts/07_migrations.md).
- To change a live column without downtime: **add** the new column → **backfill** → **switch reads** → **drop** the old one (expand/contract).
- Don't add a JSON "extra" column just in case. You'll end up with unvalidated data in it.

---

## Worked Example: Doctor Appointment Booking

### The requirement (as the business wrote it)

> "Patients should be able to find doctors by specialty and city, see available time slots, and book an appointment. Doctors work at one or more clinics. Patients can cancel. We charge a consultation fee."

### Step 0 — Questions the senior asks (and the answers)

| Question | Answer | Design impact |
|---|---|---|
| Can one account book for family members? | Yes | Separate `users` (login) from `patients` (person being treated) |
| Is the fee the same at every clinic? | No, it varies per clinic | Fee lives on `doctor_clinics`, not on `doctors` |
| Fixed slot lengths or free-form? | Fixed 15-min slots, set up by clinic staff | Pre-generated `slots` table |
| What if the fee changes after booking? | Patient pays the fee shown when they booked | Snapshot the fee on `appointments` |
| Can an appointment be rebooked after cancel? | Yes, the slot becomes free again | Uniqueness only for *active* bookings |
| Multiple countries? | India only for v1, but clinics are in different cities | `TIMESTAMPTZ` + clinic `timezone` column (cheap to add now) |
| Payments in v1? | Pay at clinic | No payments table yet (**out of scope**) |

### Step 1 — Entities

`users`, `patients`, `doctors`, `clinics`, `doctor_clinics` (M:N with its own data), `slots`, `appointments`, `appointment_events`.

### Step 2 — Invariants

1. **A slot has at most one active appointment.** ← the most important rule
2. A slot's `ends_at` is after its `starts_at`.
3. A slot can only exist for a doctor at a clinic where that doctor actually works.
4. Fees are never negative.
5. A doctor can't have two slots starting at the same time.

### Step 3 — Relationships

```
users 1 ── N patients          (an account manages several patients)
users 1 ── 0..1 doctors        (some users are doctors)
doctors N ── M clinics         (via doctor_clinics, which has the fee)
doctor_clinics 1 ── N slots
slots 1 ── N appointments      (at most 1 active; the rest are cancelled history)
patients 1 ── N appointments
appointments 1 ── N appointment_events
```

### Step 4 — Two ways to prevent double booking (a senior compares options)

| Option | How | Pick when |
|---|---|---|
| **A. Pre-generated slots** | `slots` table + partial unique index on `appointments(slot_id)` for active rows | Fixed slot lengths (**our case**); works in any SQL DB with small changes |
| **B. Time ranges** | `appointments(doctor_id, tstzrange)` + Postgres `EXCLUDE USING gist` | Variable durations, no fixed grid |

Choose **A**. It matches the business ("fixed 15-min slots"), and "show free slots" becomes a simple query.

### The schema (PostgreSQL)

```sql
CREATE TABLE users (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email       TEXT NOT NULL UNIQUE,
  phone       TEXT UNIQUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- The person being treated; one account can manage many (self, child, parent)
CREATE TABLE patients (
  id               BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  account_user_id  BIGINT NOT NULL REFERENCES users(id),
  full_name        TEXT NOT NULL,
  date_of_birth    DATE,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX patients_account_idx ON patients(account_user_id);

CREATE TABLE clinics (
  id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name      TEXT NOT NULL,
  city      TEXT NOT NULL,
  timezone  TEXT NOT NULL DEFAULT 'Asia/Kolkata'
);
CREATE INDEX clinics_city_idx ON clinics(city);

CREATE TABLE doctors (
  id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id    BIGINT NOT NULL UNIQUE REFERENCES users(id),
  full_name  TEXT NOT NULL,
  specialty  TEXT NOT NULL,
  is_active  BOOLEAN NOT NULL DEFAULT true   -- deactivate, never delete: appointments reference doctors
);
CREATE INDEX doctors_specialty_idx ON doctors(specialty) WHERE is_active;

-- M:N with its own data (fee differs per clinic)
CREATE TABLE doctor_clinics (
  doctor_id  BIGINT NOT NULL REFERENCES doctors(id),
  clinic_id  BIGINT NOT NULL REFERENCES clinics(id),
  fee_minor  BIGINT NOT NULL CHECK (fee_minor >= 0),   -- paise, never FLOAT
  currency   CHAR(3) NOT NULL DEFAULT 'INR',
  PRIMARY KEY (doctor_id, clinic_id)
);
CREATE INDEX doctor_clinics_clinic_idx ON doctor_clinics(clinic_id);

CREATE TABLE slots (
  id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  doctor_id  BIGINT NOT NULL,
  clinic_id  BIGINT NOT NULL,
  starts_at  TIMESTAMPTZ NOT NULL,
  ends_at    TIMESTAMPTZ NOT NULL,
  CHECK (ends_at > starts_at),                                          -- invariant 2
  FOREIGN KEY (doctor_id, clinic_id)
    REFERENCES doctor_clinics(doctor_id, clinic_id),                    -- invariant 3
  UNIQUE (doctor_id, starts_at)                                         -- invariant 5 + serves "free slots" query
);
-- Trade-off: UNIQUE(doctor_id, starts_at) only blocks identical start times.
-- That's enough because staff generate slots on a fixed 15-min grid. If durations
-- become variable, switch to an EXCLUDE constraint on tstzrange (option B).

CREATE TABLE appointments (
  id                 BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  slot_id            BIGINT NOT NULL REFERENCES slots(id),
  patient_id         BIGINT NOT NULL REFERENCES patients(id),
  booked_by_user_id  BIGINT NOT NULL REFERENCES users(id),
  status             TEXT NOT NULL DEFAULT 'booked'
                     CHECK (status IN ('booked', 'cancelled', 'completed', 'no_show')),
  fee_minor          BIGINT NOT NULL CHECK (fee_minor >= 0),   -- SNAPSHOT of fee at booking time
  currency           CHAR(3) NOT NULL,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Invariant 1: at most ONE non-cancelled appointment per slot.
-- Cancelled rows stay as history, and the slot can be booked again.
CREATE UNIQUE INDEX appointments_one_active_per_slot
  ON appointments(slot_id) WHERE status <> 'cancelled';

CREATE INDEX appointments_patient_idx ON appointments(patient_id);

-- Append-only audit trail: who changed what, when, and why
CREATE TABLE appointment_events (
  id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  appointment_id      BIGINT NOT NULL REFERENCES appointments(id),
  from_status         TEXT,
  to_status           TEXT NOT NULL,
  changed_by_user_id  BIGINT NOT NULL REFERENCES users(id),
  reason              TEXT,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX appointment_events_appt_idx ON appointment_events(appointment_id);
```

> **MySQL note:** MySQL has no partial indexes. A common workaround is a generated column `active_slot_id = IF(status <> 'cancelled', slot_id, NULL)` with a `UNIQUE` index on it. `NULL`s don't collide, so cancelled rows are ignored.

### Step 9 — Match each query to its index

| Query | Served by |
|---|---|
| Doctors by specialty in a city | `doctors_specialty_idx` + `clinics_city_idx` via `doctor_clinics` |
| Free slots for doctor X on date D | `UNIQUE (doctor_id, starts_at)` range scan + anti-join on `appointments_one_active_per_slot` |
| Book a slot | `appointments_one_active_per_slot` (enforces + looks up) |
| Patient's appointments | `appointments_patient_idx` |
| Monthly revenue per clinic | Rare admin query; a full scan is fine for now. Add a read replica or warehouse if it gets slow |

```sql
-- Free slots for a doctor on a given day
SELECT s.id, s.starts_at, s.ends_at
FROM slots s
WHERE s.doctor_id = $1
  AND s.starts_at >= $2 AND s.starts_at < $2 + interval '1 day'
  AND NOT EXISTS (
    SELECT 1 FROM appointments a
    WHERE a.slot_id = s.id AND a.status <> 'cancelled'
  )
ORDER BY s.starts_at;
```

### The booking flow: let the database win the race

The junior approach has a race condition:

```js
// ❌ Two users can both pass this check at the same millisecond
const taken = await prisma.appointment.findFirst({ where: { slotId, status: { not: 'cancelled' } } });
if (!taken) await prisma.appointment.create({ data: { ... } });
```

The senior approach just inserts, and lets the unique index reject the second request:

```js
// ✅ The partial unique index guarantees only one insert succeeds
try {
  await prisma.$transaction(async (tx) => {
    const appt = await tx.appointment.create({
      data: { slotId, patientId, bookedByUserId, feeMinor, currency },
    });
    await tx.appointmentEvent.create({
      data: { appointmentId: appt.id, toStatus: 'booked', changedByUserId: bookedByUserId },
    });
  });
} catch (e) {
  if (e.code === 'P2002') throw new ConflictError('Slot already booked'); // unique violation
  throw e;
}
```

> Prisma's schema language doesn't cover every partial index. Add `appointments_one_active_per_slot` in a raw SQL migration (`prisma migrate dev --create-only`, then edit the SQL).

### What we deliberately did NOT build (and when we would)

| Skipped | Add it when |
|---|---|
| `payments` table | Online payment goes into scope |
| Partitioning `slots` | Around 100M+ rows, or old-slot queries become slow |
| Soft delete on every table | Never as a blanket rule. Decide table by table |
| Separate `specialties` table | Specialties need their own data (description, icon, parent category) |
| Caching free slots | Measured p95 latency goes over budget |

### If this were MongoDB instead

The same thinking applies; only the embed-vs-reference decision is new:
- **Embed** data that is always read together and only belongs to its parent, e.g. the status history *inside* the appointment document.
- **Reference** data that is shared or grows without limit: doctor, patient, and slot are separate collections.
- Enforce invariant 1 with a **partial unique index**:
  `db.appointments.createIndex({ slotId: 1 }, { unique: true, partialFilterExpression: { status: { $in: ['booked', 'completed', 'no_show'] } } })`
  (Partial filters don't support `$ne`, so list the active statuses with `$in`, which needs MongoDB 6.0+.)

---

## Junior vs Senior: Common Mistakes

| Junior | Senior |
|---|---|
| Starts with tables | Starts with questions, invariants, and queries |
| Validates only in app code | Constraints in the DB; app validation only for friendly error messages |
| `SELECT` then `INSERT` to check uniqueness | `INSERT` and handle the unique violation |
| `FLOAT` for money | Integer minor units or `NUMERIC` |
| `price` looked up from the product for old orders | Snapshot the price onto the order |
| Five boolean flags for state | One `status` column with a `CHECK` |
| `ON DELETE CASCADE` everywhere | Picks the delete behaviour per relationship; deactivates referenced rows |
| Indexes every column, or none | One index per real access pattern |
| Designs for 1B users on day one | Does the math, designs for 10×, writes down when to revisit |
| JSON blob "for flexibility" | Real columns; JSON only for truly schemaless, rarely-queried data |

---

## Reusable Checklist

Copy this for any new business requirement:

- [ ] Wrote down open questions and got answers; listed what's out of scope
- [ ] Listed entities (nouns) and events (verbs)
- [ ] Listed invariants, each mapped to a DB constraint (or a written reason why it can't be one)
- [ ] Drew relationships with cardinality; join tables carry their own attributes
- [ ] Listed the top queries with frequency and latency needs
- [ ] Chose the store per domain, with a reason
- [ ] Every denormalized field has a written reason (snapshot / hot path / service boundary)
- [ ] Correct types: money, `TIMESTAMPTZ`, status with `CHECK`, phone as text
- [ ] Every FK has a deliberate `ON DELETE` behaviour and an index if queried
- [ ] Concurrency: the critical write path is protected by a constraint or a lock, not by a `SELECT`
- [ ] History / audit needs are covered
- [ ] Did the back-of-envelope size estimate; noted when to revisit
- [ ] Schema changes go through migrations

---

## How to Say This in an Interview

When asked to "design the DB for X", talk through the same steps out loud:

1. **"Let me ask a few clarifying questions first…"** (scale, money, multi-tenant, history, out of scope)
2. **"The core entities are… and the key invariant is…"** (name the rule that must never break)
3. **"The hot queries are…, so I'll index…"**
4. **"I'll enforce [invariant] with [constraint] so concurrent requests can't break it."**
5. **"I'm denormalizing [X] because [snapshot / hot path]."**
6. **"At [N] rows/year this fits one Postgres instance; I'd revisit partitioning at [M]."**
7. **"For v1 I'm skipping [Y]; I'd add it when [trigger]."**

Interviewers aren't looking for a perfect schema. They want to see you reason about **correctness, access patterns, and trade-offs**. More practice: [System Design Questions](../11-interview/04_system-design-questions.md).
