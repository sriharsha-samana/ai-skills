---
name: play-framework-upgrade
description: Backend-only. Detect a Java Play Framework service's current setup (Play 2.x/3.x, sbt, Java, Akka/Pekko, Ebean/JPA, Guice), move it to legacy/ and rebuild it module by module on the latest stable Play and Java LTS, with full API and behaviour parity, contract tests recorded against legacy, a switchable mock layer for external integrations, and security fixes limited to this backend. Use for any Play Framework (Java) version upgrade or migration.
---

# Java Play Framework → Latest Upgrade

Goal: work out what the service runs on today and upgrade it, one version at a time, to the latest stable Play Framework and the latest Java LTS that Play supports, without changing what the service does.

## Non-negotiables

These hold for the whole upgrade and override every default below, including the scaffold.

1. **Functional parity.** Every endpoint, background job, scheduled task, message consumer and startup hook must behave the same afterwards.
2. **Preserve business logic.** Change APIs and syntax required by the upgrade, never the logic. Keep calculations, rules, conditions, transactions and their ordering exactly. If logic looks wrong, flag it to the user; don't fix it.
3. **Preserve the API contract.** Same routes, HTTP methods, path and query params, request parsing, status codes, headers, cookies, JSON field names, null and empty handling, date and number formats, and error response bodies.
4. **Preserve application behaviour.** Same auth and session handling (cookie names, secret, expiry), same DB schema and queries, same outbound calls (URLs, payloads, timeouts, retries), same config keys and env vars, same logging of business events.
5. **Backend only.** Change only this service's backend code, build and config. Frontend code in the repo (Twirl view markup, `public/` JS, CSS and images) is ported unchanged, apart from template syntax needed to compile. Never change frontend apps, other services, the database schema, infrastructure, CI/CD or deployment. If parity seems to need such a change, stop and ask.
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
| Versions | Latest **stable** Play, plus the matching sbt, Scala (for the build) and plugin versions. Java: the latest LTS that Play version supports. Look them up when you start (Play release notes and docs) and never assume from memory. |
| Approach | Move the current code to `legacy/`, create a fresh project on the latest Play at the repo root, and port module by module, applying every migration guide between the two versions while porting. `legacy/` is the read-only reference and the parity baseline. |
| Language | Stay in Java. Modern Java features (records, `var`, switch expressions, text blocks) only in code you otherwise touch, and only when behaviour-neutral. |
| Async model | Keep the current style (`CompletionStage`, sync actions). Don't rewrite for style. |
| Persistence | Keep the current library (Ebean, JPA/Hibernate, JDBC, jOOQ, etc.), upgraded to the version compatible with the target Play. No schema changes. |
| DI | Guice, as now. Remove static or global state only where the new Play version requires it. |
| External integrations | Behind interfaces, with a switchable mock layer (Step 5) |
| Auth | The existing model and flow, ported as-is |
| Deployment | Unchanged. If the artifact (dist zip, Docker image, start script) or runtime Java version changes, tell the user; don't change CI/CD yourself. |

## Scaffold project

If the user provides a scaffold or reference project, read it first. Take from it the package layout, build settings, module and DI patterns, test setup and code conventions. Where it differs from this skill, follow it (except for the non-negotiables). List the conventions you adopted in your Step 1 report.

## Rules

- Detect and audit first. Report before changing code.
- Port one module or feature per change. After each step the build, tests and app startup must pass before moving on.
- Never edit `legacy/`. Read it to port from.
- Don't add libraries beyond what the upgrade requires and what the scaffold uses. Ask first.
- When unsure about an API or version, check the official migration guide or changelog. Do not guess.

---

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
  Examples: `chore(legacy): move existing app to legacy/`, `feat(users): port users list and detail pages`, `fix(security): sanitize v-html in ProductDescription (semgrep vue-v-html)`.
- **Never commit** secrets, real `.env` files, credentials, `node_modules`, build output or real customer data.
- **Never rewrite shared history:** no amending, rebasing or force-pushing commits that have been pushed.
- **Push** only if the user asks, or if that's the repo's established practice.
- **Milestone tags:** tag key points so they're easy to roll back to (e.g. `upgrade/legacy-moved`, `upgrade/parity-complete`, `upgrade/pre-cleanup`). Push tags only if the user asks.

## Step 0 — Detect the current setup

