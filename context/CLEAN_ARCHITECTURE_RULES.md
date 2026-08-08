# CrmApp — Clean Architecture Rules & Review Rubric

**Purpose:** a machine-readable, enforceable rulebook for this codebase. Feed it to Claude Code (or
any review tool) to audit changes, and use it yourself before opening a PR.

**Version:** 1.0 · **Last updated:** 2026-08-07 · **Applies to:** `CrmApp` (.NET 10, Aspire 13.2)

**Companions:** `BUILD_PLAN.md` (how to build it) · `PROGRESS_TRACKER.md` (where you are)

---

## How to use this document

### With Claude Code

```
Review the current diff against docs/CLEAN_ARCHITECTURE_RULES.md.
For each violation report: rule ID, file:line, severity, why it matters, and the fix.
Do not report style opinions that aren't in the rulebook.
```

Scoped variants:

```
Audit src/Application against rules AP-01..AP-12. List violations only.
Check the layer dependency rules (AR-01..AR-06) across the whole solution.
I'm adding a feature to TodoItems. Which rules apply and in what order?
```

### Severity levels

| Level | Meaning | Action |
|---|---|---|
| 🔴 **BLOCKER** | Breaks the architecture. The layering no longer holds. | Must fix before merge. |
| 🟠 **MAJOR** | Correctness, security, or performance defect. | Fix before merge, or file a ticket with a date. |
| 🟡 **MINOR** | Convention drift. Compounds if ignored. | Fix in this PR if cheap. |
| 🔵 **INFO** | Observation or trade-off worth recording. | Note in the PR description. |

### Scoring

```
Score = 100 − (BLOCKER × 25) − (MAJOR × 10) − (MINOR × 3)

≥ 90  Ship it
70–89 Fix majors, then ship
50–69 Needs rework
< 50  Architectural review required
```

---

# Part 1 — Architecture Rules (AR)

The dependency rule is the whole game: **source-code dependencies point inward only.**

```
        ┌─────────────────────────────────────────────┐
        │  Web  /  AppHost                            │  ← Frameworks & drivers
        │    ┌───────────────────────────────────┐    │
        │    │  Infrastructure                   │    │  ← Adapters
        │    │    ┌─────────────────────────┐    │    │
        │    │    │  Application            │    │    │  ← Use cases
        │    │    │    ┌───────────────┐    │    │    │
        │    │    │    │   Domain      │    │    │    │  ← Entities
        │    │    │    └───────────────┘    │    │    │
        │    │    └─────────────────────────┘    │    │
        │    └───────────────────────────────────┘    │
        └─────────────────────────────────────────────┘
                     dependencies point ──►  inward
```

### AR-01 🔴 Domain has zero project dependencies

`src/Domain/Domain.csproj` must contain **no** `<ProjectReference>` and exactly one
`<PackageReference>` (`MediatR.Contracts`, for the `INotification` marker).

```powershell
Select-String -Path "src\Domain\Domain.csproj" -Pattern "ProjectReference"   # must be empty
```

**Why:** the moment Domain depends on anything, business rules become hostage to a framework's
upgrade cycle. This is the single rule that, if broken, unravels everything else.

---

### AR-02 🔴 Application depends only on Domain

```powershell
Select-String -Path "src\Application\Application.csproj" -Pattern "Infrastructure|Web|ServiceDefaults"
# must be empty
```

**Why:** use cases must be testable without a database, web server, or DI container.

---

### AR-03 🟠 Application may reference EF Core abstractions, never a provider

**Allowed:** `Microsoft.EntityFrameworkCore` (for `DbSet<T>`, `IQueryable`, `ProjectTo`).
**Forbidden:** `Npgsql.*`, `Microsoft.EntityFrameworkCore.SqlServer`, `.Sqlite`, `.InMemory`.

```powershell
Select-String -Path "src\Application\**\*.cs" -Pattern "Npgsql|UseSqlServer|UseSqlite|UseNpgsql|UseInMemory"
```

**Why:** this is the deliberate, documented compromise this template makes. `DbSet<T>` is close
enough to a repository abstraction to be useful; a *provider* is a hard infrastructure commitment.
Keep the line exactly here — don't let it slide.

🔵 *If you'd rather have a pure Domain/Application with zero EF surface, introduce
`IRepository<T>` + Specifications. That is a valid alternative architecture, but it is **not** this
codebase's convention. Do not mix the two.*

