# Why Is DbContext Registered as Scoped by Default?

## 1. Short answer

In ASP.NET Core, `AddDbContext<TContext>()` registers the DbContext as **Scoped** by default. This gives each web request (more precisely, each DI scope) its own DbContext instance, allowing related operations in that scope to share one change tracker and unit-of-work boundary without sharing a non-thread-safe context across unrelated requests.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Default")));
```

Equivalent lifetime configuration:

```csharp
builder.Services.AddDbContext<AppDbContext>(
    options => options.UseSqlServer(connectionString),
    contextLifetime: ServiceLifetime.Scoped,
    optionsLifetime: ServiceLifetime.Scoped);
```

## 2. What does a DbContext do?

A DbContext is the primary EF Core unit for:
- Querying entities from the database.
- Tracking entity states: Added, Modified, Deleted, Unchanged, Detached.
- Applying changes through `SaveChanges()` / `SaveChangesAsync()`.
- Coordinating database operations and transactions where supported.

Example:

```csharp
public class OrdersService
{
    private readonly AppDbContext _db;

    public OrdersService(AppDbContext db) => _db = db;

    public async Task CreateAsync(Order order)
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
    }
}
```

The context tracks the entity after `Add`. Calling SaveChanges persists the pending changes.

## 3. Why Scoped?

### A. One context per request scope

In a typical ASP.NET Core application, a request creates a DI scope. Scoped services resolved within that scope share the same instance.

If `OrdersService` and `InventoryService` both receive the same scoped AppDbContext during one request, they can participate in the same unit of work.

### B. Appropriate change-tracking boundary

A request commonly performs a related set of operations. A scoped context can track those changes and persist them together when the application calls SaveChanges.

### C. Avoids cross-request state leakage

A Singleton DbContext could be shared by many requests. Its change tracker could retain entities and state across requests, creating correctness, concurrency, and memory problems.

### D. Lifetime aligns with common web application services

Application services and repositories are often scoped. A scoped DbContext can be injected into them without a captive-dependency lifetime violation.

## 4. Why not Singleton?

DbContext is **not thread-safe**. EF Core does not support multiple parallel operations on the same context instance.

Incorrect:

```csharp
builder.Services.AddDbContext<AppDbContext>(
    options => options.UseSqlServer(connectionString),
    ServiceLifetime.Singleton);
```

Potential consequences:
- Concurrent operations can throw an exception or cause undefined application behavior.
- Change-tracker state can be shared across users/requests.
- Tracked entities can remain alive much longer than intended.
- A singleton service cannot safely consume a scoped DbContext.

Never use one DbContext instance concurrently from multiple threads.

## 5. Why not Transient?

Transient means a new instance is created each time the service is resolved. It can work in some designs, but it is usually not the desired default for request-level units of work.

For example, two repositories resolving their own transient contexts may no longer share the same change tracker or unit of work. Changes made through one context are not automatically tracked by another.

Scoped gives predictable sharing within a request scope.

## 6. Scoped does not mean one transaction per HTTP request

This is an important interview distinction.

- Scoped controls the lifetime of the DbContext instance.
- It does not automatically begin a database transaction at the start of every HTTP request.
- With relational providers, a single `SaveChanges` call is transactional by default when the provider supports transactions.
- If several SaveChanges calls or other database operations must be atomic together, define an explicit transaction where appropriate.

```csharp
await using var transaction =
    await _db.Database.BeginTransactionAsync();

try
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync();

    inventory.Quantity -= order.Quantity;
    await _db.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

Use explicit transactions only when the business operation requires multiple operations to commit or roll back as one unit.

## 7. DbContext and Unit of Work

DbContext naturally implements the Unit of Work and Repository-related capabilities:
- It tracks changes to entities.
- It coordinates persistence.
- SaveChanges commits the tracked changes to the database.

You do not necessarily need to wrap EF Core in another generic repository/unit-of-work abstraction. Add abstractions when they provide a real architectural or testing benefit, not merely to duplicate DbContext.

## 8. Background services

Hosted services are commonly registered as Singleton. Injecting a scoped DbContext directly into a singleton BackgroundService is a lifetime mismatch.

Create a scope for each unit of background work:

```csharp
public sealed class OrderWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public OrderWorker(IServiceScopeFactory scopeFactory)
        => _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var scope = _scopeFactory.CreateAsyncScope();

            var db = scope.ServiceProvider
                .GetRequiredService<AppDbContext>();

            // Perform one unit of work using this context.
            await ProcessPendingOrdersAsync(db, stoppingToken);
        }
    }

    private static Task ProcessPendingOrdersAsync(
        AppDbContext db, CancellationToken cancellationToken)
    {
        // Application-specific processing.
        return Task.CompletedTask;
    }
}
```

