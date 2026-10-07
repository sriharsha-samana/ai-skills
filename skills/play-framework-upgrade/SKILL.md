---
name: play-framework-upgrade
description: Detect a Java Play Framework project's current setup (Play 2.x/3.x, sbt, Java, Akka/Pekko, Ebean/JPA, Guice) and upgrade it step by step to the latest stable Play on the latest supported Java LTS, with full API and behaviour parity, contract tests as a safety net, a switchable mock layer for external integrations, and security fixes limited to this service. Use for any Play Framework (Java) version upgrade or migration.
---

# Java Play Framework → Latest Upgrade

Goal: work out what the service runs on today and upgrade it, one version at a time, to the latest stable Play Framework and the latest Java LTS that Play supports, without changing what the service does.

## Non-negotiables

These hold for the whole upgrade and override every default below, including the scaffold.

1. **Functional parity.** Every endpoint, background job, scheduled task, message consumer and startup hook must behave the same afterwards.
2. **Preserve business logic.** Change APIs and syntax required by the upgrade, never the logic. Keep calculations, rules, conditions, transactions and their ordering exactly. If logic looks wrong, flag it to the user; don't fix it.
3. **Preserve the API contract.** Same routes, HTTP methods, path and query params, request parsing, status codes, headers, cookies, JSON field names, null and empty handling, date and number formats, and error response bodies.
4. **Preserve application behaviour.** Same auth and session handling (cookie names, secret, expiry), same DB schema and queries, same outbound calls (URLs, payloads, timeouts, retries), same config keys and env vars, same logging of business events.
5. **This service only.** Change only this repo's code, build and config. Never change other services, the database schema, infrastructure, CI/CD or deployment. If parity seems to need such a change, stop and ask.
6. **Security fixes are allowed** (see "Vulnerability fixes"), but only within this service and without breaking rules 1–5.

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
| Approach | Upgrade in place, one Play minor version at a time, following each official migration guide. No rewrite. |
| Language | Stay in Java. Modern Java features (records, `var`, switch expressions, text blocks) only in code you otherwise touch, and only when behaviour-neutral. |
| Async model | Keep the current style (`CompletionStage`, sync actions). Don't rewrite for style. |
| Persistence | Keep the current library (Ebean, JPA/Hibernate, JDBC, jOOQ, etc.), upgraded to the version compatible with the target Play. No schema changes. |
| DI | Guice, as now. Remove static or global state only where the new Play version requires it. |
| External integrations | Behind interfaces, with a switchable mock layer (Step 4) |
| Auth | The existing model and flow, ported as-is |
| Deployment | Unchanged. If the artifact (dist zip, Docker image, start script) or runtime Java version changes, tell the user; don't change CI/CD yourself. |

## Scaffold project

If the user provides a scaffold or reference project, read it first. Take from it the package layout, build settings, module and DI patterns, test setup and code conventions. Where it differs from this skill, follow it (except for the non-negotiables). List the conventions you adopted in your Step 1 report.

## Rules

- Detect and audit first. Report before changing code.
- One version step per change. After each step the build, tests and app startup must pass before moving on.
- Don't add libraries beyond what the upgrade requires and what the scaffold uses. Ask first.
- When unsure about an API or version, check the official migration guide or changelog. Do not guess.

---

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

Report a version path, e.g. `2.6 → 2.7 → 2.8 → 2.9 → 3.0 → latest`, plus the Java upgrade point(s) and any blockers (libraries with no version for the target).

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
10. A proposed order of version steps.

## Step 2 — Safety net before upgrading

- Run the existing tests and record the results. The baseline must be green, or failures must be listed as pre-existing.
- Add **contract (characterization) tests** for each endpoint in the feature inventory, written against the *current* version: status, headers that matter, and the exact JSON body (golden files are fine). Use `WithApplication` / `Helpers.route` with mocks on (Step 4), so no external system is needed.
- These tests must pass unchanged after every version step. Changing a golden file means a deviation, which needs user approval.

## Step 3 — Upgrade step by step

For each Play version step:
1. Read that version's official migration guide (Play docs: "Play 2.x Migration Guide" / "Play 3.0 Migration Guide").
2. Bump the sbt plugin, sbt, Scala (build) and Play module versions together. Upgrade companion libraries (play-ebean, play-mailer, play-ws, pac4j, etc.) to their compatible versions.
3. Fix compile errors and deprecations required for this step.
4. Run the tests and contract tests and start the app. Fix before continuing.

Key changes to expect (always confirm against the guide for the exact versions):

| Step | Typical changes |
|---|---|
| ≤ 2.6 → 2.7 | Global state deprecated (`play.Play`, `Play.current`) → inject `Config`, `Environment` and components. Static helper deprecations across the API. |
| 2.7 → 2.8 | Java `Http.Context` removed: `ctx()`, `request()`, `session()`, `flash()`, `response()` go away. Actions take `Http.Request request` (and routes declare `request: Request`). Sessions and flash via `result.addingToSession(request, ...)`, `withSession`, `flashing`. `formFactory.form(X.class).bindFromRequest(request)`. `messagesApi.preferred(request)`. Akka 2.6. |
| 2.8 → 2.9 | Java 11+ required. Scala 2.13 or 3 for the build, newer sbt. Library bumps (Jackson and others). Check JSON serialization output against the contract tests. |
| 2.9 → 3.0 | Akka → Apache Pekko: groupIds `com.typesafe.play` → `org.playframework` (including the sbt plugin), imports `akka.*` → `org.apache.pekko.*`, config `akka { }` → `pekko { }` (and Play keys that referenced Akka). Otherwise the API is close to 2.9. |
| 3.0 → latest | Follow the release notes and migration guide. Check the minimum Java version and module compatibility. |

Java upgrade: move to the newest LTS the target Play supports, in its own step. Update the `javac` release, the Docker base image *only if the user approves* (it's deployment), and fix removed JDK APIs (e.g. `javax.xml.bind` → add the JAXB dependency, Nashorn, `SecurityManager`).

## Step 4 — Mock layer for external integrations

The goal is to run and test the service without any downstream system and without changing other services or deployment.

- Put each outbound integration behind an interface. If code calls `WSClient` directly in controllers, extract a thin client class that does exactly the same calls (a behaviour-preserving refactor).
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

## Step 5 — Auth (port the existing flow, don't redesign it)

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
- [ ] Only this service changed (no schema, other-service, infra or CI/CD changes)
- [ ] Scan findings fixed or reported, and scans re-run show no new issues
- [ ] Deployment changes (Java runtime, artifact, base image) listed for the user, not applied

## Report after each step

- Version step and files changed
- What changed, and anything that deviates from current behaviour (with the reason)
- Feature inventory items verified
- Security fixes made (scanner rule or ID, file, change) and findings reported but not fixed
- Validation run and result
- Next step
