# 8 — Glossary and FAQ (architecture Q&A)

This page collects **short answers** to the kinds of “exam style” questions that show up when you study EF Core deeply—including the ones from structured notes—while pointing back to the longer documents in this folder.

**Baseline reference:** [Overview of Entity Framework Core | Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/)

---

## Glossary

| Term | Meaning |
|------|---------|
| **Model** | Metadata describing entity types, tables/columns, keys, relationships, indexes, seeds—what EF uses to generate SQL. |
| **`DbContext`** | Session + unit of work; query root via `DbSet<T>`; orchestrates tracking and `SaveChanges`. |
| **`DbSet<T>`** | Query/update endpoint for `T`; implements `IQueryable<T>`. |
| **`DbContextOptions<T>`** | Immutable configuration (provider, connection string, etc.). |
| **Expression tree** | Data structure representing code as data; enables LINQ-to-SQL translation. |
| **`IQueryable`** | Deferred LINQ surface bound to a **query provider** (EF Core). |
| **Materialization** | Building CLR objects from a `DbDataReader` row stream. |
| **Change tracker** | Tracks states/original/current values for persisted updates. |
| **Connection pool** | ADO.NET pool of reusable DB connections (per connection string key). |
| **Migration** | Versioned schema evolution script generated from model diffs. |
| **`EnsureCreated`** | Create database+tables from model if missing; not an evolution tool. |

---

## FAQ

### Q1. How does EF Core turn LINQ into SQL?

**A.** LINQ against `IQueryable` is captured as an **expression tree**, not compiled immediately to in-memory delegates. When you execute the query, EF Core runs it through an internal **query translation pipeline** that produces **provider-specific SQL** and parameters. Execution goes through **ADO.NET** (`DbCommand`, `DbDataReader`). Results are **materialized** into objects, optionally **tracked**.  

**Details:** [04-query-execution-from-linq-to-results.md](./04-query-execution-from-linq-to-results.md)

---

### Q2. What is the “query pipeline” in EF Core?

**A.** Informally: the **end-to-end compiler** from **LINQ expression** → **relational command representation** → **SQL text + parameters** → **execution** → **materialization** (and optional tracking). Exact internals evolve between versions; the **stages** above are the stable mental model.

---

### Q3. Does `DbContext` hold an open connection for its whole lifetime?

**A.** **Generally no.** It acquires a connection **when needed** for a command and releases it afterward, relying on **connection pooling**. You may hold connections longer if you start an **explicit transaction** or use APIs that require an ambient connection.

**Details:** [03-aspnet-core-di-and-request-lifecycle.md](./03-aspnet-core-di-and-request-lifecycle.md), [04-query-execution-from-linq-to-results.md](./04-query-execution-from-linq-to-results.md)

---

### Q4. Why one `DbContext` per HTTP request?

**A.** `DbContext` is **not thread-safe** and is designed as a **unit of work** for a **single logical transaction scope**. A web request maps naturally to that scope: create → handle → dispose. Isolates **change trackers** and avoids cross-request corruption.

**Details:** [03-aspnet-core-di-and-request-lifecycle.md](./03-aspnet-core-di-and-request-lifecycle.md)

---

### Q5. Why do we use `CreateScope()` at startup?

**A.** Outside an HTTP request there is **no request scope**. To resolve a **scoped** `DbContext`, you create a **manual scope** so disposal and lifetimes are correct.

**Details:** [02-startup-build-and-database-initialization.md](./02-startup-build-and-database-initialization.md)

---

### Q6. What does `CanConnectAsync` do?

**A.** A lightweight **connectivity probe** to validate configuration/reachability/authentication—often the **first real I/O**. It does **not** validate that every table matches your model.

**Details:** [02-startup-build-and-database-initialization.md](./02-startup-build-and-database-initialization.md)

---

### Q7. What does `EnsureCreated` do?

**A.** If the database does not exist, create it and create tables (and possibly seed data) from the **current model**. If the database exists, it **does not migrate** schema forward. For evolution, use **migrations**.

**Details:** [02-startup-build-and-database-initialization.md](./02-startup-build-and-database-initialization.md), [07-migrations-vs-ensurecreated-production-guidance.md](./07-migrations-vs-ensurecreated-production-guidance.md)

---

### Q8. What happens on first `SaveChanges` after modifying a tracked entity?

**A.** EF generates **`UPDATE`** statements for entities in the **`Modified`** state (with concurrency tokens if configured), executes them in a transaction, and resets states to **`Unchanged`** on success.

**Details:** [05-change-tracking-and-savechanges.md](./05-change-tracking-and-savechanges.md)

---

### Q9. How does Fluent API relate to SQL DDL?

**A.** Fluent API configures the **conceptual relational model**. That model is turned into DDL either by **`EnsureCreated`** (direct create) or by **migrations** (scripted diffs)—different machinery, same source of truth: the **EF model**.

**Details:** [06-model-building-fluent-api-and-ddl.md](./06-model-building-fluent-api-and-ddl.md)

---

## Index

Back to series index: [README.md](./README.md)
