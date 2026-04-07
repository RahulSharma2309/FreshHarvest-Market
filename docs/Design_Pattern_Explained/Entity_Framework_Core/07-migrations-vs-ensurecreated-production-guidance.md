# Part 7 — Two strategies for the database existing: “snapshot” vs “version history”

## Start with a picture

You have a **C# model** (classes + configuration). The real server has **tables**.

There are two common ways people try to line those up:

| Strategy | Plain English |
|----------|----------------|
| **`EnsureCreated`** | “If there is **no** database (or we’re in toy mode), **build from today’s model**.” |
| **Migrations** | “Keep a **numbered list of changes**; apply them in order on each environment.” |

They are **not** interchangeable for growing apps.

---

## `EnsureCreated` — when it shines, when it hurts

**Feels like:** a **big red button** labeled **“make it look like my code **right now**.”**

**Good for:**

- Personal spikes, demos, throwaway local DBs.
- Some tests where you want **zero** migration files.

**Pain points:**

- Database **already there** with old tables? `EnsureCreated` does **not** mean “alter tables until they match my new properties.” It is closer to **“if I had to create from scratch…”** — and if scratch already happened, it may **do nothing** while your code **moved on**.
- Teams cannot share a **clear upgrade path** in Git the way migrations can.
- Production startup is a **risky** place to do schema work unless you **really** know what you are doing (permissions, multiple instances starting at once, partial failures).

---

## Migrations — the story with chapters

**Feels like:** **Git commits**, but for **schema**.

- Each migration is a **chapter**: “add column,” “new table,” “rename index,” …
- New developers / CI / staging / prod run the **same ordered chapters**.
- You can **review** SQL before it touches real data.

Microsoft’s docs discuss **how** to apply migrations in deployment; the key idea for your mental model is: **treat schema changes like releases**, not like a side effect of `Main()`.

Useful entry points:

- [Migrations overview](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [Applying migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying)

---

## Simple decision guide

| You are… | Lean toward |
|----------|-------------|
| Learning / weekend experiment | `EnsureCreated` *maybe* |
| Sharing a DB with teammates | **Migrations** |
| Staging / production | **Migrations** + a deliberate apply step |
| Heavy read APIs | also remember `AsNoTracking()` (Part 4–5), indexes in SQL — ORMs do not remove the need to understand the engine |

---

## “Production” in one breath (attitude, not fear)

ORMs hide **boilerplate**, not **physics**. For real systems you still care about:

- indexes and slow queries,
- realistic integration tests against the **same engine** you deploy,
- secrets and least-privilege DB users,
- logs you can actually read when something breaks.

The [EF Core overview](https://learn.microsoft.com/en-us/ef/core/) states that kind of expectation plainly; keep it as **background wisdom**, not as something you must memorize today.

---

## What to read next

[08-glossary-and-faq.md](./08-glossary-and-faq.md) — quick words and “exam style” answers in friendly form.
