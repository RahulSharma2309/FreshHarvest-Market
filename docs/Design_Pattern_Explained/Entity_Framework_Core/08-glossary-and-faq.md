# Part 8 — Cheat sheet: words and quick answers

Use this when you **know the shape** of an idea but forgot the label, or when you want a **two-minute recap** before coding.

For the **story version** of each topic, follow the links to Parts 1–7.

---

## Glossary (friendlier than a textbook)

| Word | Think of it as… |
|------|------------------|
| **Model** | EF’s **blueprint**: classes ↔ tables, properties ↔ columns, relationships. |
| **`DbContext`** | One **workspace** for a short job (often one HTTP request): query, track, save. |
| **`DbSet<T>`** | The **named door** to query/update `T` rows through this context. |
| **`DbContextOptions`** | The **sealed settings envelope**: provider + connection string + extras. |
| **`IQueryable`** | LINQ that stays a **description** until you execute (good for SQL translation). |
| **Expression tree** | The **structured recipe** of a query EF can read and translate. |
| **Materialize** | Turn **result rows** into **C# objects**. |
| **Change tracker** | **Sticky notes** on tracked entities: new / clean / dirty / delete-me. |
| **Connection pool** | **Shared taxis** for DB connections — borrow, use, return. |
| **Migration** | A **chapter** of schema history you can replay on each environment. |
| **`EnsureCreated`** | **If missing, build from today’s blueprint** — not a full evolution story. |

---

## FAQ

### How does LINQ become SQL?

You chain LINQ on `IQueryable`. That builds an **expression tree** — a recipe EF can inspect. When you execute (`ToListAsync`, …), EF **compiles** that recipe into **provider-specific SQL**, runs it through **ADO.NET**, then **materializes** results (and may **track** them).

**Longer story:** [Part 4](./04-query-execution-from-linq-to-results.md)

---

### What is the “query pipeline” people mention?

An informal name for the **whole assembly line**: recipe → relational plan → SQL + parameters → execute → objects. Exact internals change between versions; the **stages** are what you keep.

---

### Does `DbContext` hold a connection open the whole time?

Usually **no**. It borrows a connection **around** commands and returns it to the **pool**. Longer holds happen if **you** open a transaction or use APIs that need a sustained connection.

**More:** [Part 3](./03-aspnet-core-di-and-request-lifecycle.md), [Part 4](./04-query-execution-from-linq-to-results.md)

---

### Why one `DbContext` per HTTP request?

`DbContext` is **not thread-safe**, and its tracker is **memory with opinions**. A fresh context per request matches **parallel traffic** and **stateless HTTP** — no bleed-over between users.

**More:** [Part 3](./03-aspnet-core-di-and-request-lifecycle.md)

---

### Why `CreateScope()` at startup?

Scoped services need a **scope**. A request **creates** one automatically. Startup **does not**, so you **make** a scope to resolve `DbContext` safely and **dispose** it.

**More:** [Part 2](./02-startup-build-and-database-initialization.md)

---

### What does `CanConnectAsync` tell me?

**“Can we talk to the server with this config?”** — not “are all my tables correct.”

**More:** [Part 2](./02-startup-build-and-database-initialization.md)

---

### What does `EnsureCreated` do?

**If the database is missing**, create it (and tables) from the **current** model. **If it already exists**, it is **not** your ongoing schema migration tool.

**More:** [Part 2](./02-startup-build-and-database-initialization.md), [Part 7](./07-migrations-vs-ensurecreated-production-guidance.md)

---

### I changed a property; why didn’t `SaveChanges` update the database?

Common causes: entity was **`AsNoTracking()`**, never **attached**, or you saved a **different** context instance than the one that tracked the object.

**More:** [Part 5](./05-change-tracking-and-savechanges.md)

---

### How does Fluent API relate to real SQL tables?

Fluent (and conventions/annotations) **build the model**. **`EnsureCreated`** or **migrations** then turn that model into **DDL** — different tools, **same blueprint**.

**More:** [Part 6](./06-model-building-fluent-api-and-ddl.md)

---

## Back to the series

[Index — README.md](./README.md)
