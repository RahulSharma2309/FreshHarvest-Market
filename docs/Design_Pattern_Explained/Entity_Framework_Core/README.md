# Entity Framework Core — Deep Architecture Series

This folder contains a **segmented, architecture-level** guide to Entity Framework Core (EF Core): what the major components are, **when** each phase runs, and **how** they connect to ASP.NET Core, ADO.NET, and your database provider.

## Official baseline

Start with Microsoft’s conceptual overview for terminology and recommended practices:

- [Overview of Entity Framework Core | Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/)

That page frames EF Core as an O/RM built around a **model** (entity types + `DbContext`), **LINQ querying**, **saving data**, and (for evolving schemas) **migrations**—all of which this series unpacks in more depth.

## How this series is organized

| # | Document | What you will understand |
|---|----------|-------------------------|
| 1 | [01-overview-the-model-and-dbcontext.md](./01-overview-the-model-and-dbcontext.md) | `DbContext`, `DbSet<T>`, `DbContextOptions`, the bridge to the database, responsibilities at a glance |
| 2 | [02-startup-build-and-database-initialization.md](./02-startup-build-and-database-initialization.md) | `WebApplicationBuilder` → `Build()`, host startup, manual scopes, `CanConnectAsync`, `EnsureCreated`, first model build |
| 3 | [03-aspnet-core-di-and-request-lifecycle.md](./03-aspnet-core-di-and-request-lifecycle.md) | Scoped `DbContext`, one context per HTTP request, constructor injection, disposal |
| 4 | [04-query-execution-from-linq-to-results.md](./04-query-execution-from-linq-to-results.md) | `IQueryable`, expression trees, translation to SQL, connection pooling, materialization |
| 5 | [05-change-tracking-and-savechanges.md](./05-change-tracking-and-savechanges.md) | Change tracker states, modifications, `SaveChanges`, connections “late open / early close” |
| 6 | [06-model-building-fluent-api-and-ddl.md](./06-model-building-fluent-api-and-ddl.md) | `OnModelCreating`, Fluent API → conceptual relational model → DDL shape |
| 7 | [07-migrations-vs-ensurecreated-production-guidance.md](./07-migrations-vs-ensurecreated-production-guidance.md) | `EnsureCreated` vs migrations, team workflows, production cautions (aligned with Learn) |
| 8 | [08-glossary-and-faq.md](./08-glossary-and-faq.md) | Short Q&A (e.g. “how does LINQ become SQL?”), key takeaways |

## Study notes (optional figures)

If you keep scanned notebook pages or diagrams, you can drop them under:

- `images/` (see [images/README.md](./images/README.md))

and link them from these docs for your own study copy.

## Scope

- Concepts apply across recent EF Core versions; your solution may target **.NET 8 / 9 / 10** with a matching EF Core major—always match package versions to your runtime.
- Provider details (SQL Server, PostgreSQL, etc.) differ slightly in generated SQL and type mapping; the **pipeline** is the same.

---

**Suggested reading order:** 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8.
