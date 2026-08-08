# CrmApp — Build From Scratch Plan

A step-by-step guide to rebuilding this solution by hand, so you understand every layer instead of
inheriting a template you didn't write.

**Target stack (as verified in this repo):**

| Concern | Choice |
|---|---|
| SDK | .NET 10.0.201 (`global.json`, `rollForward: latestFeature`) |
| Architecture | Clean Architecture (Jason Taylor template v10.8.0 lineage) |
| Orchestration | .NET Aspire 13.2.0 (AppHost + ServiceDefaults) |
| Database | PostgreSQL via `Npgsql.EntityFrameworkCore.PostgreSQL` |
| Mediation | MediatR 14 (CQRS + pipeline behaviours) |
| Validation | FluentValidation 12 |
| Mapping | AutoMapper 16 |
| Guards | Ardalis.GuardClauses 5 |
| API style | Minimal APIs, discovered via `IEndpointGroup` |
| API docs | Microsoft.AspNetCore.OpenApi + Scalar |
| Auth | ASP.NET Core Identity (cookie) + `MapIdentityApi` |
| Frontend | React 19 + Vite 8 + TypeScript, NSwag-generated client |
| Tests | NUnit 4 + Shouldly + Moq + Respawn + Reqnroll + Playwright |

**Related documents:**
- `PROGRESS_TRACKER.md` — tick off every checkpoint as you go.
- `CLEAN_ARCHITECTURE_RULES.md` — the rules your code is graded against.

---

## How to use this plan

Work **top to bottom**. Each phase ends with a *Verify* block — do not move on until it passes.
The dependency order matters: Domain has no dependencies, so it is built first; Web depends on
everything, so it is built last.

Each phase maps 1:1 to a section in `PROGRESS_TRACKER.md`.

**Estimated effort:** 3–5 focused days if you are new to the stack, ~1 day if experienced.

---

## Phase 0 — Prerequisites & Tooling

### 0.1 Install the toolchain

```powershell
# Verify .NET SDK 10.x
dotnet --list-sdks

# Aspire workload / CLI
dotnet workload install aspire

# Node 20+ for the React client
node --version
npm --version

# Docker Desktop must be RUNNING (Aspire starts PostgreSQL in a container)
docker info
```

### 0.2 Trust the local HTTPS certificate

```powershell
dotnet dev-certs https --trust
```

Without this the Aspire dashboard and the Vite dev server will fail their TLS handshake.

### 0.3 Create the folder skeleton

```powershell
mkdir CrmApp
cd CrmApp
mkdir src, tests, docs
```

> **Verify:** `dotnet --version` prints 10.x, `docker info` succeeds, and you have `src/`, `tests/`, `docs/`.

---

## Phase 1 — Solution Scaffolding & Build Governance

This phase sets up the *rails*. Doing it first means every project you create afterwards
automatically inherits the right settings.

### 1.1 Pin the SDK — `global.json`

```json
{
  "sdk": {
    "version": "10.0.201",
    "rollForward": "latestFeature"
  }
}
```

**Why:** guarantees every machine and CI agent compiles with the same SDK. `latestFeature` allows
patch/feature-band upgrades but blocks an accidental major-version jump.

### 1.2 Create the solution — `CrmApp.slnx`

.NET 10 supports the XML-based `.slnx` format (much cleaner diffs than `.sln`):

```powershell
dotnet new sln --name CrmApp --format slnx
```

The final structure you are aiming for:

```xml
<Solution>
  <Folder Name="/Solution Items/">
    <File Path=".editorconfig" />
    <File Path=".gitignore" />
    <File Path="Directory.Build.props" />
    <File Path="Directory.Packages.props" />
    <File Path="global.json" />
    <File Path="README.md" />
  </Folder>
  <Folder Name="/src/">
    <Project Path="src/AppHost/AppHost.csproj" DefaultStartup="true" />
    <Project Path="src/ServiceDefaults/ServiceDefaults.csproj" />
    <Project Path="src/Application/Application.csproj" />
    <Project Path="src/Domain/Domain.csproj" />
    <Project Path="src/Infrastructure/Infrastructure.csproj" />
    <Project Path="src/Shared/Shared.csproj" />
    <Project Path="src/Web/Web.csproj" />
  </Folder>
  <Folder Name="/tests/">
    <Project Path="tests/Application.FunctionalTests/Application.FunctionalTests.csproj" />
    <Project Path="tests/TestAppHost/TestAppHost.csproj" />
    <Project Path="tests/Application.UnitTests/Application.UnitTests.csproj" />
    <Project Path="tests/Domain.UnitTests/Domain.UnitTests.csproj" />
    <Project Path="tests/Infrastructure.IntegrationTests/Infrastructure.IntegrationTests.csproj" />
    <Project Path="tests/Web.AcceptanceTests/Web.AcceptanceTests.csproj" />
  </Folder>
</Solution>
```

Note `DefaultStartup="true"` on AppHost — F5 in the IDE launches the orchestrator, not the API.

### 1.3 Shared build settings — `Directory.Build.props`

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <!-- Suppress NU1608 until Npgsql.EntityFrameworkCore.PostgreSQL 10.0.0 is released -->
    <!-- NU1901-NU1904: NuGet audit vulnerability warnings kept as warnings, not build-breaking errors -->
    <WarningsNotAsErrors>NU1608;NU1901;NU1902;NU1903;NU1904</WarningsNotAsErrors>
    <ArtifactsPath>$(MSBuildThisFileDirectory)artifacts</ArtifactsPath>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

**Line-by-line rationale:**

- `TreatWarningsAsErrors` — warnings rot if they are allowed to accumulate. Fail the build instead.
- `WarningsNotAsErrors` — the *deliberate, documented* escape hatch. `NU1608` is a known version
  conflict pending an upstream release. `NU1901`–`NU1904` are NuGet security-audit warnings for
  **transitive** packages you cannot directly control; keeping them as warnings means you still see
  them without a red build. **Revisit this list every dependency bump** — it is technical debt, not
  a permanent setting.
- `ArtifactsPath` — funnels all `bin`/`obj` output into one top-level `artifacts/` folder. Simpler
  `.gitignore`, simpler CI cache.
