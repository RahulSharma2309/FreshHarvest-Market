# 2 — Application build, host startup, and database initialization

This document walks the **timeline** from `WebApplication.CreateBuilder` through `builder.Build()`, then the **optional** early startup block where many samples call `Database.CanConnectAsync()` and `Database.EnsureCreated()`.

It mirrors the “phases” from structured study notes: **pipeline exists first**, then **EF does real I/O** when you resolve a context and touch the database.

---

## 2.1 Phase A — Composition (services registration)

During `Program.cs` top-level statements (or `Startup` in older templates):

- You register services: `AddControllers`, `AddDbContext<T>`, etc.
- **No** `DbContext` instance exists yet for requests.
- **No** database connection is opened.

Architecturally: the DI container only stores **factories and descriptors**—recipes for later.

---

## 2.2 Phase B — `var app = builder.Build()`

`Build()`:

- Constructs the **host** and the **middleware pipeline** configuration.
- Still: **no per-request scope**, and typically **no** `DbContext` instance yet.

Think: *infrastructure is assembled; business database session has not started.*

---

## 2.3 Phase C — Why startup often uses `CreateScope()` manually

In ASP.NET Core, **`DbContext` is usually registered as scoped**—one logical lifetime per **HTTP request**.

At **application startup**, you are **outside** an HTTP request, so there is **no implicit scope**. If you need to run EF Core **before** the server accepts traffic (migrations, ensure database exists, seeding), you **open a scope on purpose**:

```csharp
using (var scope = app.Services.CreateScope())
{
    var services = scope.ServiceProvider;
    var context = services.GetRequiredService<OrderDbContext>();
    // ... use context ...
}
```

Why `using`:

- A scope owns **disposable** services resolved from it.
- When the scope is disposed, scoped services (including `DbContext`) are **disposed** deterministically.

**Critical point:** after `GetRequiredService<OrderDbContext>()`:

- The `DbContext` **constructor** runs with `DbContextOptions<T>` wired from DI.
- **Still** there may be **no** open connection—options are ready; connection is **lazy**.

---

## 2.4 First-time model building (inside the context)

Before EF can generate SQL, it needs a **model**—a complete description of entities, tables, keys, FKs, indexes, etc.

**First use** of a context for a given configuration triggers **model building**:

1. **Discover entity types** (from `DbSet` properties and referenced types).
2. Run **`OnModelCreating`** to apply **Fluent API** and merge **conventions**.
3. Apply **value conversions**, **relationships**, **keys**, **required/optional**, etc.
4. **Validate** the model (e.g. relationship consistency).
5. **Cache** the compiled model in memory for subsequent requests (you do not pay full build cost every time).

This is expensive relative to a simple field access—which is why EF caches it.

---

## 2.5 `Database.CanConnectAsync()` — “can I talk to the server?”

Typical call:

```csharp
var canConnect = await context.Database.CanConnectAsync();
```

Architecturally:

- This is often the **first intentional database interaction** for that startup path.
- The provider opens a connection (or borrows from the **pool**), runs a **minimal probe** (conceptually similar to a trivial query such as `SELECT 1` on SQL Server—exact mechanics are provider-specific), returns `true`/`false`, then closes / returns the connection to the pool.

It answers: **network + auth + database reachability**, not “do all my tables match the model?”

---

## 2.6 `Database.EnsureCreated()` — what it does (and what it does *not*)

Call:

```csharp
context.Database.EnsureCreated();
```

High-level behavior (conceptual):

1. **Check if the database exists** (provider-specific catalog query).
2. If **missing**, **create the database**.
3. **Create tables** (and other objects) from the **current model**—not from migration history.
4. Optionally apply **`HasData`** seed data if configured in the model.
5. If the database **already exists**, **EnsureCreated** does **not** reconcile schema changes—it effectively **does nothing** for evolution.

### Why this matters architecturally

- **`EnsureCreated`** is **not** a migrations replacement. It is a **bootstrap** helper for quick prototypes or tests.
- Microsoft’s Learn documentation and guidance consistently push **EF Core Migrations** for evolving schemas in real projects. The overview links to [Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/) and related topics.

**Document 7** contrasts **`EnsureCreated`** vs **migrations** and production rollout.

---

## 2.7 Phase D — `app.Run()` and accepting requests

After startup hooks complete:

- Kestrel (or your server) **listens** for HTTP requests.
- For each request, ASP.NET Core creates a **new DI scope** (for scoped services).
- A **new** `DbContext` instance is created for that scope (when registered with scoped lifetime).

Database schema (if you used `EnsureCreated` or applied migrations earlier) is expected to be **ready** before you rely on it under load.

---

## Mental timeline (summary)

| Moment | DbContext instance? | Connection open? |
|--------|---------------------|-------------------|
| After `AddDbContext` | No | No |
| After `Build()` | No | No |
| After `CreateScope` + `GetRequiredService` | Yes | Not necessarily |
| After `CanConnectAsync` | Yes | Briefly for probe |
| After `EnsureCreated` (first time) | Yes | During DDL |
| Per HTTP request (scoped) | New instance | Lazy per operation |

---

## Next

Continue to [03-aspnet-core-di-and-request-lifecycle.md](./03-aspnet-core-di-and-request-lifecycle.md) for the **per-request** story.
