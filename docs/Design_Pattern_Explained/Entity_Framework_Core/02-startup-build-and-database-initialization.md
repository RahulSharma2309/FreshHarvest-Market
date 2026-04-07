# Part 2 — From “registering services” to “the database exists”: a timeline

## Why this chapter exists

Beginners often think: *“I called `AddDbContext`, so EF is connected.”*  
Not really. **Registration** is “put the recipe in the cookbook.” **Connection** is “actually cook.”

This page is a **clock** you can replay in your head.

---

## Moment 1 — You register services (`AddDbContext`, …)

**What exists:** names and factories in the DI container.  
**What does not exist yet:** a live `DbContext` for a request, an open SQL connection.

**Analogy:** you filed the expediter’s job description. You did not hire them for today’s shift yet.

---

## Moment 2 — `var app = builder.Build()`

**What exists:** the host, middleware pipeline wiring, “the app can run.”  
**What still does not exist:** per-request scope, random `DbContext` instances floating around.

**Analogy:** the restaurant **built the dining room**. No customer is seated yet.

---

## Moment 3 — Startup: “I need a `DbContext` but there is no HTTP request”

In a web app, `DbContext` is usually **scoped** = **one per logical scope**.  
During a normal request, ASP.NET Core **creates that scope for you**.

At **startup**, there is **no request**, so **you** create a scope:

```csharp
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<OrderDbContext>();
    // use context...
}
```

**Why `using` matters:** when the scope ends, scoped services are **disposed**. Your context cleans up (tracker, internal state, any active connection for that context).

**What happens when `GetRequiredService` runs?**

- DI **constructs** `OrderDbContext` and passes **`DbContextOptions`**.
- That is like handing the expediter their **instructions**.
- **Still:** this alone does not mean SQL Server just felt a permanent connection. Think “person hired,” not “phone glued to ear.”

---

## Moment 4 — First time EF needs “the map” (model build)

The **first** operation that needs the full picture (query, `EnsureCreated`, etc.) triggers **model building**:

1. Find entity types (often starting from your `DbSet` properties).
2. Apply **conventions** + your **`OnModelCreating`** rules.
3. Turn that into a consistent **relational picture** (tables, keys, FKs, …).
4. **Validate** (broken relationships, conflicts).
5. **Cache** the result so the next thousand queries do not redo the heavy work.

**Relate it:** this is EF **memorizing the blueprint** after reading your drawings once.

---

## Moment 5 — `CanConnectAsync()` — “ping, are you there?”

```csharp
var ok = await context.Database.CanConnectAsync();
```

**What it is for:** a cheap sanity check—config, network, permissions, “can we talk at all?”  
**What it is not:** “do all my tables match my C# model?”

**Kitchen picture:** you dial the kitchen; someone picks up; you ask a **tiny** question; you hang up. On SQL Server this often boils down to something like opening a connection and running a minimal probe (exact details belong to the provider).

---

## Moment 6 — `EnsureCreated()` — “if the building is missing, build it from today’s blueprint”

```csharp
context.Database.EnsureCreated();
```

**Story version:**

1. Is there a database with the right name? If **no**, create it.
2. Do we need to create tables from the **current** EF model? Roughly: **yes on first creation**.
3. If you configured seed data (`HasData`), that can be part of the first-time picture.

**The catch that trips people:** if the database **already exists**, `EnsureCreated` is **not** your friend for **changing** the schema later. It does not mean “make my old DB match my new classes.” It is closer to **“if empty lot, build; if building already there, assume we’re done.”**

For **evolving** a real shared database, teams use **migrations** (Part 7).

---

## Moment 7 — `app.Run()` — customers arrive

Now the server accepts HTTP traffic. **Each** request will get its **own** scope and usually its **own** `DbContext` (Part 3).

By the time heavy traffic hits, you typically want: **schema ready**, **data strategy clear** (`EnsureCreated` for toys, migrations for grown-up apps).

---

## One table to remember

| When | Do I have a `DbContext` instance? | Is SQL necessarily busy the whole time? |
|------|-----------------------------------|----------------------------------------|
| After `AddDbContext` | No | No |
| After `Build()` | No | No |
| After `CreateScope` + `GetRequiredService` | Yes | No — only when a command runs |
| During `CanConnectAsync` | Yes | Briefly |
| During first `EnsureCreated` on empty DB | Yes | During DDL work |
| During a normal API request | Yes, **new one per request** | Only around commands |

---

## What to read next

[03-aspnet-core-di-and-request-lifecycle.md](./03-aspnet-core-di-and-request-lifecycle.md) — the same context idea, but **per HTTP request**.
