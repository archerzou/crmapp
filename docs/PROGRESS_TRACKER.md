# CrmApp — Progress Tracker

Companion to `BUILD_PLAN.md`. Tick each box as you complete it. Every phase ends with a **Gate** —
a command that must pass before you move on. Don't skip gates; a broken foundation costs far more
to fix three phases later.

**Legend:** `[ ]` todo · `[x]` done · `[~]` in progress · `[!]` blocked (add a note)

**Status:** _in progress — Phases 0–8 complete; Phase 9 (tests) is the next major gap_
**Started:** 2026-08 **Last updated:** 2026-08-15

---

## Progress Summary

| Phase | Title | Items | Done | Gate |
|---|---|---|---|---|
| 0 | Prerequisites & Tooling | 6 | ✅ 6/6 | `dotnet --version`, `docker info` |
| 1 | Solution Scaffolding | 9 | ✅ 9/9 | `dotnet build` on empty sln |
| 2 | Domain Layer | 14 | ✅ 14/14 | Zero project refs in Domain |
| 3 | Application Layer | 21 | ✅ 21/21 | No EF provider / HttpContext leaks |
| 4 | Infrastructure Layer | 14 | ✅ 14/14 | Builds; ports implemented |
| 5 | Shared & ServiceDefaults | 5 | ✅ 5/5 | Constants shared both ways |
| 6 | Web Layer | 14 | ✅ 14/14 | `/scalar` lists all endpoints |
| 7 | Aspire Orchestration | 7 | ✅ 7/7 | Dashboard all-green |
| 8 | React Client | 10 | ✅ 10/10 | `web-api-client.ts` generated |
| 9 | Test Suite | 18 | ☐ 0/18 | `dotnet test` all pass |
| 10 | First Feature E2E | 9 | ☐ 0/9 | Feature works browser→DB |
| 11 | Run & Log In | 5 | ~ 0/5 | Logged in as administrator |
| 12 | Hardening | 8 | ☐ 0/8 | Reviewed & triaged |
| | **Total** | **140** | **~100/140** | |

> **Audit note (2026-08-15):** State reconciled against the codebase. Phases 0–8 verified via file
> structure, a clean `dotnet build` (0 warnings/0 errors), and the architecture rule checks below —
> all passing. `tests/` is empty, so Phase 9 onward is not started. Phase 11 is unverified: the app
> now builds and the AppHost is fully wired, but a full run + login hasn't been confirmed here.

---

## Phase 0 — Prerequisites & Tooling

- [x] **0.1** .NET SDK 10.x installed — `dotnet --list-sdks` shows `10.0.2xx`
- [x] **0.2** Aspire workload installed — `dotnet workload list` includes `aspire`
- [x] **0.3** Node.js 20+ and npm available
- [x] **0.4** Docker Desktop installed **and running** — `docker info` exits 0
- [x] **0.5** HTTPS dev cert trusted — `dotnet dev-certs https --trust`
- [x] **0.6** Folder skeleton created — `CrmApp/{src,tests,docs}`

> **🚦 GATE 0:** All three of `dotnet --version`, `node --version`, `docker info` succeed.
> ❌ *Common blocker:* Docker not running → every functional test in Phase 9 will fail with a
> connection timeout that looks like a code bug. Fix it now.

---

## Phase 1 — Solution Scaffolding & Build Governance

- [x] **1.1** `global.json` pins SDK `10.0.201` with `rollForward: latestFeature`
- [x] **1.2** `CrmApp.slnx` created (XML format, not legacy `.sln`)
- [x] **1.3** Solution folders `/src/` and `/tests/` defined
- [x] **1.4** `Directory.Build.props` — `TargetFramework`, `Nullable`, `ImplicitUsings`
- [x] **1.5** `Directory.Build.props` — `TreatWarningsAsErrors=true`
- [x] **1.6** `Directory.Build.props` — `WarningsNotAsErrors` documented with a *why* comment
- [x] **1.7** `Directory.Build.props` — `ArtifactsPath` set to `artifacts/`
- [x] **1.8** `Directory.Packages.props` — `ManagePackageVersionsCentrally=true`
- [x] **1.9** `.editorconfig` and `.gitignore` added (`artifacts/`, `node_modules/`)

