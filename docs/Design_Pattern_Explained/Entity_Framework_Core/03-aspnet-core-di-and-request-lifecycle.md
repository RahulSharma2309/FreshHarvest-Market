# 3 — ASP.NET Core DI, scopes, and the per-request `DbContext` lifecycle

This document explains why **`DbContext` is scoped per HTTP request** in typical web apps, how constructor injection works, and what happens when the request ends.

---

## 3.1 The core rule: one `DbContext` per request

For web APIs and MVC apps, the common registration is:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

`AddDbContext` registers `AppDbContext` as **scoped** by default.

**Meaning:**

| Request | New scope? | New `DbContext`? |
|---------|------------|------------------|
| Request 1 | Yes | Instance #1 — use — dispose at end |
| Request 2 | Yes | Instance #2 — use — dispose at end |
| Request 3 | Yes | Instance #3 — use — dispose at end |

**Isolation:** change tracker state, identity map, and any cached tracked entities **do not leak** between requests.

---

## 3.2 Phase 1 — Request arrives (routing)

Example:

`GET https://localhost:5182/api/customers/1`

1. **Routing** selects an endpoint (controller action or minimal API delegate).
2. The host begins **request processing** for that single HTTP transaction.

---

## 3.3 Phase 2 — ASP.NET Core creates a request scope

Before your controller action runs, the framework:

1. Creates a **service scope** (implements `IServiceScope`).
2. Resolves **scoped** services from `scope.ServiceProvider`.

When the framework resolves `CustomerController`:

```csharp
public class CustomersController : ControllerBase
{
    private readonly AppDbContext _context;

    public CustomersController(AppDbContext context, ILogger<CustomersController> logger)
    {
        _context = context;
    }
}
```

the **same scope** provides:

- `AppDbContext` (scoped)
- `ILogger<T>` (often singleton or scoped depending on registration)

This is the **most important** EF Core + web integration concept: **the context is born with the request** (conceptually), not with the process.

---

## 3.4 Phase 3 — `DbContext` construction (still lazy on connections)

When the DI container constructs `AppDbContext`:

- It injects **`DbContextOptions<AppDbContext>`** (built at startup from `AddDbContext`).
- The base `DbContext` constructor stores that configuration.

**Still:** no guarantee a **physical** connection is open. The context holds **provider + connection string + options**.

---

## 3.5 Phase 4 — Controller action runs (query composition vs execution)

Example:

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<CustomerDto>> GetCustomer(int id, CancellationToken ct)
{
    // At method entry (conceptually):
    // - DbContext is injected
    // - Connection string / provider configured
    // - Change tracker is empty (no entities loaded yet)

    var customer = await _context.Customers
        .Include(c => c.Orders)
        .FirstOrDefaultAsync(c => c.Id == id, ct);

    // ↑ Until FirstOrDefaultAsync, the query is mostly *composed* (IQueryable chain)
    // Execution triggers translation + SQL + materialization (see doc 4)

    return customer is null ? NotFound() : Ok(Map(customer));
}
```

**`Include`:** eager loading—shapes the SQL (typically joins or split queries, depending on version/settings) so related data is retrieved with the root entity.

---

## 3.6 Phase 5 — Response

The action returns a result; middleware serializes the HTTP response.

**EF Core note:** by this point, for a read-only query, the context may have **tracked** loaded entities (default tracking) or not (if the query is `AsNoTracking()`).

---

## 3.7 Phase 6 — Cleanup (end of scope)

When the request pipeline completes:

- The **scope is disposed**.
- `DbContext.Dispose()` runs (unless something leaked a reference and prevented disposal—an anti-pattern).

Typical effects of disposal:

- **Change tracker** cleared; tracked entities released for GC.
- Internal state reset.
- Any **active** connection used by this context is **closed** / returned to the **connection pool** (ADO.NET pool is shared at the process level).

**Important:** EF generally **does not keep connections open** for the lifetime of the context. It tends to **open when needed** for a command and **close** when done (pooling makes “close” cheap).

---

## 3.8 Contrast: manual scope at startup vs request scope

| Context | Scope created by |
|---------|------------------|
| Startup seeding / `EnsureCreated` | Your code: `app.Services.CreateScope()` |
| Normal API call | ASP.NET Core: **one scope per request** |

Both patterns resolve the same scoped `DbContext` type—but **different instances** and **different lifetimes**.

---

## Next

[04-query-execution-from-linq-to-results.md](./04-query-execution-from-linq-to-results.md) — expression trees, SQL translation, pooling, materialization.
