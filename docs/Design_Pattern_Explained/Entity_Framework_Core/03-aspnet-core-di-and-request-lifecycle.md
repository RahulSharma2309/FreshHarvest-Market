# Part 3 — Web apps: why each HTTP request gets its own `DbContext`

## The idea in one sentence

**One request = one short-lived workspace.**  
That workspace is your `DbContext`. When the request ends, the workspace is **thrown away** so the next customer does not inherit the last customer’s sticky notes.

---

## How this ties to Part 1 and 2

- Part 1: `DbContext` is a **session** with the database, not the database itself.
- Part 2: at startup you **manually** create a scope because there is **no request**.
- **This part:** during normal traffic, the **framework** creates the scope **for you**.

Same type (`OrderDbContext`), **different instance** every time.

---

## What `AddDbContext` really registers

Typical line:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

**Plain meaning:** “Whenever someone asks for `AppDbContext` **inside a scope**, build one with these options. **Dispose** it when the scope ends.”

Default lifetime is **scoped** — on purpose.

---

## Walk through one API call

Imagine:

`GET /api/customers/42`

**Step A — Routing**  
The framework picks **which method** will run (your controller action or minimal API).

**Step B — A new “bubble” appears: the request scope**  
Inside that bubble, DI can create **scoped** services.

**Step C — Your controller is created**  
Constructor injection:

```csharp
public CustomersController(AppDbContext db) { _db = db; }
```

The `db` you get **belongs to this request’s bubble only**.

**Step D — Your action runs**  
You write LINQ. Some lines **compose** a query; the line with `FirstOrDefaultAsync` (or similar) **executes** it (see Part 4).

**Step E — Response goes out**  
HTTP result is produced.

**Step F — The bubble pops**  
Scope disposal → `DbContext` disposal.

**What disposal gives you:**

- Change tracker cleared (no ghost state for the next request).
- Connections used by this context go back to the **pool** (they are not “destroyed”; they become available again).

---

## “But why not one global `DbContext` for the whole app?”

Three practical reasons:

1. **`DbContext` is not thread-safe.** Web servers handle **many** requests **at the same time**. One shared context would be like **one** notepad passed between **twenty** people writing at once.

2. **The change tracker is memory with opinions.** It remembers what you loaded. If Request A loaded a `Customer` and Request B reused the same context, their **edits and assumptions** would collide.

3. **Lifetime matches HTTP.** HTTP is **stateless** at the server level. A fresh context per request **matches** that story: each request starts clean.

---

## Connection myth-busting

**Myth:** “The context keeps SQL open for the whole request.”

**Closer to truth:** EF tends to **borrow** a connection from the pool **when a command runs**, then **return** it. You can hold a connection longer if you start a **transaction** yourself—then you are asking for a longer phone call on purpose.

---

## Startup scope vs request scope — same tool, different boss

| Situation | Who creates the scope? |
|-----------|-------------------------|
| `EnsureCreated` / seeding at startup | **Your** `CreateScope()` |
| Normal controller / minimal API | **ASP.NET Core** per request |

Both paths resolve the **same registered type**, but **different instances** and **different lifetimes**.

---

## What to read next

[04-query-execution-from-linq-to-results.md](./04-query-execution-from-linq-to-results.md) — what actually happens when you “run” the LINQ.
