# 01 - Why Is DbContext Registered as Scoped by Default?

## 1. الفكرة باختصار | Short Answer

في ASP.NET Core، الدالة `AddDbContext<TContext>()` تسجّل `DbContext` كـ **Scoped** افتراضياً.

يعني غالباً كل HTTP Request يحصل على نسخة مستقلة من `DbContext`، والخدمات التي تعمل داخل نفس الـ Request Scope يمكنها مشاركة النسخة نفسها.

**Why Scoped?** A request scope is a natural unit-of-work boundary. It lets related services share one change tracker without sharing a non-thread-safe context across unrelated requests.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Default")));
```

> ملاحظة: المقصود بدقة هو نسخة واحدة لكل DI Scope، وليس بالضرورة لكل HTTP Request في كل أنواع التطبيقات. في تطبيقات الويب المعتادة، يُنشأ Scope لكل Request.

## 2. ما وظيفة DbContext؟ | What does it do?

`DbContext` هو المكوّن الأساسي في EF Core للتعامل مع قاعدة البيانات، ويقوم بـ:

- تنفيذ الاستعلامات وجلب البيانات.
- تتبّع حالة الكيانات (Change Tracking)، مثل Added وModified وDeleted.
- حفظ التغييرات باستخدام `SaveChangesAsync()`.
- إدارة عمليات الحفظ والمعاملات وفق قواعد EF Core ومزوّد قاعدة البيانات.

مثال:

```csharp
public class OrdersService
{
    private readonly AppDbContext _db;

    public OrdersService(AppDbContext db)
    {
        _db = db;
    }

    public async Task CreateAsync(Order order)
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
    }
}
```

عند استدعاء `Add` تصبح حالة الكيان Added، وعند `SaveChangesAsync` يحاول EF Core حفظ التغييرات في قاعدة البيانات.

## 3. لماذا Scoped؟ | Why not share one context across requests?

تخيّل أن طلب HTTP واحد يحتاج إلى خدمتين:

- `OrdersService`
- `InventoryService`

إذا كانتا ضمن نفس الـ Scope وتم حقن `AppDbContext` فيهما، فستحصلان على نفس النسخة.

**الفائدة:** يمكنهما مشاركة Change Tracker والعمل ضمن وحدة عمل (Unit of Work) واحدة.

أما Request آخر فيحصل على Scope ونسخة DbContext مختلفين.

```text
HTTP Request 1  -> Scope 1 -> DbContext A
   OrdersService   ─┐
                    ├── نفس DbContext A
   InventoryService ┘

HTTP Request 2  -> Scope 2 -> DbContext B
```

هذا يقلل مشاركة حالة التتبع بين المستخدمين والطلبات المختلفة، ويجعل عمر السياق مناسباً لعملية العمل المعتادة.

## 4. لماذا لا نستخدم Singleton؟ | Why not Singleton?

`DbContext` **ليس Thread-Safe**؛ أي لا يدعم تنفيذ عمليات متزامنة متعددة على نفس النسخة.

لو جعلناه Singleton فقد تستخدم Requests مختلفة نفس النسخة في الوقت نفسه، وقد ينتج عن ذلك:

- أخطاء عند تنفيذ عمليات متزامنة.
- مشاركة Change Tracker وبيانات متتبعة بين طلبات مختلفة.
- احتفاظ الكائنات المتتبعة بالذاكرة مدة أطول من المطلوب.

قد يظهر الخطأ التالي عند بدء عملية قبل انتهاء عملية أخرى على نفس السياق:

```text
A second operation was started on this context instance
before a previous operation completed.
```

**الخلاصة:** لا تشارك نسخة DbContext واحدة بين Threads أو Requests متزامنة.

## 5. لماذا ليس Transient؟ | Why not Transient?

`Transient` يعني أن الـ DI Container ينشئ نسخة جديدة عند كل عملية Resolve للخدمة.

قد يؤدي ذلك إلى أن خدمتين في نفس الطلب تحصلان على نسختين مختلفتين من DbContext، وبالتالي لكل نسخة Change Tracker مستقل.

مثلاً: إذا أضافت خدمة كياناً إلى سياقها، فلن يكون السياق الآخر على علم بهذا الكيان تلقائياً.

لذلك يكون Scoped أنسب غالباً عندما نريد سياقاً واحداً مشتركاً داخل وحدة عمل الطلب.

## 6. Scoped لا يعني Transaction تلقائياً

هذه نقطة مهمة في المقابلات:

- **Scoped** يحدد عمر الكائن.
- **Transaction** تحدد مجموعة عمليات قاعدة البيانات التي يجب أن تنجح أو تفشل معاً.
- لا يبدأ ASP.NET Core تلقائياً Transaction واحدة لكل HTTP Request لمجرد أن DbContext مسجل Scoped.
- مع مزوّد علائقي يدعم المعاملات، يقوم EF Core عادةً بتنفيذ كل استدعاء `SaveChanges` داخل Transaction واحدة.
- إذا احتجت إلى جعل عدة عمليات حفظ مستقلة جزءاً من عملية ذرّية واحدة، استخدم Transaction صريحة عند الحاجة.

مثال مختصر:

```csharp
await using var transaction =
    await _db.Database.BeginTransactionAsync();