- `ImplicitUsings` / `Nullable` — modern C# defaults; nullable reference types are load-bearing for
  the domain model's correctness.

### 1.4 Central Package Management — `Directory.Packages.props`

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Ardalis.GuardClauses" Version="5.0.0" />
    <PackageVersion Include="AutoMapper" Version="16.1.1" />
    <PackageVersion Include="FluentValidation.DependencyInjectionExtensions" Version="12.1.1" />
    <PackageVersion Include="MediatR" Version="14.1.0" />
    <PackageVersion Include="MediatR.Contracts" Version="2.0.1" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.0.5" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.1" />
    <!-- ...one line per package, no versions in any .csproj... -->
  </ItemGroup>
</Project>
```

**Why:** with CPM, individual `.csproj` files reference packages *without* a `Version` attribute.
One place to bump a version; impossible for two projects to silently disagree.

**Rule:** if you ever type `Version=` inside a `.csproj`, you have broken CPM.

### 1.5 `.editorconfig` and `.gitignore`

Add an `.editorconfig` (the standard .NET one is a good base) so formatting is enforced by tooling
rather than argued about in review. Add `artifacts/`, `node_modules/`, `.env.local` to `.gitignore`.

> **Verify:** `dotnet build` succeeds on the empty solution. `git status` shows no `bin`/`obj`.

---

## Phase 2 — Domain Layer (the centre; zero dependencies)

**Golden rule:** `Domain.csproj` references **no other project** and only one NuGet package
(`MediatR.Contracts`, purely for the `INotification` marker on domain events).

```powershell
dotnet new classlib -o src/Domain
dotnet sln CrmApp.slnx add src/Domain/Domain.csproj
```

`src/Domain/Domain.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <RootNamespace>CrmApp.Domain</RootNamespace>
    <AssemblyName>CrmApp.Domain</AssemblyName>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="MediatR.Contracts" />
  </ItemGroup>
</Project>
```

### 2.1 Build order inside Domain

Build in this sequence — each step depends only on the previous ones:

```
Common/BaseEvent.cs        →  Common/BaseEntity.cs     →  Common/BaseAuditableEntity.cs
Common/ValueObject.cs      →  ValueObjects/Colour.cs   →  Exceptions/UnsupportedColourException.cs
Enums/PriorityLevel.cs     →  Events/TodoItemCompletedEvent.cs
Entities/TodoList.cs, Entities/TodoItem.cs
Constants/Roles.cs
GlobalUsings.cs
```

### 2.2 `Common/BaseEntity.cs` — identity + domain events

```csharp
using System.ComponentModel.DataAnnotations.Schema;

namespace CrmApp.Domain.Common;

public abstract class BaseEntity
{
    // This can easily be modified to be BaseEntity<T> and public T Id to support different key types.
    // Using non-generic integer types for simplicity
    public int Id { get; set; }

    private readonly List<BaseEvent> _domainEvents = new();

    [NotMapped]
    public IReadOnlyCollection<BaseEvent> DomainEvents => _domainEvents.AsReadOnly();