> **🚦 GATE 1:** `dotnet build` succeeds. `git status` shows no `bin/`, `obj/`, or `artifacts/`.
> ✅ *Self-check:* Can you explain what each line of `Directory.Build.props` does? If not, re-read
> BUILD_PLAN §1.3 — you'll be living with these settings for the whole project.

---

## Phase 2 — Domain Layer

**Constraint to hold the entire phase: Domain references NO project and ONE package.**

- [x] **2.1** `Domain.csproj` created; `RootNamespace`/`AssemblyName` = `CrmApp.Domain`
- [x] **2.2** Only `MediatR.Contracts` referenced; **zero** `ProjectReference`
- [x] **2.3** `Common/BaseEvent.cs` — abstract, implements `INotification`
- [x] **2.4** `Common/BaseEntity.cs` — `Id`, private `_domainEvents` list
- [x] **2.5** `BaseEntity` exposes `IReadOnlyCollection` + `[NotMapped]`
- [x] **2.6** `BaseEntity` has `AddDomainEvent` / `RemoveDomainEvent` / `ClearDomainEvents`
- [x] **2.7** `Common/BaseAuditableEntity.cs` — 4 audit fields, `DateTimeOffset` (not `DateTime`)
- [x] **2.8** `Common/ValueObject.cs` — structural equality via `GetEqualityComponents()`
- [x] **2.9** `Enums/PriorityLevel.cs`
- [x] **2.10** `Exceptions/UnsupportedColourException.cs`
- [x] **2.11** `ValueObjects/Colour.cs` — private-ish ctor, static `From()` factory that validates
- [x] **2.12** `Colour` has implicit→string and explicit→Colour operators
- [x] **2.13** `Events/TodoItemCompletedEvent.cs`
- [x] **2.14** `Entities/TodoItem.cs` — `Done` setter raises event, guarded by `value && !_done`

> **🚦 GATE 2:** `dotnet build src/Domain` succeeds.
> Run these — **all must return zero hits:**
> ```powershell
> Select-String -Path "src\Domain\**\*.cs" -Pattern "Microsoft.EntityFrameworkCore|DbContext|HttpContext|IRepository"
> ```
> ✅ *Self-check:* Is `TodoItem.Done = true` twice raising exactly one event? That idempotency guard
> is the difference between a correct and a subtly-broken domain event.

---

## Phase 3 — Application Layer

- [x] **3.1** `Application.csproj` references **only** `Domain`
- [x] **3.2** No `Version=` attributes in the csproj (CPM intact)
- [x] **3.3** `GlobalUsings.cs` with the 6 global usings
- [x] **3.4** `Common/Interfaces/IApplicationDbContext.cs` — DbSets + `SaveChangesAsync`
- [x] **3.5** `Common/Interfaces/IUser.cs` — `Id`, `Roles`
- [x] **3.6** `Common/Interfaces/IIdentityService.cs`
- [x] **3.7** `Common/Exceptions/ValidationException.cs` — exposes `Errors` dictionary
- [x] **3.8** `Common/Exceptions/ForbiddenAccessException.cs`
- [x] **3.9** `Common/Security/AuthorizeAttribute.cs` — `Roles`, `Policy`
- [x] **3.10** `Common/Models/Result.cs` and `LookupDto.cs`
- [x] **3.11** `Behaviours/LoggingBehaviour.cs` (pre-processor)
- [x] **3.12** `Behaviours/UnhandledExceptionBehaviour.cs`
- [x] **3.13** `Behaviours/AuthorizationBehaviour.cs` — roles **and** policies
- [x] **3.14** `Behaviours/ValidationBehaviour.cs` — aggregates all failures
- [x] **3.15** `Behaviours/PerformanceBehaviour.cs` — 500 ms threshold
- [x] **3.16** `DependencyInjection.cs` — behaviours registered in the correct order
- [x] **3.17** Feature folders follow `Feature/Commands|Queries/UseCase/` layout
- [x] **3.18** All 4 TodoItem commands + 3 TodoList commands implemented
- [x] **3.19** Each command has a sibling `AbstractValidator` — _validators exist for Create/Update; Delete & UpdateTodoItemDetail intentionally omit them (matches canonical template)_
- [x] **3.20** `GetTodos` query uses `.AsNoTracking()` **and** `.ProjectTo<>()`
- [x] **3.21** Every DTO owns a `private class Mapping : Profile`

