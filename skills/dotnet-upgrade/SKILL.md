---
name: dotnet-upgrade
description: Backend-only. Detect a .NET service's current setup (.NET Framework 4.x or .NET Core / .NET 5+, ASP.NET MVC / Web API / Core, WCF, EF6 / EF Core, DI container, auth, hosting), move it to legacy/ and rebuild it project by project on the latest .NET LTS and ASP.NET Core, with full API and behaviour parity, contract tests recorded against legacy, a switchable mock layer for external integrations, and security fixes limited to this backend. Use for any .NET Framework → .NET or .NET version upgrade or migration.
---

# .NET Framework / .NET → Latest LTS Upgrade

Goal: work out what the service runs on today and rebuild it beside a `legacy/` copy on the latest .NET LTS and ASP.NET Core, without changing what the service does.

## Non-negotiables

These hold for the whole upgrade and override every default below, including the scaffold.

1. **Functional parity.** Every endpoint, background job, scheduled task, message consumer, SOAP operation and startup task must behave the same afterwards.
2. **Preserve business logic.** Change APIs and syntax required by the upgrade, never the logic. Keep calculations, rules, conditions, transactions and their ordering exactly. If logic looks wrong, flag it to the user; don't fix it.
3. **Preserve the API contract.** Same routes, HTTP methods, model binding results, status codes, headers, cookies, JSON and XML shape (property names and casing, null handling, date, enum and number formats), validation error format and error response bodies.
4. **Preserve application behaviour.** Same auth and session handling, same DB schema and queries, same outbound calls (URLs, payloads, timeouts, retries), same config keys and connection string names, same logging of business events.
5. **Backend only.** Change only this service's backend code, build and config. Frontend files the service serves (Razor views' markup, `wwwroot` / `Content` / `Scripts` assets) are ported unchanged, apart from syntax needed to compile. Never change frontend apps, other services, the database schema, infrastructure, CI/CD or deployment. If parity seems to need such a change, stop and ask.
6. **Security fixes are allowed** (see "Vulnerability fixes"), but only within this backend and without breaking rules 1–5.

Any deviation, however small, must be listed in the step report with its reason and approved by the user.

## Precedence

1. The non-negotiables above
2. Explicit user instructions
3. The scaffold project, if the user provides one
4. The defaults in this skill

## Target (defaults)

| Area | Default |
|---|---|
| Versions | Latest **LTS** .NET and ASP.NET Core, the matching C# language version, and the latest compatible package versions. Look them up when you start (dotnet.microsoft.com support policy, `dotnet --list-sdks`) and never assume from memory. Pin the SDK in `global.json`. |
| Approach | Move the current code to `legacy/`, create a fresh solution at the repo root, and port project by project and feature by feature, applying the migration notes below. `legacy/` is the read-only reference and the parity baseline. |
| Project format | SDK-style `.csproj`, `PackageReference`. Central Package Management (`Directory.Packages.props`) for multi-project solutions. |
| Hosting | `WebApplication.CreateBuilder` (minimal hosting) in `Program.cs`, with controllers. Don't convert controllers to Minimal APIs unless the user asks. |
| Language | Modern C# features only in code you otherwise touch, and only when behaviour-neutral. Nullable reference types enabled with warnings, not errors, unless the user asks otherwise. |
| JSON | Whatever produces the *same* output as legacy. For Web API 2 / Newtonsoft-based services that means `AddNewtonsoftJson()` with the same settings. Switching to System.Text.Json is a separate step, only if the user asks. |
| Data access | Keep the current technology at the latest version that runs on modern .NET: EF6 (6.4+) runs on .NET, Dapper and ADO.NET carry over. Moving EF6 → EF Core is a separate, opt-in project. No schema changes and no new migrations. |
| DI | Built-in container. Keep Autofac (or another container with modern .NET support) if registrations rely on its features. Same lifetimes. |
| External integrations | Behind interfaces, with a switchable mock layer (Step 5) |
| Auth | The existing model and flow, ported as-is |
| Deployment | Unchanged. If the hosting model (IIS module, Windows service, container image, runtime) changes, tell the user; don't change CI/CD yourself. |

## Scaffold project