    public void AddDomainEvent(BaseEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void RemoveDomainEvent(BaseEvent domainEvent) => _domainEvents.Remove(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

**Key design points:**
- `_domainEvents` is a private field exposed only as `IReadOnlyCollection` — callers must go through
  `AddDomainEvent`. Encapsulation, not a public `List`.
- `[NotMapped]` keeps EF Core from trying to persist the event list. This is the *one* acceptable
  persistence-flavoured attribute in Domain because it is a pure "ignore me" marker.

### 2.3 `Common/BaseAuditableEntity.cs`

```csharp
namespace CrmApp.Domain.Common;

public abstract class BaseAuditableEntity : BaseEntity
{
    public DateTimeOffset Created { get; set; }
    public string? CreatedBy { get; set; }
    public DateTimeOffset LastModified { get; set; }
    public string? LastModifiedBy { get; set; }
}
```

These fields are populated automatically by an EF interceptor in Phase 4 — never set them by hand
in a handler.

### 2.4 `Common/ValueObject.cs` and `ValueObjects/Colour.cs`

`ValueObject` is the classic DDD base implementing structural equality via
`GetEqualityComponents()`. `Colour` is the worked example:

```csharp
namespace CrmApp.Domain.ValueObjects;

public class Colour(string code) : ValueObject
{
    public static Colour From(string code)
    {
        var colour = new Colour(code);
        if (!SupportedColours.Contains(colour))
            throw new UnsupportedColourException(code);
        return colour;
    }

    public static Colour Red => new("#E05C4D");
    public static Colour Green => new("#4CAF50");
    // ... Orange, Teal, Blue, Purple, Grey

    public string Code { get; private set; } = string.IsNullOrWhiteSpace(code) ? "#000000" : code;

    public static implicit operator string(Colour colour) => colour.ToString();
    public static explicit operator Colour(string code) => From(code);

    public override string ToString() => Code;

    public static IEnumerable<Colour> SupportedColours
    {
        get
        {
            yield return Red; yield return Orange; yield return Green;
            yield return Teal; yield return Blue; yield return Purple; yield return Grey;
        }
    }

    protected override IEnumerable<object> GetEqualityComponents() { yield return Code; }
}
```

**Why this shape:** invalid state is unrepresentable. You cannot construct an unsupported `Colour`
through `From`. The `implicit`→string / `explicit`→Colour pair means EF conversion and DTO mapping
stay terse while the narrowing direction (which can throw) stays explicit.

### 2.5 `Entities/TodoItem.cs` — where the domain event is raised

```csharp
namespace CrmApp.Domain.Entities;

public class TodoItem : BaseAuditableEntity
{
    public int ListId { get; set; }
    public string? Title { get; set; }
    public string? Note { get; set; }
    public PriorityLevel Priority { get; set; }

    private bool _done;
    public bool Done
    {
        get => _done;
        set
        {
            if (value && !_done)
                AddDomainEvent(new TodoItemCompletedEvent(this));
            _done = value;
        }
    }

    public TodoList List { get; set; } = null!;
}
```

**This is the heart of the pattern.** The transition false→true raises `TodoItemCompletedEvent`
*inside the entity*. No handler, service, or controller has to remember to do it. The guard
`value && !_done` makes it idempotent — setting `Done = true` twice raises one event.

> **Verify:** `dotnet build src/Domain` succeeds. Open `Domain.csproj` and confirm there is exactly
> one `PackageReference` and zero `ProjectReference`.

---

## Phase 3 — Application Layer (use cases; depends only on Domain)

```powershell
dotnet new classlib -o src/Application
dotnet sln CrmApp.slnx add src/Application/Application.csproj
dotnet add src/Application reference src/Domain
```

`src/Application/Application.csproj` — note **no versions** (CPM):

```xml
<ItemGroup>
  <PackageReference Include="Ardalis.GuardClauses" />
  <PackageReference Include="AutoMapper" />
  <PackageReference Include="FluentValidation.DependencyInjectionExtensions" />
  <PackageReference Include="MediatR" />
  <PackageReference Include="Microsoft.EntityFrameworkCore" />
  <PackageReference Include="Microsoft.Extensions.Hosting" />
</ItemGroup>
<ItemGroup>
  <ProjectReference Include="..\Domain\Domain.csproj" />
</ItemGroup>
```

> **"Why does Application reference EF Core?"** Only for the `DbSet<T>` and `IQueryable`
> *abstractions* used by `IApplicationDbContext` and `ProjectTo`. It never references a *provider*
> (Npgsql). This is the pragmatic trade-off the template makes; see
> `CLEAN_ARCHITECTURE_RULES.md` → AR-03 for the boundary.

### 3.1 `GlobalUsings.cs`

```csharp
global using Ardalis.GuardClauses;
global using AutoMapper;
global using AutoMapper.QueryableExtensions;
global using Microsoft.EntityFrameworkCore;
global using FluentValidation;
global using MediatR;
```

Every handler file then avoids six repetitive `using` lines.

### 3.2 Ports (interfaces the outer layers must implement)

`Common/Interfaces/IApplicationDbContext.cs` — the persistence port:

```csharp
public interface IApplicationDbContext
{
    DbSet<TodoList> TodoLists { get; }
    DbSet<TodoItem> TodoItems { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken);
}
```

`Common/Interfaces/IUser.cs` — the current-user port (`string? Id`, `IEnumerable<string>? Roles`).

`Common/Interfaces/IIdentityService.cs` — user management + `AuthorizeAsync(userId, policy)`.

**This is Dependency Inversion in practice:** Application *declares* what it needs; Infrastructure
and Web *supply* it. The arrow points inward.

### 3.3 Cross-cutting: the MediatR pipeline

Build these five behaviours in `Common/Behaviours/`. They run in registration order, wrapping every
request like layers of an onion.

| Behaviour | Responsibility |
|---|---|
| `LoggingBehaviour<T>` | Pre-processor. Logs request name + user before handling. |
| `UnhandledExceptionBehaviour<T,R>` | Catches, logs, rethrows. Outermost safety net. |
| `AuthorizationBehaviour<T,R>` | Reads `[Authorize]` off the request; enforces roles/policies. |
| `ValidationBehaviour<T,R>` | Runs all `IValidator<TRequest>`; throws `ValidationException`. |
| `PerformanceBehaviour<T,R>` | Stopwatch; warns on requests over 500 ms. |

`ValidationBehaviour` is the one to study:

```csharp
public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next,
    CancellationToken cancellationToken)
{
    if (_validators.Any())
    {
        var validationResults = await Task.WhenAll(
            _validators.Select(v => v.ValidateAsync(new ValidationContext<TRequest>(request), cancellationToken)));

        var failures = validationResults.Where(r => r.Errors.Any()).SelectMany(r => r.Errors).ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);
    }
    return await next();
}
```

**Why it matters:** no handler ever validates its own input. Validation is declarative
(a separate `AbstractValidator` class) and enforced centrally. A handler that starts with
`if (string.IsNullOrEmpty(request.Title)) throw ...` is a rule violation.

`AuthorizationBehaviour` reflects over `[Authorize]` attributes on the *request type*, checks
`_user.Id != null`, then role membership, then policies via `IIdentityService`. Throws
`UnauthorizedAccessException` or `ForbiddenAccessException`.

### 3.4 Wire it up — `DependencyInjection.cs`

```csharp
namespace Microsoft.Extensions.DependencyInjection;

public static class DependencyInjection
{
    public static void AddApplicationServices(this IHostApplicationBuilder builder)
    {
        builder.Services.AddAutoMapper(cfg => cfg.AddMaps(Assembly.GetExecutingAssembly()));
        builder.Services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

        builder.Services.AddMediatR(cfg => {
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
            cfg.AddOpenRequestPreProcessor(typeof(LoggingBehaviour<>));
            cfg.AddOpenBehavior(typeof(UnhandledExceptionBehaviour<,>));
            cfg.AddOpenBehavior(typeof(AuthorizationBehaviour<,>));
            cfg.AddOpenBehavior(typeof(ValidationBehaviour<,>));
            cfg.AddOpenBehavior(typeof(PerformanceBehaviour<,>));
        });
    }
}
```

Note the namespace trick: declaring `namespace Microsoft.Extensions.DependencyInjection` means
`Program.cs` gets `builder.AddApplicationServices()` with no extra `using`. **Order is
deliberate** — exception handling wraps auth, which wraps validation, which wraps performance.

### 3.5 Feature slices — the CQRS folder convention

```
TodoItems/
  Commands/
    CreateTodoItem/
      CreateTodoItem.cs                  ← command record + handler, one file
      CreateTodoItemCommandValidator.cs
    UpdateTodoItem/
    UpdateTodoItemDetail/
    DeleteTodoItem/
  EventHandlers/
    LogTodoItemCompleted.cs
TodoLists/
  Commands/...
  Queries/
    GetTodos/
      GetTodos.cs
      TodosVm.cs
      TodoListDto.cs
      TodoItemDto.cs
```

**Vertical slices, not horizontal layers.** Everything for "create a todo item" lives in one folder.

### 3.6 A command, in full

`TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs`:

```csharp
public record CreateTodoItemCommand : IRequest<int>
{
    public int ListId { get; init; }
    public string? Title { get; init; }
}

public class CreateTodoItemCommandHandler : IRequestHandler<CreateTodoItemCommand, int>
{
    private readonly IApplicationDbContext _context;

    public CreateTodoItemCommandHandler(IApplicationDbContext context) => _context = context;

    public async Task<int> Handle(CreateTodoItemCommand request, CancellationToken cancellationToken)
    {
        var entity = new TodoItem
        {
            ListId = request.ListId,
            Title = request.Title,
            Done = false
        };

        _context.TodoItems.Add(entity);
        await _context.SaveChangesAsync(cancellationToken);

        return entity.Id;
    }
}
```

**Observe what is absent:** no validation (behaviour), no logging (behaviour), no try/catch
(behaviour), no auth check (behaviour), no `IApplicationDbContext` concrete type. The handler is
~10 lines of pure use-case logic. **That is the target for every handler you write.**

Its validator sits beside it:

```csharp
public class CreateTodoItemCommandValidator : AbstractValidator<CreateTodoItemCommand>
{
    public CreateTodoItemCommandValidator()
    {
        RuleFor(v => v.Title).MaximumLength(200).NotEmpty();
    }
}
```

### 3.7 A query with projection

`TodoLists/Queries/GetTodos/GetTodos.cs` shows `[Authorize]` on the request and `ProjectTo`:

```csharp
[Authorize]
public record GetTodosQuery : IRequest<TodosVm>;

// in Handle:
Lists = await _context.TodoLists
    .AsNoTracking()
    .ProjectTo<TodoListDto>(_mapper.ConfigurationProvider)
    .OrderBy(t => t.Title)
    .ToListAsync(cancellationToken)
```

**Two non-negotiables for queries:**
1. `.AsNoTracking()` — reads never need the change tracker.
2. `.ProjectTo<TDto>()` — translates to SQL `SELECT` of only the needed columns. Never
   `.ToListAsync()` then `_mapper.Map<>()`; that fetches whole entities and maps in memory.

### 3.8 DTO-owned mapping profiles

Put the AutoMapper `Profile` *inside* the DTO as a private nested class:

```csharp
public class TodoListDto
{
    public TodoListDto() { Items = []; }

    public int Id { get; init; }
    public string? Title { get; init; }
    public string? Colour { get; init; }
    public IReadOnlyCollection<TodoItemDto> Items { get; init; }

    private class Mapping : Profile
    {
        public Mapping() { CreateMap<TodoList, TodoListDto>(); }
    }
}
```

**Why:** the mapping lives next to the shape it produces. Delete the DTO, the mapping goes with it.
No central `MappingProfile.cs` god-file that everyone edits and conflicts on.

> **Verify:** `dotnet build src/Application` succeeds. Grep for `Npgsql`, `DbContext` (concrete),
> or `HttpContext` in `src/Application` — all must return zero hits.

---

## Phase 4 — Infrastructure Layer (adapters; implements Application's ports)

```powershell
dotnet new classlib -o src/Infrastructure
dotnet sln CrmApp.slnx add src/Infrastructure/Infrastructure.csproj
dotnet add src/Infrastructure reference src/Application
```

### 4.1 `Data/ApplicationDbContext.cs`

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>, IApplicationDbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }

    public DbSet<TodoList> TodoLists => Set<TodoList>();
    public DbSet<TodoItem> TodoItems => Set<TodoItem>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);   // ← Identity's own mappings; must come first
        builder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}
