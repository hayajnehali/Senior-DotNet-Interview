# Model Binding & Validation in ASP.NET Core Web API

## 1. الفكرة الأساسية

عندما يرسل العميل HTTP Request، تكون البيانات داخل Route أو Query String أو Headers أو Body أو Form. يحتاج ASP.NET Core إلى تحويل هذه البيانات إلى Parameters أو C# Objects، ثم التحقق من صحتها قبل تنفيذ منطق التطبيق.

- **Model Binding:** يقرأ بيانات HTTP ويحوّلها إلى قيم Parameters أو Properties في C#.
- **Model Validation:** يفحص القيم بعد الربط ويتأكد أنها تحقق قواعد التحقق المحددة.
- **ModelState:** يحتفظ بالقيم وأخطاء Model Binding وModel Validation.

مثال:

```http
GET /api/products?page=2&pageSize=20
```

```csharp
public IActionResult Get([FromQuery] int page, [FromQuery] int pageSize)
{
    return Ok(new { page, pageSize });
}
```

يحوّل Model Binding النصوص `"2"` و`"20"` إلى أعداد صحيحة.

## 2. مصادر Model Binding

| Attribute | المصدر | الاستخدام |
|---|---|---|
| `[FromRoute]` | Route | `/api/products/25` |
| `[FromQuery]` | Query String | Pagination, filtering, sorting |
| `[FromBody]` | Request Body | JSON غالباً |
| `[FromForm]` | Form Data | Multipart/form uploads |
| `[FromHeader]` | HTTP Headers | Correlation ID أو Header مخصص |
| `[FromServices]` | Dependency Injection | الحصول على Service وليس بيانات العميل |

مثال:

```csharp
[HttpGet("{id:guid}")]
public IActionResult Get(
    [FromRoute] Guid id,
    [FromQuery] bool includeDetails,
    [FromHeader(Name = "X-Correlation-ID")] string? correlationId)
{
    return Ok();
}
```

**مهم:** لا تستخدم أكثر من Parameter واحد يعتمد على `[FromBody]` في Action عادي؛ فالـ Request Body يُقرأ بواسطة Input Formatter، وليس مصدراً مستقلاً لكل Parameter. اجمع البيانات في Request DTO واحد.

## 3. كيف تتم العملية؟

بصورة مبسطة:

1. يصل HTTP Request إلى ASP.NET Core.
2. يتم تحديد Endpoint.
3. يحدد Framework مصدر كل Parameter وفق Attributes وقواعد الاستنتاج.
4. يقرأ البيانات ويحوّلها إلى النوع المطلوب (Binding/Deserialization).
5. تُنفّذ قواعد Validation.
6. إذا كانت البيانات صالحة، يبدأ تنفيذ Controller Action.
7. إذا كان Controller يستخدم `[ApiController]` ووجد ModelState أخطاء، يرجع Framework استجابة 400 تلقائياً قبل تنفيذ Action، ما دام السلوك الافتراضي مفعّلاً.

### الفرق بين Binding وValidation

| الحالة | Model Binding | Model Validation |
|---|---|---|
| `price = 100` | ينجح | ينجح إذا وافق القواعد |
| `price = -100` | ينجح | يفشل مع `[Range]` المناسب |
| `price = "abc"` إلى `decimal` | يفشل التحويل | يُسجل خطأ في ModelState |
| JSON غير صالح نحوياً | يفشل في القراءة/Deserialization | لا يصل إلى Validation الطبيعي للنموذج |

## 4. ApiController وModelState

`[ApiController]` يفعّل سلوكيات خاصة بـ Web APIs، منها Automatic HTTP 400 Responses عند وجود أخطاء ModelState. يتم ذلك افتراضياً عبر `ModelStateInvalidFilter`.

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateProductRequest request)
    {
        // لا يصل التنفيذ إلى هنا إذا فشل ModelState
        return Ok();
    }
}
```

لذلك لا تحتاج عادةً إلى تكرار:

```csharp
if (!ModelState.IsValid)
    return BadRequest(ModelState);
