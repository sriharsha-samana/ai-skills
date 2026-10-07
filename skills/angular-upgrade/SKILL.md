---
name: angular-upgrade
description: Frontend-only. Detect an app's current Angular setup (AngularJS 1.x or Angular 2+, builder, modules vs standalone, RxJS, NgRx, UI library, SSR) and upgrade it to the latest stable Angular with standalone components, signals and the built-in control flow, keeping full feature parity, a switchable mock API layer and frontend-only security fixes. AngularJS apps are rebuilt beside a legacy/ folder; Angular 2+ apps are upgraded in place one major at a time. Use for any AngularJS or Angular version upgrade or migration.
---

# AngularJS / Angular → Latest Upgrade

Goal: work out what the app runs on today, pick the right upgrade path, and bring it to the latest stable Angular without changing what the app does.

## Non-negotiables

These hold for the whole upgrade and override every default below, including the scaffold.

1. **Feature and functional parity.** Every feature, screen, flow, validation, permission check, edge case and error message in the current version must work the same afterwards. Nothing is dropped, merged or "simplified away".
2. **Preserve business logic.** Port calculations, rules, conditions, formatting and data transforms exactly, including quirks the code depends on. Change syntax, never the logic. If logic looks wrong, flag it to the user; don't fix it.
3. **Preserve APIs.** Same endpoints, methods, headers, query params, request bodies, response handling, error handling and call order or timing (e.g. debounce, polling, retries).
4. **Preserve application behaviour.** Same URLs (including hash or hashbang style), redirects, auth flow, session handling, stored keys (localStorage, cookies), rendering mode (SPA / SSR / prerender), loading and empty states, and side effects (analytics, downloads, notifications).
5. **Frontend only.** Change only frontend code. Never change backend code, even if it lives in the same repo, and never change API contracts, the database, infrastructure, CI/CD or deployment. If parity seems to need a non-frontend change, stop and ask.
6. **Security fixes are allowed** (see "Vulnerability fixes"), but only within the frontend and without breaking rules 1–5.

Any deviation, however small, must be listed in the step report with its reason and approved by the user.

## Precedence

1. The non-negotiables above
2. Explicit user instructions
3. The scaffold project, if the user provides one
4. The defaults in this skill

## Target stack (defaults)

| Area | Default |
|---|---|
| Versions | Latest **stable** Angular, Angular CLI, TypeScript, RxJS and zone.js versions supported by that Angular, plus the matching UI library and NgRx versions. Look them up when you start (`npm view @angular/core version`, angular.dev version compatibility table). Never assume versions from memory. |
| Components | Standalone components (no NgModules), `inject()` for dependencies, signals (`signal`, `computed`, `input()`, `output()`, `model()`) for component state |
| Templates | Built-in control flow (`@if`, `@for` with `track`, `@switch`, `@defer`) |
| Routing | `provideRouter` with lazy-loaded feature routes. Functional guards and resolvers. All existing URLs stay the same. |
| HTTP | `provideHttpClient(withInterceptors([...]))` with functional interceptors. All calls go through feature API services, with a switchable mock mode. |
| State | Signal-based services for app and feature state. If the app already uses NgRx, keep it (upgraded) unless the user asks to replace it. |
| Forms | Keep the current forms approach (template-driven or reactive). Use typed reactive forms in new and ported reactive code. |
| UI | The current UI library's latest Angular version (e.g. AngularJS Material → Angular Material, UI Bootstrap → ng-bootstrap). Built-in components and theming first. Ask if there is none. |
| Styling | As little custom CSS as possible. Colours, typography and density come from the UI library's theme. |
| Build | The CLI's current application builder (esbuild-based) |
| Change detection | Keep zone.js unless the user asks for zoneless |
| Auth | The existing auth model and flow, ported as-is |
| Rendering | Same mode as now. SPA stays SPA; Angular Universal → `@angular/ssr` with the same behaviour. |
| Deployment, backend, other services | Unchanged. Ask before touching CI/CD, hosting or env config. |

## Scaffold project

If the user provides a scaffold project (path, repo or files), read it before writing any code. Take from it:

- folder structure and file naming
- `angular.json`, `app.config.ts`, theme, layout shell
- the patterns for API services, state, guards and interceptors
- lint, format, test and TypeScript settings
- component and page examples to copy

