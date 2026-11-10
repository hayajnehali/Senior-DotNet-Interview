# Thread, Task, Async/Await & State Machine

> هدف الملف: فهم ما يحدث فعلياً داخل .NET عند تنفيذ عملية غير متزامنة، خصوصاً في ASP.NET Core وEF Core، مع أمثلة مناسبة لمقابلات Senior .NET.
>
> **ملاحظة:** الشرح الداخلي مبسّط لتوضيح الفكرة. الكود الذي يولّده C# compiler قد يختلف في تفاصيله بين إصدارات .NET، لذلك لا تعتبر أسماء الحقول أو أرقام الحالات أدناه عقداً ثابتاً.



## Definitions | التعريفات (English first, then Arabic)

### Thread
**English:** A thread is an execution path within a process that the operating system schedules to run instructions on a CPU core.

**العربي:** الـ Thread هو مسار تنفيذ داخل البرنامج، يقوم نظام التشغيل بجدولته لتنفيذ التعليمات على نواة المعالج.

### ThreadPool
**English:** The .NET ThreadPool is a managed collection of reusable worker threads used to execute queued work without creating a new thread for every operation.

**العربي:** الـ ThreadPool هي مجموعة Threads يديرها .NET ويعيد استخدامها لتنفيذ الأعمال المجدولة، بدلاً من إنشاء Thread جديد لكل عملية.

### Task
**English:** A Task represents an asynchronous operation and its completion state. A Task<T> can also provide a result when the operation completes.

**العربي:** الـ Task تمثل عملية غير متزامنة وحالة اكتمالها. أما Task<T> فتحمل أيضاً النتيجة التي تنتج عند اكتمال العملية.

### async
**English:** The async modifier enables the await keyword in a method and allows the compiler to transform the method into an asynchronous state machine.

**العربي:** الكلمة async تسمح باستخدام await داخل الدالة، وتجعل المترجم يحول الدالة إلى State Machine تدير تنفيذها غير المتزامن. وهي بحد ذاتها لا تنشئ Thread جديداً.

### await
**English:** The await operator checks whether an awaitable operation has completed. If it has not, it suspends the method's continuation and arranges for it to resume when the operation completes.

**العربي:** العامل await يفحص هل العملية المنتظرة اكتملت. إذا لم تكتمل، يعلّق استمرار الدالة ويسجل استئنافها عند اكتمال العملية، دون حجز Thread لمجرد انتظار I/O غير المتزامن.

### State Machine
**English:** A compiler-generated structure that tracks an async method's execution state, preserves the values needed across await points, and resumes the method when awaited operations complete.

**العربي:** بنية يولّدها المترجم لتتبع مرحلة تنفيذ دالة async، وحفظ القيم التي تحتاجها بعد await، ثم استكمال الدالة عند اكتمال العمليات المنتظرة.

### Awaiter
**English:** An awaiter exposes the completion status of an awaitable operation, provides its result, and participates in registering the continuation.

**العربي:** الـ Awaiter يتيح فحص اكتمال العملية، والحصول على نتيجتها أو الخطأ، والمشاركة في تسجيل الكود الذي سيكمل التنفيذ.

### Continuation
**English:** A continuation is the work scheduled to run after an asynchronous operation completes, allowing the suspended method to continue.

**العربي:** الـ Continuation هو الجزء الذي تتم جدولته للتنفيذ بعد اكتمال العملية غير المتزامنة، حتى تتابع الدالة من المكان الذي توقفت عنده.

### AsyncTaskMethodBuilder
**English:** AsyncTaskMethodBuilder is a compiler-facing helper that manages the Task returned by an async method and completes it with a result, exception, or cancellation.

**العربي:** AsyncTaskMethodBuilder هو مكوّن يستخدمه الكود الذي يولّده المترجم لإدارة الـ Task التي ترجعها دالة async، وإكمالها بنتيجة أو خطأ أو إلغاء.