```

يمكن تخصيص الاستجابة عبر `InvalidModelStateResponseFactory` عند الحاجة إلى توحيد API error contract.

## 5. Data Annotations

تستخدم لقواعد التحقق البسيطة على Properties:

| Attribute | الوظيفة |
|---|---|
| `[Required]` | القيمة مطلوبة |
| `[StringLength]` | أقل/أكبر طول للنص |
| `[MinLength]`, `[MaxLength]` | حدود الطول |
| `[Range]` | مجال القيم |
| `[EmailAddress]` | فحص صيغة البريد |
| `[RegularExpression]` | فحص نمط Regex |
| `[Compare]` | مقارنة Property بأخرى |

مثال:

```csharp
public sealed class CreateProductRequest
{
    [Required]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; init; } = string.Empty;

    [Required]
    [RegularExpression(@"^[A-Z0-9-]{3,20}$")]
    public string Sku { get; init; } = string.Empty;

    [Range(typeof(decimal), "0.01", "999999.99")]
    public decimal Price { get; init; }

    [Range(0, 1000000)]
    public int InitialStock { get; init; }
}
```

### Nullable Reference Types وملاحظة Value Types

في MVC، الـ Non-Nullable Reference Properties تُعامل افتراضياً كأن عليها Required ضمنياً، مع تفاصيل مرتبطة بإعدادات MVC وNullable Context.

أما `int` و`decimal` و`Guid` فهي Value Types ولها قيم افتراضية. إذا كان يجب التمييز بين عدم إرسال القيمة وإرسال صفر، استخدم Nullable Value Type:

```csharp
[Required]
[Range(1, 100)]
public int? Quantity { get; init; }
```

## 6. IValidatableObject وCustom Attributes

### IValidatableObject

يُستخدم عندما تعتمد القاعدة على أكثر من Property، وتكون قابلة للتنفيذ بشكل متزامن:

```csharp
public sealed class CreateProductRequest : IValidatableObject
{
    public bool TrackInventory { get; init; }
    public int InitialStock { get; init; }

    public IEnumerable<ValidationResult> Validate(
        ValidationContext validationContext)
    {
        if (!TrackInventory && InitialStock > 0)
        {
            yield return new ValidationResult(
                "Initial stock must be zero when tracking is disabled.",
                new[] { nameof(InitialStock) });
        }
    }
}
```

لا تضع Database Queries أو عمليات خارجية Async هنا؛ فـ `IValidatableObject.Validate` متزامنة.

### Custom Validation Attribute

مفيد لقاعدة صيغة قابلة لإعادة الاستخدام، مثل تنسيق SKU. لا تجعله مسؤولاً عن الوصول إلى قاعدة البيانات أو التحقق من قواعد أعمال تعتمد على خدمات خارجية.

## 7. Model Validation vs Business Validation

هذه نقطة معمارية مهمة:

| Model/Input Validation | Business Validation |
|---|---|
| يفحص شكل وصحة المدخلات | يفحص قواعد العمل وحالة النظام |
| Required, Range, StringLength | SKU غير مكرر، المنتج موجود، الرصيد كافٍ |
| غالباً متزامن ولا يحتاج DB | قد يحتاج DB أو خدمة أخرى |
| Request DTO / Validation | Application أو Domain حسب طبيعة القاعدة |

مثال: `[EmailAddress]` يفحص شكل البريد، لكنه لا يثبت أن البريد غير مستخدم. فحص وجود البريد يتم في Application Service.

### منع التكرار بشكل صحيح

فحص التطبيق وحده لا يمنع Race Condition:

1. طلبان يفحصان SKU في الوقت نفسه.
2. كلاهما يجد أن SKU غير موجود.
3. كلاهما يحاول الحفظ.

لذلك استخدم Application check لتحسين رسالة الخطأ، وUnique Index في قاعدة البيانات لضمان القيد:

```csharp
modelBuilder.Entity<Product>()
    .HasIndex(x => x.Sku)
    .IsUnique();
```

ويجب معالجة تعارض Unique Constraint أثناء الحفظ وإرجاع استجابة مناسبة، مثل 409 Conflict.

## 8. مثال API متكامل مختصر

### Request وResponse DTOs

```csharp
public sealed class CreateProductRequest
{
    [Required]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; init; } = string.Empty;

    [Required]
    [RegularExpression(@"^[A-Z0-9-]{3,20}$")]
    public string Sku { get; init; } = string.Empty;

    [Range(typeof(decimal), "0.01", "999999.99")]
    public decimal Price { get; init; }
}

public sealed record ProductResponse(
    Guid Id,
    string Name,
    string Sku,
    decimal Price);
```

### Controller

```csharp
[ApiController]
[Route("api/products")]
public sealed class ProductsController : ControllerBase
{
    private readonly IProductService _service;

    public ProductsController(IProductService service)
        => _service = service;

    [HttpPost]
    public async Task<ActionResult<ProductResponse>> Create(
        CreateProductRequest request,
        CancellationToken cancellationToken)
    {
        var result = await _service.CreateAsync(
            request, cancellationToken);

        if (result.SkuAlreadyExists)
            return Conflict(new ProblemDetails
            {
                Title = "SKU already exists.",
                Status = StatusCodes.Status409Conflict
            });

        return CreatedAtAction(
            nameof(GetById),
            new { id = result.Product!.Id },
            result.Product);
    }