Where the scaffold differs from this skill, follow the scaffold (except for the non-negotiables). List the conventions you adopted in your Step 1 report. If no scaffold is provided, use the defaults in this skill.

## Rules

- Detect and audit first. Report before changing code.
- Port or upgrade one feature, route or concern at a time. The app must build and run after each step.
- Never edit `legacy/` (rebuild path). Read it to port from.
- Don't add libraries beyond the target stack and what the scaffold uses. Ask first.
- When unsure about an API or version, check angular.dev, the update guide (angular.dev/update-guide) or the library changelog. Do not guess.

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
  Examples: `chore(legacy): move existing app to legacy/`, `feat(users): port users list and detail pages`, `fix(security): remove bypassSecurityTrustHtml in ProductDescription (semgrep angular-bypass)`.
- **Never commit** secrets, real `.env` files, credentials, `node_modules`, build output or real customer data.
- **Never rewrite shared history:** no amending, rebasing or force-pushing commits that have been pushed.
- **Push** only if the user asks, or if that's the repo's established practice.
- **Milestone tags:** tag key points so they're easy to roll back to (e.g. `upgrade/legacy-moved`, `upgrade/parity-complete`, `upgrade/pre-cleanup`). Push tags only if the user asks.

## Step 0 — Detect the current setup

| Check | Where to look | Possible results |
|---|---|---|
| Framework | `angular` (AngularJS) vs `@angular/core` version | AngularJS 1.x / Angular 2–latest |
| Hybrid | `@angular/upgrade`, `UpgradeModule`, `downgradeComponent` | ngUpgrade hybrid app |
| Build | `angular.json` builders, `webpack.config.*`, Gulp or Grunt, `karma.conf.js` | CLI (browser / application builder) / custom |
| Module style | `@NgModule` vs `standalone: true` / `bootstrapApplication` | modules / standalone / mixed |
| Templates | `*ngIf`, `*ngFor` vs `@if`, `@for` | — |
| Router | `ngRoute`, `ui-router` (AngularJS); `RouterModule` / `provideRouter`; `useHash` | — |
| HTTP | `$http`, `$resource`, `HttpClient`, interceptors (class or functional) | — |
| State | NgRx, NGXS, Akita, services with Subjects or signals | — |
| Forms | template-driven / reactive / untyped reactive | — |
| UI library | Angular Material (including legacy, pre-MDC components), AngularJS Material, PrimeNG, ng-bootstrap, UI Bootstrap, `@angular/flex-layout` | — |
| RxJS | version; `toPromise`, deep `rxjs/operators` imports | — |
| SSR | `@nguniversal/*`, `@angular/ssr`, `server.ts`, prerender config | SPA / SSR / prerender |
| Auth | interceptors, guards, token storage, OIDC libraries (`angular-oauth2-oidc`, MSAL, etc.) | — |
| Tests and lint | Karma/Jasmine, Jest, Vitest; Protractor, Cypress, Playwright; TSLint vs angular-eslint | — |

Then choose the path:

| Current setup | Path |
|---|---|
| AngularJS 1.x, or an ngUpgrade hybrid | **R: rebuild** as a new Angular app beside `legacy/` |
| Angular 2+ | **U: upgrade in place** with `ng update`, one major version at a time. No `legacy/` move unless the user asks. |

On path U, ask whether to run the modernization migrations (standalone, control flow, `inject()`, signal inputs and queries) across the whole app now. The default is to apply them to the files you otherwise touch. State which path you chose and why, and wait for the user to confirm.

## Step 1 — Audit (report only, no edits)