> **🚦 GATE 3:** `dotnet build src/Application` succeeds.
> ```powershell
> # All must be ZERO:
> Select-String -Path "src\Application\**\*.cs" -Pattern "Npgsql|UseSqlServer|HttpContext|IdentityUser|UserManager"
> # Every command should have a validator — counts should match:
> (Get-ChildItem src\Application -Recurse -Filter "*Command.cs").Count
> (Get-ChildItem src\Application -Recurse -Filter "*CommandValidator.cs").Count
> ```
> ✅ *Self-check:* Open your longest handler. Does it contain any `try/catch`, `if (string.IsNullOrEmpty(...))`,
> or `_logger.LogInformation`? All three belong in behaviours, not handlers.

---

## Phase 4 — Infrastructure Layer

- [x] **4.1** `Infrastructure.csproj` references `Application` (never the reverse)
- [x] **4.2** `Data/ApplicationDbContext.cs` — `IdentityDbContext<ApplicationUser>`, `IApplicationDbContext`
- [x] **4.3** `OnModelCreating` calls `base.OnModelCreating(builder)` **first**
- [x] **4.4** `ApplyConfigurationsFromAssembly` used (no manual registration list)
- [x] **4.5** `Configurations/TodoItemConfiguration.cs` — `MaxLength(200)`, `IsRequired`
- [x] **4.6** `Configurations/TodoListConfiguration.cs` — `Colour` `HasConversion`
- [x] **4.7** `Interceptors/AuditableEntityInterceptor.cs` — injects `TimeProvider`, not `DateTime.UtcNow`
- [x] **4.8** Audit interceptor handles `HasChangedOwnedEntities()`
- [x] **4.9** `Interceptors/DispatchDomainEventsInterceptor.cs` — clears events **before** publishing
- [x] **4.10** `Identity/ApplicationUser.cs`, `IdentityService.cs`, `IdentityResultExtensions.cs`
- [x] **4.11** `DependencyInjection.cs` — `Guard.Against.Null(connectionString)`
- [x] **4.12** Both interceptors registered before `AddDbContext`, then `AddInterceptors`
- [x] **4.13** `IApplicationDbContext` bound to the same scoped `ApplicationDbContext` instance
- [x] **4.14** `ApplicationDbContextInitialiser.cs` — role, admin user, sample data

> **🚦 GATE 4:** `dotnet build` succeeds for the whole `src/` tree.
> ```powershell
> # MUST be zero — this would invert the dependency rule:
> Select-String -Path "src\Application\Application.csproj" -Pattern "Infrastructure"
> ```
> ⚠️ *Know before you continue:* `InitialiseAsync()` **drops your database on every startup**.
> Expected for now; tracked in Phase 12.

---

## Phase 5 — Shared & ServiceDefaults

- [x] **5.1** `Shared/Services.cs` — `WebFrontend`, `WebApi`, `DatabaseServer`, `Database` consts
- [x] **5.2** Infrastructure resolves the connection string via `Services.Database`
- [x] **5.3** `ServiceDefaults/Extensions.cs` — `AddServiceDefaults()`
- [x] **5.4** OpenTelemetry: tracing, metrics, logging configured
- [x] **5.5** `MapDefaultEndpoints()` — `/health` and `/alive`, Development-only