| Check | Where to look |
|---|---|
| Play version and groupId | `project/plugins.sbt`: `com.typesafe.play % sbt-plugin` (2.x) vs `org.playframework % sbt-plugin` (3.x) |
| sbt | `project/build.properties` |
| Scala version (build) | `build.sbt` `scalaVersion` |
| Java version | `.java-version`, `.sdkmanrc`, `javacOptions`, Dockerfile base image, CI config |
| Build tool | sbt (standard) or Gradle (legacy Play Gradle plugin, much rarer) |
| Akka / Pekko | `akka.*` or `org.apache.pekko.*` imports, `akka { }` / `pekko { }` config blocks, actors, streams |
| Persistence | `play-ebean`, `javaJpa` + Hibernate, `javaJdbc`, jOOQ, `evolutions`, Flyway or Liquibase |
| HTTP client | `ws` / `play-ws`, OkHttp, Apache HttpClient, etc. |
| Global state | `Http.Context`, `ctx()`, static `request()`, `session()`, `flash()`, `play.Play`, `Play.current` |
| Templates | Twirl views (`app/views/*.scala.html`) |
| Auth | `Security.Authenticator`, action composition (`@With`, `@Security.Authenticated`), `pac4j`, `deadbolt`, JWT libraries, session config |
| Tests | JUnit 4/5, `WithApplication`, `Helpers`, Mockito, test coverage level |
| Other modules | play-mailer, play-json, filters (CSRF, CORS, AllowedHosts, security headers), caching, i18n |

Report the version gap (e.g. Play 2.6 / Java 8 → latest Play / Java LTS), which porting notes from Step 4 apply, and any blockers (libraries with no version for the target).

## Step 1 — Audit (report only, no edits)

1. **Endpoints:** every route in `conf/routes` (and included routers), with its controller action, request body type, response shape, status codes, filters and auth applied.
2. **Outbound integrations:** each HTTP API, message broker, email or SMS provider, file storage or other external service, with its client code, config keys, timeouts and retries.
3. **Auth:** session cookie name and config, secret key source, authenticators and action composition, roles and permissions, token validation.
4. **Background work:** Akka/Pekko actors, schedulers, startup tasks (eager singletons), consumers.
5. **Config:** `application.conf` keys and env overrides that the upgrade will touch.
6. **Deployment:** build command, artifact, runtime Java version, start flags. Record these; don't change them.
7. **Feature inventory** (the parity baseline): every endpoint and job, its business rules, side effects (DB writes, outbound calls, emails) and error cases.
8. **Security findings:** dependency scan (e.g. `sbt dependencyCheck`, Snyk or whichever scanner the user names), plus manual checks from "Vulnerability fixes". Give each finding a severity and a proposed fix.
9. Conventions taken from the scaffold, if provided.
10. A proposed porting order, starting with the smallest and lowest-risk features.

## Step 2 — Move existing code to `legacy/`

1. Use `git mv` so file history is kept.
2. **Keep at the repo root:** `.git`, CI/CD config, deploy and hosting config (Dockerfile, k8s manifests, Procfile and similar), `.env*` files, `README`, `LICENSE`. Before moving, list what stays and what moves, and confirm with the user.
3. Move everything else (`app/`, `conf/`, `public/`, `project/`, `build.sbt`, `test/` and so on) into `legacy/`.
4. Make sure the new build and tooling ignore `legacy/` (sbt only builds the root project; also exclude it from IDE, lint and scanner configs that live in the repo).
5. `legacy/` is a read-only reference. Never edit it, and don't delete it until the user approves. It can still run on its own (`cd legacy && sbt run`, on its original Java version) for side-by-side comparison.
6. Root deploy config (e.g. the Dockerfile) still describes the legacy build. List what will need to change at cut-over for the user; don't change it yourself.

## Step 3 — Contract tests (the parity safety net)

- Run the legacy tests and record the results as the baseline (pre-existing failures listed).
- In the new project, add **black-box HTTP contract tests** (e.g. `test/contract/`) for each endpoint in the feature inventory: status, headers that matter, cookies, and the exact JSON or HTML body (golden files are fine). The tests take the target from `CONTRACT_BASE_URL`.
- **Record the goldens against the running legacy app.** Don't add a mock toggle to `legacy/`. If its downstream URLs come from config or env, point them at a local stub server (e.g. WireMock) that serves the same fixtures as the new app's mocks. Otherwise record against the dev environment.
- Run the same tests against the new app (mocks on) for every ported feature. They must pass unchanged. Changing a golden file means a deviation, which needs user approval.

