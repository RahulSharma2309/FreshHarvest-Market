# Part 5 — Changing data: sticky notes, then `SaveChanges`

## Mental model

After Part 4, you can **load** objects. Now: **what if you change them?**

EF keeps a **change tracker** — think **sticky notes on each tracked object** saying:

- what it is (`Customer` id 7),
- whether it is **new**, **clean**, **dirty**, or **marked delete**,
- for updates, what the **original** snapshot looked like.

**`SaveChanges`** is the button: **“flush every sticky note to the database in a sensible order.”**

---

## States you actually care about (beginner set)

| State | Plain English | Typical cause |
|-------|----------------|---------------|
| **Added** | “Insert this row.” | `DbSet.Add(...)` |
| **Unchanged** | “Loaded from DB; I haven’t edited it.” | default after a tracked query |
| **Modified** | “This row already exists; some columns changed.” | you changed properties on a tracked entity |
| **Deleted** | “Remove this row.” | `DbSet.Remove(...)` on a tracked entity |
| **Detached** | “EF is not managing this instance.” | new `new Customer()`, or explicitly detached |

(EF also uses other states in advanced scenarios; the table above is enough to reason about most apps.)

---

## Tiny story: load, edit, save

```csharp
var customer = await _context.Customers.FirstAsync(c => c.Id == 1);
// Tracker: "Customer 1 is unchanged."

customer.FirstName = "Rahul";
// Tracker: "Customer 1 is modified; here is old vs new for columns."

await _context.SaveChangesAsync();
// EF issues UPDATE(s), then marks those entities unchanged again if all went well.
```

**Relate it to Part 4:** tracking only applies if the object was **tracked** when loaded. If you used `AsNoTracking()`, changing properties **does nothing** on save unless you **attach** and manage states yourself.

---

## What `SaveChanges` tries to do (conceptually)

1. Look at **all** pending entries.
2. Build **INSERT / UPDATE / DELETE** commands.
3. Order them so **dependencies** make sense (parents/children rules).
4. Run them inside a **transaction** by default (so you do not half-finish a story).
5. Refresh tracker state for things that succeeded (for example, store-generated ids).

---

## Transactions in one paragraph

**Default:** one `SaveChanges` → one transaction.  
**Custom:** you can start `Database.BeginTransaction()` when you need **multiple** rounds (EF + raw SQL, multiple saves, …) to succeed **together** or **not at all**.

---

## Concurrency (super short)

If you configure a **concurrency token** (row version, etc.), EF can generate updates that mean: **“change this row only if it still looks like when I read it.”** If someone else changed it first, you can get **`DbUpdateConcurrencyException`** — that is optimistic locking doing its job.

---

## Pitfall: `Update(entity)` on a big object graph

Calling `Update` on a detached graph can mark **more** than you meant as modified. Safer patterns for APIs:

- **Load** tracked entity, **mutate**, **save**, or
- Use **DTOs** and map explicitly, or
- Learn attachment rules when you need graphs.

---

## How this connects backward and forward

- **Backward:** Part 4 explains **when** objects become tracked.
- **Forward:** Part 6 explains **where** EF learned table/column shapes; Part 7 explains **how** those shapes get created in the server over time.

---

## Official reference (optional)

[Saving data — EF Core docs](https://learn.microsoft.com/en-us/ef/core/saving/)