If the user provides a scaffold or reference project, read it first. Take from it the solution and project layout, `Directory.Build.props`, analyzers and code style, logging and error-handling patterns, test setup and naming. Where it differs from this skill, follow it (except for the non-negotiables). List the conventions you adopted in your Step 1 report.

## Rules

- Detect and audit first. Report before changing code.
- Port one project, module or feature per change. After each step the build, tests and app startup must pass before moving on.
- Never edit `legacy/`. Read it to port from.
- Don't add packages beyond what the upgrade requires and what the scaffold uses. Ask first.
- When unsure about an API or version, check Microsoft Learn (the migration guides and "Breaking changes in .NET N" pages) or the package changelog. Do not guess.

## Git commits (save points)

Commit at every logical save point, so any change can be reverted on its own with `git revert <sha>`.

- **Branch:** use the branch the user names. If none is named, ask before the first commit.
- **When to commit:** after each completed unit of work that builds and passes its tests. Typical units are the audit report (if saved to the repo), the `legacy/` move, the new project setup, the API/mock layer, auth, each store or module, each feature or route group, each version step, and each security fix. Never let uncommitted work pile up across several features.
- **One concern per commit.** Don't mix a refactor, a feature port and a security fix in one commit.
- **The `legacy/` move is its own commit,** containing only `git mv` renames and no content changes, so Git keeps file history.
- **Every security fix is its own commit,** so it can be reverted independently.
- **Every commit must leave the project building and its tests passing.** If that's impossible for a step, say so in the commit body.
- **Message format** (Conventional Commits):
  ```
  <type>(<scope>): <short summary, imperative, ≤ 72 chars>

  What changed and why.
  Parity: what was verified (contract tests, manual checks).
  Deviations: none | <description + user approval>
  Refs: <feature inventory item / scanner rule ID / migration guide section>
  ```
  Types: `chore` (moves, setup, deps), `feat` (feature ported), `fix` (bug or security fix), `refactor`, `test`, `build`, `docs`.
  Examples: `chore(legacy): move existing app to legacy/`, `feat(orders): port OrdersController endpoints`, `fix(security): parameterize order search query (CodeQL cs/sql-injection)`.
- **Never commit** secrets, real `.env` files, credentials, `bin/`/`obj/`, build output or real customer data.
- **Never rewrite shared history:** no amending, rebasing or force-pushing commits that have been pushed.
- **Push** only if the user asks, or if that's the repo's established practice.
- **Milestone tags:** tag key points so they're easy to roll back to (e.g. `upgrade/legacy-moved`, `upgrade/parity-complete`, `upgrade/pre-cleanup`). Push tags only if the user asks.

## Step 0 — Detect the current setup

| Check | Where to look |
|---|---|
| Target framework | `<TargetFramework(s)>` / `<TargetFrameworkVersion>` in `.csproj`; `global.json` |
| Project format | SDK-style vs legacy verbose `.csproj`; `packages.config` vs `PackageReference` |
| App type | `Global.asax` + `System.Web` (MVC 5 / Web API 2 / Web Forms `.aspx`), OWIN `Startup`, `.svc` (WCF), ASP.NET Core `Program.cs`/`Startup.cs`, console / Windows service / worker |
| Config | `Web.config` / `App.config` (`appSettings`, `connectionStrings`, custom sections, transforms), `appsettings*.json` |
| Data access | EF6 (code-first or EDMX), EF Core, Dapper, ADO.NET, stored procedures |
| DI | Unity, Autofac, Ninject, SimpleInjector, StructureMap, built-in |
| Auth | Forms auth, OWIN cookie, ASP.NET Identity (version), Windows auth, JWT bearer, OAuth/OIDC, custom `HttpModule` or `DelegatingHandler` |
| Pipeline | `HttpModule`s, `HttpHandler`s, `DelegatingHandler`s, global filters, `Application_*` events, OWIN middleware |
| Serialization | Newtonsoft settings (contract resolver, date handling, `TypeNameHandling`), XML formatters, `DataContract` |
| Background work | Hangfire, Quartz, `HostingEnvironment.QueueBackgroundWorkItem`, Windows services, timers |
| Logging | log4net, NLog, Serilog, `System.Diagnostics.Trace` |
| Windows-only APIs | `System.Drawing`, registry, WMI, `System.DirectoryServices`, COM interop, `AppDomain`, Remoting, `BinaryFormatter` |
| Tests | MSTest, NUnit, xUnit, Moq, coverage level |
| Hosting | IIS (classic/integrated), self-host, Windows service, containers, Azure App Service |