```

`ApplyConfigurationsFromAssembly` auto-discovers every `IEntityTypeConfiguration<T>` — you never
maintain a registration list.

### 4.2 `Data/Configurations/` — Fluent API, one file per entity

```csharp
public class TodoItemConfiguration : IEntityTypeConfiguration<TodoItem>
{
    public void Configure(EntityTypeBuilder<TodoItem> builder)
    {
        builder.Property(t => t.Title).HasMaxLength(200).IsRequired();
    }
}
```

**Rule:** persistence constraints live *here*, never as `[MaxLength]` attributes on the domain
entity. That is what keeps Domain persistence-ignorant.

`TodoListConfiguration` additionally converts the `Colour` value object:

```csharp
builder.Property(t => t.Colour)
    .HasConversion(c => c.Code, code => Colour.From(code))
    .HasMaxLength(7);
```

### 4.3 `Data/Interceptors/AuditableEntityInterceptor.cs`

Populates `Created`/`CreatedBy`/`LastModified`/`LastModifiedBy` on every save:

```csharp
foreach (var entry in context.ChangeTracker.Entries<BaseAuditableEntity>())
{
    if (entry.State is EntityState.Added or EntityState.Modified || entry.HasChangedOwnedEntities())
    {
        var utcNow = _dateTime.GetUtcNow();
        if (entry.State == EntityState.Added)
        {
            entry.Entity.CreatedBy = _user.Id;
            entry.Entity.Created = utcNow;
        }
        entry.Entity.LastModifiedBy = _user.Id;
        entry.Entity.LastModified = utcNow;
    }
}
```

Injects `TimeProvider` (not `DateTime.UtcNow`) — that is what makes time mockable in tests.
`HasChangedOwnedEntities()` catches the case where only an owned value object changed, which EF
otherwise reports as `Unchanged` on the parent.

### 4.4 `Data/Interceptors/DispatchDomainEventsInterceptor.cs`

This is the bridge from Phase 2's `AddDomainEvent` to actual handlers:

```csharp
public async Task DispatchDomainEvents(DbContext? context)
{
    if (context == null) return;

    var entities = context.ChangeTracker.Entries<BaseEntity>()
        .Where(e => e.Entity.DomainEvents.Any())
        .Select(e => e.Entity);

    var domainEvents = entities.SelectMany(e => e.DomainEvents).ToList();

    entities.ToList().ForEach(e => e.ClearDomainEvents());   // ← clear BEFORE publish

    foreach (var domainEvent in domainEvents)
        await _mediator.Publish(domainEvent);
}
```

**Subtle and important:** events are collected and cleared *before* publishing. If a handler
triggers another save, the same event cannot be dispatched twice.

### 4.5 Identity

- `Identity/ApplicationUser.cs` — `public class ApplicationUser : IdentityUser { }`
- `Identity/IdentityService.cs` — implements `IIdentityService`, wrapping `UserManager` and
  `IAuthorizationService`. **This is why `IIdentityService` exists**: `UserManager<ApplicationUser>`
  is an ASP.NET type that Application must never see.
- `Identity/IdentityResultExtensions.cs` — maps `IdentityResult` → the Application-layer `Result`.

### 4.6 `DependencyInjection.cs`

```csharp
public static void AddInfrastructureServices(this IHostApplicationBuilder builder)
{
    var connectionString = builder.Configuration.GetConnectionString(Services.Database);
    Guard.Against.Null(connectionString, message: $"Connection string '{Services.Database}' not found.");

    builder.Services.AddScoped<ISaveChangesInterceptor, AuditableEntityInterceptor>();
    builder.Services.AddScoped<ISaveChangesInterceptor, DispatchDomainEventsInterceptor>();

    builder.Services.AddDbContext<ApplicationDbContext>((sp, options) =>
    {
        options.AddInterceptors(sp.GetServices<ISaveChangesInterceptor>());
        options.UseNpgsql(connectionString);
        options.ConfigureWarnings(w => w.Ignore(RelationalEventId.PendingModelChangesWarning));
    });

    builder.EnrichNpgsqlDbContext<ApplicationDbContext>();   // Aspire: health checks, retries, tracing

    builder.Services.AddScoped<IApplicationDbContext>(p => p.GetRequiredService<ApplicationDbContext>());
    builder.Services.AddScoped<ApplicationDbContextInitialiser>();

    builder.Services.AddAuthentication(options =>
    {
        options.DefaultScheme = IdentityConstants.ApplicationScheme;
        options.DefaultSignInScheme = IdentityConstants.ExternalScheme;
    }).AddIdentityCookies();

    builder.Services.AddAuthorizationBuilder();

    builder.Services.AddIdentityCore<ApplicationUser>()
        .AddRoles<IdentityRole>()
        .AddEntityFrameworkStores<ApplicationDbContext>()
        .AddSignInManager()
        .AddDefaultTokenProviders()
        .AddApiEndpoints();

    builder.Services.AddSingleton(TimeProvider.System);
    builder.Services.AddTransient<IIdentityService, IdentityService>();
}
```

The `AddScoped<IApplicationDbContext>(p => p.GetRequiredService<ApplicationDbContext>())` line is
the port-to-adapter binding — one `DbContext` instance serving both interfaces per scope.

### 4.7 `Data/ApplicationDbContextInitialiser.cs`

Seeds the `Administrator` role, the `administrator@localhost` / `Administrator1!` user, and a sample
todo list.

> ⚠️ **Know this behaviour:** `InitialiseAsync()` calls `EnsureDeletedAsync()` then
> `EnsureCreatedAsync()` — **the database is dropped and recreated on every dev startup.** Fine for
> a template; if you want persistence between runs, switch to `MigrateAsync()` and generate EF
> migrations instead.

> **Verify:** `dotnet build src/Infrastructure`. Confirm Application does *not* reference
> Infrastructure (the dependency arrow points inward only).

---

## Phase 5 — Shared & ServiceDefaults

### 5.1 `src/Shared/Services.cs` — resource-name constants

```csharp
public static class Services
{
    public const string WebFrontend = "webfrontend";
    public const string WebApi = "webapi";
    public const string DatabaseServer = "dbserver";
    public const string Database = "CrmAppDb";
}
```

**Why a whole project for four strings:** AppHost registers a resource under a name, and
Infrastructure looks up a connection string by that same name. A typo would only surface at
runtime. Sharing the constant makes it a compile-time error instead.

### 5.2 `src/ServiceDefaults/Extensions.cs`

The standard Aspire shared-configuration project: OpenTelemetry (traces/metrics/logs), health check
endpoints (`/health`, `/alive`), service discovery, and HTTP resilience handlers. Exposes
`builder.AddServiceDefaults()` and `app.MapDefaultEndpoints()`.

Health endpoints are mapped **only in Development** — do not expose them publicly without auth.

---

## Phase 6 — Web Layer (Minimal APIs)

```powershell
dotnet new web -o src/Web
dotnet add src/Web reference src/Application src/Infrastructure src/ServiceDefaults
```

### 6.1 The endpoint discovery mechanism

`Infrastructure/IEndpointGroup.cs`:

```csharp
public interface IEndpointGroup
{
    static virtual string? RoutePrefix => null;      // defaults to /api/{ClassName}
    static abstract void Map(RouteGroupBuilder groupBuilder);
}
```

Uses C# 11 static abstract interface members. `WebApplicationExtensions.MapEndpoints` scans the
assembly, finds implementations, creates `app.MapGroup(routePrefix).WithTags(groupName)`, and
invokes `Map`. **Adding a new endpoint group requires zero changes to `Program.cs`.**

`EndpointRouteBuilderExtensions` wraps `MapGet`/`MapPost`/etc. to auto-derive
`.WithName(handler.Method.Name)` — that name becomes the OpenAPI `operationId`, which becomes the
**method name in the NSwag-generated TypeScript client**. Rename a handler → the frontend method
renames. `Guard.Against.AnonymousMethod(handler)` enforces named static methods so this always works.

Note the signature asymmetry: `pattern` is optional for GET/POST (collection-level) and **required**
for PUT/PATCH/DELETE (resource-level, needs `{id}`).

### 6.2 `Endpoints/TodoLists.cs`

```csharp
public class TodoLists : IEndpointGroup
{
    public static void Map(RouteGroupBuilder groupBuilder)
    {
        groupBuilder.RequireAuthorization();

        groupBuilder.MapGet(GetTodoLists);
        groupBuilder.MapPost(CreateTodoList);
        groupBuilder.MapPut(UpdateTodoList, "{id}");
        groupBuilder.MapDelete(DeleteTodoList, "{id}");
    }