### Blocking
**English:** Blocking occurs when a thread is forced to wait and cannot perform other work until the awaited operation finishes.

**العربي:** يحدث Blocking عندما يبقى الـ Thread منتظراً ولا يستطيع تنفيذ عمل آخر حتى تنتهي العملية.

### I/O-bound operation
**English:** An I/O-bound operation spends most of its time waiting for an external resource, such as a database, network service, or file system.

**العربي:** عملية تعتمد على الإدخال والإخراج، وتقضي معظم وقتها بانتظار مورد خارجي مثل قاعدة البيانات أو الشبكة أو الملفات.

### CPU-bound operation
**English:** A CPU-bound operation spends most of its time performing calculations and consuming processor time.

**العربي:** عملية تعتمد على المعالج، وتقضي معظم وقتها في تنفيذ الحسابات واستهلاك وقت الـ CPU.

### ThreadPool starvation
**English:** ThreadPool starvation occurs when queued work cannot obtain worker threads promptly, often because existing pool threads are blocked for long periods.

**العربي:** يحدث ThreadPool starvation عندما لا تجد الأعمال الجديدة Threads متاحة بسرعة، وغالباً بسبب انشغال Threads الموجودة لفترات طويلة بعمليات Blocking.

## 1. Thread

### التعريف | Definition

**Thread:** مسار تنفيذ داخل العملية (Process)، ينفّذ التعليمات ويُجدول بواسطة نظام التشغيل على نواة المعالج (CPU Core).

**بالعربي:** الثريد هو المسار الذي ينفّذ الكود فعلياً. لكل Thread مكدس تنفيذ (Stack) وحالة تنفيذ خاصة به. يستطيع المعالج التبديل بين عدة Threads، ويمكن لعدة Threads العمل بالتوازي عندما تتوفر عدة أنوية.

### نقاط مهمة

- الـ Thread مورد تنفيذي، وليس نتيجة عملية.
- إنشاء Thread جديد له تكلفة؛ لذلك لا يُنشأ Thread جديد لكل طلب HTTP.
- في ASP.NET Core تُستخدم Threads من ThreadPool لمعالجة العمل.
- عند حجز Thread بانتظار I/O بشكل متزامن، لا يستطيع هذا الـ Thread خدمة عمل آخر خلال فترة الانتظار.

## 2. ThreadPool

**ThreadPool:** مجموعة Threads يديرها .NET ويعيد استخدامها لتنفيذ الأعمال المجدولة، بدلاً من إنشاء Thread جديد لكل عملية صغيرة.

**بالعربي:** عندما يحتاج التطبيق إلى تنفيذ عمل، يمكنه استخدام Thread متاح من ThreadPool. وعندما ينتهي العمل يعود Thread إلى المجموعة ليُستخدم مرة أخرى.

**في ASP.NET Core:** الطلبات تُعالج باستخدام ThreadPool. عند استخدام I/O غير متزامن بشكل صحيح، لا يبقى Thread محجوزاً طوال انتظار قاعدة البيانات أو HTTP.

## 3. Task

### التعريف | Definition

**Task:** كائن يمثل عملية (Operation) غير متزامنة وحالتها، ويمكن أن يمثل اكتمالها أو فشلها أو إلغاءها، كما يمكن أن يحمل نتيجة.

**بالعربي:** الـ Task لا تعني Thread. هي تمثيل للعملية التي ننتظر نتيجتها أو اكتمالها.

- `Task`: عملية لا تعيد قيمة للمستدعي.
- `Task<T>`: عملية تعيد نتيجة من النوع `T` عند اكتمالها.
- يمكن أن تمثل Task عملية I/O لا تحتاج إلى Thread طوال فترة الانتظار.
- `Task.Run` يجدول عملاً على ThreadPool؛ لكنه ليس مطلوباً عادةً لعمليات I/O التي توفر API غير متزامنة أصلاً.

### مثال

```csharp
Task<List<Student>> task = GetStudentsAsync();
```