try
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync();

    product.Stock -= order.Quantity;
    await _db.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

إذا أمكن تنفيذ التغييرات المرتبطة كلها ثم استدعاء `SaveChangesAsync` مرة واحدة، فهذا غالباً أبسط؛ لأن EF Core يوفّر Transaction للحفظ الواحد مع المزوّدات العلائقية الداعمة.

## 7. DbContext و Unit of Work

يوفّر `DbContext` وظائف Unit of Work بشكل طبيعي:

1. يتتبع التغييرات على الكيانات.
2. يجمع التغييرات المعلّقة.
3. يحفظها عند استدعاء `SaveChanges`.

لا تحتاج دائماً إلى إنشاء Generic Repository أو Unit of Work إضافي فوق EF Core؛ أضف هذه الطبقات عندما تقدم فائدة معمارية حقيقية، لا لمجرد تكرار وظائف DbContext.

## 8. ماذا لو احتجنا DbContext داخل BackgroundService؟

غالباً تكون خدمات `BackgroundService` طويلة العمر (Singleton). لذلك لا تحقن DbContext المسجل Scoped مباشرة في Constructor الخاص بها؛ فهذا يسبب تعارضاً في Lifetimes يسمى **Captive Dependency**.

أنشئ Scope لكل عملية عمل:

```csharp
public sealed class OrderWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public OrderWorker(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        await using var scope = _scopeFactory.CreateAsyncScope();

        var db = scope.ServiceProvider
            .GetRequiredService<AppDbContext>();

        await ProcessPendingOrdersAsync(db, stoppingToken);
    }

    private static Task ProcessPendingOrdersAsync(
        AppDbContext db, CancellationToken cancellationToken)
    {
        // تنفيذ عملية العمل
        return Task.CompletedTask;
    }
}
```

في Worker يعمل باستمرار، أنشئ Scope جديداً لكل دورة أو وحدة عمل، ولا تجعل Scope واحداً يمتد بلا داعٍ طوال عمر التطبيق.

## 9. متى نستخدم IDbContextFactory؟

نستخدم `IDbContextFactory<TContext>` عندما نحتاج إنشاء DbContext والتحكم بعمره يدوياً، مثل:

- تنفيذ أكثر من Unit of Work مستقلة داخل نفس Scope.
- تنفيذ عمليات متوازية، بحيث تحصل كل عملية على DbContext مستقل.
- بعض سيناريوهات Blazor أو Background Processing.

التسجيل:

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

الاستخدام:

```csharp
public class ReportService
{
    private readonly IDbContextFactory<AppDbContext> _factory;

    public ReportService(IDbContextFactory<AppDbContext> factory)
    {
        _factory = factory;
    }

    public async Task<int> CountOrdersAsync()
    {
        await using var db = await _factory.CreateDbContextAsync();
        return await db.Orders.CountAsync();
    }
}
```

**مهم:** الكود الذي ينشئ Context من Factory هو المسؤول عن التخلص منه (Dispose)، لذلك استخدم `using` أو `await using`.

## 10. العمليات المتوازية | Parallel Operations

لا تنفّذ عمليتين متوازيتين على نفس DbContext:

```csharp
// خطأ: العمليتان تستخدمان نفس DbContext بالتزامن
var ordersTask = _db.Orders.ToListAsync();
var usersTask = _db.Users.ToListAsync();

await Task.WhenAll(ordersTask, usersTask);
```

إذا كان التوازي ضرورياً، استخدم Context منفصلاً لكل عملية، مثلاً عبر `IDbContextFactory`:

```csharp
await using var db1 = await _factory.CreateDbContextAsync();
await using var db2 = await _factory.CreateDbContextAsync();

var ordersTask = db1.Orders.ToListAsync();
var usersTask = db2.Users.ToListAsync();

await Task.WhenAll(ordersTask, usersTask);
```