    [EndpointSummary("Get all Todo Lists")]
    [EndpointDescription("Retrieves all todo lists along with their items.")]
    public static async Task<Ok<TodosVm>> GetTodoLists(ISender sender)
    {
        var vm = await sender.Send(new GetTodosQuery());
        return TypedResults.Ok(vm);
    }

    [EndpointSummary("Update a Todo List")]
    public static async Task<Results<NoContent, BadRequest>> UpdateTodoList(
        ISender sender, int id, UpdateTodoListCommand command)
    {
        if (id != command.Id) return TypedResults.BadRequest();
        await sender.Send(command);
        return TypedResults.NoContent();
    }
    // ...
}
```

**The endpoint contract:** inject `ISender`, build/forward the request, return `TypedResults`.
The only permitted logic is HTTP-shaped concerns like the `id != command.Id` route/body consistency
check. **No business logic, no `DbContext`, no `if` on domain state.**

Return concrete `TypedResults` types (`Ok<T>`, `Created<int>`, `Results<NoContent, BadRequest>`) —
these are what let OpenAPI describe response schemas accurately without manual `.Produces<>()` calls.

### 6.3 `Endpoints/Users.cs`

```csharp
groupBuilder.MapIdentityApi<ApplicationUser>();
groupBuilder.MapPost(Logout, "logout").RequireAuthorization();
```

One line gives you register/login/refresh/confirm-email/2FA. The custom `Logout` is added because
`MapIdentityApi` does not ship one for cookie auth.

### 6.4 Error handling

`Infrastructure/ProblemDetailsExceptionHandler.cs` implements `IExceptionHandler`, translating
Application exceptions to RFC 7807 responses:

| Exception | Status |
|---|---|
| `ValidationException` | 400 + `errors` dictionary |
| `UnauthorizedAccessException` | 401 |
| `ForbiddenAccessException` | 403 |
| `NotFoundException` / `KeyNotFoundException` | 404 |

Registered with `builder.Services.AddExceptionHandler<...>()` + `app.UseExceptionHandler(options => { })`.
The empty lambda is required — without it ASP.NET expects a fallback path.

**This is the payoff for throwing exceptions in the Application layer**: handlers stay clean, and
one adapter owns the HTTP translation.

### 6.5 `Services/CurrentUser.cs`

```csharp
public class CurrentUser(IHttpContextAccessor accessor) : IUser
{
    public string? Id => accessor.HttpContext?.User?.FindFirstValue(ClaimTypes.NameIdentifier);
    public IEnumerable<string>? Roles => /* claims of type Role */;
}
```

The adapter for the `IUser` port. `HttpContext` stops here and never leaks inward.

### 6.6 `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();
builder.AddKeyVaultIfConfigured();
builder.AddApplicationServices();
builder.AddInfrastructureServices();
builder.AddWebServices();