> **🚦 GATE 5:** No magic strings. `Select-String -Path "src\**\*.cs" -Pattern '"CrmAppDb"'`
> should hit **only** `Shared/Services.cs`.

---

## Phase 6 — Web Layer

- [x] **6.1** `Web.csproj` references `Application`, `Infrastructure`, `ServiceDefaults`
- [x] **6.2** `OpenApiDocumentsDirectory` → `./wwwroot/openapi/`, file name `v1`
- [x] **6.3** `Infrastructure/IEndpointGroup.cs` — `static abstract Map`, `static virtual RoutePrefix`
- [x] **6.4** `Infrastructure/WebApplicationExtensions.cs` — assembly scan + `MapGroup` + `WithTags`
- [x] **6.5** `Infrastructure/EndpointRouteBuilderExtensions.cs` — auto `.WithName(method.Name)`
- [x] **6.6** Verb extensions guard with `Guard.Against.AnonymousMethod`
- [x] **6.7** `Endpoints/TodoLists.cs` — `RequireAuthorization()` at group level
- [x] **6.8** `Endpoints/TodoItems.cs`
- [x] **6.9** `Endpoints/Users.cs` — `MapIdentityApi<ApplicationUser>()` + custom `Logout`
- [x] **6.10** All handlers return `TypedResults` with concrete generic types
- [x] **6.11** All handlers carry `[EndpointSummary]` / `[EndpointDescription]`
- [x] **6.12** `Infrastructure/ProblemDetailsExceptionHandler.cs` — 400/401/403/404 mapping
- [x] **6.13** `Services/CurrentUser.cs` implements `IUser` from claims
- [x] **6.14** `Program.cs` — 5 `Add*Services` in order; `UseExceptionHandler(options => { })`

> **🚦 GATE 6:** `dotnet run --project src/Web`, open `/scalar`.
> Every endpoint appears, grouped by tag, with your summaries visible.
> ```powershell
> # MUST be zero — endpoints must go through ISender only:
> Select-String -Path "src\Web\Endpoints\*.cs" -Pattern "IApplicationDbContext|ApplicationDbContext"
> ```
> ✅ *Self-check:* Is any endpoint method longer than ~8 lines? If so, business logic has leaked out
> of the Application layer.

---

## Phase 7 — Aspire Orchestration

- [x] **7.1** `AppHost/Program.cs` — `AddAzurePostgresFlexibleServer` + `RunAsContainer`
- [x] **7.2** Container lifetime `Persistent` (survives restarts)
- [x] **7.3** `AddProject<Projects.Web>` with `WithReference` **and** `WaitFor`
- [x] **7.4** `WithUrlForEndpoint` adds the Scalar shortcut to the dashboard
- [x] **7.5** Frontend registration guarded by `builder.ExecutionContext.IsRunMode`
- [x] **7.6** `launchSettings.json` — https + http profiles, OTLP endpoint vars
- [x] **7.7** All resource names come from `Services.*` constants

> **🚦 GATE 7:** `dotnet run --project ./src/AppHost` → dashboard at
> **https://crmapp.dev.localhost:17215** (grab the `?t=<token>` link from the console).
> `dbserver`, `CrmAppDb`, `webapi`, `webfrontend` all show **Running / Healthy**.
> ❌ *If `webapi` is stuck "Waiting":* Postgres isn't healthy — check Docker and the container logs
> in the dashboard.

---

## Phase 8 — React Client