كل Context له Change Tracker مستقل ويجب التخلص منه بعد الانتهاء.

## 11. مقارنة Lifetimes

| Lifetime | المعنى | هل يناسب DbContext عادةً؟ |
|---|---|---|
| Transient | نسخة جديدة عند كل Resolve | ممكن، لكنه قد يفصل وحدة العمل بين الخدمات |
| Scoped | نسخة واحدة داخل Scope | نعم، وهو التسجيل الافتراضي المعتاد |
| Singleton | نسخة واحدة طوال عمر التطبيق | لا، لأنه غير Thread-Safe |

## 12. أسئلة مقابلات Senior | Interview Q&A

### Q1. Why is DbContext registered as Scoped by default?

**Answer:** A request scope provides a natural unit-of-work boundary. Services in the same scope can share one change tracker, while separate requests get separate DbContext instances.

**بالعربي:** لأن Scoped يسمح لخدمات الطلب الواحد بمشاركة نفس السياق، بينما يحصل كل طلب مستقل على سياق خاص به.

### Q2. Is DbContext thread-safe?

**Answer:** No. EF Core does not support multiple concurrent operations on the same DbContext instance.

**بالعربي:** لا. لا تشغّل عمليات متزامنة على نفس النسخة؛ استخدم Context منفصلاً لكل عملية متوازية.

### Q3. Does Scoped mean one transaction per HTTP request?

**Answer:** No. Scoped controls object lifetime, not transaction boundaries. A relational provider normally wraps one SaveChanges call in a transaction.

**بالعربي:** لا. Scoped يحدد عمر DbContext فقط، ولا ينشئ Transaction تلقائياً لكل Request.

### Q4. Can a Singleton service inject a Scoped DbContext?

**Answer:** Not directly. Use IServiceScopeFactory to create a scope, resolve the DbContext inside it, and dispose the scope after the work.

**بالعربي:** لا تحقن Scoped مباشرة في Singleton؛ أنشئ Scope مستقلاً للعملية باستخدام IServiceScopeFactory.

### Q5. When would you use IDbContextFactory?

**Answer:** When explicit context creation is needed, such as independent units of work or parallel operations that require separate contexts.

**بالعربي:** عندما تحتاج التحكم بإنشاء DbContext والتخلص منه يدوياً، أو تحتاج سياقات مستقلة لعمليات متوازية.

### Q6. What happens when two services inject the same Scoped DbContext?

**Answer:** If both are resolved from the same scope, they receive the same DbContext instance and can share tracked changes.

**بالعربي:** إذا كانتا ضمن نفس Scope فستحصلان على نفس النسخة، لكن يجب ألا تنفذا عمليات متزامنة عليها.

## 13. تمرين عملي | Practical Exercise

**Scenario:** طلب API ينشئ Order ويقلل كمية المنتج في المخزون. يجب أن تنجح العمليتان معاً أو تفشلا معاً.

المطلوب:

1. احقن نفس Scoped DbContext في الخدمة المسؤولة.
2. اقرأ المنتج وتأكد من توفر الكمية.
3. أضف الطلب وحدّث كمية المخزون.
4. حاول حفظ التغييرات المرتبطة باستدعاء واحد لـ `SaveChangesAsync`.
5. إذا احتجت عدة عمليات حفظ ويجب أن تكون كلها ذرّية، استخدم Transaction صريحة.
6. لا تنفذ عمليات EF متوازية على نفس DbContext.

**Key takeaway:** Scoped gives one shared DbContext per scope. It does not automatically create a transaction for the entire HTTP request.

## 14. أخطاء شائعة | Common Mistakes

- الاعتقاد أن Scoped يعني Transaction تلقائية لكل Request.
- تسجيل DbContext كـ Singleton.
- تشغيل استعلامين بالتزامن على نفس DbContext.
- حقن DbContext Scoped مباشرة داخل Singleton أو BackgroundService.
- افتراض أن الخدمات التي تستخدم Contexts مختلفة تشارك Change Tracker واحداً.
- إنشاء Context من Factory وعدم التخلص منه.

## 15. مراجع Microsoft الرسمية | Official References

- [DbContext lifetime, configuration, and initialization](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [DbContext is not thread-safe](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/#the-dbcontext-lifetime)
- [Using transactions](https://learn.microsoft.com/en-us/ef/core/saving/transactions)
- [Use a DbContext factory](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/#use-a-dbcontext-factory)
- [Dependency injection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)