var app = builder.Build();

if (app.Environment.IsDevelopment())
    await app.InitialiseDatabaseAsync();
else
    app.UseHsts();

app.UseHttpsRedirection();
app.UseCors(static b => b.AllowAnyMethod().AllowAnyHeader().AllowAnyOrigin());
app.UseFileServer();

app.MapOpenApi();
app.MapScalarApiReference();

app.UseExceptionHandler(options => { });

app.MapDefaultEndpoints();
app.MapEndpoints(typeof(Program).Assembly);
app.MapFallbackToFile("index.html");

app.Run();
```

Five `Add*Services` calls, one per layer, in dependency order. `Program.cs` stays under 50 lines
because each layer owns its own registration.

> ⚠️ **Production note:** `AllowAnyOrigin()` is wide open. Restrict this before deploying.

> **Verify:** `dotnet build`. Then `dotnet run --project src/Web` and hit `/scalar` — you should see
> your endpoints with the summaries you wrote.

---

## Phase 7 — Aspire Orchestration

### 7.1 `src/AppHost/Program.cs`

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddAzureContainerAppEnvironment("aca-env");

var databaseServer = builder
    .AddAzurePostgresFlexibleServer(Services.DatabaseServer)
    .WithPasswordAuthentication()
    .RunAsContainer(container => container.WithLifetime(ContainerLifetime.Persistent))
    .AddDatabase(Services.Database);

var web = builder.AddProject<Projects.Web>(Services.WebApi)
    .WithReference(databaseServer)
    .WaitFor(databaseServer)
    .WithExternalHttpEndpoints()
    .WithAspNetCoreEnvironment()
    .WithUrlForEndpoint("http", url =>
    {
        url.DisplayText = "Scalar API Reference";
        url.Url = "/scalar";
    });

if (builder.ExecutionContext.IsRunMode)
{
    builder.AddJavaScriptApp(Services.WebFrontend, "./../Web/ClientApp")
        .WithRunScript("start")
        .WithReference(web)
        .WaitFor(web)
        .WithHttpEndpoint(env: "PORT")
        .WithExternalHttpEndpoints();
}

builder.Build().Run();
```

**Key concepts:**
- `RunAsContainer(...Persistent)` — locally it's a Docker container that **survives restarts**;
  when published to Azure it becomes a real Flexible Server. Same code, two targets.
- `WithReference` injects the connection string; `WaitFor` blocks startup until the DB is healthy.
- `IsRunMode` guard — the Vite dev server runs only locally; in production the SPA is built into
  `wwwroot` by the `PublishRunWebpack` MSBuild target in `Web.csproj`.

### 7.2 `launchSettings.json`

```json
"https": {
  "commandName": "Project",
  "launchBrowser": true,
  "applicationUrl": "https://crmapp.dev.localhost:17215;http://crmapp.dev.localhost:15083",
  "environmentVariables": {
    "ASPNETCORE_ENVIRONMENT": "Development",
    "DOTNET_ENVIRONMENT": "Development",
    "DOTNET_DASHBOARD_OTLP_ENDPOINT_URL": "https://localhost:21120",
    "DOTNET_RESOURCE_SERVICE_ENDPOINT_URL": "https://localhost:22042"
  }
}
```

Dashboard lands on **https://crmapp.dev.localhost:17215**. All `*.localhost` names resolve to
127.0.0.1, so no hosts-file edit is needed. On first run the console prints a
`Login to the dashboard at ...?t=<token>` URL — you need that token.

> **Verify:** `dotnet run --project ./src/AppHost` → dashboard shows `dbserver`, `CrmAppDb`,
> `webapi`, and `webfrontend` all green.