1. **Routes:** every route with its path, URL style (hash or hashbang), params, guards, resolvers, redirects, lazy modules, nesting and named outlets.
2. **API endpoints:** every call with its method, path, params, body, response shape and the file that makes it, plus the base URL and environment config.
3. **Auth:** where the token or session is stored, how it's attached (interceptor), the login, logout and refresh flows, 401/403 handling, guards, and role or permission checks.
4. **State:** stores, services holding state, `$rootScope` usage, events (`$broadcast`/`$emit`).
5. **Global pieces:** directives, filters and pipes, decorators, `run` and `config` blocks, `APP_INITIALIZER`s, global styles.
6. **Third-party libraries** and their latest-Angular equivalents. Blockers go first.
7. **Deployment:** the build command, output path (`dist/<app>` vs `dist/<app>/browser` with the application builder), base href, hosting, env files and CI config. Record these; don't change them.
8. Conventions taken from the scaffold, if one was provided.
9. **Feature inventory** (the parity baseline): every screen and its user actions, business rules (validations, calculations, conditional display, permission checks), side effects, and error and empty states, with the source file for each. Each item is later checked off when ported or upgraded.
10. **Security findings:** `npm audit` or whichever scanner the user names, plus manual checks from "Vulnerability fixes". Give each finding a severity and a proposed fix.
11. A proposed order of work, starting with the smallest and lowest-risk.

## Step 2 — Move existing code to `legacy/` (rebuild path only)

1. Use `git mv` so file history is kept.
2. **Keep at the repo root:** `.git`, CI/CD config, deploy and hosting config (Dockerfile, nginx config, `netlify.toml`, `vercel.json`, `staticwebapp.config.json` and similar), `.env*` files, `README`, `LICENSE`. Before moving, list what stays and what moves, and confirm with the user.
3. Move everything else into `legacy/`.
4. Make sure no tooling picks up `legacy/`: tsconfig `exclude`, ESLint ignores, test config.
5. `legacy/` is a read-only reference. Never edit it, and don't delete it until the user approves. It can still run on its own for side-by-side comparison.

## Step 3 — Set up the target app

**Rebuild path:** copy the scaffold if one was provided. Otherwise create a new CLI app (`ng new`, standalone, the same style language as legacy: CSS or SCSS, routing on, SSR only if legacy had it) and use this structure:

```
src/
  main.ts                 # bootstrapApplication(AppComponent, appConfig)
  app/
    app.config.ts         # provideRouter, provideHttpClient(withInterceptors), UI library providers
    app.routes.ts         # top-level routes, lazy feature routes
    core/                 # auth, interceptors, guards, layout shell, api helpers
    shared/               # reusable components, pipes, directives
    features/<feature>/   # pages, components, <feature>.api.ts, <feature>.store.ts, <feature>.routes.ts
    mocks/                # mock handlers + fixtures (loaded only in mock mode)
  environments/           # environment.ts, environment.mock.ts
```

**In-place path:** for each major version:
1. Follow the Angular update guide for that version (angular.dev/update-guide). Check the required Node and TypeScript versions.
2. Run `ng update @angular/core@<N> @angular/cli@<N>` and the matching updates for the UI library, NgRx and other `@angular/*` packages. Review every automatic migration.
3. Fix the build and run the tests and e2e before the next major.

Notable changes on the way (confirm against the guide for the exact versions): Angular Material's MDC-based components (DOM and style changes; the legacy components were removed); typed forms (`ng update` converts to `UntypedFormGroup`, which you then type gradually); class-based guards and resolvers deprecated in favour of functional ones; the move from the browser builder to the application builder (output path changes to `dist/<app>/browser`); `@angular/flex-layout` deprecated (replace with CSS flex/grid utilities or the UI library's layout); Protractor removed (ask the user which e2e tool to keep or adopt); TSLint → angular-eslint; RxJS 7 (`toPromise` → `firstValueFrom`/`lastValueFrom`).

If the build output path or command changes, tell the user the deploy pipeline will need updating. Do not change it yourself.

## Step 4 — API layer and mock mode

All HTTP calls go through feature API services using `HttpClient`. Components never call `HttpClient`, `fetch` or `$http` directly.

### Toggle

- **Default:** `apiMock` in the environment files. `ng serve -c mock` uses `environment.mock.ts` (`apiMock: true`) through a `mock` configuration with `fileReplacements` in `angular.json`.
- **Dev-only override:** add `?mock=on` or `?mock=off` to any URL. The choice is remembered in localStorage and ignored in production builds.
- When mocks are on, show a small "MOCK API" badge using the UI library (e.g. a warning-coloured chip in the toolbar).

```ts
// src/app/core/api/mock-toggle.ts
import { isDevMode } from '@angular/core'
import { environment } from '../../../environments/environment'

export function isMockEnabled(): boolean {
  if (isDevMode() && typeof window !== 'undefined') {
    try {
      const q = new URLSearchParams(window.location.search).get('mock')
      if (q === 'on' || q === 'off') localStorage.setItem('apiMock', q)
      const saved = localStorage.getItem('apiMock')
      if (saved) return saved === 'on'
    } catch {}
  }
  return environment.apiMock === true
}
```