هذا المتغير يمثل عملية جلب الطلاب. عند اكتمالها بنجاح، تكون النتيجة قائمة من الطلاب ويمكن الحصول عليها عبر `await` أو `Result` (مع تجنب Result في المسارات غير المتزامنة).

### حالات Task الشائعة

- `RanToCompletion`: اكتملت بنجاح.
- `Faulted`: انتهت بخطأ.
- `Canceled`: أُلغيت.

هذه أمثلة على الحالات النهائية؛ توجد أيضاً حالات غير نهائية مثل `WaitingForActivation` و`Running` وغيرها بحسب نوع العملية وطريقة إنشائها.

## 4. async وawait

### async

**async:** كلمة تسمح باستخدام `await` داخل الدالة، ويحوّل المترجم الدالة إلى بنية State Machine لإدارة الاستكمال.

**بالعربي:** `async` لا تنشئ Thread بحد ذاتها، ولا تجعل كل الكود داخل الدالة يعمل بالتوازي.

### await

**await:** ينتظر اكتمال Awaitable، وإذا لم يكن مكتملاً يعلّق استمرار الدالة ويسجل ما يلزم لاستئنافها لاحقاً، دون حجز Thread لمجرد الانتظار غير المتزامن.

**بالعربي:** عند `await` يفحص التنفيذ هل العملية انتهت:
1. إذا انتهت، يستخرج النتيجة ويكمل مباشرة.
2. إذا لم تنتهِ، يحفظ حالة الدالة ويسجل الاستكمال، ثم يعيد التحكم إلى المستدعي.
3. عند اكتمال العملية، يُجدول الاستكمال لتكمل الدالة من بعد `await`.

> `await` لا يحوّل عملية متزامنة إلى غير متزامنة. يجب استخدام API غير متزامنة فعلاً مثل `ToListAsync` أو `HttpClient.GetAsync`.

## 5. State Machine

### التعريف | Definition

**State Machine:** بنية يولّدها C# compiler لإدارة مراحل تنفيذ دالة `async`، وحفظ الحالة اللازمة لاستكمالها بعد `await`.

**بالعربي:** هي التي تتذكر أين وصلت الدالة، وما المتغيرات التي تحتاجها بعد الانتظار، وكيف تستأنف تنفيذ الكود عند اكتمال العملية.

### مثال الدالة الأصلية

```csharp
public async Task<int> GetStudentCountAsync()
{
    int departmentId = 5;

    var students = await _context.Students
        .Where(s => s.DepartmentId == departmentId)
        .ToListAsync();

    int count = students.Count;
    return count;
}
```

### ما الذي تحتاج State Machine إلى الاحتفاظ به؟

عندما تكون العملية عند `await` غير مكتملة، قد تحتاج البنية المولّدة إلى الاحتفاظ بمعلومات مثل:

- **State:** المرحلة التي وصلت إليها الدالة.
- **DepartmentId:** المتغير المحلي إذا كان مطلوباً بعد `await`.
- **Awaiter:** لمتابعة الـ awaitable والحصول على نتيجته أو إعادة رمي الخطأ.
- **Builder:** لإدارة الـ Task الخارجية التي ترجعها الدالة غير المتزامنة.

ليس كل متغير محلي يُحفظ؛ المتغيرات التي لا تحتاجها الدالة بعد نقطة التعليق لا يلزم عادةً نقلها إلى حقول الحالة.

### نموذج تعليمي مبسّط جداً

الكود التالي يشرح الفكرة فقط، وليس الكود الحرفي الذي يولّده المترجم:

```csharp
// Conceptual illustration only
struct GetStudentCountStateMachine
{
    public int State;
    public int DepartmentId;
    public TaskAwaiter<List<Student>> Awaiter;
    public AsyncTaskMethodBuilder<int> Builder;

    public void MoveNext()
    {
        // The compiler-generated implementation
        // checks the state, resumes after await,
        // obtains the result, and completes the builder.
    }
}
```

