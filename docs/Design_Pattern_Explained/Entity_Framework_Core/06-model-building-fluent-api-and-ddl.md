# Part 6 — Turning your C# intentions into “what the database should look like”

## The bridge you are learning

You write **classes** and sometimes **Fluent** configuration.  
The database has **tables**, **columns**, **keys**, **foreign keys**.

EF’s job is to keep a **single clear picture** in the middle: **the model**.

- **Queries** use the model to generate **SELECT** shapes.
- **`SaveChanges`** uses it to generate **INSERT/UPDATE/DELETE** shapes.
- **`EnsureCreated` / migrations** use it to talk **DDL** (create/alter objects).

**Same blueprint, different tools** depending on what you are doing.

---

## Three ways you influence the blueprint

1. **Do nothing fancy — conventions**  
   EF guesses: `Id` might be a key, `CustomerId` on `Order` might be a foreign key, etc. Good for prototypes; not always what you want forever.

2. **Attributes on properties** (data annotations)  
   Quick labels: required, max length, …

3. **`OnModelCreating` + Fluent API**  
   Your **explicit** overrides: exact keys, delete behaviors, table names, precision, relationships that are hard to guess.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>(entity =>
    {
        entity.HasKey(e => e.Id);
        entity.Property(e => e.FirstName).HasMaxLength(30);
        entity.HasMany(e => e.Orders)
              .WithOne(o => o.Customer)
              .HasForeignKey(o => o.CustomerId)
              .OnDelete(DeleteBehavior.Cascade);
    });
}
```

**How to read that without memorizing jargon:**

- **HasKey** — “this column is the **main name tag** for a row.”
- **HasMaxLength(30)** — “this text column is **not endless**.”
- **HasMany / WithOne / HasForeignKey** — “many orders **belong to** one customer; the **foreign key column** lives on the order side.”
- **OnDelete Cascade** — “if the parent goes away, **what happens to children**?”

---

## From Fluent lines to SQL-ish shapes (not letter-perfect, but the idea)

Your brain can map:

| Fluent idea | Database idea |
|-------------|----------------|
| Primary key | `PRIMARY KEY` constraint |
| Required string with max length | `NVARCHAR(30) NOT NULL` (SQL Server-ish) |
| Foreign key + relationship | `FOREIGN KEY ... REFERENCES ...` |
| Cascade delete | `ON DELETE CASCADE` |

EF’s generated SQL will not always match **exactly** what you would type by hand in every edge case, but **the relational picture** is what matters: **what tables exist, how they point at each other**.

---

## First build, then cache

The first time EF needs the full model, it:

1. **Collects** entities.
2. **Applies** conventions + your Fluent rules.
3. **Checks** for nonsense (broken relationships, conflicts).
4. **Caches** the result.

So the expensive “figure out the blueprint” work is **not** repeated on every single query.

---

## Seeds (`HasData`)

**Plain idea:** “When this blueprint is applied, also **insert these starter rows**.”

- With **`EnsureCreated`**, seeds may appear when the schema is first created.
- With **migrations**, seeds become **script steps** you can review in source control.

---

## How this connects to Part 7

- **`EnsureCreated`** says: **“Make reality match today’s blueprint if the database is missing.”** It does not **evolve** an old database when you change classes.
- **Migrations** say: **“Here is the **history** of blueprint changes; upgrade step by step.”**

Same **source of truth** (the model). **Different life cycle strategy.**

---

## Optional official reading

[Creating a model — EF Core docs](https://learn.microsoft.com/en-us/ef/core/modeling/)
