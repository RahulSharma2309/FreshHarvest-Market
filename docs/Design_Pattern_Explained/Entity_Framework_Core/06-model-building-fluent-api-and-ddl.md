# 6 — Model building: conventions, Fluent API, and how configuration becomes DDL

This document explains **`OnModelCreating`**, how EF Core builds its **internal model**, and how that model relates to **CREATE TABLE**-style DDL—conceptually the same bridge your notes drew from Fluent configuration to SQL.

---

## 6.1 Where configuration lives

EF discovers:

- **Entity types** via `DbSet<T>` properties and references from those types.
- **Conventions** (defaults for keys, FKs, table names, etc.).
- **Data annotations** on properties (optional).
- **`OnModelCreating(ModelBuilder modelBuilder)`** for **Fluent API** overrides.

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

---

## 6.2 First-time model build (cached afterward)

When the first query or `EnsureCreated` / migration build needs the model:

1. **Scan** entity types and `DbSet` roots.
2. **Apply conventions** (naming, keys, relationship inference).
3. **Invoke `OnModelCreating`**—Fluent calls **mutate** the `ModelBuilder`’s internal metadata.
4. **Finalize** mappings: CLR types ↔ column types, nullability, lengths, precision, indexes, etc.
5. **Validate** (conflicting relationships, missing FKs, etc.).
6. **Cache** the compiled model for the lifetime of the app domain (for that options configuration).

---

## 6.3 From Fluent API to SQL shape (conceptual)

Your notes illustrated:

- `HasKey` → primary key constraint.
- `HasMaxLength(30)` → `NVARCHAR(30)` (SQL Server example).
- `HasMany` / `WithOne` / `HasForeignKey` → **foreign key column** + **relationship** in the relational model.
- `OnDelete(Cascade)` → `ON DELETE CASCADE` on the FK.

EF Core does **not** always emit SQL identical to handwritten DDL for every edge case, but the **relational model** it builds is what both **`EnsureCreated`** and **migrations** consume—through different pipelines:

- **`EnsureCreated`**: “make the database look like this model now” (no history).
- **Migrations**: “evolve from snapshot A to snapshot B with upgrade/downgrade scripts.”

---

## 6.4 Relationships: navigation properties vs shadow FKs

EF can use:

- **Explicit FK property** on the dependent (`CustomerId`).
- **Shadow properties** (no CLR property) when inferred—less obvious in debugging.

Fluent API can **rename** columns, **split** tables, map to views, configure **owned types**, **table-per-hierarchy**, etc.—all still part of the same **model** abstraction.

---

## 6.5 Seeds: `HasData`

`HasData` attaches **seed rows** to the model. How they are applied:

- **`EnsureCreated`**: may insert seed data when creating schema.
- **Migrations**: generates **InsertData** operations in migration `Up()` methods.

---

## Next

[07-migrations-vs-ensurecreated-production-guidance.md](./07-migrations-vs-ensurecreated-production-guidance.md) — choosing a strategy for real deployments.