    [HttpGet("{id:guid}")]
    public async Task<ActionResult<ProductResponse>> GetById(
        Guid id,
        CancellationToken cancellationToken)
    {
        var product = await _service.GetByIdAsync(
            id, cancellationToken);

        return product is null ? NotFound() : Ok(product);
    }
}
```

التدفق: Binding/Deserialization → Validation → Action → Application Service → Business Rules/Database → HTTP Response.

## 9. HTTP Status Codes

| Status | الاستخدام |
|---|---|
| 400 Bad Request | الطلب غير صالح أو فشل Binding/Validation |
| 409 Conflict | تعارض مع الحالة الحالية، مثل SKU مكرر |
| 422 Unprocessable Content | خيار لعقد API يميز محتوى مفهوم الصيغة لكنه غير قابل للمعالجة وفق قواعد محددة |

مع `[ApiController]`، الاستجابة الافتراضية لأخطاء ModelState هي 400. لا تغيّرها إلى 422 إلا إذا كان ذلك جزءاً مقصوداً ومتسقاً من API contract.

## 10. أخطاء شائعة وممارسات Senior

- لا تستقبل EF Entity مباشرةً من العميل؛ استخدم Request DTO لتجنب Overposting وفصل API Contract عن Persistence Model.
- لا تعتمد على Angular/Frontend Validation؛ يمكن تجاوز الواجهة وإرسال HTTP Request مباشرة.
- لا تضع Database Queries داخل Data Annotation أو `IValidatableObject`.
- لا تكرر `ModelState.IsValid` عندما يكون Automatic 400 مفعّلاً.
- لا تفترض أن `[Required]` على `int` يميز بين عدم الإرسال وإرسال صفر؛ استخدم `int?` عند الحاجة.
- لا تكشف Stack Traces أو SQL أو تفاصيل داخلية في الاستجابة.
- افصل Validation عن Authorization؛ صحة البيانات لا تعني أن المستخدم مخوّل.
- ضع قيوداً على أطوال النصوص وحجم الطلبات والملفات.
- اختبر HTTP behavior باستخدام Integration Tests، وليس Validators فقط.
- اجعل API error response contract متسقاً، واستخدم Trace ID لتسهيل التتبع.

## 11. Testing

اختبر على الأقل:
1. بيانات صحيحة تعيد 201 Created.
2. سعر سالب يعيد 400 Bad Request.
3. SKU مكرر يعيد 409 Conflict.
4. فشل Validation يمنع استدعاء Application Service.
5. شكل `ValidationProblemDetails` متوافق مع العقد.

مثال Integration Test (يفترض وجود `_client` وتهيئة اختبار التطبيق):

```csharp
[Fact]
public async Task CreateProduct_WithInvalidPrice_Returns400()
{
    var request = new
    {
        name = "Laptop",
        sku = "LAP-100",
        price = -100
    };

    var response = await _client.PostAsJsonAsync(
        "/api/products", request);

    Assert.Equal(
        HttpStatusCode.BadRequest,
        response.StatusCode);
}
```

## 12. .NET 10: Minimal API Validation

في ASP.NET Core 10 أضيف دعم Validation مدمج لـ Minimal APIs عبر حزمة `Microsoft.Extensions.Validation`:

```csharp
builder.Services.AddValidation();
```

ثم يمكن استخدام Data Annotations على Request Model:

```csharp
public record CreateProductRequest(
    [Required, StringLength(100, MinimumLength = 3)]
    string Name,
    [Range(0.01, 999999.99)]
    decimal Price);

app.MapPost("/api/products", (CreateProductRequest request) =>
    Results.Ok(request));
