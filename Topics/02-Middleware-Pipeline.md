# ASP.NET Core Middleware Pipeline & Request Lifecycle

## 1. What is middleware?

Middleware is software assembled into the ASP.NET Core HTTP request pipeline. Each middleware component can:
- Read or modify the incoming HttpContext.
- Perform work before calling the next component.
- Call the next component to continue the pipeline.
- Perform work after the next component returns.
- Short-circuit the pipeline and produce a response without calling the next component.

A simplified pipeline:

```text
Request
  |
  v
Middleware A (before)
  |
  v
Middleware B (before)
  |
  v
Endpoint
  |
  v
Middleware B (after)
  |
  v
Middleware A (after)
  |
  v
Response
```

## 2. Registering middleware

### Use

`Use` adds middleware that can call the next component:

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");
    await next(context);
    Console.WriteLine("After");
});
```

The code before `next` runs on the way in; code after it runs while the response unwinds.

### Run

`Run` adds a terminal middleware. It does not receive a next delegate, so it ends the pipeline:

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello");
});
```

Middleware registered after this terminal delegate will not execute for that branch.

### Map

`app.Map("/path", ...) ` creates a path-based branch in the middleware pipeline:

```csharp
app.Map("/health-check", branch =>
{
    branch.Run(async context =>
        await context.Response.WriteAsync("Healthy"));
});
```

Important distinction:
- `app.Map("/path", ...)` branches the middleware pipeline based on the request path.
- `app.MapGet("/path", ...)` maps an HTTP GET endpoint through endpoint routing.

They are not the same API or concept.

## 3. Middleware order matters

Middleware executes in the order it is registered for the request, then in reverse order after the next delegate returns.

Example:

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("A - before");
    await next(context);
    Console.WriteLine("A - after");
});

app.Use(async (context, next) =>
{
    Console.WriteLine("B - before");
    await next(context);
    Console.WriteLine("B - after");
});

app.Run(async context =>
{
    Console.WriteLine("Endpoint");
    await context.Response.WriteAsync("OK");
});
```

Expected log:

```text
A - before
B - before
Endpoint
B - after
A - after
```

If middleware does not call `next`, it short-circuits the pipeline.

## 4. Short-circuiting

A middleware may decide that the request should not continue:

```csharp
app.Use(async (context, next) =>
{
    if (!context.Request.Headers.ContainsKey("X-Client-Id"))
    {
        context.Response.StatusCode = StatusCodes.Status400BadRequest;
        await context.Response.WriteAsync("Missing client id");
        return;
    }

    await next(context);
});
```

Short-circuiting is useful for rejecting invalid requests, serving cached responses, or handling a request directly. Be careful not to bypass required security or observability middleware.

## 5. A typical middleware order

A common explicit setup is:

```csharp
var app = builder.Build();

app.UseExceptionHandler("/error");

app.UseHttpsRedirection();

app.UseStaticFiles();

app.UseRouting();

app.UseCors("FrontendPolicy");

app.UseAuthentication();

app.UseRateLimiter();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

This is an illustrative order, not a universal copy-and-paste recipe. The correct order depends on the features used:
- Exception handling should be early enough to catch downstream exceptions.
- Routing must run before middleware that needs the selected endpoint or route metadata.
- CORS should be placed so endpoint routing and endpoint metadata are handled correctly.
- Authentication establishes the user identity.
- Authorization evaluates access using that identity.
- Rate limiting placement depends on the policy. Endpoint-specific policies require routing first; policies that use authenticated user information should run after authentication.
- Static files are commonly served before authorization when they are intended to be public. Protected files need an authorization-aware approach.

## 6. Authentication vs Authorization

- **Authentication:** Who is the caller? It establishes the current user identity.
- **Authorization:** Is this caller allowed to perform this operation?

Typical order:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Authorization needs the authenticated principal when a policy depends on the user's identity or claims.

## 7. Custom conventional middleware

