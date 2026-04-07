# 4 — Query execution: from LINQ composition to SQL, connection, and materialization

This document explains **what happens when** you write a LINQ query against a `DbSet<T>`—the path from **`IQueryable`** and **expression trees** to **provider SQL**, **ADO.NET execution**, and **C# objects** back in memory.

It aligns with the conceptual “querying” section of the [EF Core overview](https://learn.microsoft.com/en-us/ef/core/) while going deeper into **timing** and **components**.

---

## 4.1 `IQueryable<T>` vs `IEnumerable<T>` (why it matters)

- **`IEnumerable<T>`** LINQ (with `Func<>` delegates) runs **in memory** when you enumerate—**not** what you want for database-backed queries.
- **`IQueryable<T>`** LINQ uses **expression trees** (`Expression<Func<>>`) so EF can **inspect** the query and translate it.

`DbSet<T>` implements **`IQueryable<T>`**—that is the extension point for the EF **query provider**.

---

## 4.2 Step 1 — Query composition (deferred execution)

Example:

```csharp
var query = _context.Customers
    .Include(c => c.Orders)
    .Where(c => c.Id == id);
```

Until you call a **terminal** operator (`ToListAsync`, `FirstOrDefaultAsync`, `CountAsync`, …), EF is mostly **building a description** of work:

- Each LINQ operator adds to an **`IQueryable`** whose **Expression** property is an **expression tree**.
- Think of it as a **recipe**: “start from Customers DbSet, include Orders, filter by Id.”

**No SQL yet** (in the typical deferred path). **No round trip** to the database.

---

## 4.3 Step 2 — Translation: expression tree → database command

When you execute:

```csharp
var customer = await query.FirstOrDefaultAsync(ct);
```

EF Core’s **query pipeline** (internally: compiler, query translation, SQL generation—version details evolve) does roughly:

1. **Normalize** the expression tree (includes, filters, ordering, projections).
2. **Translate** to a **relational** representation (conceptually: “what tables/joins/filters/projections?”).
3. Generate **provider-specific SQL** (SQL Server dialect, PostgreSQL dialect, etc.).
4. Optionally use or build a **cached query plan** for equivalent query shapes (performance optimization; details are internal and version-dependent).

**Result:** a command text + parameters bound through ADO.NET APIs (`DbCommand`).

This is the “**LINQ becomes SQL**” moment.

---

## 4.4 Step 3 — Connection acquisition (pooling)

EF asks the ADO.NET provider for a **`DbConnection`**:

- Usually the connection is **taken from the connection pool** (for connection-string pools in the process).
- The physical TCP connection may already exist from earlier requests—**pooling** amortizes cost.

**Note:** pooling is fundamentally an **ADO.NET / driver** concern; EF sits above it.

---

## 4.5 Step 4 — Command execution

EF opens the connection if needed, executes the **`DbCommand`**, and reads a **`DbDataReader`** (forward-only, streaming).

Depending on settings:

- **Single query** vs **split queries** for multiple includes (avoids Cartesian explosion in some cases).
- **Tracking** vs **no-tracking** changes what happens after materialization.

---

## 4.6 Step 5 — Materialization (rows → objects)

The EF **materializer** maps:

- Result columns → **CLR properties**
- Related result shapes → **navigation properties** (for includes/joins)

You receive a graph of objects (e.g. `Customer` with `Orders` populated) or `null` if no row matched.

---

## 4.7 Step 6 — Change tracking (default)

For **tracked** queries, EF’s **change tracker** attaches entities:

- New entities from the database typically start in **`Unchanged`** state (they match the store as far as EF knows).
- If you mutate properties on a tracked entity, the tracker can mark them **`Modified`** (EF Core term; some notes say “Changed”).

**`AsNoTracking()`** skips this attachment—ideal for read-only endpoints and reduces memory.

---

## 4.8 Step 7 — Connection return

After the reader completes:

- EF **closes** the connection (logical close) → returns to the **pool**.
- The **`DbContext`** may still be alive for the rest of the HTTP request, but it is **not** holding an open connection for that whole time (unless you explicitly begin a transaction or use behaviors that keep it open).

---

## 4.9 Async operators

`FirstOrDefaultAsync`, `ToListAsync`, etc.:

- Use **async ADO.NET** (`ExecuteReaderAsync`) under the hood.
- Free threads during I/O—important for scalability in ASP.NET Core.

---

## 4.10 “Compiled query” and caching (conceptual)

EF Core can reuse work across executions when query shapes repeat. The exact caching layers are internal, but the **mental model** from study notes is right:

- **Expression tree** → **compiled plan** (conceptually) → **SQL** → execute → materialize.

---

## Next

[05-change-tracking-and-savechanges.md](./05-change-tracking-and-savechanges.md) — mutations, states, `SaveChanges`, transactions.