```ts
// src/app/core/api/mock.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http'
import { from, switchMap } from 'rxjs'
import { environment } from '../../../environments/environment'

export const mockInterceptor: HttpInterceptorFn = (req, next) => {
  if (!req.url.startsWith(environment.apiBase)) return next(req)
  return from(import('../../mocks')).pipe(switchMap(({ handleMock }) => handleMock(req)))
}
```

```ts
// src/app/app.config.ts (excerpt)
provideHttpClient(withInterceptors([
  authInterceptor,                               // ported from legacy, same behaviour
  ...(isMockEnabled() ? [mockInterceptor] : []), // last, so auth headers are still applied
])),
```

```ts
// src/app/mocks/index.ts
import { HttpErrorResponse, HttpRequest, HttpResponse } from '@angular/common/http'
import { environment } from '../../environments/environment'
import users from './fixtures/users.json'

type Ctx = { params: Record<string, string>; req: HttpRequest<unknown> }
const routes: Record<string, (ctx: Ctx) => unknown> = {
  // One entry per endpoint found in the Step 1 audit — same paths, same response shapes
  'POST /auth/login': () => ({ token: 'mock-token', user: users[0] }),
  'GET /users': () => users,
  'GET /users/:id': ({ params }) => {
    const user = users.find(u => String(u.id) === params['id'])
    if (!user) throw new HttpErrorResponse({ status: 404, statusText: 'Not Found' })
    return user
  },
}

export async function handleMock(req: HttpRequest<unknown>): Promise<HttpResponse<unknown>> {
  const path = req.url.slice(environment.apiBase.length).split('?')[0]
  for (const [key, handler] of Object.entries(routes)) {
    const [method, pattern] = key.split(' ')
    const params = method === req.method ? match(pattern, path) : null
    if (params) {
      await new Promise(r => setTimeout(r, 200)) // simulate latency
      return new HttpResponse({ status: 200, body: await handler({ params, req }), url: req.url })
    }
  }
  throw new HttpErrorResponse({ status: 501, statusText: `No mock for ${req.method} ${path}`, url: req.url })
}

function match(pattern: string, path: string) {
  const p = pattern.split('/'), a = path.split('/')
  if (p.length !== a.length) return null
  const params: Record<string, string> = {}
  for (let i = 0; i < p.length; i++) {
    if (p[i].startsWith(':')) params[p[i].slice(1)] = decodeURIComponent(a[i])
    else if (p[i] !== a[i]) return null
  }
  return params
}
```

Mock rules:
- Mock every endpoint from the audit, including auth. With mocks on, nothing goes to the network. A missing mock fails loudly (the 501 error above).
- Build fixtures from the response shapes found in the current code and its types. Use fake data only: no real user data, tokens or secrets.
- Cover error states too (401, 404, validation errors) so the UI's error paths can be tested.
- Mocks are loaded with a dynamic import, so with the toggle off they never run, and the backend, deploy and dependencies stay unchanged.

## Step 5 — Auth (port the existing flow, don't redesign it)

- Use the same storage keys, header, endpoints, refresh logic and logout behaviour. Keep the same OIDC or MSAL library (upgraded) and configuration if one is used.
- Auth state lives in an `AuthService` built on signals (or the existing NgRx slice).
- Port interceptors as functional interceptors with identical behaviour, including the order relative to other interceptors.
- Port guards as functional guards (`CanActivateFn`, `CanMatchFn`) with the same redirect targets and query params.
- Don't introduce a new auth library or model unless the user asks.

## Step 6 — State

- **AngularJS** services and factories → `@Injectable({ providedIn: 'root' })` services using `inject()`. State that changes over time lives in signals; expose read-only signals plus methods that update them.
- `$rootScope` data and `$broadcast`/`$emit` events → a shared service (signals, or a `Subject` for one-off events).
- Feature state used by one page stays in that component.
- **NgRx** (if present): upgrade it with `ng update @ngrx/store` per Angular major. Keep the actions, reducers and effects logic as-is. Selectors can be read with `store.selectSignal()`.
- RxJS stays where streams are genuinely needed (HTTP, websockets, debounce). Convert to signals at the component edge with `toSignal()`.