### ماذا يحدث عند await خطوة بخطوة؟

1. تبدأ الدالة وتكون قيمة `departmentId` هي 5.
2. يبدأ EF Core/مزود قاعدة البيانات عملية الاستعلام، وتُرجع `ToListAsync` كائناً قابلاً للانتظار (Task).
3. يفحص Awaiter هل العملية اكتملت.
4. إذا لم تكتمل، تحفظ State Machine الحالة والمتغيرات اللازمة والـ Awaiter.
5. يسجّل المترجم الاستكمال (Continuation)، أي العودة إلى تنفيذ منطق State Machine عند اكتمال العملية.
6. تنتهي الدالة مؤقتاً من جهة المستدعي وتُعاد الـ Task الخارجية. لا يبقى Thread محجوزاً لمجرد انتظار I/O.
7. عندما تكتمل عملية قاعدة البيانات، تُجدول متابعة State Machine، وتُستدعى آلية الاستكمال مثل `MoveNext`.
8. تحصل الدالة على قائمة الطلاب، وتحسب العدد، ثم يكمل الـ Builder الـ Task الخارجية بنتيجة العدد.

**مهم:** State Machine ليست Thread، ولا تنشئ Thread مخصصاً. كما أن الاستكمال قد يعمل على Thread مختلف عن Thread الذي بدأ الدالة. في ASP.NET Core لا يوجد عادةً SynchronizationContext خاص بالطلب، لكن لا تعتمد على هوية Thread بعينها.

## 6. الفرق بين Task الداخلية والخارجية

في المثال:

```csharp
public async Task<int> GetCountAsync()
{
    var students = await GetStudentsAsync();
    return students.Count;
}
```

هناك عمليتان ممثلتان:

- **Task الداخلية:** التي ترجعها `GetStudentsAsync()`، وتمثل جلب قائمة الطلاب.
- **Task الخارجية:** التي ترجعها `GetCountAsync()`، وتمثل تنفيذ الدالة كاملة وإرجاع العدد.

عندما تكتمل Task الداخلية، لا يعني ذلك أن Task الخارجية اكتملت فوراً؛ فما زال على الدالة حساب `students.Count` ثم إرجاعه. بعد انتهاء الدالة بنجاح، تكتمل Task الخارجية وتحمل العدد.

## 7. سيناريو ASP.NET Core + EF Core

```csharp
[HttpGet("students/count")]
public async Task<IActionResult> GetStudentsCount()
{
    var count = await _context.Students.CountAsync();
    return Ok(count);
}
```

التسلسل:

1. يصل HTTP request ويبدأ تنفيذ الـ Controller على Thread من ThreadPool.
2. يبدأ EF Core ومزود SQL عملية قاعدة البيانات غير المتزامنة.
3. ترجع العملية Task.
4. إذا لم تكتمل Task، تحفظ State Machine ما يلزم وتعيد الدالة التحكم؛ فيصبح Thread متاحاً لعمل آخر.
5. عند اكتمال استعلام SQL، تُجدول متابعة الدالة.
6. تستخرج الدالة العدد وتُرجع HTTP response.

> استخدم `CountAsync()` عندما تحتاج العدد فقط؛ لا تجلب كل الصفوف بـ `ToListAsync()` ثم تحسب `Count` دون حاجة.

## 8. Blocking vs Non-blocking

### Blocking

```csharp
var students = _context.Students.ToList();
```

**بالعربي:** استدعاء متزامن؛ Thread الذي ينفذ الاستدعاء يبقى مشغولاً أثناء انتظار نتيجة قاعدة البيانات.

### Non-blocking asynchronous I/O

```csharp
var students = await _context.Students.ToListAsync();
```

**بالعربي:** عند انتظار I/O غير المكتمل، لا يبقى Thread محجوزاً لمجرد الانتظار؛ يمكنه تنفيذ عمل آخر، ثم تستأنف الدالة لاحقاً.