---

## Phase 8 — React Client

```powershell
cd src/Web
npm create vite@latest ClientApp -- --template react
cd ClientApp
npm install @picocss/pico lucide-react react-router-dom
npm install -D nswag sass typescript @types/node
```

### 8.1 The generated API client — the critical piece

`package.json`:

```json
"scripts": {
  "prestart": "npm run generate-api",
  "start": "vite",
  "prebuild": "npm run generate-api",
  "build": "vite build",
  "generate-api": "nswag run /runtime:Net100"
}
```

`prestart`/`prebuild` are npm lifecycle hooks — they run **automatically** before `start`/`build`.
So every dev-server launch regenerates `src/web-api-client.ts` from the live OpenAPI document.

**The full contract chain:**

```
Handler method name  →  .WithName()  →  OpenAPI operationId  →  NSwag  →  TS client method
    C# return type   →  TypedResults →  OpenAPI schema      →  NSwag  →  TS interface
```

Rename `GetTodoLists` in C# and the TypeScript compiler will error at the call site. This is
**end-to-end type safety with no hand-written API types**, and it is why the endpoint conventions in
Phase 6 are strict.

`Web.csproj` emits the document at build time:

```xml
<OpenApiDocumentsDirectory>./wwwroot/openapi/</OpenApiDocumentsDirectory>
<OpenApiGenerateDocumentsOptions>--file-name v1</OpenApiGenerateDocumentsOptions>
```

### 8.2 Component structure

```
src/
  main.jsx, App.jsx, AppRoutes.jsx, styles.scss
  web-api-client.ts               ← GENERATED; never edit, never commit conflicts into it
  components/
    Layout.jsx, NavMenu.jsx, Home.jsx, Counter.jsx, Weather.jsx, Todo.jsx
    ThemeContext.jsx, ThemeToggle.jsx
    api-authorization/
      AuthContext.jsx, LoginPage.jsx, RegisterPage.jsx, ProtectedRoute.jsx
```

`AuthContext` holds session state; `ProtectedRoute` wraps authenticated routes and redirects to
`/login`. Auth uses the **cookie** issued by `MapIdentityApi`, so `fetch` calls need
`credentials: 'include'`.

> **Verify:** `npm run generate-api` produces a non-empty `web-api-client.ts` containing a
> `getTodoLists` method (API must be running).

---

## Phase 9 — Test Suite

Six projects, each with a distinct job:

| Project | Scope | Speed |
|---|---|---|
| `Domain.UnitTests` | Value objects, entity invariants. No mocks needed. | ms |
| `Application.UnitTests` | Behaviours, validators, AutoMapper config. Moq for ports. | ms |
| `Infrastructure.IntegrationTests` | Interceptors, EF configurations against a real DB. | seconds |
| `Application.FunctionalTests` | Full vertical slice via MediatR against real Postgres. | seconds |
| `Web.AcceptanceTests` | Reqnroll (BDD) + Playwright through the browser. | minutes |
| `TestAppHost` | Cut-down Aspire host — **DB only**, no web/frontend. | n/a |

### 9.1 `Domain.UnitTests` — the simplest possible test

```csharp
[Test]
public void ShouldReturnCorrectColourCode()
{
    var code = "#FFFFFF";
    var colour = Colour.From(code);
    colour.Code.ShouldBe(code);
}
```

No DI, no mocks, no database. **If your domain tests need mocks, your domain has dependencies it
shouldn't have.**

### 9.2 `Application.UnitTests/Common/Mappings/MappingTests.cs`

```csharp
[Test]
public void ShouldHaveValidConfiguration() => _configuration!.AssertConfigurationIsValid();

[TestCase(typeof(TodoList), typeof(TodoListDto))]
[TestCase(typeof(TodoItem), typeof(TodoItemDto))]
public void ShouldSupportMappingFromSourceToDestination(Type source, Type destination)
{
    var instance = GetInstanceOf(source);
    _mapper!.Map(instance, source, destination);
}
```

`AssertConfigurationIsValid()` catches every unmapped destination property across the whole
assembly. **Highest value-per-line test in the solution** — add a DTO property and forget the
mapping, this fails immediately.

### 9.3 `Application.FunctionalTests` — real database, real pipeline

`FunctionalTestSetup.cs` (`[SetUpFixture]`, runs once):

```csharp
var builder = await DistributedApplicationTestingBuilder
    .CreateAsync<Projects.TestAppHost>(args: [], configureBuilder: (options, _) =>
    {
        options.DisableDashboard = true;
    });

builder.Configuration["ASPIRE_ALLOW_UNSECURED_TRANSPORT"] = "true";

_app = await builder.BuildAsync(ct).WaitAsync(ct);
await _app.StartAsync(ct).WaitAsync(ct);
await _app.ResourceNotifications.WaitForResourceHealthyAsync(Services.Database, ct);

var connectionString = (await _app.GetConnectionStringAsync(Services.Database))!;
_factory = new WebApiFactory(connectionString);
ScopeFactory = _factory.Services.GetRequiredService<IServiceScopeFactory>();
DbResetter = await DatabaseResetter.CreateAsync(connectionString);
```

`TestApp.cs` provides the ergonomics: `SendAsync(request)`, `RunAsDefaultUserAsync()`,
`RunAsAdministratorAsync()`, `FindAsync<T>()`, `AddAsync<T>()`, `CountAsync<T>()`, `ResetState()`.

`TestBase` resets before **every** test via Respawn:

```csharp
public abstract class TestBase
{
    [SetUp]
    public async Task SetUp() => await TestApp.ResetState();
}
```

A typical test:

```csharp
public class CreateTodoItemTests : TestBase
{
    [Test]
    public async Task ShouldRequireMinimumFields()
    {
        var command = new CreateTodoItemCommand();
        await Should.ThrowAsync<ValidationException>(() => TestApp.SendAsync(command));
    }

    [Test]
    public async Task ShouldCreateTodoItem()
    {
        var userId = await TestApp.RunAsDefaultUserAsync();
        var listId = await TestApp.SendAsync(new CreateTodoListCommand { Title = "New List" });

        var itemId = await TestApp.SendAsync(new CreateTodoItemCommand { ListId = listId, Title = "Tasks" });

        var item = await TestApp.FindAsync<TodoItem>(itemId);

        item.ShouldNotBeNull();
        item.ListId.ShouldBe(listId);
        item.CreatedBy.ShouldBe(userId);        // ← proves the audit interceptor ran
        item.Created.ShouldBeGreaterThan(DateTimeOffset.MinValue);
    }
}
```

