# 5 — Change tracking, mutations, and `SaveChanges`

This document focuses on what happens **after** entities are loaded: how EF Core **detects changes** and how **`SaveChanges`** turns those into **INSERT/UPDATE/DELETE** statements.

---

## 5.1 The change tracker

Each `DbContext` owns a **change tracker** (`ChangeTracker` API):

- It stores **entity entries** for tracked instances.
- Each entry has:
  - **State** (`Detached`, `Unchanged`, `Added`, `Modified`, `Deleted`)
  - **Original values** (snapshot for updates/concurrency)
  - **Current values**

**Per-request scope** means tracker state is **isolated** per `DbContext` instance.

---

## 5.2 Loading as `Unchanged`

When EF materializes an entity from a query (tracking mode):

```csharp
var customer = await _context.Customers.FirstAsync(c => c.Id == 1);
// Entry state: Unchanged (conceptually)
```

Mutate a property:

```csharp
customer.FirstName = "Rahul";
```

The change tracker **detects** the difference between **original** and **current** values and marks the entity **`Modified`** (for scalar properties).

---

## 5.3 Insert and delete states

- **`Add` / `AddAsync`:** new entity → **`Added`** → `SaveChanges` emits **INSERT**.
- **`Remove`:** tracked entity → **`Deleted`** → **DELETE**.

---

## 5.4 `SaveChanges` / `SaveChangesAsync`

When you call:

```csharp
await _context.SaveChangesAsync(ct);
```

EF Core:

1. **Builds commands** for all pending changes (inserts/updates/deletes).
2. Orders them respecting **relationships** and **dependencies** (e.g. parent before child, or provider rules).
3. Typically wraps them in a **transaction** (implicit transaction if you did not start one explicitly).
4. Opens a connection (or uses an existing ambient transaction/connection), executes commands, accepts results (e.g. store-generated keys).
5. Updates **tracker state** back to **`Unchanged`** for persisted entities.

---

## 5.5 Connections and transactions

- **Implicit transaction:** a single `SaveChanges` call → one transaction by default.
- **Explicit transaction:** you can begin a transaction on `Database.BeginTransaction()` and enlist multiple `SaveChanges` calls or raw SQL—useful when you need atomicity across EF and hand-written SQL.

---

## 5.6 Concurrency tokens

If you configure **row version / concurrency token** properties, EF can generate **`UPDATE ... WHERE`** clauses that fail when the row changed—throwing **`DbUpdateConcurrencyException`**. Architecturally: optimistic concurrency control at the database level.

---

## 5.7 No-tracking queries

```csharp
var rows = await _context.Customers.AsNoTracking().ToListAsync();
```

- Faster for read-heavy paths; less memory.
- Mutations to returned objects are **not** persisted unless you **`Attach`** and mark states manually.

---

## 5.8 Detached graphs and `Update` pitfalls

Calling **`Update(entity)`** on a detached graph can mark **many** navigations as modified. For APIs, prefer:

- Load tracked entity, mutate, `SaveChanges`, or
- Use DTOs + explicit mapping, or
- Use patterns like **patch** with careful attachment rules.

---

## Next

[06-model-building-fluent-api-and-ddl.md](./06-model-building-fluent-api-and-ddl.md) — how configuration becomes schema.
