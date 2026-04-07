# Part 4 — From LINQ on the screen to rows in memory: the journey

## What feels confusing at first

You write C#. The database speaks SQL. Somewhere in the middle, something has to **understand your C# well enough** to write **correct SQL**.

That “something” is EF’s query side. This page is the **storyboard** of one query.

---

## Two kinds of LINQ (why `IQueryable` matters)

**`IEnumerable` + normal delegates**  
Think: **LINQ to Objects**. Filters run in **your process** on **things already in RAM**.

**`IQueryable` + expression trees**  
Think: **LINQ as a description**. EF can **read** the description and **translate** it to SQL.

`DbSet<T>` is `IQueryable`. That is the hook.

---

## Scene 1 — Composing: you are writing a recipe, not eating

```csharp
var query = _context.Customers
    .Include(c => c.Orders)
    .Where(c => c.Id == id);
```

Until you call something that **forces execution** (`ToListAsync`, `FirstOrDefaultAsync`, `CountAsync`, …), you are mostly stacking **intent**:

- Start from **Customers**
- **Also bring** related **Orders** (eager load)
- **Filter** to one id

**No trip to SQL yet** in the usual deferred path.  
**Analogy:** you are writing the **order ticket**; the kitchen has not started cooking.

---

## Scene 2 — Execution: “go run this now”

```csharp
var customer = await query.FirstOrDefaultAsync(ct);
```

Now EF must produce **real work**:

1. **Look at the whole expression tree** (the stacked intent).
2. **Turn it into a relational plan** — which tables, joins, filters, projections.
3. **Generate SQL** your provider understands (SQL Server dialect vs PostgreSQL dialect, etc.).
4. **Add parameters** so values are not clumsily pasted into the string (good for safety and caching).

**This is the “LINQ becomes SQL” moment** people draw in notebooks.

---

## Scene 3 — Borrowing a line to the database (pooling)

EF asks the ADO.NET layer for a connection. Usually that means: **grab a free connection from the pool** for this connection string.

**Analogy:** a **shared taxi stand**. You do not manufacture a new car per trip; you take the next available cab and return it when done.

---

## Scene 4 — Execute and stream results

EF runs the command and reads through a **forward-only reader** (think: **read rows one by one from a firehose**, not load a giant array blindly without control).

Details like **single query vs split queries** for heavy `Include` trees affect **how many** round trips and **how wide** each result set is—worth tuning when performance matters.

---

## Scene 5 — Materialization: rows become objects

EF builds **real C# instances**:

- Column values → properties  
- Related rows → navigation properties (when you used `Include` / joins)

You get either an object graph or `null` if nothing matched.

---

## Scene 6 — Tracking (default): “I’m watching this instance”

If you did **not** say `AsNoTracking()`, EF **attaches** loaded entities to the **change tracker**:

- They start life as **unchanged** relative to the database (as far as EF knows).
- If you change properties later, EF can notice and mark them **modified** for `SaveChanges` (Part 5).

**Read-only APIs** often use `AsNoTracking()` to say: **“don’t keep sticky notes on these objects.”**

---

## Scene 7 — Give the connection back

After the reader is done, EF **closes** the logical connection → it **returns to the pool**.  
Your `DbContext` may still exist for the rest of the request, but it is **not** assumed to hold an open pipe the entire time.

---

## String it together as a mantra

**Compose → (later) translate → connect → execute → materialize → (maybe) track → release connection.**

---

## How this connects to other parts

| Part | Link |
|------|------|
| Where does `DbContext` come from? | Part 3 |
| What happens when I edit and save? | Part 5 |
| Where does table/column knowledge come from? | Part 6 |

---

## Official deep dives (optional)

[Querying data — EF Core docs](https://learn.microsoft.com/en-us/ef/core/querying/) (for edge cases and APIs)