## Step 7 — Routing

- AngularJS `ngRoute` or `ui-router` states → `Routes` in `app.routes.ts` and `features/*/<feature>.routes.ts`, lazy loaded with `loadComponent` / `loadChildren`.
- **Keep URLs identical.** AngularJS hashbang URLs (`#!/users`) → `withHashLocation()`. Add redirects so old `#!/` links still land on the same screen; this is the one place where URL handling may need adapting, so flag it to the user.
- `resolve` → functional resolvers. `$stateParams` / `$routeParams` → `withComponentInputBinding()` + `input()`, or `ActivatedRoute`.
- `ui-router` nested views and named views → child routes and named outlets.
- `otherwise` → `{ path: '**', ... }` with the same target.
- Check every route from the audit resolves to the same screen.

## Step 8 — Components and templates

AngularJS → Angular:

| AngularJS | Angular |
|---|---|
| controller + `$scope` / controllerAs | standalone component; fields → `signal()`, derived values → `computed()` |
| `.component()` bindings `<`, `@`, `&`, `=` | `input()`, `input()`, `output()`, `model()` |
| directive (element) | component |
| directive (attribute behaviour) | attribute directive |
| filter | pure pipe |
| `ng-if`, `ng-show` / `ng-hide` | `@if`, `[hidden]` / class binding (keep the same DOM semantics) |
| `ng-repeat` (+ `track by`) | `@for (item of items; track item.id)` |
| `ng-switch` | `@switch` |
| `ng-model` | `[(ngModel)]` (template-driven) or reactive forms, matching the current validation behaviour |
| `ng-click`, `ng-change`, `ng-submit` | `(click)`, `(ngModelChange)`, `(ngSubmit)` |
| `ng-class`, `ng-style` | `[class.x]`, `[style.x]` or `[ngClass]` |
| `ng-include` | child component |
| `$watch` / `$watchCollection` | `computed()` / `effect()` (keep the same trigger semantics) |
| `$timeout`, `$interval` | `setTimeout` / `setInterval` cleaned up via `DestroyRef`, or RxJS `timer` |
| `$q`, promises | `async/await` or Observables, preserving resolution order |
| `$http` / `$resource` | feature API service with `HttpClient` |
| `$sce.trustAsHtml` | `[innerHTML]` with Angular sanitization; `DomSanitizer` only if strictly required (see security) |
| `$onInit`, `$onChanges`, `$onDestroy` | `ngOnInit`, `ngOnChanges` / `effect()`, `DestroyRef` / `ngOnDestroy` |

Angular 2+ modernization (path U files you touch, or everything if the user approved): `*ngIf` / `*ngFor` → `@if` / `@for` (`ng g @angular/core:control-flow`); NgModules → standalone (`ng g @angular/core:standalone`); constructor injection → `inject()` (`ng g @angular/core:inject`); `@Input` / `@Output` / queries → signal APIs (the matching `ng g @angular/core:*` migrations). Review each migration diff for behaviour changes, especially `@for` tracking and `ngOnChanges` logic.

## Step 9 — UI library and minimal custom CSS