```

لا تفترض وجود نفس الدعم المدمج في Minimal APIs في .NET 8 و.NET 9. هذه الميزة تخص Minimal APIs؛ أما Controllers فكان لديها Model Binding وModel Validation قبل .NET 10. راجع توثيق الإصدار المستهدف عند توزيع DTOs على Assemblies أو تخصيص Validation.

## 13. Senior Interview Q&A

### Q1. What is the difference between Model Binding and Model Validation?

**Answer:** Model Binding maps incoming HTTP request data to action parameters or model properties. Model Validation checks whether the bound values satisfy the defined validation rules.

**الشرح:** Binding يحوّل بيانات HTTP إلى C# values/objects. Validation يفحص القيم بعد الربط.

### Q2. What happens when validation fails in a controller decorated with ApiController?

**Answer:** ASP.NET Core automatically returns a 400 Bad Request response when ModelState contains errors, provided the automatic invalid-model-state behavior is enabled. The action is not executed.

**الشرح:** يتم فحص ModelState قبل Action، وعند وجود أخطاء يرجع Framework 400 تلقائياً.

### Q3. What is ModelState?

**Answer:** ModelState stores attempted values and model binding or validation errors.

**الشرح:** يحتفظ بالقيم وأخطاء التحويل والتحقق.

### Q4. When would you use IValidatableObject?

**Answer:** I use it for synchronous validation rules that depend on multiple properties of the same model. I use an application service for asynchronous or database-dependent rules.

**الشرح:** استخدمه لقواعد متزامنة تربط عدة خصائص، وليس لاستعلامات DB أو عمليات Async.

### Q5. Should database-dependent validation be implemented inside a ValidationAttribute?

**Answer:** Generally, no. It introduces infrastructure concerns into input validation. I handle it in the application layer and enforce critical invariants with database constraints.

**الشرح:** افصل التحقق المعتمد على DB في Application Service، وضع القيود الحرجة في قاعدة البيانات أيضاً.

### Q6. Why use DTOs instead of binding EF Core entities directly?

**Answer:** DTOs define an explicit API contract, prevent overposting, and decouple external requests from persistence models.

**الشرح:** DTO يحدد الحقول المسموح للعميل إرسالها ويحمي خصائص Entity الداخلية.

### Q7. What is the difference between HTTP 400 and 409?

**Answer:** 400 indicates an invalid or malformed request. 409 indicates a conflict with the current state of the resource.

**الشرح:** 400 لبيانات/طلب غير صالح، و409 لتعارض مثل SKU موجود مسبقاً.

### Q8. Does client-side validation replace server-side validation?

**Answer:** No. Client-side validation improves user experience, but server-side validation is mandatory because clients can bypass the frontend.

**الشرح:** تحقق Angular لتحسين التجربة فقط؛ السيرفر يجب أن يتحقق دائماً.

### Q9. What is the difference between FromBody and FromQuery?

**Answer:** FromBody reads and deserializes the HTTP request body using an input formatter. FromQuery binds values from the URL query string.

**الشرح:** الأول يقرأ JSON أو Body، والثاني يقرأ Query String مثل `?page=1`.

### Q10. What changed for Minimal API validation in .NET 10?

**Answer:** .NET 10 introduced built-in validation support for Minimal APIs through Microsoft.Extensions.Validation. Calling AddValidation enables validation of supported request models before the handler executes.

**الشرح:** .NET 10 أضاف دعماً مدمجاً لـ Minimal API Validation عبر `AddValidation()`؛ لا تفترض توفر السلوك نفسه في .NET 8 و9.

## 14. Practical Exercise

أنشئ `CreateCustomerRequest` للحقول التالية:
- `FullName`: Required، من 3 إلى 100 حرف.
- `Email`: Required وEmailAddress.
- `Age`: من 18 إلى 100.
- `Password`: Required وطوله 8 على الأقل.
- `ConfirmPassword`: يجب أن يطابق Password باستخدام `[Compare]`.

استخدم DTO وليس Customer Entity. اجعل Controller يستخدم `[ApiController]`. فكّر أين تتحقق من أن البريد غير مستخدم، وأين تنشئ الحساب وتخزن كلمة المرور بشكل آمن.

### Suggested Solution

```csharp
public sealed class CreateCustomerRequest
{
    [Required]
    [StringLength(100, MinimumLength = 3)]
    public string FullName { get; init; } = string.Empty;

    [Required]
    [EmailAddress]
    public string Email { get; init; } = string.Empty;

    [Range(18, 100)]
    public int Age { get; init; }

    [Required]
    [MinLength(8)]
    public string Password { get; init; } = string.Empty;

    [Required]
    [Compare(nameof(Password))]
    public string ConfirmPassword { get; init; } = string.Empty;
}
```

- فحص صيغة البريد يتم عبر `[EmailAddress]`.
- فحص أن البريد غير مستخدم يتم في Application Service، مع Unique Index إذا كان البريد يجب أن يكون فريداً.
- إنشاء الحساب وتجزئة كلمة المرور يتمان عبر ASP.NET Core Identity أو Password Hasher مناسب؛ لا تخزن كلمة المرور كنص صريح.
- لا حاجة لفحص `ModelState.IsValid` يدوياً ما دام `[ApiController]` والسلوك التلقائي مفعّلين.

## 15. Official References

- [Model Binding in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/model-binding)
- [Model Validation in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/validation)
- [Create web APIs with ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/web-api/)
- [Minimal API validation in ASP.NET Core 10](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/parameter-binding#validation)
- [What's new in ASP.NET Core 10](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-10.0)