---

### AR-04 🔴 No inward leakage of framework types

These types must never appear in `src/Domain` or `src/Application`:

| Type | Belongs in | The port to use instead |
|---|---|---|
| `HttpContext`, `IHttpContextAccessor` | Web | `IUser` |
| `ControllerBase`, `IActionResult`, `TypedResults` | Web | return plain types/DTOs |
| `UserManager<T>`, `SignInManager<T>`, `IdentityUser` | Infrastructure | `IIdentityService` |
| `ApplicationDbContext` (concrete) | Infrastructure | `IApplicationDbContext` |
| `DateTime.Now` / `DateTime.UtcNow` | nowhere | `TimeProvider` |

```powershell
Select-String -Path "src\Domain\**\*.cs","src\Application\**\*.cs" `
  -Pattern "HttpContext|IActionResult|ControllerBase|UserManager|SignInManager|IdentityUser|ApplicationDbContext"
```

---

### AR-05 🟠 Ports are declared by the consumer, implemented by the outer layer

Interfaces live in `src/Application/Common/Interfaces/`; implementations live in
`src/Infrastructure/` or `src/Web/Services/`.

| Port (Application) | Adapter | Location |
|---|---|---|
| `IApplicationDbContext` | `ApplicationDbContext` | Infrastructure |
| `IIdentityService` | `IdentityService` | Infrastructure |
| `IUser` | `CurrentUser` | Web |

**Why:** the interface belongs to whoever *needs* it, not whoever *provides* it. That inversion is
what lets the arrow point inward.

---

### AR-06 🟡 Shared constants over magic strings

Aspire resource names must come from `src/Shared/Services.cs`.

```powershell
Select-String -Path "src\**\*.cs" -Pattern '"CrmAppDb"|"webapi"|"dbserver"|"webfrontend"'
# should hit ONLY src\Shared\Services.cs
```

**Why:** AppHost registers a resource by name; Infrastructure resolves a connection string by that
name. Without a shared constant a typo is a runtime failure instead of a compile error.

---

# Part 2 — Domain Rules (DM)

### DM-01 🔴 Entities inherit the right base

`BaseEntity` for identity + domain events; `BaseAuditableEntity` when who/when matters.
Never re-declare `Id` or the audit fields on a derived entity.

---

### DM-02 🔴 Domain events are raised inside the entity

```csharp
// ✅ CORRECT — the rule lives with the data it governs
private bool _done;
public bool Done
{
    get => _done;
    set
    {
        if (value && !_done)                       // idempotency guard
            AddDomainEvent(new TodoItemCompletedEvent(this));
        _done = value;
    }
}
```

```csharp
// ❌ WRONG — the handler must remember; the next caller won't
public async Task Handle(UpdateTodoItemCommand request, CancellationToken ct)
{
    entity.Done = request.Done;
    if (request.Done)
        entity.AddDomainEvent(new TodoItemCompletedEvent(entity));   // violation
}
```

**Why:** if raising the event is the caller's job, the event is only as reliable as the least
careful caller. Put it in the setter and it fires from every path — handlers, seeders, tests.

**Check:** `AddDomainEvent` outside `src/Domain` is a violation.

```powershell
Select-String -Path "src\Application\**\*.cs","src\Infrastructure\**\*.cs","src\Web\**\*.cs" -Pattern "AddDomainEvent"
```

---

### DM-03 🟠 Domain event state transitions must be idempotent

Guard with the current state (`value && !_done`), so setting the same value twice raises one event.

---

### DM-04 🟠 Value objects validate in a static factory and are immutable

```csharp
public static Colour From(string code)
{
    var colour = new Colour(code);
    if (!SupportedColours.Contains(colour))
        throw new UnsupportedColourException(code);
    return colour;
}
```

Properties use `private set` or `init`. Equality comes from `GetEqualityComponents()`, never
reference equality.

**Why:** invalid state becomes unrepresentable. Validation happens once, at the boundary, instead of
at every use site.

---

### DM-05 🟠 Domain is persistence-ignorant

No `[Table]`, `[Column]`, `[Key]`, `[MaxLength]`, `[Required]`, `[ForeignKey]` on domain entities.
All of it goes in `IEntityTypeConfiguration<T>`.

**Sole exception:** `[NotMapped]` on `BaseEntity.DomainEvents` — a pure "ignore me" marker with no
schema opinion.

```powershell
Select-String -Path "src\Domain\**\*.cs" -Pattern "\[Table|\[Column|\[Key\]|\[MaxLength|\[Required|\[ForeignKey"
```

---

### DM-06 🟡 `DateTimeOffset` over `DateTime`

Always. `DateTime` loses offset information and produces bugs that only appear when your users cross
a timezone.

---

### DM-07 🟡 Collections exposed read-only

Back with a private field, expose `IReadOnlyCollection<T>`, mutate through methods. A public
`List<T>` setter means any caller can bypass your invariants.

---

# Part 3 — Application Rules (AP)

### AP-01 🔴 One use case = one folder = one file for request + handler

```
TodoItems/Commands/CreateTodoItem/
    CreateTodoItem.cs                  ← command record AND handler
    CreateTodoItemCommandValidator.cs