Dispose the scope after the unit of work so its scoped services, including DbContext, are disposed.

## 9. IDbContextFactory

Use `IDbContextFactory<TContext>` when the application needs to create and dispose contexts explicitly, for example:
- Blazor components with lifetimes that do not match a request.
- Multiple independent units of work inside one scope.
- Parallel operations that each require their own context.
- Background processing where an explicit context boundary is useful.

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

public sealed class ReportService
{
    private readonly IDbContextFactory<AppDbContext> _factory;

    public ReportService(IDbContextFactory<AppDbContext> factory)
        => _factory = factory;

    public async Task<int> CountOrdersAsync()
    {
        await using var db = await _factory.CreateDbContextAsync();
        return await db.Orders.CountAsync();
    }
}
```

A context created by the factory is the caller's responsibility to dispose.

## 10. Parallel queries

Do not do this with one context:

```csharp
// Incorrect: concurrent operations on the same DbContext.
var ordersTask = _db.Orders.ToListAsync();
var usersTask = _db.Users.ToListAsync();
await Task.WhenAll(ordersTask, usersTask);
```

Prefer sequential operations, or create separate contexts for genuinely parallel work:

```csharp
await using var db1 = await _factory.CreateDbContextAsync();
await using var db2 = await _factory.CreateDbContextAsync();

var ordersTask = db1.Orders.ToListAsync();
var usersTask = db2.Users.ToListAsync();

await Task.WhenAll(ordersTask, usersTask);
```

Each context has its own tracking state and must be disposed.

## 11. Lifetime comparison

| Lifetime | Instance behavior | Typical DbContext suitability |
|---|---|---|
| Transient | New instance per resolution | Possible, but can split a unit of work |
| Scoped | One instance per DI scope | Default and recommended for typical web requests |
| Singleton | One instance for application lifetime | Not suitable for DbContext |

## 12. Common interview mistakes

- Saying Scoped means a transaction automatically wraps the whole HTTP request.
- Claiming DbContext is thread-safe.
- Running parallel EF operations on one context.
- Injecting a scoped DbContext into a singleton BackgroundService.
- Assuming two transient repositories necessarily share one context.
- Treating IDbContextFactory as a reason to keep contexts alive indefinitely.

## 13. Senior interview Q&A

### Q1. Why is DbContext registered as Scoped by default?
**Answer:** Because a request scope provides a natural unit-of-work boundary. Services in the same scope can share one change tracker, while separate requests receive separate contexts. This avoids sharing a non-thread-safe DbContext across requests.

### Q2. Is DbContext thread-safe?
**Answer:** No. EF Core does not support concurrent operations on the same DbContext instance. I use separate contexts for parallel operations.

### Q3. Does Scoped mean one transaction per request?
**Answer:** No. Scoped defines object lifetime, not transaction boundaries. A relational provider normally wraps an individual SaveChanges call in a transaction. Multiple operations requiring atomicity need an explicit transaction.

### Q4. Can a Singleton service use DbContext?
**Answer:** Not by directly constructor-injecting a scoped context. It can create a scope through IServiceScopeFactory, resolve the context inside that scope, and dispose the scope after the work.

### Q5. When would you use IDbContextFactory?
**Answer:** When context creation must be explicit, such as background jobs, Blazor, multiple units of work in one scope, or independent contexts for parallel work.

### Q6. Why not register DbContext as Transient?
**Answer:** Transient can create separate contexts for separate resolutions, which may split tracking and persistence across repositories. Scoped provides one shared context within the request scope.

### Q7. What happens if two services inject the same scoped DbContext?
**Answer:** When resolved in the same scope, they receive the same instance and can share tracked changes. They still must not execute concurrent operations on it.

## 14. Practical exercise

**Scenario:** An API request creates an order and decreases product stock. Both operations must succeed or fail together.

1. Inject the same scoped DbContext into the application service.
2. Load the product and validate available stock.
3. Add the order and update stock.
4. Call SaveChanges once if possible, so the related changes are persisted together.
5. If the workflow requires multiple SaveChanges calls, use an explicit transaction.
6. Do not run the two EF operations concurrently on the same context.

**Key takeaway:** Scoped gives a shared request-level DbContext; transaction boundaries must still be designed deliberately.

## 15. Official references

- [DbContext lifetime, configuration, and initialization](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [DbContext is not thread-safe](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/#the-dbcontext-lifetime)
- [Using transactions](https://learn.microsoft.com/en-us/ef/core/saving/transactions)
- [Use a DbContext factory](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/#use-a-dbcontext-factory)
- [Dependency injection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)