A conventional middleware class typically receives `RequestDelegate` in its constructor and exposes `Invoke` or `InvokeAsync`:

```csharp
public sealed class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        _logger.LogInformation(
            "Incoming {Method} {Path}",
            context.Request.Method,
            context.Request.Path);

        await _next(context);

        _logger.LogInformation(
            "Response {StatusCode}",
            context.Response.StatusCode);
    }
}
```

Register it:

```csharp
app.UseMiddleware<RequestLoggingMiddleware>();
```

### Important DI lifetime rule

Conventional middleware is normally constructed when the application pipeline is built, not once per request. Therefore, do not inject a scoped service such as DbContext into its constructor.

Instead, inject a scoped dependency into `InvokeAsync`:

```csharp
public sealed class AuditMiddleware
{
    private readonly RequestDelegate _next;

    public AuditMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(
        HttpContext context,
        AppDbContext db)
    {
        // Use this request-scoped DbContext here.
        await _next(context);
    }
}
```

The InvokeAsync parameter is resolved from the request's service scope.

## 8. IMiddleware

An alternative is implementing `IMiddleware`:

```csharp
public sealed class TenantMiddleware : IMiddleware
{
    private readonly ITenantService _tenantService;

    public TenantMiddleware(ITenantService tenantService)
        => _tenantService = tenantService;

    public async Task InvokeAsync(
        HttpContext context,
        RequestDelegate next)
    {
        await _tenantService.ResolveTenantAsync(context);
        await next(context);
    }
}
```

Register it with DI:

```csharp
builder.Services.AddScoped<TenantMiddleware>();
app.UseMiddleware<TenantMiddleware>();
```

The middleware activation is handled by the middleware factory, so its dependencies can follow the registered DI lifetime. Choose the lifetime intentionally; scoped is common when it depends on scoped services.

## 9. Exception handling middleware

Centralized exception handling avoids repeating try/catch blocks in every controller:

```csharp
app.UseExceptionHandler();
```

In modern ASP.NET Core versions, exception handling can be configured with `AddExceptionHandler<T>` and an `IExceptionHandler` implementation, or with an appropriate exception-handler path/configuration.

A simplified custom example:

```csharp
app.Use(async (context, next) =>
{
    try
    {
        await next(context);
    }
    catch (Exception)
    {
        if (context.Response.HasStarted)
            throw;

        context.Response.Clear();
        context.Response.StatusCode =
            StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(new
        {
            message = "An unexpected error occurred."
        });
    }
});
```

Production exception responses should not expose stack traces, SQL details, secrets, or internal implementation information. Log the exception with a correlation/trace identifier.

## 10. Endpoint filters are not middleware

Endpoint filters are designed for minimal API endpoints. They can run before and after an endpoint handler and are useful for endpoint-specific concerns such as validation or logging.

```csharp
app.MapPost("/orders", (CreateOrderRequest request) =>
{
    return Results.Ok();
})
.AddEndpointFilter(async (context, next) =>
{
    // Before endpoint handler
    var result = await next(context);
    // After endpoint handler
    return result;
});
```

Middleware is broader: it can apply across the HTTP pipeline. Endpoint filters are attached to selected endpoints or route groups.

## 11. Request lifecycle overview

A simplified request lifecycle:
1. The server (typically Kestrel) accepts the HTTP request.
2. ASP.NET Core creates/uses the HttpContext and request service scope.
3. Middleware components execute in configured order.
4. Routing selects an endpoint.
5. Authentication and authorization middleware process identity and access according to pipeline order.
6. The selected endpoint executes (controller action, minimal API handler, etc.).
7. The response unwinds through middleware in reverse order.
8. The request scope is disposed after request processing completes.

This is a conceptual overview; hosting and endpoint details vary by application configuration.

## 12. Common mistakes