## Step 4 — New project and porting

1. Create the new project at the root from the scaffold if one was provided, otherwise from the official Play Java seed for the latest version (`sbt new playframework/play-java-seed.g8`), on the latest sbt and Java LTS. Keep the same package names.
2. Port in this order, one change at a time:
   1. Build: the same libraries as legacy, at their latest versions compatible with the target Play.
   2. Config: `application.conf` with the same keys and env overrides. Set explicitly any value whose default changed between versions.
   3. Guice modules and DI bindings.
   4. Filters: the same set, in the same order, with the same settings.
   5. Auth (Step 6).
   6. Outbound clients and mocks (Step 5).
   7. Persistence: the same entities, mappings and queries. Copy evolutions or migrations unchanged and keep `play.evolutions` settings identical, so no new scripts run against existing databases.
   8. Routes and controllers, feature by feature, with the same paths in `conf/routes`.
   9. Views and `public/` assets, copied unchanged except for syntax needed to compile.
   10. Background jobs, actors and schedulers.
3. While porting each file, apply the changes for every version between legacy and target (table below), always confirmed against the official migration guides ("Play 2.x Migration Guide" / "Play 3.0 Migration Guide").
4. Run the contract tests for each ported feature.

Porting reference (changes to apply when code comes from that version or older):

| Step | Typical changes |
|---|---|
| ≤ 2.6 → 2.7 | Global state deprecated (`play.Play`, `Play.current`) → inject `Config`, `Environment` and components. Static helper deprecations across the API. |
| 2.7 → 2.8 | Java `Http.Context` removed: `ctx()`, `request()`, `session()`, `flash()`, `response()` go away. Actions take `Http.Request request` (and routes declare `request: Request`). Sessions and flash via `result.addingToSession(request, ...)`, `withSession`, `flashing`. `formFactory.form(X.class).bindFromRequest(request)`. `messagesApi.preferred(request)`. Akka 2.6. |
| 2.8 → 2.9 | Java 11+ required. Scala 2.13 or 3 for the build, newer sbt. Library bumps (Jackson and others). Check JSON serialization output against the contract tests. |
| 2.9 → 3.0 | Akka → Apache Pekko: groupIds `com.typesafe.play` → `org.playframework` (including the sbt plugin), imports `akka.*` → `org.apache.pekko.*`, config `akka { }` → `pekko { }` (and Play keys that referenced Akka). Otherwise the API is close to 2.9. |
| 3.0 → latest | Follow the release notes and migration guide. Check the minimum Java version and module compatibility. |

Java: the new project targets the newest LTS the target Play supports. Fix removed JDK APIs while porting (e.g. `javax.xml.bind` → add the JAXB dependency, Nashorn, `SecurityManager`). The runtime and Docker base image are deployment: list the change for the user.

## Step 5 — Mock layer for external integrations

The goal is to run and test the service without any downstream system and without changing other services or deployment.

- Put each outbound integration behind an interface. Where legacy calls `WSClient` directly in controllers, port it as a thin client class that makes exactly the same calls.
- Bind the real or mock implementation in one Guice module, chosen by one config flag.

```hocon
# conf/application.conf
mock.externals = false
mock.externals = ${?MOCK_EXTERNALS}
play.modules.enabled += "modules.ExternalsModule"
```

```java
// app/modules/ExternalsModule.java
public class ExternalsModule extends AbstractModule {
  private static final Logger log = LoggerFactory.getLogger(ExternalsModule.class);
  private final boolean mock;

  public ExternalsModule(Environment env, Config config) {
    this.mock = config.getBoolean("mock.externals");
  }

  @Override
  protected void configure() {
    if (mock) {
      log.warn("MOCK EXTERNALS ENABLED — no calls go to real downstream systems");
      bind(PaymentClient.class).to(MockPaymentClient.class);
      // one line per integration
    } else {
      bind(PaymentClient.class).to(HttpPaymentClient.class);
    }
  }
}
```

Mock rules:
- Toggle: `MOCK_EXTERNALS=true sbt run`, or set it in the environment. The default is off.
- Mock every outbound integration from the audit. Mocks return fixtures (e.g. `conf/mocks/*.json`) with the real response shapes, and can simulate error cases (timeouts, 4xx/5xx) for testing.
- Fake data only: no real customer data, tokens or secrets.
- The database stays real (local or dev). Swapping in an in-memory DB changes SQL behaviour, so only do it if the user asks.
- Contract tests run with mocks on.

