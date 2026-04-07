# 7 — Migrations vs `EnsureCreated`, and production-oriented guidance

This document contrasts **`Database.EnsureCreated()`** with **EF Core Migrations** and summarizes **production** guidance echoed in Microsoft’s EF Core documentation ([overview](https://learn.microsoft.com/en-us/ef/core/), [migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/), [applying migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying)).

---

## 7.1 `EnsureCreated` — strengths and limits

### What it is good for

- **Local prototypes** and some **tests** where you want “create DB + tables from current model” **without** migration files.
- Very small throwaway environments.

### What it is bad at

- **Schema evolution:** if the database already exists, **`EnsureCreated` does not alter tables** to match a changed model.
- **Team workflow:** no versioned upgrade path, no downgrade, no repeatable pipeline.
- **Production:** risky to tie schema creation to app startup without careful concurrency and permission planning.

Microsoft explicitly calls out that **applying migrations at startup** can have **concurrency** and **permission** implications—prefer controlled deployment processes for production. See [Applying Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying).

---

## 7.2 Migrations — strengths

- **Versioned** schema changes checked into source control.
- **Up** and **Down** (when authored) scripts.
- **Team alignment**: everyone applies the same sequence.
- **CI/CD**: migrate as a dedicated step (job, container init, release pipeline).

---

## 7.3 Practical recommendation

| Scenario | Prefer |
|----------|--------|
| Learning / spike | `EnsureCreated` *maybe* |
| Shared dev DB | Migrations |
| Staging / production | Migrations + automated apply strategy |
| Integration tests | Migrations or test-specific database strategies (test containers, etc.) |

---

## 7.4 Other production considerations (from Learn)

The [EF Core overview](https://learn.microsoft.com/en-us/ef/core/) lists O/RM considerations including:

- Deep **database** knowledge for performance and debugging.
- **Integration tests** that mirror production engine/version.
- **Load testing** for naive query patterns (`Include` explosions, N+1, etc.).
- **Security** (secrets, SQL injection with raw SQL, least-privilege DB users).
- **Logging/diagnostics** (query tags, Application Insights).
- **Migration review** before touching production data.

---

## Next

[08-glossary-and-faq.md](./08-glossary-and-faq.md) — condensed Q&A and glossary.