- Putting authentication after authorization.
- Registering endpoint-dependent middleware before routing.
- Assuming `app.Map("/x")` and `app.MapGet("/x")` are equivalent.
- Forgetting that middleware after a terminal `Run` does not execute in that pipeline branch.
- Injecting DbContext into the constructor of conventional middleware.
- Assuming middleware after a short-circuit will still execute.
- Returning detailed exception messages or stack traces to clients.
- Running expensive synchronous work on the request thread.

## 13. Senior interview Q&A

### Q1. What is middleware in ASP.NET Core?
**Answer:** Middleware is a component in the HTTP request pipeline. It can inspect or modify the request and response, call the next component, or short-circuit the pipeline.

### Q2. Why does middleware order matter?
**Answer:** Each component sees the request at its position in the pipeline. Some middleware depends on work performed earlier, such as authorization depending on the authenticated user and endpoint metadata being available after routing.

### Q3. What is the difference between Use, Run, and Map?
**Answer:** Use adds middleware that can call the next component. Run adds terminal middleware. Map creates a path-based pipeline branch. MapGet maps an HTTP endpoint and is part of endpoint routing.

### Q4. What is short-circuiting?
**Answer:** A middleware handles the request without calling the next delegate. Later middleware and the endpoint are skipped for that request path.

### Q5. Can I inject a scoped DbContext into middleware?
**Answer:** In conventional middleware, not through its constructor because the middleware instance is generally created when the pipeline is built. Inject it into InvokeAsync instead, or implement IMiddleware and register it with DI.

### Q6. What is the difference between authentication and authorization?
**Answer:** Authentication establishes who the caller is. Authorization decides whether that caller is allowed to access a resource or perform an action.

### Q7. Middleware vs endpoint filters?
**Answer:** Middleware applies at the HTTP pipeline level and can affect broad request processing. Endpoint filters wrap selected minimal API handlers and are more endpoint-focused.

### Q8. What happens when middleware throws an exception?
**Answer:** The exception propagates outward through the pipeline until an earlier exception-handling component catches it, provided the response has not already started. A centralized handler can log the error and return a safe response.

## 14. Practical exercise

Build a small pipeline with:
1. A request logging middleware.
2. A middleware that rejects requests missing an `X-Client-Id` header.
3. Authentication and authorization in the correct order (if the sample uses them).
4. A controller or minimal API endpoint.
5. Centralized exception handling.

Verify:
- The log shows inbound and outbound processing.
- A request without the header is short-circuited.
- A valid request reaches the endpoint.
- Exceptions are converted to safe error responses.
- Middleware order is visible in the logs.

## 15. Official references