## Step 6 — Auth (port the existing flow, don't redesign it)

- Keep the same session cookie name and settings, the secret key source, authenticators, action composition, roles and token validation.
- If a new version rejects the current setup (e.g. secret key length rules, changed cookie defaults), stop and tell the user. Changing the secret invalidates all existing sessions.
- Explicitly set any session or cookie config whose *default* changed between versions, so behaviour stays the same.
- Don't introduce a new auth library or model unless the user asks.

## Vulnerability fixes

Fix issues reported by common scans (OWASP Dependency-Check, Snyk, SonarQube, Semgrep, CodeQL, SpotBugs/Find Security Bugs) when the fix stays inside this service and keeps parity.

| Finding | Fix |
|---|---|
| Vulnerable dependencies | Patched version with the same API (the upgrade covers most) |
| SQL injection (string-concatenated queries) | Parameterized queries / bind params with identical SQL semantics |
| XXE in XML parsing | Disable external entities and DTDs on the parser |
| Insecure deserialization (`ObjectInputStream` on untrusted input) | Allow-list classes or switch to the existing JSON mapping, with the same payload |
| Command injection (`Runtime.exec` with input) | Argument arrays, no shell; validate input |
| Path traversal (files built from request input) | Normalize and check against the base directory |
| SSRF (user-controlled outbound URLs) | Restrict to the hosts the feature legitimately uses |
| Open redirect | Allow only relative or allow-listed targets; fall back to the current default |
| Hardcoded secrets | Move to config or env (`${?ENV_VAR}`). Tell the user to rotate anything that was committed. |
| Secrets or PII in logs | Remove or mask them |
| JWT validation gaps (no signature check, `alg: none`, no expiry check) | Pin the algorithm in use and verify the signature and expiry |

Rules for security fixes:
- A fix must not change the API contract or business logic, except to block the exploit itself.
- **Report but don't change** findings whose fix changes behaviour clients rely on, or that live outside this service: enabling CSRF, CORS or security-header filters, password hashing scheme changes (they break existing hashes), session model changes, TLS and infrastructure. Describe the risk and the recommended fix for the user to decide.
- List every fix with the scanner rule or ID, the file, and the change.

## Verify after each step

- [ ] Compiles with no new deprecation warnings
- [ ] Existing tests and contract tests pass unchanged (any golden-file change approved)
- [ ] App starts in dev and prod mode (`sbt run`, `sbt stage` / `dist`)
- [ ] Works with mocks on and against real dev dependencies (if the user can run them)
- [ ] Auth: login, session, protected routes and roles behave the same
- [ ] Background jobs, schedulers and consumers start and run

## Done checklist

- [ ] On the latest stable Play and the newest supported Java LTS (versions recorded in the report)
- [ ] Feature inventory fully checked off; any deviations approved
- [ ] Every outbound integration mockable; the toggle works through `MOCK_EXTERNALS`
- [ ] Only backend code changed (no frontend apps, schema, other-service, infra or CI/CD changes)
- [ ] `legacy/` untouched; deletion only on user approval
- [ ] Scan findings fixed or reported, and scans re-run show no new issues
- [ ] Deployment changes (Java runtime, artifact, base image) listed for the user, not applied
- [ ] All work committed at logical save points; the working tree is clean
- [ ] Mocks and `legacy/` kept; they're removed only through the cleanup step, when the user asks

## Final step — Cleanup (only when the user asks)

Never do this automatically, and never as part of another step. Only start it when the user explicitly asks, after the done checklist is complete. The user may ask for both parts or just one.

1. Confirm the scope with the user (mocks, `legacy/`, or both) and list exactly what will be removed.
2. Tag the current commit first (e.g. `upgrade/pre-cleanup`) so everything can be restored.
3. **Remove the mocks** (its own commit, e.g. `chore(mocks): remove mock API layer`):
   - Delete the `Mock*` client classes and the `conf/mocks/` fixtures.
   - In `ExternalsModule`, remove the `mock` branch and the warning, keeping only the real bindings (or move the real bindings to the module that normally holds them).
   - Remove `mock.externals` from `application.conf`.
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

- Module or feature ported, and files changed
- What changed, and anything that deviates from current behaviour (with the reason)
- Feature inventory items verified
- Security fixes made (scanner rule or ID, file, change) and findings reported but not fixed
- Validation run and result
- Commits made (SHA + message) and tags created
- Next step
