# Part 1 — The big picture: `DbContext`, `DbSet`, and “the map in EF’s head”

## What you are trying to learn

You want two skills at once:

1. **Use** EF without feeling like magic.
2. **Picture** what happens when: objects, SQL, and timing.

This part gives you **characters** and **roles**. Later parts put them on a **timeline**.

---

## Analogy: three roles in a restaurant

| Role | In EF terms | Job |
|------|-------------|-----|
| **You (chef’s mind)** | Your entity classes (`Customer`, `Order`) | You think in *objects* and *business rules*. |
| **The expediter** | `DbContext` | Carries orders to the kitchen, brings plates back, remembers what changed on the table. |
| **The kitchen** | Database + provider | Speaks **SQL** and stores **rows**. |

EF Core is the **expediter**. It does not *replace* the kitchen; it **translates** between you and the kitchen.

---

## `DbContext` — the expediter (session + unit of work)

**Plain definition:** `DbContext` is the object you hold for a **short, focused piece of work** (often one HTTP request). It:

- Knows **how** to reach the database (connection string, SQL Server vs PostgreSQL, etc.).
- Exposes **entry points** for each kind of row you care about (`DbSet<Customer>`, …).
- Can **remember** entities you loaded or added, so it can **save** changes in one batch.

**It is not the database.** It is **one conversation** with the database.

**Important timing detail:** when the constructor runs, you usually have **configuration**, not an open wire to SQL yet. Think: the expediter has the **address** of the kitchen, but the **phone line** is not permanently plugged in. EF opens the line when it needs to run a command, then hangs up (connection pooling makes “hang up” cheap).

---

## `DbSet<T>` — the menu section for one table (conceptually)

**Plain definition:** `DbSet<Customer>` is “everything EF knows about **Customer** rows for **this** context.”

Two habits that confuse beginners:

1. **It is not a list in memory.**  
   `_context.Customers` is more like **“the idea of the Customers table, queryable.”** Until you run `ToListAsync`, `FirstOrDefaultAsync`, etc., you have not necessarily hit the database.

2. **It is your LINQ starting point.**  
   Because it behaves like `IQueryable`, you can chain `.Where`, `.OrderBy`, `.Include`. That chain is **a recipe**, not **the meal**.

**Mental shortcut:** `DbSet` = **named door** into one kind of entity for this context.

---

## `DbContextOptions<T>` and `: base(options)` — the sealed envelope

When you see:

```csharp
public class OrderDbContext : DbContext
{
    public OrderDbContext(DbContextOptions<OrderDbContext> options) : base(options) { }
}
```

Think of `options` as a **small sealed envelope** that already says:

- which **database family** (SQL Server, SQLite, …),
- **where** it lives (connection string),
- and extra knobs (timeouts, retries, …).

The base `DbContext` **stores** that envelope. **Still:** no guarantee a connection is open. The envelope is **ready**; the **call** happens later.

---

## The “model” — the map EF builds from your code

Before EF can write good SQL, it builds an internal **map**:

- Which class → which table  
- Which property → which column  
- Keys, relationships, required fields, string lengths, …

That map comes from:

- **Conventions** (EF guesses a lot),
- **Data annotations** on properties (optional),
- **`OnModelCreating`** with the Fluent API (you override guesses).

**First time** your app needs that map (first query, `EnsureCreated`, migrations tooling, …), EF **builds** it, **checks** it, then **caches** it. You pay the “thinking cost” once per app run (for that configuration), not on every single query.

---

## How this part connects to the next parts

| You just learned | Next you will see |
|------------------|-------------------|
| Context = short-lived workspace | **Part 2:** why startup uses `CreateScope()` manually |
| `DbSet` = query door | **Part 4:** what “execute” really means |
| Options ≠ open connection | **Part 3:** same idea inside an HTTP request |
| Model = internal map | **Part 6:** Fluent API filling in that map |

---

## Official reference (when you need exact API text)

[EF Core overview — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/)