Report the version gap (e.g. .NET Framework 4.7.2 / Web API 2 → latest .NET LTS / ASP.NET Core), which migration notes from Step 4 apply, and any blockers. **Web Forms pages, WCF server features without CoreWCF support, and Windows-only APIs on a non-Windows target are blockers: report them first and ask how to proceed.**

## Step 1 — Audit (report only, no edits)

1. **Endpoints:** every route (attribute and convention), with its controller action, model binding sources, response type and shape, status codes, filters and auth applied. Include SOAP operations for WCF.
2. **Outbound integrations:** each HTTP API, SOAP client, queue, email or SMS provider, file storage or other external service, with its client code, config keys, timeouts and retries.
3. **Auth:** scheme(s), cookie names and settings, machine key or data protection setup, Identity schema version and password hasher, token validation parameters, roles, claims and policies.
4. **Pipeline:** modules, handlers, delegating handlers, filters and OWIN middleware, **in execution order**.
5. **Background work:** jobs, schedules, services, consumers.
6. **Config:** every `appSettings` key, connection string name and custom section, plus the env and transform overrides.
7. **Deployment:** build and publish commands, artifact, hosting model, runtime, IIS settings. Record these; don't change them.
8. **Feature inventory** (the parity baseline): every endpoint and job, its business rules, side effects (DB writes, outbound calls, emails) and error cases.
9. **Security findings:** `dotnet list package --vulnerable`, or whichever scanner the user names, plus manual checks from "Vulnerability fixes". Give each finding a severity and a proposed fix.
10. Conventions taken from the scaffold, if provided.
11. A proposed porting order (shared libraries first, then the host, then features from smallest to largest risk).

## Step 2 — Move existing code to `legacy/`

1. Use `git mv` so file history is kept.
2. **Keep at the repo root:** `.git`, CI/CD config, deploy and hosting config (Dockerfile, pipeline YAML, publish profiles used by CI, k8s manifests and similar), env and secret templates, `README`, `LICENSE`. Before moving, list what stays and what moves, and confirm with the user.
3. Move everything else (the solution, projects, `packages/`, tests) into `legacy/`.
4. Make sure the new solution and tooling ignore `legacy/`: it isn't in the new `.sln`/`.slnx`, and is excluded from `Directory.Build.props` globbing, analyzers and scanners.
5. `legacy/` is a read-only reference. Never edit it, and don't delete it until the user approves. It can still build and run on its own (on its original framework) for side-by-side comparison.
6. Root deploy config still describes the legacy build. List what will need to change at cut-over for the user; don't change it yourself.

## Step 3 — Contract tests (the parity safety net)

- Run the legacy tests and record the results as the baseline (pre-existing failures listed).
- In the new solution, add a **black-box HTTP contract test project** (e.g. xUnit + `HttpClient`) covering each endpoint in the feature inventory: status, relevant headers, cookies, and the exact JSON or XML body (approved snapshots are fine). Include edge cases for model binding, validation errors, casing, nulls, dates and enums. The tests take the target from `CONTRACT_BASE_URL`.
- **Record the snapshots against the running legacy app.** Don't add a mock toggle to `legacy/`. If its downstream URLs come from config, point them at a local stub server (e.g. WireMock.Net) that serves the same fixtures as the new app's mocks. Otherwise record against the dev environment.
- Run the same tests against the new app (mocks on) for every ported feature. They must pass unchanged. Changing a snapshot means a deviation, which needs user approval.

## Step 4 — New solution and porting

1. Create the new solution at the root from the scaffold if one was provided. Otherwise use `dotnet new` templates (`webapi` with controllers, `classlib`, `worker`, `xunit`) on the latest LTS, with the same project and namespace names as legacy.
2. Port in this order, one change at a time:
   1. Shared class libraries (domain, utilities). These are often near copy-paste; multi-target only if the user asks.
   2. Config: `appsettings.json` with the same keys (flatten `appSettings` as top-level keys, or use the options pattern with the same names), the same `ConnectionStrings` names, and environment-specific files replacing transforms.
   3. DI registrations with the same lifetimes.
   4. Pipeline: middleware, filters and handlers, **in the same order** as legacy.
   5. Error handling, with the same error response format and status codes.
   6. Auth (Step 6).
   7. Outbound clients and mocks (Step 5).
   8. Data access: the same models, mappings and queries. Point at the existing database. No new migrations.
   9. Controllers and endpoints, feature by feature, with the same routes.
   10. Razor views and static assets, copied unchanged except for syntax needed to compile.
   11. Background jobs, workers and consumers.