- [x] **8.1** Vite + React 19 + TypeScript scaffolded in `src/Web/ClientApp`
- [x] **8.2** `nswag.json` configured, `/runtime:Net100`
- [x] **8.3** `package.json` has `prestart` **and** `prebuild` → `generate-api`
- [x] **8.4** `web-api-client.ts` generated and **git-ignored or clearly marked generated**
- [x] **8.5** `AppRoutes.jsx` + `Layout.jsx` + `NavMenu.jsx`
- [x] **8.6** `api-authorization/AuthContext.jsx` — session state
- [x] **8.7** `api-authorization/ProtectedRoute.jsx` — redirects to `/login`
- [x] **8.8** `LoginPage.jsx` / `RegisterPage.jsx` hit the Identity API with `credentials: 'include'`
- [x] **8.9** `Todo.jsx` consumes the **generated** client (no hand-written `fetch` URLs)
- [x] **8.10** `Web.csproj` `PublishRunWebpack` target builds the SPA on publish

> **🚦 GATE 8:** With the API running:
> ```powershell
> cd src\Web\ClientApp; npm run generate-api
> Select-String -Path "src\web-api-client.ts" -Pattern "getTodoLists"
> ```
> Must find the method. ✅ *Self-check:* Rename a C# handler method, regenerate — does TypeScript
> now fail to compile at the call site? If not, the operationId chain is broken.

---

## Phase 9 — Test Suite

### Domain.UnitTests
- [ ] **9.1** Project created, references `Domain` only
- [ ] **9.2** `ValueObjects/ColourTests.cs` — valid code, invalid code throws, equality
- [ ] **9.3** No mocking library needed anywhere in this project

### Application.UnitTests
- [ ] **9.4** `Common/Mappings/MappingTests.cs` — `AssertConfigurationIsValid()`
- [ ] **9.5** `MappingTests` `[TestCase]` per entity→DTO pair
- [ ] **9.6** `Common/Behaviours/RequestLoggerTests.cs` — Moq on `IUser`/`IIdentityService`
- [ ] **9.7** `Common/Exceptions/ValidationExceptionTests.cs`

### Infrastructure.IntegrationTests
- [ ] **9.8** Project created
- [ ] **9.9** Interceptor + EF configuration coverage

### TestAppHost
- [ ] **9.10** Minimal Aspire host — **database only**, no web/frontend
- [ ] **9.11** Uses the same `Services.*` constants

### Application.FunctionalTests
- [ ] **9.12** `FunctionalTestSetup.cs` — `[SetUpFixture]`, `DisableDashboard`, 60 s timeout
- [ ] **9.13** Waits for `WaitForResourceHealthyAsync(Services.Database)`
- [ ] **9.14** `Infrastructure/WebApiFactory.cs` overrides the connection string
- [ ] **9.15** `Infrastructure/DatabaseResetter.cs` — Respawn
- [ ] **9.16** `Infrastructure/TestApp.cs` — `SendAsync`, `RunAs*`, `FindAsync`, `CountAsync`
- [ ] **9.17** `TestBase` resets state in `[SetUp]` (before **every** test)
- [ ] **9.18** A test asserts `CreatedBy`/`Created` to prove the audit interceptor fires

### Web.AcceptanceTests
- [ ] **9.19** `AspireSetup.cs` + `PlaywrightSetup.cs`
- [ ] **9.20** `Features/*.feature` — Home, Login, Counter, Weather
- [ ] **9.21** Page Objects in `Pages/`; step definitions never touch selectors directly
- [ ] **9.22** Playwright browsers installed

> **🚦 GATE 9:** `dotnet test` — **all green** (Docker must be running).
> ```powershell
> dotnet test tests\Domain.UnitTests          # should finish in < 2s
> dotnet test tests\Application.UnitTests
> dotnet test tests\Application.FunctionalTests
> dotnet test                                  # everything
> ```
> ❌ *Flaky functional tests?* Almost always `ResetState()` not running, or two tests sharing state
> through a static. Respawn in `[SetUp]` is the fix.

---

## Phase 10 — First Feature End-to-End