**This is the most valuable test type here.** It exercises validation behaviour, authorization
behaviour, the handler, EF, interceptors, and domain events — against real PostgreSQL, with no mocks.

### 9.4 `Web.AcceptanceTests` — BDD through the browser

`Features/Login.feature`:

```gherkin
Feature: Login
  Scenario: Successful login
    Given I am on the login page
    When I log in as "administrator@localhost" with password "Administrator1!"
    Then I should be redirected to the home page
```

Step definitions drive Playwright via the Page Object Model (`Pages/LoginPage.cs` etc.).
`AspireSetup.cs` boots the full AppHost; `PlaywrightSetup.cs` manages the browser lifecycle.

```powershell
# One-time: install browser binaries
pwsh artifacts/bin/Web.AcceptanceTests/debug/playwright.ps1 install
```

### 9.5 Running tests

```powershell
dotnet test                                              # everything
dotnet test tests/Domain.UnitTests                       # fast inner loop
dotnet test --filter "FullyQualifiedName~TodoItems"      # one feature
```

> **Verify:** `dotnet test` — all green. Docker must be running for functional/acceptance tests.

---

## Phase 10 — Adding a Feature (the repeatable workflow)

Once the skeleton exists, **every** new feature follows this exact loop. Worked example:
*"Add a `DueDate` to TodoItem and let users filter by overdue items."*

```
1. DOMAIN      src/Domain/Entities/TodoItem.cs
               + public DateTimeOffset? DueDate { get; set; }
               If a rule attaches (e.g. raise TodoItemOverdueEvent), put it in the setter.

2. CONFIG      src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs
               builder.Property(t => t.DueDate);
               (drop/recreate happens automatically in dev)

3. COMMAND     src/Application/TodoItems/Commands/UpdateTodoItemDetail/UpdateTodoItemDetail.cs
               + public DateTimeOffset? DueDate { get; init; }
               + entity.DueDate = request.DueDate;

4. VALIDATOR   ...Commands/UpdateTodoItemDetail/UpdateTodoItemDetailCommandValidator.cs
               RuleFor(v => v.DueDate).GreaterThan(_ => DateTimeOffset.UtcNow)
                   .When(v => v.DueDate.HasValue);

5. QUERY/DTO   src/Application/TodoLists/Queries/GetTodos/TodoItemDto.cs
               + public DateTimeOffset? DueDate { get; init; }
               AutoMapper picks it up by convention (name match).

6. ENDPOINT    Usually NOTHING to change — the command record is the request body.
               Only touch src/Web/Endpoints/ if you need a NEW route.

7. TEST        tests/Application.FunctionalTests/TodoItems/Commands/
                   UpdateTodoItemDetailTests.cs
               + ShouldRejectPastDueDate, ShouldPersistDueDate

8. FRONTEND    cd src/Web/ClientApp && npm run generate-api
               web-api-client.ts now exposes dueDate; update Todo.jsx.

9. VERIFY      dotnet test && dotnet run --project ./src/AppHost
```

**Scaffolding shortcut** (from `src/Application/`):

```powershell
dotnet new ca-usecase --name CreateTodoList --feature-name TodoLists --usecase-type command --return-type int
dotnet new ca-usecase -n GetTodos -fn TodoLists -ut query -rt TodosVm

# If "No templates matching 'ca-usecase'":
dotnet new install Clean.Architecture.Solution.Template::10.8.0
```

### The direction of change

Notice the flow is always **inside-out**: Domain → Application → Infrastructure/Web → Client.
If you find yourself starting at the endpoint and working inward, you are probably about to put
business logic in the wrong layer.

---

## Phase 11 — Run, Seed & Log In

```powershell
dotnet run --project ./src/AppHost
```

1. Aspire dashboard opens at **https://crmapp.dev.localhost:17215** (use the `?t=<token>` URL from
   the console on first run).
2. Postgres container starts; `webapi` waits for it to be healthy.
3. In Development, `Program.cs` calls `InitialiseDatabaseAsync()` → drops, recreates, and seeds.
4. Click the `webfrontend` URL in the dashboard.

**Seeded credentials:**

| Field | Value |
|---|---|
| Email | `administrator@localhost` |
| Password | `Administrator1!` |
| Role | `Administrator` |

Also available: `/scalar` for interactive API docs, and `src/Web/Web.http` for a scripted
login-then-call flow (cookie name `.AspNetCore.Identity.Application`).

---

## Phase 12 — Hardening Checklist

Items the template leaves as exercises. Address before any real deployment:

- [ ] **CORS** — replace `AllowAnyOrigin()` in `Program.cs` with an explicit origin list.
- [ ] **Migrations** — swap `EnsureDeleted`/`EnsureCreated` for `dotnet ef migrations add` +
      `MigrateAsync()`. Required for any environment with data you care about.
- [ ] **NuGet audit debt** — remove `NU1901;NU1902;NU1903;NU1904` from `WarningsNotAsErrors` and
      pin patched versions of `Microsoft.OpenApi`, `System.Security.Cryptography.Xml`,
      `MessagePack`, and `OpenTelemetry.*`.
- [ ] **Seed credentials** — never ship `Administrator1!`. Gate seeding on `IsDevelopment()` (already
      done) and use Key Vault for real environments (`AddKeyVaultIfConfigured` is wired).
- [ ] **Rate limiting** — add `AddRateLimiter` on the Identity endpoints.
- [ ] **Health endpoints** — currently Development-only; add auth before exposing publicly.
- [ ] **`PendingModelChangesWarning`** — currently ignored; remove the suppression once on migrations.
- [ ] **CI** — `dotnet build` + `dotnet test` on PR; ensure a Docker-enabled agent.
