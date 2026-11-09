# 04 - Dependency Injection (DI)

## 1. ما هي Dependency Injection؟

**Dependency Injection (DI)** هي طريقة نعطي فيها الكلاس الاعتماديات التي يحتاجها من الخارج، بدلاً من أن ينشئها بنفسه.

مثلاً: `OrderService` تحتاج خدمة لإرسال الإشعارات. بدلاً من إنشاء `EmailService` داخلها، نمرر لها Interface تمثل خدمة الإشعارات.

**الهدف الأساسي:** تقليل الترابط بين الكلاسات (**Loose Coupling**) وتسهيل التغيير والاختبار والصيانة.

## 2. ما المشكلة التي تحلها DI؟

### بدون DI

```csharp
public class OrderService
{
    private readonly EmailService _emailService = new EmailService();

    public void CreateOrder()
    {
        // Create order
        _emailService.Send();
    }
}
```

هنا `OrderService` مرتبطة مباشرة بـ `EmailService`. إذا أردنا استخدام SMS بدلاً من Email، نحتاج إلى تعديل `OrderService`.

### مع DI

أولاً نعرّف العقد (Interface):

```csharp
public interface INotificationService
{
    void Send();
}
```

تنفيذ البريد:

```csharp
public class EmailService : INotificationService
{
    public void Send()
    {
        Console.WriteLine("Email sent");
    }
}
```

تنفيذ الرسائل:

```csharp
public class SmsService : INotificationService
{
    public void Send()
    {
        Console.WriteLine("SMS sent");
    }
}
```

الخدمة تعتمد على Interface، وليس على تنفيذ محدد:

```csharp
public class OrderService
{
    private readonly INotificationService _notificationService;

    public OrderService(INotificationService notificationService)
    {
        _notificationService = notificationService;
    }

    public void CreateOrder()
    {
        // Create order
        _notificationService.Send();
    }
}
```

نسجل التنفيذ في `Program.cs`:

```csharp
builder.Services.AddScoped<INotificationService, EmailService>();
builder.Services.AddScoped<OrderService>();
```

إذا أردنا استخدام SMS، نغيّر التسجيل فقط:

```csharp
builder.Services.AddScoped<INotificationService, SmsService>();
```

لا نحتاج إلى تعديل `OrderService`.

## 3. من ينشئ الكائنات ويمررها؟

المسؤول هو **DI Container** في ASP.NET Core، وليس `OrderService` ولا الـ Controller.

عند وصول HTTP Request واحتياج ASP.NET Core إلى إنشاء Controller:

1. يكتشف أن الـ Controller يحتاج `OrderService`.
2. يطلب من DI Container إنشاء `OrderService`.
3. يكتشف الـ Container أن الخدمة تحتاج `INotificationService`.
4. يرجع إلى التسجيلات في `Program.cs)، ويجد أن التنفيذ هو `EmailService`.
5. ينشئ `EmailService` ويمررها إلى Constructor الخاص بـ `OrderService`.
6. يمرر `OrderService` إلى الـ Controller.

هذا الأسلوب يسمى **Constructor Injection**، وهو الأسلوب المفضل غالباً لأنه يجعل اعتماديات الكلاس واضحة.

## 4. فوائد DI

- **Loose Coupling:** لا يرتبط الكلاس بتنفيذ محدد.
- **Maintainability:** تغيير التنفيذ وصيانة الكود أسهل.
- **Testability:** يمكن اختبار الخدمة باستخدام Mock بدلاً من تشغيل خدمة حقيقية.
- **Flexibility:** يمكن تبديل Email بـ SMS دون تعديل منطق الطلبات.

مثال الاختبار: نمرر Mock من `INotificationService` إلى `OrderService)، ونتأكد أن `Send()` تم استدعاؤها، دون إرسال بريد حقيقي.

## 5. Service Lifetimes

عند تسجيل الخدمات نحدد مدة حياة الكائن:

| Lifetime | المعنى |
|---|---|
| Transient | كائن جديد كلما طُلبت الخدمة من الـ Container. |
| Scoped | نسخة واحدة داخل Scope واحد، وغالباً Scope لكل HTTP Request. |
| Singleton | نسخة واحدة طوال عمر التطبيق. |

### Scoped عملياً

```csharp
builder.Services.AddScoped<INotificationService, EmailService>();
builder.Services.AddScoped<OrderService>();
```

- إذا طُلبت الخدمة أكثر من مرة داخل نفس HTTP Request، يحصل الطلب على نفس نسخة `EmailService`.
- عند وصول HTTP Request جديد، يتم إنشاء Scope جديد، وبالتالي نسخة جديدة من `EmailService`.

### لماذا DbContext مسجل Scoped افتراضياً؟

`AddDbContext` يسجل `DbContext` كـ Scoped افتراضياً:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

وبذلك يحصل كل HTTP Request عادةً على نسخة مستقلة من `DbContext`، ويمكن لخدمات الطلب نفسه مشاركة النسخة.

**مهم:** `DbContext` غير Thread-Safe؛ لا تشغّل عمليتين متزامنتين على نفس النسخة، مثل استخدام `Task.WhenAll` مع استعلامين على نفس الـ Context.

## 6. لماذا لا نسجل DbContext كـ Singleton؟

لأن Singleton واحداً قد يُستخدم من عدة Requests بالتزامن، بينما `DbContext` لا يدعم الاستخدام المتزامن من عدة Threads.

قد يظهر الخطأ:

```text
A second operation was started on this context instance
before a previous operation completed.
```

لذلك نستخدم Scoped عادةً مع طلبات الويب.

## 7. Captive Dependency

**Captive Dependency** تحدث عندما تحتفظ خدمة طويلة العمر باعتمادية أقصر عمراً، مثل Singleton تحتفظ بخدمة Scoped.

مثال غير صحيح:

```csharp
builder.Services.AddSingleton<ReportService>();
builder.Services.AddScoped<AppDbContext>();
```

```csharp
public class ReportService
{
    private readonly AppDbContext _context;

    public ReportService(AppDbContext context)
    {
        _context = context;
    }
}
```

المشكلة: `ReportService` تعيش طوال عمر التطبيق، بينما `AppDbContext` يفترض أن يعيش ضمن Scope. لا يصح أن تحتفظ Singleton مباشرة باعتمادية Scoped.

مع تفعيل التحقق من Scopes، يستطيع DI Container اكتشاف هذا التعارض وإظهار خطأ عند بناء الخدمة أو حلّها.

### الحل الأول: اجعل الخدمة Scoped

إذا كانت طبيعة الخدمة مرتبطة بطلب HTTP:

```csharp
builder.Services.AddScoped<ReportService>();
builder.Services.AddDbContext<AppDbContext>();
```

### الحل الثاني: IServiceScopeFactory

إذا كان هناك سبب فعلي يجعل الخدمة Singleton، وتحتاج إلى DbContext عند تنفيذ عملية، أنشئ Scope للعملية:

```csharp
public class ReportService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public ReportService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task GenerateReport()
    {
        using var scope = _scopeFactory.CreateScope();

        var context = scope.ServiceProvider
            .GetRequiredService<AppDbContext>();

        var orders = await context.Orders.ToListAsync();
    }
}
```

هنا يتم إنشاء Scope جديد، ويُستخدم DbContext ضمن هذا الـ Scope. لا تحتفظ بالـ Context بعد انتهاء الـ Scope.

## 8. لماذا لا نستخدم IServiceProvider في كل Service؟

استخدام `IServiceProvider` لطلب الاعتماديات يدوياً من داخل الكلاس قد يحوّل التصميم إلى **Service Locator Pattern**.

مثال:

```csharp
public class OrderService
{
    private readonly IServiceProvider _provider;

    public OrderService(IServiceProvider provider)
    {
        _provider = provider;
    }

    public void CreateOrder()
    {
        var notification = _provider
            .GetRequiredService<INotificationService>();

        notification.Send();
    }
}
```