### تجنب Sync over Async

```csharp
var students = GetStudentsAsync().Result; // Avoid
GetStudentsAsync().Wait();                 // Avoid
```

**بالعربي:** `Result` و`Wait()` يحجبان Thread حتى تكتمل العملية، وقد يسببان ThreadPool starvation، كما قد يؤدي الحجب إلى deadlocks في بيئات تملك SynchronizationContext مناسباً لذلك.

## 9. CPU-bound vs I/O-bound

| النوع | المعنى | مثال | النهج المعتاد |
|---|---|---|---|
| I/O-bound | انتظار مورد خارجي | SQL, HTTP, file I/O | استخدم API غير متزامنة مباشرة |
| CPU-bound | الحساب يستهلك المعالج | ضغط ملف، معالجة صور، حساب ثقيل | نفّذ العمل على Thread مناسب؛ وقد يفيد Task.Run في تطبيقات العميل أو عند الحاجة المدروسة |

### مثال صحيح لقاعدة البيانات

```csharp
var students = await _context.Students.ToListAsync();
```

### مثال غير مناسب عادةً

```csharp
await Task.Run(() => _context.Students.ToList());
```

**لماذا؟** لأن هذا لا يحوّل استعلام SQL المتزامن إلى I/O غير متزامن؛ بل يحجز ThreadPool Thread لتنفيذ الاستدعاء المتزامن. استخدم `ToListAsync` بدلاً منه.

## 10. Task.Run وTask.WhenAll

### Task.Run

```csharp
int result = await Task.Run(() => CalculateReport());
```

**بالعربي:** يجدول عملية حسابية على ThreadPool ثم ينتظر نتيجتها. لا تستخدمه تلقائياً حول استدعاءات EF Core أو HttpClient غير المتزامنة.

### Task.WhenAll

```csharp
Task<Customer> customerTask = customerService.GetAsync(id);
Task<List<Order>> ordersTask = orderService.GetByCustomerAsync(id);

await Task.WhenAll(customerTask, ordersTask);

var customer = await customerTask;
var orders = await ordersTask;
```

**بالعربي:** يبدأ العمليتين المستقلتين ثم ينتظر اكتمالهما معاً. `WhenAll` لا ينشئ Threads بذاته. لا تنفّذ عمليات EF Core متزامنة بالتوازي على نفس `DbContext`؛ فهو غير مصمم للاستخدام المتزامن.

## 11. ThreadPool Starvation

**ThreadPool Starvation:** حالة لا تتوفر فيها Threads كافية من ThreadPool بسرعة لتلبية الأعمال الجديدة، غالباً بسبب حجب عدد كبير من Threads.

**سيناريو:** عدد كبير من طلبات API يستخدم `.Result` أو استدعاءات I/O متزامنة. تبقى Threads محجوزة وهي تنتظر، فتتأخر معالجة طلبات أخرى وتزداد أوقات الاستجابة.

**المعالجة:** استخدم async end-to-end، وتجنب الحجب، ولا تستخدم `Task.Run` كحل شكلي لاستدعاء API متزامنة.

## 12. مقارنة سريعة للمقابلات

| المفهوم | مسؤوليته |
|---|---|
| Thread | مسار تنفيذ فعلي |
| ThreadPool | إدارة وإعادة استخدام Threads |
| Task | تمثيل عملية وحالتها ونتيجتها |
| async | تمكين await وتحويل الدالة إلى نمط async |
| await | انتظار غير حاجب عند عدم اكتمال العملية |
| State Machine | حفظ حالة الدالة وإدارة استكمالها |
| Awaiter | فحص الاكتمال والحصول على النتيجة وتسجيل الاستمرار |
| Continuation | الكود الذي يُنفّذ لاستكمال الدالة |
| Builder | إدارة وإكمال الـ Task التي ترجعها دالة async |

## 13. أسئلة مقابلات Senior .NET

### Q1: What is the difference between a Thread and a Task?