```

**Why:** vertical slices. Everything for one behaviour is co-located; deleting a feature is deleting
a folder.

---

### AP-02 🔴 Handlers contain **only** use-case logic

A handler must **not** contain:

| Anti-pattern | Where it belongs |
|---|---|
| `if (string.IsNullOrEmpty(...)) throw` | `AbstractValidator` |
| `try { } catch { }` | `UnhandledExceptionBehaviour` |
| `_logger.LogInformation(...)` | `LoggingBehaviour` |
| `if (!user.IsInRole(...))` | `[Authorize]` + `AuthorizationBehaviour` |
| `Stopwatch` | `PerformanceBehaviour` |

**Benchmark:** `CreateTodoItemCommandHandler.Handle` is ~10 lines. If yours is over ~30, cross-cutting
concerns have leaked in.

```csharp
// ✅ CORRECT
public async Task<int> Handle(CreateTodoItemCommand request, CancellationToken cancellationToken)
{
    var entity = new TodoItem { ListId = request.ListId, Title = request.Title, Done = false };
    _context.TodoItems.Add(entity);
    await _context.SaveChangesAsync(cancellationToken);
    return entity.Id;
}
```

---

### AP-03 🔴 Every command has a validator

Note the file-naming convention: the command *record* is `CreateTodoItemCommand` but it lives in a
file named `CreateTodoItem.cs`. So count the **types**, not the filenames:

```powershell
# Commands declared:
(Get-ChildItem src\Application -Recurse -Filter *.cs |
  Select-String -Pattern "record \w+Command\b.*:.*IRequest").Count
# Validators declared:
(Get-ChildItem src\Application -Recurse -Filter *.cs |
  Select-String -Pattern "class \w+CommandValidator\b").Count
```

Queries need one only when they take parameters.

🔵 *Current state: 7 commands, 4 validators. The three delete commands
(`DeleteTodoItem`, `DeleteTodoList`, plus `UpdateTodoItemDetail`) take only an `int Id` and rely on
`Guard.Against.NotFound` in the handler (AP-08). That is an accepted pattern — a mismatch in these
counts is a prompt to check, not an automatic violation.*

---

### AP-04 🔴 Requests are `record`s with `init` properties

```csharp
public record CreateTodoItemCommand : IRequest<int>
{
    public int ListId { get; init; }
    public string? Title { get; init; }
}
```

**Why:** a request is a message. Mutating it mid-pipeline (a behaviour changing what the handler
sees) is a class of bug that `init` makes impossible.

---

### AP-05 🔴 `CancellationToken` threaded to every async call

`Handle` receives one; pass it to `SaveChangesAsync`, `ToListAsync`, `FirstOrDefaultAsync`, and every
downstream call. A dropped token means work continues after the client disconnects.

---

### AP-06 🟠 Queries use `AsNoTracking()` + `ProjectTo<TDto>()`

```csharp
// ✅ CORRECT — SQL selects only the DTO's columns
Lists = await _context.TodoLists
    .AsNoTracking()
    .ProjectTo<TodoListDto>(_mapper.ConfigurationProvider)
    .OrderBy(t => t.Title)
    .ToListAsync(cancellationToken)