- [ASP.NET Core middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [Write custom ASP.NET Core middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/write)
- [Middleware ordering](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/#middleware-order)
- [Dependency injection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)
- [Minimal API endpoint filters](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/min-api-filters)


---

# 16. Middleware and Dependency Injection: Lifetime Problem and Solutions

> Each explanation is written in English first, followed immediately by its Arabic explanation.

## 16.1 The scenario

**English:** Imagine an e-commerce API that must write an audit log for every incoming HTTP request. An AuditService performs the database operation, and it depends on AppDbContext.

**العربية:** تخيّل أن عندك API لمتجر إلكتروني، وتريد تسجيل كل طلب HTTP يصل إلى السيرفر داخل قاعدة البيانات. عندك خدمة اسمها AuditService مسؤولة عن الحفظ، وهذه الخدمة تعتمد على AppDbContext.

~~~csharp
public class AuditService
{
    private readonly AppDbContext _db;

    public AuditService(AppDbContext db)
    {
        _db = db;
    }

    public async Task LogAsync(string message)
    {
        _db.AuditLogs.Add(new AuditLog
        {
            Message = message,
            CreatedAt = DateTime.UtcNow
        });

        await _db.SaveChangesAsync();
    }
}
~~~

**English:** Register the DbContext and the audit service as Scoped.

**العربية:** نسجّل DbContext وAuditService بعمر Scoped، بحيث تكون لكل HTTP Request نسخة خدمات خاصة به ضمن Request Scope.

~~~csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

builder.Services.AddScoped<AuditService>();
~~~

## 16.2 The incorrect conventional middleware

**English:** Conventional middleware is normally constructed when the application builds the middleware pipeline. It is not constructed again for every HTTP request. Injecting a scoped service into its constructor creates a lifetime mismatch.

**العربية:** يتم إنشاء Conventional Middleware عادةً عند بناء Middleware Pipeline، وليس من جديد مع كل HTTP Request. لذلك، عندما تحقن خدمة Scoped داخل Constructor الخاص بها، يصبح عندك تعارض بين الأعمار: Middleware طويلة العمر تحتفظ بخدمة المفروض أن تكون مرتبطة بطلب واحد.

~~~csharp
// Incorrect for conventional middleware
public class AuditMiddleware
{
    private readonly RequestDelegate _next;
    private readonly AuditService _auditService;

    public AuditMiddleware(
        RequestDelegate next,
        AuditService auditService)
    {
        _next = next;
        _auditService = auditService;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        await _auditService.LogAsync(
            $"Request: {context.Request.Path}");

        await _next(context);
    }
}
~~~

**English:** The problem is not that middleware cannot use Dependency Injection. The problem is storing a scoped dependency in a middleware instance that lives beyond an individual request.

**العربية:** المشكلة ليست أن Middleware لا تدعم Dependency Injection؛ فهي تدعمها. المشكلة أنك خزّنت خدمة Scoped داخل كائن Middleware عمره أطول من عمر الطلب الواحد.

## 16.3 Understand the lifetime mismatch

**English:** A scoped service is created once per DI scope. In a typical ASP.NET Core web application, each HTTP request has its own request scope. A conventional middleware instance, however, is normally shared by requests.

**العربية:** خدمة Scoped يتم إنشاؤها مرة واحدة داخل كل DI Scope. وفي تطبيق ASP.NET Core المعتاد، كل HTTP Request له Scope خاص فيه. أما نسخة Conventional Middleware فعادةً تكون مشتركة بين الطلبات.

**English:** If the middleware captures a scoped service in its constructor, that service can be retained beyond the request scope for which it was created. This is commonly called a captive dependency and can cause incorrect lifetime behavior and request-state leakage.

**العربية:** إذا احتفظت Middleware بخدمة Scoped داخل Constructor، فقد تبقى هذه الخدمة مرتبطة بكائن أطول عمراً من الـ Scope الذي أُنشئت فيه. تسمى هذه الحالة غالباً Captive Dependency، وقد تسبب استخدام خدمة خارج عمرها الصحيح وتسريب حالة مرتبطة بطلب إلى طلبات أخرى.

**Important clarification — English:** Do not think that ASP.NET Core creates a new conventional middleware instance for every request. The key point is that the middleware instance is normally created as the pipeline is built, while scoped dependencies belong to each request scope.

**توضيح مهم — العربية:** لا تفترض أن ASP.NET Core تنشئ نسخة جديدة من Conventional Middleware مع كل طلب. الفكرة الأساسية أن نسخة Middleware تُنشأ عادةً عند بناء Pipeline، بينما خدمات Scoped تتبع Scope كل Request.

## 16.4 Solution 1: Inject the scoped dependency into InvokeAsync

**English:** Keep dependencies such as RequestDelegate in the conventional middleware constructor. Pass scoped services as parameters to InvokeAsync. ASP.NET Core resolves those parameters from the current request's service scope.

**العربية:** اترك الاعتماديات المناسبة لعمر Middleware، مثل RequestDelegate، داخل Constructor. أما الخدمات Scoped فمرّرها كـ Parameters إلى InvokeAsync. عند تنفيذ الطلب، يقوم ASP.NET Core بحل هذه الخدمات من Scope الخاص بالطلب الحالي.

~~~csharp
public class AuditMiddleware
{
    private readonly RequestDelegate _next;

    public AuditMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(
        HttpContext context,
        AuditService auditService)
    {
        await auditService.LogAsync(
            $"Request: {context.Request.Path}");

        await _next(context);
    }
}
~~~

Register it:

~~~csharp
builder.Services.AddScoped<AuditService>();

var app = builder.Build();

app.UseMiddleware<AuditMiddleware>();

app.MapControllers();

app.Run();
~~~

**English:** The middleware instance can be shared, but the AuditService parameter is resolved for the current request. Request A receives the scoped service from Scope A; Request B receives the scoped service from Scope B.

**العربية:** نسخة Middleware يمكن أن تبقى مشتركة، لكن AuditService التي تصل إلى InvokeAsync يتم حلّها من Scope الطلب الحالي. طلب أحمد يأخذ الخدمة من Scope أحمد، وطلب محمد يأخذ الخدمة من Scope محمد.

### Visualize two requests

~~~text
                 One conventional middleware instance
                              |
                 +------------+------------+
                 |                         |
             Request A                 Request B
                 |                         |
             Scope A                   Scope B
                 |                         |
          AuditService A             AuditService B
                 |                         |
           DbContext A                DbContext B
~~~

**English:** The middleware does not store either request's scoped service in its fields. Each invocation receives the dependency belonging to its own request scope.

**العربية:** Middleware لا تخزّن خدمة أي طلب داخل Fields الخاصة بها. كل استدعاء لـ InvokeAsync يستقبل الخدمة التابعة للـ Scope الخاص بذلك الطلب.

## 16.5 Solution 2: Implement IMiddleware

**English:** Another option is to implement IMiddleware. Unlike conventional middleware activation, IMiddleware instances are activated through the Dependency Injection container. This allows constructor injection of scoped services when the middleware itself is registered with a compatible lifetime, commonly Scoped.

**العربية:** يوجد حل آخر وهو تطبيق الواجهة IMiddleware. بخلاف طريقة إنشاء Conventional Middleware، يتم إنشاء نسخ IMiddleware عن طريق حاوية Dependency Injection. لذلك تستطيع حقن خدمات Scoped داخل Constructor، بشرط تسجيل Middleware نفسها بعمر مناسب، وغالباً Scoped.

~~~csharp
public class AuditMiddleware : IMiddleware
{
    private readonly AuditService _auditService;

    public AuditMiddleware(AuditService auditService)
    {
        _auditService = auditService;
    }

    public async Task InvokeAsync(
        HttpContext context,
        RequestDelegate next)
    {
        await _auditService.LogAsync(
            $"Request: {context.Request.Path}");

        await next(context);
    }
}
~~~

Register both services:

~~~csharp
builder.Services.AddScoped<AuditService>();
builder.Services.AddScoped<AuditMiddleware>();

var app = builder.Build();

app.UseMiddleware<AuditMiddleware>();

app.MapControllers();

app.Run();
~~~

**English:** Because AuditMiddleware is registered as Scoped, the DI container can create an instance for the request scope and inject that scope's AuditService. Do not register this middleware as Singleton while it depends on scoped services.

**العربية:** بما أن AuditMiddleware مسجلة Scoped، يستطيع DI إنشاء نسخة ضمن Scope الطلب وحقن AuditService التابعة لنفس الـ Scope. لا تسجل Middleware كـ Singleton وهي تعتمد على خدمات Scoped.

## 16.6 Compare the two approaches

| Point | Conventional Middleware | IMiddleware |
|---|---|---|
| Activation | Normally created when the pipeline is built | Created through DI |
| Scoped dependency in constructor | Avoid it | Supported with a compatible middleware lifetime |
| Scoped dependency in InvokeAsync | Supported | Supported |
| Register middleware type in DI | Not required for the conventional pattern | Required |
| Typical use | Simple middleware with stable dependencies | Middleware that needs constructor-injected DI services |

**English:** Both approaches support Dependency Injection. Choose conventional middleware with InvokeAsync injection when you want a simple pipeline component. Choose IMiddleware when DI-managed activation and constructor injection are useful.

**العربية:** الطريقتان تدعمان Dependency Injection. استخدم Conventional Middleware مع حقن الخدمات Scoped في InvokeAsync عندما تريد Middleware بسيطة. واستخدم IMiddleware عندما تحتاج أن تتم إدارتها عن طريق DI وأن تحقن اعتمادياتها في Constructor.

## 16.7 The rule to remember

**English:** The issue is not whether middleware supports Dependency Injection. The issue is where the dependency is injected and whether its lifetime is compatible with the middleware instance.

**العربية:** المشكلة ليست هل Middleware تدعم Dependency Injection أم لا. المشكلة هي أين تحقن Dependency، وهل عمرها مناسب لعمر كائن Middleware أم لا.

- **Conventional middleware:** Inject scoped services into InvokeAsync, not its constructor.
  - **العربية:** في Conventional Middleware، احقن خدمات Scoped داخل InvokeAsync وليس Constructor.
- **IMiddleware:** Constructor injection is valid when the middleware is registered with a compatible lifetime, such as Scoped.
  - **العربية:** في IMiddleware، حقن الخدمات داخل Constructor صحيح عندما تسجل Middleware بعمر مناسب مثل Scoped.
- **Never let a long-lived middleware or singleton retain a scoped service beyond its scope.**
  - **العربية:** لا تجعل Middleware طويلة العمر أو Singleton تحتفظ بخدمة Scoped خارج عمر الـ Scope الخاص بها.

## 16.8 Senior interview questions

### Q1. Why should you not inject DbContext into the constructor of conventional middleware?

**Answer (English):** Conventional middleware is normally instantiated when the application builds the pipeline and is reused across requests. DbContext is scoped and should belong to the current request scope. Constructor injection would capture a scoped dependency in a longer-lived middleware instance.

**الشرح (العربية):** لأن Conventional Middleware يتم إنشاؤها عادةً عند بناء Pipeline وتُستخدم مع عدة Requests، بينما DbContext عمرها Scoped وتخص الطلب الحالي. حقنها في Constructor يجعل Middleware تحتفظ باعتمادية Scoped بعمر أطول من عمرها الصحيح.

### Q2. How can conventional middleware use a scoped service?

**Answer (English):** Inject the scoped service as a parameter of InvokeAsync. ASP.NET Core resolves it from the current request's service scope.

**الشرح (العربية):** نمرّر الخدمة Scoped كـ Parameter إلى InvokeAsync. عندها يقوم ASP.NET Core بإحضارها من Scope الطلب الحالي.

### Q3. How does IMiddleware solve the constructor injection problem?

**Answer (English):** IMiddleware is activated by the DI container. When it is registered as Scoped, its scoped dependencies can be injected into its constructor and share the same request scope.

**الشرح (العربية):** يتم إنشاء IMiddleware عن طريق DI. وعندما نسجلها Scoped، يمكن حقن الخدمات Scoped داخل Constructor، وتكون تابعة لنفس Request Scope.

### Q4. What is a captive dependency?

**Answer (English):** A captive dependency occurs when a service with a longer lifetime retains a dependency with a shorter lifetime, causing the shorter-lived dependency to be used beyond its intended scope.

**الشرح (العربية):** تحدث Captive Dependency عندما تحتفظ خدمة طويلة العمر بخدمة أقصر عمراً منها، مما قد يؤدي إلى استخدام الخدمة الأقصر خارج الـ Scope المخصص لها.

## 16.9 Quick self-check

**English:** You have conventional middleware and AppDbContext is registered as Scoped. Where should you inject the DbContext?

**العربية:** عندك Conventional Middleware وAppDbContext مسجلة Scoped. أين يجب أن تحقن DbContext؟

**Correct answer:** As a parameter of InvokeAsync, not in the middleware constructor.

**الإجابة الصحيحة:** كـ Parameter داخل InvokeAsync وليس داخل Constructor الخاص بالـ Middleware.