Styling order of preference:
1. A UI library component with its inputs (appearance, color, density, size).
2. Theme and global component defaults (e.g. an Angular Material theme with `mat.theme`, or the UI library's global config), for anything repeated.
3. Layout with simple flex and grid utilities (from the UI library, the scaffold, or one small shared stylesheet).
4. Only then minimal component styles using theme tokens or CSS variables, never hard-coded colours. Explain why they were needed.

Drop custom CSS when a library equivalent exists, and don't port global stylesheets wholesale. Visual output may move toward the stock UI library; behaviour may not. For Angular Material upgrades across the MDC switch, check every screen for layout and spacing changes and fix them with theme or density settings, not overrides.

## Vulnerability fixes

Fix issues reported by common code and dependency scans (e.g. `npm audit`, Snyk, SonarQube, Semgrep, CodeQL, OWASP ZAP findings that trace to frontend code), as long as the fix stays in the frontend and keeps parity.

| Finding | Fix |
|---|---|
| Vulnerable or outdated packages | The upgrade replaces most of them. For the rest, use a patched version with the same API. |
| `bypassSecurityTrust*` on untrusted data | Remove the bypass and let Angular sanitize. If HTML must be kept, sanitize first (DOMPurify is the one allowed new dependency; tell the user). |
| Direct DOM writes (`nativeElement.innerHTML`, `document.write`) | Use template bindings or `Renderer2` |
| AngularJS expression or template injection (user input compiled with `$compile` or server-rendered into templates) | Removed by the rebuild. Never build templates from user input. |
| `eval`, `new Function`, string `setTimeout` | Replace with equivalent non-dynamic code |
| Open redirect (e.g. unvalidated `returnUrl`) | Allow only same-origin relative paths; otherwise fall back to the current default page |
| Secrets, API keys or credentials in environment files | Remove them from the frontend. If a secret was ever shipped to the browser, tell the user it must be rotated. |
| `target="_blank"` without `rel` | Add `rel="noopener noreferrer"` |
| Sensitive data in URLs, logs or `console.*` | Remove it from logs; leave URLs alone if the API contract needs them, and flag it |
| JSONP or insecure `postMessage` handlers | Check `event.origin`; keep JSONP only if the API contract requires it, and flag it |

Rules for security fixes:
- A fix must not change visible behaviour, API calls or business logic, except to block the exploit itself.
- **Report but don't change** findings whose fix would alter the auth model, an API contract, the backend or infrastructure. Examples: the token is stored in localStorage, missing CSP or security headers set by the server, CORS, server-side validation. Describe the risk and the recommended fix for the user to decide.
- List every security fix in the step report with the scanner rule or ID, the file, and the change.

## Step 10 — Verify each feature

- [ ] Matches the feature inventory entry: every action, business rule, validation, edge case and error/empty state behaves as before
- [ ] Business logic compared line by line with the original source (calculations, conditions, formatting)
- [ ] `ng build` with no errors or new warnings; lint clean
- [ ] Unit and e2e tests pass
- [ ] Works in mock mode, including error states
- [ ] Works against the real API (if the user can run it)
- [ ] Same URLs, same API calls and payloads, same rendering mode
- [ ] Auth: login, logout, token refresh, protected-route redirect and role gating behave the same
- [ ] Responsive at mobile and desktop widths
- [ ] No new custom CSS without a stated reason

## Done checklist

- [ ] On the latest stable Angular (versions recorded in the report)
- [ ] Feature inventory fully checked off; any deviations approved by the user
- [ ] Every endpoint mocked; the toggle works through `ng serve -c mock` and `?mock=`
- [ ] Standalone, signals and built-in control flow wherever agreed
- [ ] Only frontend files changed (no backend, API contract, infra or CI/CD changes)
- [ ] Scan findings fixed or reported, and the scans re-run show no new issues
- [ ] Deploy changes (build command, output path) listed for the user, not applied
- [ ] All work committed at logical save points; the working tree is clean
- [ ] Mocks and `legacy/` kept; they're removed only through the cleanup step, when the user asks

## Final step — Cleanup (only when the user asks)

Never do this automatically, and never as part of another step. Only start it when the user explicitly asks, after the done checklist is complete. The user may ask for both parts or just one.

1. Confirm the scope with the user (mocks, `legacy/`, or both) and list exactly what will be removed.
2. Tag the current commit first (e.g. `upgrade/pre-cleanup`) so everything can be restored.
3. **Remove the mocks** (its own commit, e.g. `chore(mocks): remove mock API layer`):
   - Delete `src/app/mocks/` (handlers and fixtures), `mock.interceptor.ts` and `mock-toggle.ts`.
   - In `app.config.ts`, remove the conditional `mockInterceptor` entry; keep the other interceptors in the same order.
   - Remove `apiMock` from the environment files, `environment.mock.ts`, the `mock` configuration in `angular.json`, and the "MOCK API" badge.
   - On the in-place path there is no `legacy/`, so cleanup covers mocks only.
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

- Files changed
- What was ported or changed, and anything that deviates from current behaviour (with the reason)
- Feature inventory items completed
- Security fixes made (scanner rule or ID, file, change) and findings reported but not fixed
- Validation run and result
- Commits made (SHA + message) and tags created
- Next step