```

```csharp
// ❌ WRONG — loads full entities, maps in memory, tracks everything
var lists = await _context.TodoLists.ToListAsync(cancellationToken);
return _mapper.Map<List<TodoListDto>>(lists);
```

**Why:** `ProjectTo` translates into the SQL `SELECT`. The wrong version reads every column of every
row into memory — invisible on 10 rows, fatal on 100,000.

---

### AP-07 🟠 No client-side evaluation

Never `.ToList()` / `.AsEnumerable()` before `.Where()` / `.OrderBy()`. Filter in the database.

```csharp
// ❌ Loads the whole table, then filters in memory
var items = (await _context.TodoItems.ToListAsync(ct)).Where(i => !i.Done);
```

---

### AP-08 🟠 Guard clauses for not-found

```csharp
var entity = await _context.TodoItems.FindAsync([request.Id], cancellationToken);
Guard.Against.NotFound(request.Id, entity);
```

Consistent 404 translation via `ProblemDetailsExceptionHandler`. Don't hand-roll null checks that
produce a different shape of error.

---

### AP-09 🟠 Authorization is declarative

`[Authorize]` on the *request type*, enforced centrally by `AuthorizationBehaviour`.

```csharp
[Authorize]
public record GetTodosQuery : IRequest<TodosVm>;

[Authorize(Roles = Roles.Administrator)]
public record DeleteTodoListCommand(int Id) : IRequest;
```

Role names come from `Domain/Constants/Roles.cs` — never a string literal.

---

### AP-10 🟡 Mapping profiles nested in their DTO

```csharp
public class TodoListDto
{
    public int Id { get; init; }
    // ...
    private class Mapping : Profile
    {
        public Mapping() { CreateMap<TodoList, TodoListDto>(); }
    }
}
```

**Why:** no central `MappingProfile.cs` god-file to conflict on. Delete the DTO, the mapping goes
with it.

---

### AP-11 🟡 Domain exceptions, not HTTP results

Application throws `ValidationException`, `ForbiddenAccessException`, `NotFoundException`. The Web
layer translates. An Application file that knows about status codes is a layering violation.

---

### AP-12 🟡 Behaviour registration order is load-bearing

```csharp
cfg.AddOpenRequestPreProcessor(typeof(LoggingBehaviour<>));
cfg.AddOpenBehavior(typeof(UnhandledExceptionBehaviour<,>));   // outermost
cfg.AddOpenBehavior(typeof(AuthorizationBehaviour<,>));
cfg.AddOpenBehavior(typeof(ValidationBehaviour<,>));
cfg.AddOpenBehavior(typeof(PerformanceBehaviour<,>));          // innermost
```

Auth before validation (don't leak field-level errors to unauthorized callers). Exception handling
outermost so it catches everything. Changing this order changes observable behaviour.

---

# Part 4 — Infrastructure Rules (IN)

### IN-01 🔴 EF configuration in `IEntityTypeConfiguration<T>`, one file per entity

Discovered via `ApplyConfigurationsFromAssembly` — never a manual registration list.

---

### IN-02 🔴 `base.OnModelCreating(builder)` called first

```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);      // Identity's mappings — MUST be first
    builder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
}
```

Omitting it silently breaks the Identity schema.

---

### IN-03 🟠 Interceptors registered before `AddDbContext`

```csharp
builder.Services.AddScoped<ISaveChangesInterceptor, AuditableEntityInterceptor>();
builder.Services.AddScoped<ISaveChangesInterceptor, DispatchDomainEventsInterceptor>();

builder.Services.AddDbContext<ApplicationDbContext>((sp, options) =>
{
    options.AddInterceptors(sp.GetServices<ISaveChangesInterceptor>());
    options.UseNpgsql(connectionString);
});
```

The `(sp, options)` overload is required — you need the service provider to resolve them.

---

### IN-04 🟠 Domain events cleared before publishing

```csharp
var domainEvents = entities.SelectMany(e => e.DomainEvents).ToList();
entities.ToList().ForEach(e => e.ClearDomainEvents());     // BEFORE publish
foreach (var domainEvent in domainEvents)
    await _mediator.Publish(domainEvent);
```

**Why:** a handler that triggers another `SaveChanges` would otherwise re-dispatch the same event —
an infinite loop in the worst case.

---

### IN-05 🟠 `TimeProvider`, never `DateTime.UtcNow`

Injected and registered as `AddSingleton(TimeProvider.System)`. This is what makes time mockable.

```powershell
Get-ChildItem src -Recurse -Filter *.cs |
  Where-Object { $_.FullName -notmatch 'node_modules' } |
  Select-String -Pattern "DateTime\.UtcNow|DateTime\.Now"