3. Run the contract tests for each ported feature.

### Migration notes: .NET Framework / ASP.NET → ASP.NET Core

| Legacy | ASP.NET Core | Parity note |
|---|---|---|
| `Global.asax`, OWIN `Startup` | `Program.cs` with `WebApplication.CreateBuilder` | Move `Application_Start` logic to startup code, in the same order |
| `Web.config` `appSettings` / `connectionStrings` | `appsettings.json` + env vars, `IConfiguration` / options | Same key and connection string names |
| `HttpContext.Current` | inject `IHttpContextAccessor` or pass `HttpContext` | |
| `HttpModule` / `HttpHandler` / `DelegatingHandler` | middleware / endpoints | Same order and short-circuit behaviour |
| Web API 2 `ApiController` | `[ApiController]` + `ControllerBase` | `[ApiController]` returns automatic 400 ProblemDetails on invalid model state. Set `SuppressModelStateInvalidFilter = true` and keep the legacy validation response. Binding source inference also differs, so add explicit `[FromBody]`/`[FromQuery]` where legacy relied on Web API rules. |
| MVC 5 `Controller` | `Controller` (MVC) | `ActionResult` types and HTML helpers mostly carry over; check `ChildAction` → view components |
| Convention routes (`MapHttpRoute`, `MapRoute`) | `MapControllerRoute` with the same templates; attribute routes unchanged | Check trailing slashes, case and optional segments in the contract tests |
| Web API 2 JSON (Newtonsoft, PascalCase by default) | `AddNewtonsoftJson()` with the same settings | System.Text.Json defaults to camelCase and different date, enum and null handling, so don't switch implicitly |
| XML formatter | `AddXmlSerializerFormatters()` / `AddXmlDataContractSerializerFormatters()` matching legacy | |
| `IHttpActionResult` / `HttpResponseMessage` | `IActionResult` / `ActionResult<T>` | Same status codes and headers |
| Exception filters, `ExceptionHandler` | exception-handling middleware or filters | Same error body |
| Bundling (`System.Web.Optimization`) | static files with the same output URLs (or the scaffold's bundler) | Frontend: port unchanged |
| `Session` (System.Web) | `ISession` + distributed cache | Stores bytes/strings only, so serialize objects the same way. Flag if behaviour can't match. |
| Forms auth / OWIN cookie auth | cookie authentication with the same cookie name, path, expiry and sliding settings | Existing cookies won't decrypt (different ticket format). Report this as a cut-over impact. Shared-cookie interop is possible but needs user approval. |
| ASP.NET Identity 2 | ASP.NET Core Identity mapped onto the **existing tables**, with the password hasher in V2-compatible mode | If parity needs a schema change, stop and ask |
| Windows auth | `Negotiate` authentication (or IIS Windows auth) | |
| WCF service | CoreWCF with the same contracts, bindings and addresses | Check for unsupported bindings and report them |
| WCF client | `System.ServiceModel.*` client packages or regenerated clients (`dotnet-svcutil`) | Same endpoints and timeouts |
| Windows service | Worker service + `UseWindowsService()` | Same schedule and service name (the service name is deployment, so report it) |
| `HostingEnvironment.QueueBackgroundWorkItem` | `BackgroundService` / hosted service | |
| log4net / NLog / Serilog | the same library with its ASP.NET Core integration | Same sinks, formats and levels |
| `BinaryFormatter`, Remoting, AppDomains, CAS | removed. Replace with the existing alternative serializer for the same payload, or report as a blocker. | |
| `System.Drawing` on non-Windows | keep Windows hosting, or report | Changing the hosting OS is deployment |
| Unity / Ninject / StructureMap | built-in DI (or Autofac) | Same lifetimes and named or keyed registrations (keyed services exist in modern .NET) |
| `packages.config` | `PackageReference`, latest compatible versions | Check each package's changelog for changed defaults |

### Migration notes: .NET Core / .NET 5+ → latest LTS

- Read "Breaking changes in .NET N" (runtime, ASP.NET Core, EF Core) for every version between legacy and target, and apply what affects the ported code.
- `Startup.cs` can be kept via the generic host or folded into `Program.cs`; either is behaviour-neutral. Follow the scaffold.
- Replace obsolete APIs flagged by the compiler (`SYSLIB*` and `ASPDEPR*` warnings) with their recommended equivalents, keeping behaviour.
- EF Core major upgrades: check query translation changes (e.g. split vs single queries, `Contains` translation, value converters) in the contract tests.
- Check `System.Text.Json` behaviour changes between versions (e.g. property ordering, reference handling, number handling). Pin the old options explicitly where output differs.

## Step 5 — Mock layer for external integrations

The goal is to run and test the service without any downstream system and without changing other services or deployment.

- Put each outbound integration behind an interface. Where legacy creates `HttpClient`/`WebClient` or SOAP clients inline, port it as a typed client class that makes exactly the same calls.
- Register the real or mock implementation in one place, chosen by one config value.

```json
// appsettings.Development.json (default false everywhere else)
{ "MockExternals": false }
```

```csharp
// Program.cs (excerpt)
var mockExternals = builder.Configuration.GetValue<bool>("MockExternals");

if (mockExternals)
{
    builder.Services.AddSingleton<IPaymentClient, MockPaymentClient>();
    // one line per integration
}
else
{
    builder.Services.AddHttpClient<IPaymentClient, HttpPaymentClient>(c =>
    {
        // same base address, headers and timeout as legacy
    });
}

var app = builder.Build();
if (mockExternals)
    app.Logger.LogWarning("MOCK EXTERNALS ENABLED — no calls go to real downstream systems");
```

Mock rules:
- Toggle: `MockExternals=true dotnet run`, or set it in `launchSettings.json` / `appsettings.Development.json`. The default is off.
- Mock every outbound integration from the audit. Mocks return fixtures (e.g. `Mocks/Fixtures/*.json`, copied to output) with the real response shapes, and can simulate error cases (timeouts, 4xx/5xx, SOAP faults).
- Fake data only: no real customer data, tokens or secrets.
- The database stays real (local or dev). Swapping in an in-memory provider changes query behaviour, so only do it if the user asks.
- Contract tests run with mocks on.

## Step 6 — Auth (port the existing flow, don't redesign it)

- Keep the same schemes, cookie names and options, token validation parameters (issuer, audience, signing keys, clock skew), claims mapping (`MapInboundClaims` affects claim names, so match legacy), roles and policies.
- Keep existing password hashes verifiable: use the hasher compatibility mode that matches the legacy Identity version.
- Data protection: if cookies or tokens must survive the cut-over, that needs shared keys or interop. Report the options; don't decide alone.
- If the new framework rejects the current setup (e.g. a weak key or an unsupported scheme), stop and tell the user. Changing keys invalidates sessions and tokens.
- Don't introduce a new auth library or model unless the user asks.

## Vulnerability fixes

Fix issues reported by common scans (`dotnet list package --vulnerable`, Snyk, SonarQube, Semgrep, CodeQL, Security Code Scan / .NET analyzers) when the fix stays inside this backend and keeps parity.

| Finding | Fix |
|---|---|
| Vulnerable packages | Patched version with the same API (the upgrade covers most) |
| SQL injection (string-built `SqlCommand` / raw SQL) | Parameters (`SqlParameter`, Dapper params, `FromSqlInterpolated`) with identical SQL semantics |
| `BinaryFormatter` / `NetDataContractSerializer` / Newtonsoft `TypeNameHandling.All/Auto` on untrusted input | Remove type-name handling or restrict it with a `SerializationBinder`, keeping the same payload format |
| XXE (`XmlDocument`/`XmlTextReader` with a resolver) | `XmlResolver = null`, `DtdProcessing.Prohibit` |
| Path traversal (files from request input) | `Path.GetFullPath` + check against the base directory |
| Open redirect (`Redirect(returnUrl)`) | `Url.IsLocalUrl` / `LocalRedirect`; otherwise fall back to the current default |
| SSRF (user-controlled outbound URLs) | Restrict to the hosts the feature legitimately uses |
| Command injection (`Process.Start` with input) | Argument lists, no shell; validate input |
| Hardcoded secrets or connection string passwords | Move to configuration (env vars, user secrets in dev, the existing secret store). Tell the user to rotate anything that was committed. |
| Secrets or PII in logs | Remove or mask them |
| JWT validation gaps (no signature, issuer or lifetime check) | Enable the missing validation with the parameters legacy intended |
| Weak random for tokens (`System.Random`) | `RandomNumberGenerator` with the same output format |

Rules for security fixes:
- A fix must not change the API contract or business logic, except to block the exploit itself.
- **Report but don't change** findings whose fix changes behaviour clients rely on, or that live outside this service: adding antiforgery validation, CORS or security-header changes, HTTPS redirection or HSTS, password hashing scheme changes, cookie flag changes, TLS and infrastructure. Describe the risk and the recommended fix for the user to decide.
- **Note for .NET Framework sources:** ASP.NET request validation (blocking HTML in input) doesn't exist in ASP.NET Core. Report it, check that Razor output encoding covers each place input is rendered, and ask before adding any input filtering.
- List every fix with the scanner rule or ID, the file, and the change.

## Verify after each step

- [ ] `dotnet build` with no errors and no new warnings (treat new analyzer warnings as findings)
- [ ] Existing tests and contract tests pass unchanged (any snapshot change approved)
- [ ] App starts with the production configuration (`dotnet run -c Release` / published output)
- [ ] Works with mocks on and against real dev dependencies (if the user can run them)
- [ ] Auth: login, cookies or tokens, protected endpoints, roles and policies behave the same
- [ ] Background jobs, workers and consumers start and run

## Done checklist

- [ ] On the latest .NET LTS (versions recorded in the report; SDK pinned in `global.json`)
- [ ] Feature inventory fully checked off; any deviations approved
- [ ] Every outbound integration mockable; the toggle works through `MockExternals`
- [ ] Only backend code changed (no frontend apps, schema, other-service, infra or CI/CD changes)
- [ ] Scan findings fixed or reported, and scans re-run show no new issues
- [ ] Deployment changes (runtime, hosting model, publish command, service name) listed for the user, not applied
- [ ] All work committed at logical save points; the working tree is clean
- [ ] Mocks and `legacy/` kept; they're removed only through the cleanup step, when the user asks

## Final step — Cleanup (only when the user asks)

Never do this automatically, and never as part of another step. Only start it when the user explicitly asks, after the done checklist is complete. The user may ask for both parts or just one.

1. Confirm the scope with the user (mocks, `legacy/`, or both) and list exactly what will be removed.
2. Tag the current commit first (e.g. `upgrade/pre-cleanup`) so everything can be restored.
3. **Remove the mocks** (its own commit, e.g. `chore(mocks): remove mock API layer`):
   - Delete the `Mock*` client classes and `Mocks/Fixtures/`.
   - In `Program.cs`, remove the `mockExternals` branch and warning, keeping only the real registrations.
   - Remove `MockExternals` from `appsettings*.json` and `launchSettings.json`.
   - Keep the real implementations and their behaviour exactly as they are.
   - Tests that relied on mocks: ask the user whether to convert them to test-only doubles inside the test folder, or remove them. Don't silently delete test coverage.
   - Remove the mock flag from `.env.example` and docs. Leave real `.env` files and deployment config to the user, and tell them which variable is no longer used.
4. **Remove `legacy/`** (its own commit, e.g. `chore(legacy): remove legacy app after cut-over`):
   - `git rm -r legacy/`.
   - Remove `legacy/` entries from ignore lists (build config, tsconfig, lint, test runner, scanners).
   - Remove contract-test notes or scripts that only applied to recording against legacy. Keep the contract tests themselves if they still run.
   - If deploy config at the root still references legacy paths, don't change it. Report it to the user.
5. Build, run all tests and start the app after each removal commit.
6. Report what was removed, the commit SHAs, and the tag to restore from.

## Report after each step

- Project, module or feature ported, and files changed
- What changed, and anything that deviates from current behaviour (with the reason)
- Feature inventory items verified
- Security fixes made (scanner rule or ID, file, change) and findings reported but not fixed
- Validation run and result
- Commits made (SHA + message) and tags created
- Next step