**Answer:** A Thread is an execution path managed by the operating system. A Task represents an asynchronous operation and its completion; it does not necessarily own a Thread.

**بالعربي:** الـ Thread ينفذ التعليمات فعلياً، أما Task فتمثل العملية ونتيجتها أو اكتمالها، وقد تنتظر I/O دون امتلاك Thread طوال الانتظار.

### Q2: Does async create a new thread?

**Answer:** No. The async keyword does not create a new thread. It enables await and causes the compiler to generate a state machine.

**بالعربي:** async لا تنشئ Thread؛ المترجم يولّد State Machine لإدارة الاستكمال.

### Q3: What happens when execution reaches an incomplete await?

**Answer:** The method stores the state required for resumption, registers a continuation, and returns control to its caller. When the awaited operation completes, the continuation resumes the method.

**بالعربي:** تحفظ الدالة حالتها، وتسجل الاستكمال، ثم تعيد التحكم للمستدعي. بعد اكتمال العملية تستأنف التنفيذ.

### Q4: What is the role of the async state machine?

**Answer:** It tracks the method's execution state, preserves locals needed across suspension points, and resumes execution after awaited operations complete.

**بالعربي:** تحفظ مكان توقف الدالة والمتغيرات اللازمة وتستكمل التنفيذ بعد انتهاء العمليات المنتظرة.

### Q5: Does every Task require a Thread?

**Answer:** No. A Task can represent asynchronous I/O that completes without occupying a thread while waiting. CPU-bound work scheduled with Task.Run typically uses a ThreadPool thread.

**بالعربي:** لا. عمليات I/O غير المتزامنة لا تحتاج Thread محجوزاً طوال الانتظار، بينما الأعمال الحسابية المجدولة بـ Task.Run تستخدم عادةً ThreadPool.

### Q6: Why should we avoid .Result and .Wait() in ASP.NET Core?

**Answer:** They block threads while waiting, reducing thread availability and potentially causing ThreadPool starvation. Async/await allows non-blocking waits.

**بالعربي:** لأنها تحجز Threads أثناء الانتظار، وقد تسبب نقصاً في Threads المتاحة. استخدم await مع API غير متزامنة.

### Q7: Is Task.WhenAll creating multiple threads?

**Answer:** No. Task.WhenAll coordinates multiple tasks and completes when all of them complete. The underlying operations determine how work is executed.

**بالعربي:** لا. هي تنسّق انتظار عدة Tasks؛ ولا تنشئ Threads بنفسها.

### Q8: Can we execute multiple EF Core queries concurrently on the same DbContext?

**Answer:** No. EF Core does not support multiple parallel operations on the same DbContext instance. Await each operation before starting another, or use separate DbContext instances when parallelism is required.

**بالعربي:** لا يجوز تشغيل استعلامات متوازية على نفس DbContext. انتظر انتهاء العملية أو استخدم instances منفصلة عند الحاجة وبشكل مدروس.

## 14. الخلاصة التي يجب تذكرها

- **Thread:** من ينفّذ الكود.
- **Task:** تمثيل العملية واكتمالها ونتيجتها.
- **await:** ينتظر دون حجز Thread أثناء انتظار I/O غير المكتمل.
- **State Machine:** تتذكر حالة الدالة وكيف تستكمل.
- **ThreadPool:** يدير Threads ويعيد استخدامها.
- **async لا تعني parallel:** التزامن/التوازي ليسا نتيجة تلقائية لمجرد كتابة async.

## 15. مصادر Microsoft الرسمية

1. [C# await operator](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/await)
2. [Asynchronous programming scenarios](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/async-scenarios)
3. [Asynchronous programming with async and await](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)
4. [The managed thread pool](https://learn.microsoft.com/en-us/dotnet/standard/threading/the-managed-thread-pool)
5. [ASP.NET Core best practices](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices)
6. [EF Core asynchronous programming](https://learn.microsoft.com/en-us/ef/core/miscellaneous/async)