```

✅ **Codebase is clean on this rule.** `GetWeatherForecastsQueryHandler` previously used
`DateTime.Now.AddDays(index)` — untestable against a fixed clock, and local time rather than UTC.
Fixed 2026-08-07 by injecting `TimeProvider` and using `_dateTime.GetUtcNow().UtcDateTime`. The
public `WeatherForecast.Date` contract is unchanged, so no client regeneration was needed.

🔵 *Further improvement available:* `WeatherForecast.Date` is still `DateTime`, not `DateTimeOffset`
(DM-06). Changing it would alter the OpenAPI schema and require `npm run generate-api`; the
JavaScript `new Date(...)` consumer in `Weather.jsx` parses either form, so it is safe but not free.

---

### IN-06 🟠 Fail fast on missing configuration

```csharp
var connectionString = builder.Configuration.GetConnectionString(Services.Database);
Guard.Against.Null(connectionString, message: $"Connection string '{Services.Database}' not found.");
```

Crash at startup with a clear message, not on the first request with a `NullReferenceException`.

---

### IN-07 🟡 One `DbContext` instance per scope serving both interfaces

```csharp
builder.Services.AddScoped<IApplicationDbContext>(p => p.GetRequiredService<ApplicationDbContext>());
```

Registering it twice would give you two contexts, two change trackers, and a very confusing bug.

---

### IN-08 🔵 `EnsureDeleted`/`EnsureCreated` is dev-only and destructive

`ApplicationDbContextInitialiser.InitialiseAsync()` **drops the database on every startup**.
Acceptable for the template; must become `MigrateAsync()` + EF migrations before any environment
holds data you care about. Tracked as Phase 12.2.

---

# Part 5 — Web Rules (WB)

### WB-01 🔴 Endpoints are thin — `ISender` in, `TypedResults` out

```csharp
// ✅ CORRECT
public static async Task<Ok<TodosVm>> GetTodoLists(ISender sender)
{
    var vm = await sender.Send(new GetTodosQuery());
    return TypedResults.Ok(vm);
}
```

The **only** permitted logic is HTTP-shaped, e.g. route/body consistency:

```csharp
if (id != command.Id) return TypedResults.BadRequest();
```

```powershell
# MUST be zero:
Select-String -Path "src\Web\Endpoints\*.cs" -Pattern "IApplicationDbContext|ApplicationDbContext|_context"
```

**Benchmark:** more than ~8 lines in an endpoint method means logic has leaked out of Application.

---

### WB-02 🔴 Endpoint groups implement `IEndpointGroup`

Auto-discovered by `MapEndpoints`. Never hand-register a route in `Program.cs`.

---

### WB-03 🔴 Named static handler methods

Enforced by `Guard.Against.AnonymousMethod(handler)`.

**Why — the full chain:**

```
handler method name → .WithName() → OpenAPI operationId → NSwag → TypeScript client method name
```

A lambda has no usable name, so the generated client method would be garbage. This rule is what
makes the frontend type-safe.

---

### WB-04 🟠 Concrete `TypedResults` generic types

```csharp
Task<Ok<TodosVm>>                            // ✅
Task<Created<int>>                           // ✅
Task<Results<NoContent, BadRequest>>         // ✅
Task<IResult>                                // ❌ — OpenAPI can't infer the schema
```

Accurate schemas → accurate generated client. `IResult` erases the type.

---

### WB-05 🟠 Authorization at the group level

```csharp
public static void Map(RouteGroupBuilder groupBuilder)
{
    groupBuilder.RequireAuthorization();     // first line — secure by default
    groupBuilder.MapGet(GetTodoLists);
}
```

Opt **out** per endpoint with `.AllowAnonymous()`. Never opt in per endpoint — someone will forget.

---

### WB-06 🟠 Exceptions → RFC 7807 in one place

`ProblemDetailsExceptionHandler` owns all exception→status mapping. No `try/catch` in endpoints.

---

### WB-07 🟡 Every endpoint documented

`[EndpointSummary]` and `[EndpointDescription]` on every handler — they surface in Scalar and in the
generated client's doc comments.

---

### WB-08 🟡 `Program.cs` stays declarative

Five `Add*Services()` calls in dependency order. If it grows past ~50 lines, something that belongs
in a layer's `DependencyInjection.cs` has leaked upward.

---

### WB-09 🟠 CORS must not be `AllowAnyOrigin()` in production

Currently wide open (`Program.cs`). 🔵 Known debt — Phase 12.1.

---

# Part 6 — Testing Rules (TS)

### TS-01 🔴 Test type matches the concern

| Testing… | Project | Mocks? |
|---|---|---|
| Value objects, entity invariants | `Domain.UnitTests` | **None** |
| Behaviours, validators, mappings | `Application.UnitTests` | Moq on ports |
| Interceptors, EF config | `Infrastructure.IntegrationTests` | Real DB |
| Use cases end-to-end | `Application.FunctionalTests` | Real DB, no mocks |
| User journeys | `Web.AcceptanceTests` | Real browser |

🟠 **If a Domain test needs a mock, the Domain has a dependency it shouldn't have.**

---

### TS-02 🔴 State reset before every functional test

```csharp
public abstract class TestBase
{
    [SetUp]
    public async Task SetUp() => await TestApp.ResetState();
}
```

Respawn truncates tables; `_userId`/`_roles` statics are cleared. Skipping this produces order-
dependent tests that pass locally and fail in CI.

---

### TS-03 🟠 `AssertConfigurationIsValid()` is mandatory

```csharp
[Test]
public void ShouldHaveValidConfiguration() => _configuration!.AssertConfigurationIsValid();
```

Highest value-per-line test in the solution: catches every unmapped destination property across the
whole assembly. Add a DTO property, forget the mapping — this fails immediately.

---

### TS-04 🟠 Commands tested for both validation failure and happy path

```csharp
[Test]
public async Task ShouldRequireMinimumFields()
{
    var command = new CreateTodoItemCommand();
    await Should.ThrowAsync<ValidationException>(() => TestApp.SendAsync(command));
}
```

---

### TS-05 🟠 Assert audit fields to prove interceptors ran

```csharp
item.CreatedBy.ShouldBe(userId);
item.Created.ShouldBeGreaterThan(DateTimeOffset.MinValue);
```

Without this, a silently-unregistered interceptor passes every other test.

---

### TS-06 🟡 Shouldly, not `Assert.*`

`item.ShouldNotBeNull()` reads better and produces far better failure messages than
`Assert.IsNotNull(item)`.

---

### TS-07 🟡 Page Object Model in acceptance tests

Selectors live in `Pages/*.cs`. Step definitions call page methods and never touch a selector
directly. One markup change, one file to update.

---

# Part 7 — Cross-Cutting Rules (XC)

### XC-01 🔴 Central Package Management — no versions in `.csproj`

```powershell
Select-String -Path "src\**\*.csproj","tests\**\*.csproj" -Pattern 'PackageReference.*Version='
# must be empty
```

---

### XC-02 🟠 `WarningsNotAsErrors` entries need a dated justification

Every suppression is technical debt. Current state:

| Code | Reason | Exit condition |
|---|---|---|
| `NU1608` | Version conflict | Npgsql.EFCore.PostgreSQL 10.0.0 released |
| `NU1901`–`NU1904` | Transitive CVEs | Patched versions pinned (Phase 12.4) |

Adding a code without a comment explaining why is a violation.

---

### XC-03 🟠 Nullable annotations are honest

`Nullable` is enabled solution-wide. `!` (null-forgiving) is allowed only where genuinely
guaranteed — e.g. `public TodoList List { get; set; } = null!;` for an EF navigation property.
Sprinkling `!` to silence the compiler defeats the feature.

---

### XC-04 🟡 `async`/`await` all the way

No `.Result`, no `.Wait()`, no `.GetAwaiter().GetResult()` in request paths.

🔵 *Known exception:* `DispatchDomainEventsInterceptor.SavingChanges` (the sync overload) uses
`.GetAwaiter().GetResult()` because EF's sync API gives no alternative. Prefer `SaveChangesAsync`
everywhere so the async path is the one that runs.

---

### XC-05 🟡 Generated code is never hand-edited

`src/Web/ClientApp/src/web-api-client.ts` is regenerated by `prestart`/`prebuild`. Editing it means
your change vanishes on the next `npm start`. Change the C# and regenerate.

---

### XC-06 🟠 No secrets in source

Connection strings come from Aspire; production secrets from Key Vault
(`AddKeyVaultIfConfigured`). Seeded dev credentials (`Administrator1!`) are Development-only and
must never be reachable in another environment.

---

# Automated Verification

**→ Run `pwsh ./scripts/check-architecture.ps1`.** It exits non-zero on any violation, so it can
gate a PR in CI directly.

**Status as of 2026-08-07: all rules pass** (one baselined deviation, KD-06).

```
ok   AR-01  Domain has no project references
ok   AR-02  Application depends only on Domain
ok   AR-04  No framework types leaked inward
ok   DM-02  Domain events raised only inside Domain
ok   DM-05  Domain free of persistence attributes
ok   IN-05  TimeProvider used instead of DateTime.Now
ok   WB-01  Endpoints contain no data access
ok   XC-01  Central Package Management intact
base XC-04  No sync-over-async (1 accepted)
info AP-03  commands=7 validators=4
```

### The baseline mechanism

The script holds an explicit `$Baseline` hashtable of accepted deviations, keyed by rule ID and
pinned to `File:Line`:

```powershell
$Baseline = @{
    'XC-04' = @(
        # KD-06: EF's sync SavingChanges override has no async hook.
        'DispatchDomainEventsInterceptor.cs:19'
    )
}
```

A baselined hit reports as `base` (not a failure). **A hit anywhere else still fails.** This is the
point: it is far better than widening the grep to hide the violation, because a *new* instance of
the same anti-pattern is still caught. Every baseline entry must have a matching KD-xx row below.

### The checks, inline

If you'd rather run them ad hoc than via the script:

> ⚠️ `Select-String -Path "src\**\*.cs"` does **not** recurse in PowerShell — the `**` glob only
> matches one level. Always pipe from `Get-ChildItem -Recurse` as shown, and exclude
> `node_modules` or you will scan the entire React dependency tree.

```powershell
$root = $PSScriptRoot ? (Split-Path $PSScriptRoot) : "."
$src  = Get-ChildItem "$root\src" -Recurse -Filter *.cs |
          Where-Object { $_.FullName -notmatch '\\(node_modules|artifacts|bin|obj)\\' }

function Check($id, $result) {
    if ($result) {
        Write-Host "FAIL $id" -ForegroundColor Red
        $result | Select-Object -First 5 | ForEach-Object { "      $($_.Filename):$($_.LineNumber)" }
    } else {
        Write-Host "ok   $id" -ForegroundColor Green
    }
}

# ── AR-01/02: dependency direction ────────────────────────────────
Check "AR-01" (Select-String -Path "$root\src\Domain\Domain.csproj" -Pattern "ProjectReference")
Check "AR-02" (Select-String -Path "$root\src\Application\Application.csproj" -Pattern "Infrastructure|Web")

# ── AR-03/04: framework leakage inward ────────────────────────────
Check "AR-04" (Get-ChildItem "$root\src\Domain","$root\src\Application" -Recurse -Filter *.cs |
    Select-String -Pattern "HttpContext|IActionResult|ControllerBase|UserManager|SignInManager|Npgsql|UseSqlServer")

# ── DM-02: domain events raised outside Domain ────────────────────
Check "DM-02" (Get-ChildItem "$root\src\Application","$root\src\Infrastructure","$root\src\Web" -Recurse -Filter *.cs |
    Where-Object { $_.FullName -notmatch 'node_modules' } | Select-String -Pattern "AddDomainEvent")

# ── DM-05: persistence attributes in Domain ───────────────────────
Check "DM-05" (Get-ChildItem "$root\src\Domain" -Recurse -Filter *.cs |
    Select-String -Pattern "\[Table|\[Column|\[Key\]|\[MaxLength|\[Required")

# ── WB-01: data access in endpoints ───────────────────────────────
Check "WB-01" (Get-ChildItem "$root\src\Web\Endpoints" -Filter *.cs |
    Select-String -Pattern "IApplicationDbContext|ApplicationDbContext|_context")

# ── IN-05: unmockable time  (currently 1 known hit — KD-07) ───────
Check "IN-05" ($src | Select-String -Pattern "DateTime\.UtcNow|DateTime\.Now")

# ── XC-01: CPM intact ─────────────────────────────────────────────
Check "XC-01" (Get-ChildItem "$root\src","$root\tests" -Recurse -Filter *.csproj |
    Select-String -Pattern 'PackageReference.*Version=')

# ── XC-04: sync-over-async  (1 known hit in the interceptor — KD-06) ──
Check "XC-04" ($src | Select-String -Pattern "\.Result\b|\.Wait\(\)|GetAwaiter\(\)\.GetResult")
```

Command/validator parity (AP-03 — a mismatch is a prompt to check, not an automatic failure):

```powershell
"Commands  : " + ($src | Select-String -Pattern "record \w+Command\b.*:.*IRequest").Count
"Validators: " + ($src | Select-String -Pattern "class \w+CommandValidator\b").Count
```

Standard gates:

```powershell
dotnet build                                              # zero warnings
dotnet test                                               # all green
dotnet list package --vulnerable --include-transitive     # audit debt
dotnet list package --outdated                            # drift
```

---

# Review Checklist (paste into a PR)

```markdown
## Architecture
- [ ] AR-01/02  Dependency direction intact (Domain ← Application ← Infrastructure/Web)
- [ ] AR-04     No framework types leaked into Domain/Application
- [ ] AR-05     New interfaces declared in Application, implemented outward
- [ ] AR-06     No magic strings for Aspire resource names

## Domain
- [ ] DM-02/03  Domain events raised in the entity, idempotently
- [ ] DM-04     Value objects validate in a static factory; immutable
- [ ] DM-05     No persistence attributes

## Application
- [ ] AP-02     Handler contains only use-case logic
- [ ] AP-03     Every new command has a validator
- [ ] AP-05     CancellationToken threaded everywhere
- [ ] AP-06/07  Queries use AsNoTracking + ProjectTo; no client-side evaluation
- [ ] AP-09     Authorization declarative via [Authorize]

## Infrastructure
- [ ] IN-01     EF config in IEntityTypeConfiguration<T>
- [ ] IN-05     TimeProvider, not DateTime.UtcNow

## Web
- [ ] WB-01     Endpoints thin; ISender in, TypedResults out
- [ ] WB-03/04  Named static handlers; concrete result types
- [ ] WB-05     Group-level RequireAuthorization
- [ ] WB-07     [EndpointSummary]/[EndpointDescription] present

## Testing
- [ ] TS-01     Right test type for the concern
- [ ] TS-04     Validation failure AND happy path covered
- [ ] TS-05     Audit fields asserted where relevant

## Cross-cutting
- [ ] XC-01     No versions in .csproj
- [ ] XC-02     Any new warning suppression is justified
- [ ] XC-05     Generated client regenerated, not hand-edited
- [ ] XC-06     No secrets committed

**Score:** ___ / 100
```

---

# Known Deviations (living list)

Accepted trade-offs. Review each time this document is updated.

| ID | Deviation | Rationale | Exit condition |
|---|---|---|---|
| KD-01 | Application references EF Core | `DbSet<T>`/`ProjectTo` ergonomics beat a hand-rolled repository | None — deliberate (AR-03) |
| KD-02 | `EnsureDeleted`/`EnsureCreated` in dev | Fast template iteration | Move to migrations (Phase 12.2) |
| KD-03 | `AllowAnyOrigin()` CORS | Local dev convenience | Lock down before deploy (Phase 12.1) |
| KD-04 | NU1901–1904 suppressed | Transitive CVEs, no patched versions yet | Pin patches (Phase 12.4) |
| KD-05 | `PendingModelChangesWarning` ignored | No migrations in use yet | Resolves with KD-02 |
| KD-06 | Sync `GetAwaiter().GetResult()` in `DispatchDomainEventsInterceptor.SavingChanges:19` | EF's sync `SavingChanges` overload has no async hook. There are currently **zero** sync `SaveChanges()` callers, but deleting the override would silently drop domain events the moment someone adds one — keeping the safety net is the lesser evil. | Only if EF exposes an async interception point for the sync path. Baselined in `check-architecture.ps1`. |
| ~~KD-07~~ | ~~`DateTime.Now` in `GetWeatherForecastsQuery.cs:20`~~ | — | ✅ **Resolved 2026-08-07** — `TimeProvider` injected. |

---

# Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-08-07 | Initial rulebook derived from the CrmApp codebase |

> **Keeping this current:** when you accept a new trade-off, add it to *Known Deviations* with an
> exit condition. When you add a rule, give it an ID, a severity, a *why*, and a grep check. A rule
> without an automated check will not be followed.
