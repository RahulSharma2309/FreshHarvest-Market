# 1 — Overview: The model, `DbContext`, and `DbSet<T>`

This document anchors the mental model Microsoft describes in the [EF Core overview](https://learn.microsoft.com/en-us/ef/core/): data access is done through a **model** made of **entity classes** and a **context** that represents a session with the database.

---

## 1.1 What problem does EF Core solve?

EF Core is an **object–relational mapper (O/RM)**. It lets you:

- Work with **.NET objects** instead of hand-writing most ADO.NET/SQL for CRUD.
- Express queries with **LINQ**; the provider turns that into **SQL** for your database engine.
- Track changes to loaded entities and persist them with **`SaveChanges`** / **`SaveChangesAsync`**.

Microsoft’s overview emphasizes that you still need solid knowledge of the underlying database for production systems (indexes, constraints, profiling, migrations strategy). See the “EF O/RM considerations” section on [Learn](https://learn.microsoft.com/en-us/ef/core/).

---

## 1.2 The two pillars of the model

### Entity types (POCOs / domain classes)

Plain classes whose properties map to **columns** (value types, strings, enums, owned types, etc.) and whose navigation properties map to **relationships** (foreign keys, joins).

### `DbContext`

The **unit of work** and **session** abstraction:

- It is **not** “the database”—it is an **API surface** that knows how to talk to the database through a **provider** (SQL Server, Npgsql, SQLite, …).
- It exposes **`DbSet<T>`** properties. Each `DbSet<T>` is the **gateway** for querying and attaching entities of type `T` for that context.

**Mnemonic:** *`DbContext` is the heart of EF Core in your app—it is the bridge between C# objects and the database.*

---

## 1.3 What is `DbSet<T>`?

`DbSet<TEntity>`:

- Represents **the collection of entities** of type `TEntity` that EF will map to a **table** (or a view/query, depending on configuration).
- Implements **`IQueryable<TEntity>`**, which is why you can compose **LINQ**; the query is **not executed** until you use an **operator that forces execution** (`ToListAsync`, `FirstOrDefaultAsync`, `CountAsync`, …).

So: **`DbSet` = logical table + query root**, not “rows already in memory.”

---

## 1.4 Why `DbContextOptions<TContext>` and `DbContext`’s base constructor?

When you write:

```csharp
public class OrderDbContext : DbContext
{
    public OrderDbContext(DbContextOptions<OrderDbContext> options) : base(options) { }
}
```

you are wiring **immutable configuration** into the base `DbContext`:

- **Which database provider** (`UseSqlServer`, `UseNpgsql`, …).
- **Connection string** (or connection factory).
- **Provider-specific options** (retry, command timeout, etc.).
- Optional **caching**, **interceptors**, **logging**, etc.

**Important architectural point:** constructing a `DbContext` with options **does not** mean a physical connection is open yet. The context holds **how** to connect; connections are typically acquired **lazily** when a query or `SaveChanges` needs them.

---

## 1.5 Responsibilities of `DbContext` (checklist)

At an architecture level, `DbContext` orchestrates:

| Responsibility | What it means |
|----------------|----------------|
| **Model** | Knows the **entity types**, **keys**, **relationships**, **indexes**, **conventions**, and **Fluent API** configuration from `OnModelCreating`. |
| **Query compilation** | Turns **LINQ** on `IQueryable` into a **query plan** / SQL via the provider’s **query pipeline**. |
| **Change tracking** | Tracks instances it loads (when tracking is enabled) so it can detect **inserts/updates/deletes** for `SaveChanges`. |
| **Persistence** | Batches commands (INSERT/UPDATE/DELETE) and applies them in a transaction (default behavior). |
| **Connection management** | Uses ADO.NET under the hood; participates in **connection pooling** (pooling is primarily an ADO.NET/provider concern). |
| **Concurrency tokens / transactions** | Can use transactions you provide or implicit transactions around `SaveChanges`. |

---

## 1.6 How this fits the next documents

- **Document 2** — Host startup: when the **first** real work happens (`Build`, scopes, `EnsureCreated` / migrations).
- **Document 3** — Web apps: **scoped** `DbContext` per HTTP request.
- **Document 4** — What happens **inside** a LINQ query from composition to SQL.

---

## Further reading (Microsoft Learn)

- [Creating a model](https://learn.microsoft.com/en-us/ef/core/modeling/)
- [DbContext configuration](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