المشكلة أن اعتماديات `OrderService` لم تعد واضحة في Constructor؛ نعرف فقط أنها تحتاج `IServiceProvider`، وقد نكتشف الاعتماديات الحقيقية عند قراءة جسم الكلاس أو أثناء التشغيل.

**الأفضل غالباً:** Constructor Injection، لأنه يوضح كل ما يحتاجه الكلاس مباشرة.

استخدام `IServiceScopeFactory` له حالات مشروعة، مثل Background Services التي تحتاج إنشاء Scope لكل عملية. لا تخلط بين هذه الحالة وبين استخدام Service Locator عشوائياً في كل كلاس.

## 9. خلاصة سريعة للمقابلة

- DI تفصل إنشاء الاعتماديات عن الكلاس الذي يستخدمها.
- DI Container ينشئ الخدمات ويمرر اعتمادياتها.
- Constructor Injection يجعل الاعتماديات واضحة.
- Transient: نسخة جديدة عند كل طلب من الـ Container.
- Scoped: نسخة واحدة لكل Scope.
- Singleton: نسخة واحدة طوال عمر التطبيق.
- DbContext يكون Scoped افتراضياً لأنه غير Thread-Safe.
- لا تجعل Singleton تعتمد مباشرة على Scoped؛ انتبه إلى Captive Dependency.
- استخدم IServiceScopeFactory عندما تحتاج Singleton إلى Scope مستقل، مثل Background Service.
- تجنب Service Locator عندما يكون Constructor Injection أوضح.

## 10. Senior Interview Questions

### Q1: What is the main purpose of Dependency Injection?

**Answer:** To reduce tight coupling by providing dependencies from outside a class, making the code easier to test, maintain, and extend.

الشرح: نعطي الكلاس ما يحتاجه بدلاً من أن ينشئ الاعتماديات بنفسه.

### Q2: Who creates and injects the registered services in ASP.NET Core?

**Answer:** The built-in Dependency Injection Container.

الشرح: الـ Container يقرأ التسجيلات وينشئ الكائنات ويمررها إلى Constructors.

### Q3: What is the difference between Scoped and Singleton?

**Answer:** Scoped creates one instance per scope, while Singleton creates one instance for the application's lifetime.

الشرح: في تطبيق الويب، غالباً كل HTTP Request له Scope مستقل.

### Q4: Why is DbContext not thread-safe?

**Answer:** A DbContext instance does not support multiple concurrent operations. Use separate instances for concurrent operations.

الشرح: لا تشغّل استعلامين بالتزامن على نفس نسخة DbContext.

### Q5: What is a Captive Dependency?

**Answer:** It occurs when a service with a longer lifetime captures a dependency with a shorter lifetime, such as a Singleton holding a Scoped service.

الشرح: مثالها Singleton تحتفظ بـ DbContext مسجل Scoped.

### Q6: Why should we avoid the Service Locator pattern?

**Answer:** It hides a class's actual dependencies and makes the code harder to understand and test.

الشرح: Constructor Injection يوضح اعتماديات الكلاس من البداية.

## 11. تمرين عملي

لديك `OrderService` تحتاج إلى إرسال إشعار عند إنشاء الطلب.

1. أنشئ `INotificationService`.
2. أنشئ تنفيذين: `EmailService` و`SmsService`.
3. اجعل `OrderService` تعتمد على Interface باستخدام Constructor Injection.
4. سجّل `EmailService` في DI Container.
5. بدّل التسجيل إلى `SmsService` دون تعديل `OrderService`.

### الحل

```csharp
builder.Services.AddScoped<INotificationService, EmailService>();
builder.Services.AddScoped<OrderService>();
```

لاستخدام SMS بدلاً من Email:

```csharp
builder.Services.AddScoped<INotificationService, SmsService>();
```

## Official References

- [Dependency injection in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [Dependency injection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)
- [DbContext configuration and lifetime](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