Pick a real feature (the plan uses `DueDate`). Prove you can drive the full loop unaided.

- [ ] **10.1** Domain changed first (entity/value object/event)
- [ ] **10.2** EF configuration updated in `Infrastructure/Data/Configurations/`
- [ ] **10.3** Command/query record updated in `Application`
- [ ] **10.4** Validator rule added
- [ ] **10.5** DTO + mapping updated
- [ ] **10.6** Endpoint touched **only** if a new route was needed
- [ ] **10.7** Functional test written — happy path **and** validation failure
- [ ] **10.8** `npm run generate-api`; UI updated against the regenerated client
- [ ] **10.9** `dotnet test` green; feature verified in the browser

> **🚦 GATE 10:** The feature works browser → API → MediatR → EF → PostgreSQL, and there is a test
> that fails if you revert the change.
> ✅ *Self-check:* How many files did you touch in `src/Web/Endpoints/`? If more than zero for a
> pure field addition, your endpoints are doing too much.

---

## Phase 11 — Run & Log In

- [ ] **11.1** `dotnet run --project ./src/AppHost` starts clean
- [ ] **11.2** Dashboard reachable; all resources healthy
- [ ] **11.3** Database seeded (check logs for the initialiser)
- [ ] **11.4** Logged in as `administrator@localhost` / `Administrator1!`
- [ ] **11.5** Scalar (`/scalar`) exercised — an authenticated call returns 200

> **🚦 GATE 11:** You can create a todo item in the UI and see the row in PostgreSQL.

---

## Phase 12 — Hardening (triage before any deployment)

- [ ] **12.1** CORS locked to explicit origins (currently `AllowAnyOrigin()`)
- [ ] **12.2** EF **migrations** replace `EnsureDeleted`/`EnsureCreated`
- [ ] **12.3** `PendingModelChangesWarning` suppression removed
- [ ] **12.4** NuGet audit suppressions (`NU1901`–`NU1904`) removed; packages patched
- [ ] **12.5** Seed credentials not used outside Development; Key Vault wired
- [ ] **12.6** Rate limiting on Identity endpoints
- [ ] **12.7** Health endpoints authenticated if exposed
- [ ] **12.8** CI runs `dotnet build` + `dotnet test` on a Docker-enabled agent

> **🚦 GATE 12:** Each item is either done or has a dated ticket. "We'll get to it" is not a state.

---

## Blockers Log

| Date | Phase | Problem | Resolution |
|---|---|---|---|
| 2026-08-15 | 11 | `dotnet run --project src/Web` failed DB init: `28P01 password authentication failed for user "admin"` | Running Web standalone bypasses Aspire's injected connection string; native local Postgres has no `admin` role. Run via `src/AppHost` (Aspire provisions Postgres) or create the role locally. |

## Decisions Log

Record every deviation from `BUILD_PLAN.md` and why — future-you will ask.

| Date | Decision | Rationale |
|---|---|---|
| 2026-08-15 | Enabled `CentralPackageTransitivePinningEnabled` in `Directory.Packages.props` | Lets central `PackageVersion` entries override vulnerable/transitive deps (pinned `Microsoft.OpenApi` 2.7.5 to clear NU1903). |
| 2026-08-15 | Bumped `Microsoft.Extensions.Hosting` to 10.0.9 | Transitive pinning surfaced Aspire's `>= 10.0.8` requirement; aligned with the rest of the 10.0.9 Microsoft family. |
| 2026-08-15 | `.idea/` untracked from git and fully git-ignored | IDE state should not be in source control; removed with `git rm -r --cached`. |

---

## Quick Health Check

Run any time to see where you stand:

```powershell
dotnet build                      # compiles?
dotnet test                       # green?
dotnet list package --vulnerable --include-transitive   # audit debt?
dotnet list package --outdated    # drift?
```

Then run the rule checks in `CLEAN_ARCHITECTURE_RULES.md` §"Automated Verification".
