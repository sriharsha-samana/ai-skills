---
name: react-upgrade
description: Frontend-only. Detect an app's current React setup (React 15–19, Create React App / webpack / Vite / Next.js, class vs function components, React Router, Redux/MobX/Context, UI library, test tools) and upgrade it to the latest stable React with function components and hooks, keeping full feature parity, a switchable mock API layer and frontend-only security fixes. Older or CRA-based apps are rebuilt on Vite beside a legacy/ folder; modern Vite or Next.js apps are upgraded in place. Use for any React, CRA, React Router, Redux or Next.js version upgrade or migration.
---

# React → Latest Upgrade

Goal: work out what the app runs on today, pick the right upgrade path, and bring it to the latest stable React without changing what the app does.

## Non-negotiables

These hold for the whole upgrade and override every default below, including the scaffold.

1. **Feature and functional parity.** Every feature, screen, flow, validation, permission check, edge case and error message in the current version must work the same afterwards. Nothing is dropped, merged or "simplified away".
2. **Preserve business logic.** Port calculations, rules, conditions, formatting and data transforms exactly, including quirks the code depends on. Change syntax, never the logic. If logic looks wrong, flag it to the user; don't fix it.
3. **Preserve APIs.** Same endpoints, methods, headers, query params, request bodies, response handling, error handling and call order or timing (e.g. debounce, polling, retries).
4. **Preserve application behaviour.** Same URLs, redirects, auth flow, session handling, stored keys (localStorage, cookies), rendering mode (SPA / SSR / static export), loading and empty states, and side effects (analytics, downloads, notifications).
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
| Versions | Latest **stable** React and React DOM, plus the latest compatible versions of the router, state library, UI library and test tools. Look them up when you start (`npm view react version`, react.dev upgrade guides) and never assume versions from memory. |
| Build | SPA: Vite with the official React plugin. Next.js apps: stay on Next.js (latest stable). |
| Language | TypeScript, `strict` on in new code. Typing fixes must be behaviour-neutral. |
| Components | Function components and hooks. Custom hooks replace HOCs and mixins. Error boundaries stay class components (React still requires that). |
| Routing | React Router (latest stable) for SPAs; Next.js keeps its current router (Pages or App). All existing URLs stay the same. |
| State | Keep the current approach, upgraded: Redux → Redux Toolkit (`configureStore`, `createSlice`) with the same state shape and action types; MobX, Zustand, Jotai, Context and React Query stay as they are. Don't add a state library unless the user asks. |
| Data | All HTTP goes through one `api` module, with a mock mode switched on or off by one setting |
| UI | The current UI library's latest version (e.g. MUI, Ant Design, Chakra, React Bootstrap). Built-in components and theming first. Ask if there is none. |
| Styling | As little custom CSS as possible. Colours, typography and spacing come from the UI library's theme. |
| Tests | React Testing Library. Enzyme can't test React 18+, so its tests are rewritten to test the same behaviour. Jest or Vitest, following the scaffold (Vitest by default on Vite). |
| React Compiler and new features | Not enabled unless the user asks. Server Components or Actions are only adopted if the user asks. |
| Auth | The existing auth model and flow, ported as-is |
| Rendering | Same mode as now. CRA/Vite SPA stays SPA; Next.js keeps SSR, SSG or static export as configured. |
| Deployment, backend, other services | Unchanged. Ask before touching CI/CD, hosting or env config. |

## Scaffold project

If the user provides a scaffold project (path, repo or files), read it before writing any code. Take from it:

- folder structure and file naming
- `vite.config` / `next.config`, `tsconfig`, theme, layout shell
- the patterns for the API module, state, hooks, routing and guards
- lint, format and test settings
- component and page examples to copy

Where the scaffold differs from this skill, follow the scaffold (except for the non-negotiables). List the conventions you adopted in your Step 1 report. If no scaffold is provided, use the defaults in this skill.

## Rules

- Detect and audit first. Report before changing code.
- Port or upgrade one feature, route or concern at a time. The app must build and run after each step.
- Never edit `legacy/` (rebuild path). Read it to port from.
- Don't add libraries beyond the target stack and what the scaffold uses. Ask first.
- When unsure about an API or version, check react.dev, the library's migration guide or changelog. Do not guess.

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
  Examples: `chore(legacy): move existing app to legacy/`, `feat(users): port users list and detail pages`, `fix(security): sanitize dangerouslySetInnerHTML in ProductDescription (semgrep react-dangerouslysetinnerhtml)`.
- **Never commit** secrets, real `.env` files, credentials, `node_modules`, build output or real customer data.
- **Never rewrite shared history:** no amending, rebasing or force-pushing commits that have been pushed.
- **Push** only if the user asks, or if that's the repo's established practice.
- **Milestone tags:** tag key points so they're easy to roll back to (e.g. `upgrade/legacy-moved`, `upgrade/parity-complete`, `upgrade/pre-cleanup`). Push tags only if the user asks.

## Step 0 — Detect the current setup

| Check | Where to look | Possible results |
|---|---|---|
| React | `react`, `react-dom` versions; `ReactDOM.render` vs `createRoot` | 15 / 16 / 17 / 18 / 19 |
| Build | `react-scripts` (CRA), `react-app-rewired`/`craco`, `webpack.config.*`, `vite.config.*`, `next` | CRA / custom webpack / Vite / Next.js (Pages or App router) |
| Language | `tsconfig.json`, `.ts(x)` vs `.js(x)`, PropTypes | JS / TS / mixed |
| Components | count of `class ... extends (React.)Component` vs function components; HOCs, mixins (`createReactClass`) | — |
| Legacy APIs | string refs, legacy context (`contextTypes`), `findDOMNode`, `UNSAFE_*` / `componentWillMount`, `defaultProps` on function components | — |
| Router | `react-router-dom` version (v3/v4/v5 vs v6/v7), `HashRouter` vs `BrowserRouter`, `connected-react-router` | — |
| State | `redux` (+ `redux-thunk`, `redux-saga`), Redux Toolkit, MobX, Zustand, Recoil, Context, React Query / SWR | — |
| HTTP | `axios`, `fetch`, `superagent`, interceptors | — |
| UI library | `@material-ui/*` (MUI v4) vs `@mui/*`, antd, Chakra, React Bootstrap, Semantic UI, styled-components, emotion, CSS modules | — |
| Env vars | `REACT_APP_*`, `process.env.*`, `.env*` files | — |
| Tests | Jest, Vitest, Enzyme, React Testing Library, Cypress, Playwright | — |
| Auth | interceptors, route guards, token storage, OIDC libraries (`oidc-client-ts`, MSAL, Auth0, NextAuth/Auth.js) | — |
| Rendering | Next.js `getServerSideProps` / `getStaticProps` / App Router / `output: 'export'`; SSR setup for custom builds | SPA / SSR / static |

Then choose the path:

| Current setup | Path |
|---|---|
| CRA, custom webpack or an old React (≤ 17) SPA, or a mostly class-based app | **R: rebuild** on Vite beside `legacy/` |
| Vite SPA on React 18+ | **U: upgrade in place**, one major version at a time |
| Next.js | **U: upgrade in place** on Next.js, one major version at a time. Keep the current router (moving Pages → App Router only if the user asks). |

On path U, ask whether to convert existing class components to function components now. The default is to convert only the files you otherwise touch. State which path you chose and why, and wait for the user to confirm.

## Step 1 — Audit (report only, no edits)

1. **Routes:** every route with its path, params, nested routes, guards or wrapper components, redirects, lazy loading, and the router type (browser or hash). For Next.js, the page files plus `middleware`, `rewrites` and `redirects`.
2. **API endpoints:** every call with its method, path, params, body, response shape and the file that makes it, plus the base URL and env vars.
3. **Auth:** where the token or session is stored, how it's attached, the login, logout and refresh flows, 401/403 handling, guards, and role or permission checks.
4. **State:** stores and slices (state shape, action types, middleware, sagas and thunks), contexts, and where each is used.
5. **Global pieces:** providers, HOCs, custom hooks, global styles, polyfills, analytics, error boundaries.
6. **Third-party libraries** and their latest-React-compatible versions. Blockers go first (e.g. libraries with no React 18/19 support).
7. **Deployment:** build command, output directory (CRA `build/`), `homepage` / base path, env vars, hosting and CI config. Record these; don't change them.
8. Conventions taken from the scaffold, if one was provided.
9. **Feature inventory** (the parity baseline): every screen and its user actions, business rules (validations, calculations, conditional display, permission checks), side effects, and error and empty states, with the source file for each. Each item is later checked off when ported or upgraded.
10. **Security findings:** `npm audit` or whichever scanner the user names, plus manual checks from "Vulnerability fixes". Give each finding a severity and a proposed fix.
11. A proposed order of work, starting with the smallest and lowest-risk.

## Step 2 — Move existing code to `legacy/` (rebuild path only)

1. Use `git mv` so file history is kept.
2. **Keep at the repo root:** `.git`, CI/CD config, deploy and hosting config (Dockerfile, nginx config, `netlify.toml`, `vercel.json`, `staticwebapp.config.json` and similar), `.env*` files, `README`, `LICENSE`. Before moving, list what stays and what moves, and confirm with the user.
3. Move everything else into `legacy/`.
4. Make sure no tooling picks up `legacy/`: tsconfig `include`/`exclude`, ESLint ignores, test runner roots.
5. `legacy/` is a read-only reference. Never edit it, and don't delete it until the user approves. It can still run on its own (`cd legacy && npm install && npm start`) for side-by-side comparison.

## Step 3 — Set up the target app

**Rebuild path:** copy the scaffold if one was provided. Otherwise create a Vite React TypeScript app and use this structure:

```
index.html                # was public/index.html (remove %PUBLIC_URL%)
src/
  main.tsx                # createRoot(...).render(<App />) inside <StrictMode>
  app/                    # App, providers, router, layout shell, error boundary
  features/<feature>/     # pages, components, hooks, <feature>.api.ts, <feature>Slice.ts
  components/             # shared components
  hooks/                  # shared custom hooks (replace HOCs)
  lib/api/                # api client + mock toggle
  mocks/                  # mock handlers + fixtures (loaded only in mock mode)
  store/                  # Redux store (if used)
  utils/  types/
public/                   # static files, served as-is
vite.config.ts
```

Keep CRA-era behaviour that deployment depends on:

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  base: '/',                              // same as legacy "homepage" / PUBLIC_URL
  envPrefix: ['REACT_APP_', 'VITE_'],      // existing env var names keep working
  build: { outDir: 'build' },              // same output dir as CRA, so deploy is unchanged
  server: {
    port: 3000,                            // same dev port
    proxy: { /* port the "proxy" field from legacy package.json */ },
  },
})
```

- `process.env.REACT_APP_X` → `import.meta.env.REACT_APP_X` (same variable names, so deploy env config is unchanged).
- Absolute imports (`jsconfig`/`tsconfig` `baseUrl`) → the same aliases in `vite.config.ts` and `tsconfig` `paths`.
- `import { ReactComponent } from './x.svg'` needs an SVG plugin (e.g. `vite-plugin-svgr`). That's an allowed build dependency; tell the user.

**In-place path:** for each major version:
1. Follow the official upgrade guide (react.dev blog "React N Upgrade Guide"; Next.js "Upgrading" guides) and run the official codemods (e.g. `npx codemod@latest react/19/migration-recipe`, `npx types-react-codemod`, `npx @next/codemod upgrade`). Review every change.
2. Upgrade the router, state, UI library and test tools to versions compatible with the new React.
3. Fix the build and run the tests before the next major.

React version changes to apply while porting or upgrading (confirm against the guides):

| From | Change | Parity note |
|---|---|---|
| ≤ 16 | New JSX transform (no `import React` needed); event delegation moves to the root; no event pooling | Check code calling `e.persist()` or relying on `document`-level listeners |
| ≤ 17 | `ReactDOM.render` → `createRoot`; automatic batching everywhere | Batching can change intermediate renders. Use `flushSync` only where legacy relied on a synchronous update. |
| ≤ 17 | StrictMode runs effects twice in development | Make effects clean up properly; don't "fix" by removing StrictMode |
| ≤ 17 (TS) | `React.FC` no longer adds `children` implicitly | Declare `children` in props |
| ≤ 18 | Removed: `propTypes` checks and `defaultProps` on function components, string refs, legacy context, `findDOMNode`, `ReactDOM.render`/`hydrate`/`unmountComponentAtNode`, `react-test-renderer/shallow` | `defaultProps` → default values in destructuring (same values); string refs → `useRef`/callback refs |
| ≤ 18 | `ref` is a normal prop for function components; `forwardRef` no longer needed | |
| ≤ 18 (TS) | `useRef` requires an argument; global `JSX` namespace → `React.JSX` | |

## Step 4 — API layer and mock mode

Every HTTP call goes through `src/lib/api`. Components never call `fetch` or `axios` directly; they use feature hooks or API functions (or the existing React Query / RTK Query setup, which calls `api`).

### Toggle

- **Default:** `VITE_API_MOCK=true` or `false` in `.env` (for Next.js: `NEXT_PUBLIC_API_MOCK`).
- **Dev-only override:** add `?mock=on` or `?mock=off` to any URL. The choice is remembered in localStorage and ignored in production builds.
- When mocks are on, show a small "MOCK API" badge using the UI library (e.g. a warning-coloured chip in the app bar).

```ts
// src/lib/api/client.ts
type ApiInit = { method?: string; query?: Record<string, unknown>; body?: unknown; headers?: Record<string, string> }

const mockOn = isMockEnabled()

export async function api<T>(path: string, init: ApiInit = {}): Promise<T> {
  if (mockOn) return (await import('../../mocks')).handleMock<T>(path, init)
  return http<T>(path, init)
}

async function http<T>(path: string, init: ApiInit): Promise<T> {
  // Port the legacy client exactly: same base URL, headers, auth token source,
  // interceptors, error mapping, timeouts (e.g. wrap the existing axios instance).
  throw new Error('port legacy HTTP client here')
}

function isMockEnabled(): boolean {
  if (import.meta.env.DEV && typeof window !== 'undefined') {
    try {
      const q = new URLSearchParams(window.location.search).get('mock')
      if (q === 'on' || q === 'off') localStorage.setItem('apiMock', q)
      const saved = localStorage.getItem('apiMock')
      if (saved) return saved === 'on'
    } catch {}
  }
  return import.meta.env.VITE_API_MOCK === 'true'
}

export const apiMockOn = mockOn
```

```ts
// src/mocks/index.ts
import users from './fixtures/users.json'

export class ApiError extends Error {
  constructor(public status: number, message: string) { super(message) }
}

type Ctx = { params: Record<string, string>; query: unknown; body: unknown }
const routes: Record<string, (ctx: Ctx) => unknown> = {
  // One entry per endpoint found in the Step 1 audit — same paths, same response shapes
  'POST /auth/login': () => ({ token: 'mock-token', user: users[0] }),
  'GET /users': () => users,
  'GET /users/:id': ({ params }) => {
    const user = users.find(u => String(u.id) === params.id)
    if (!user) throw new ApiError(404, 'Not found')
    return user
  },
}

export async function handleMock<T>(url: string, init: { method?: string; query?: unknown; body?: unknown } = {}): Promise<T> {
  const method = (init.method ?? 'GET').toUpperCase()
  const path = url.split('?')[0]
  for (const [key, handler] of Object.entries(routes)) {
    const [m, pattern] = key.split(' ')
    const params = m === method ? match(pattern, path) : null
    if (params) {
      await new Promise(r => setTimeout(r, 200)) // simulate latency
      return (await handler({ params, query: init.query, body: init.body })) as T
    }
  }
  throw new ApiError(501, `No mock for ${method} ${path}`)
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
- Errors thrown by mocks must look like the errors the real client throws (same shape and status field), so the UI's error handling runs the same way.
- Mock every endpoint from the audit, including auth. With mocks on, nothing goes to the network. A missing mock fails loudly (the 501 error above).
- Build fixtures from the response shapes found in the current code and its types. Use fake data only: no real user data, tokens or secrets.
- Cover error states too (401, 404, validation errors) so the UI's error paths can be tested.
- Mocks are loaded with a dynamic import, so with the toggle off they never run, and the backend, deploy and dependencies stay unchanged.

## Step 5 — Auth (port the existing flow, don't redesign it)

- Use the same storage keys, header, endpoints, refresh logic and logout behaviour. Keep the same OIDC, MSAL or Auth.js library (upgraded) and configuration if one is used.
- Auth state lives where it lives today (Context, Redux slice or store), exposed through a `useAuth()` hook.
- Route guards become wrapper or layout routes (e.g. a `<RequireAuth>` element around protected routes, or a loader) with the same redirect targets and query params. Next.js keeps its `middleware` or page-level checks.
- Don't introduce a new auth library or model unless the user asks.

## Step 6 — State

- **Redux → Redux Toolkit:** `createStore` + `combineReducers` → `configureStore`; reducers and action creators → `createSlice` with the **same state shape and action type strings** (other code, persistence or devtools may rely on them); thunks → `createAsyncThunk` or plain thunks with the same dispatch order; sagas stay (upgraded) unless the user asks. `connect` → `useSelector`/`useDispatch` in converted components.
- Persisted state (`redux-persist` keys, localStorage keys) must stay identical.
- MobX, Zustand, Jotai, Context, React Query and SWR: upgrade to compatible versions and keep their usage.
- State used by one page stays local (`useState`/`useReducer`).

## Step 7 — Routing

React Router v5 (or older) → latest:

| v5 and older | Latest |
|---|---|
| `<Switch>` | `<Routes>` |
| `<Route path component={X}>` / `render` | `<Route path element={<X />}>` |
| `exact` | removed (matching is exact by default; use `/*` for descendant routes) |
| `useHistory().push(x)` / `replace` | `useNavigate()(x)` / `navigate(x, { replace: true })` |
| `<Redirect to>` | `<Navigate to replace />` |
| `withRouter(C)` | `useNavigate` / `useLocation` / `useParams` hooks |
| `match.params`, `match.url` | `useParams()`, relative links |
| nested `<Switch>` inside components | nested `<Route>` + `<Outlet />` (or descendant `<Routes>` under `/*`) |
| `<Prompt>` | `useBlocker` (data router) — check behaviour matches |
| `connected-react-router` | router hooks; remove the router state from Redux only if nothing reads it |

- Keep the same router type (`BrowserRouter` vs `HashRouter`) and `basename`, so URLs don't change.
- `React.lazy` + `<Suspense>` for code splitting, with the same loading UI.
- Check every route from the audit resolves to the same screen.
- Next.js: keep the current router. Follow the Next.js upgrade guides for each major (e.g. `Link` no longer needs a child `<a>`; in recent versions some App Router request APIs became async and caching defaults changed). Set caching options explicitly to match current behaviour.

## Step 8 — Components (function components + hooks)

| Class component | Function component |
|---|---|
| `this.state` / `setState` | `useState` / `useReducer` (keep the same update semantics; use functional updates where legacy read previous state) |
| `componentDidMount` | `useEffect(() => { ... }, [])` |
| `componentDidUpdate(prevProps)` | `useEffect` with the relevant dependencies (compare with a ref when legacy compared previous values) |
| `componentWillUnmount` | `useEffect` cleanup |
| `setState(x, callback)` | `useEffect` reacting to the changed state |
| `getDerivedStateFromProps` | compute during render, or `key` to reset state |
| `shouldComponentUpdate` / `PureComponent` | `React.memo` (+ `useMemo` / `useCallback` where needed for the same render behaviour) |
| `componentWillMount` / `UNSAFE_*` | effects or render-time computation |
| `createRef` / string refs | `useRef` |
| instance fields | `useRef` |
| HOCs / mixins | custom hooks |
| `static contextType` / `contextTypes` | `useContext` |
| `componentDidCatch` / error boundaries | stay as a class component |

Keep render output identical: same DOM structure where tests, CSS or analytics depend on it, the same keys in lists, and the same conditional rendering.

## Step 9 — UI library and minimal custom CSS

Styling order of preference:
1. A UI library component with its props (variant, color, size, density).
2. Theme and global component defaults (e.g. MUI `createTheme` with `components` defaults) for anything repeated.
3. The UI library's layout primitives and spacing system (e.g. MUI `Stack`, `Grid`, `sx` spacing).
4. Only then minimal component styles using theme tokens or CSS variables, never hard-coded colours. Explain why they were needed.

Drop custom CSS when a library equivalent exists, and don't port global stylesheets wholesale. Visual output may move toward the stock UI library; behaviour may not.

UI library upgrades: follow the library's migration guide and codemods (e.g. MUI v4 → latest: `@material-ui/*` → `@mui/*`, `makeStyles`/`withStyles` (JSS) → `sx` / `styled`, `createMuiTheme` → `createTheme`, `@mui/codemod`). Check every screen for layout and spacing changes and fix them with theme settings, not overrides.

## Vulnerability fixes

Fix issues reported by common code and dependency scans (e.g. `npm audit`, Snyk, SonarQube, Semgrep, CodeQL, OWASP ZAP findings that trace to frontend code), as long as the fix stays in the frontend and keeps parity.

| Finding | Fix |
|---|---|
| Vulnerable or outdated packages (CRA's toolchain is a common source) | The upgrade replaces most of them. For the rest, use a patched version with the same API. |
| XSS through `dangerouslySetInnerHTML` | Render as text, or sanitize first if HTML is genuinely needed (DOMPurify is the one allowed new dependency; tell the user) |
| `javascript:` URLs from data in `href`/`src` | Allow only `http(s)`, `mailto` and relative URLs |
| Direct DOM writes (`innerHTML` via refs, `document.write`) | Use React rendering |
| `eval`, `new Function`, string `setTimeout` | Replace with equivalent non-dynamic code |
| Open redirect (e.g. unvalidated `?redirect=`) | Allow only same-origin relative paths; otherwise fall back to the current default page |
| Secrets in `REACT_APP_*` / `VITE_*` / `NEXT_PUBLIC_*` vars or source (they're bundled into the browser) | Remove them from the frontend. Tell the user they must be rotated. |
| `target="_blank"` without `rel` | Add `rel="noopener noreferrer"` |
| Sensitive data in URLs, logs or `console.*` | Remove it from logs; leave URLs alone if the API contract needs them, and flag it |
| Insecure `postMessage` handlers | Check `event.origin` |

Rules for security fixes:
- A fix must not change visible behaviour, API calls or business logic, except to block the exploit itself.
- **Report but don't change** findings whose fix would alter the auth model, an API contract, the backend or infrastructure. Examples: the token is stored in localStorage, missing CSP or security headers set by the server, CORS, server-side validation. Describe the risk and the recommended fix for the user to decide.
- List every security fix in the step report with the scanner rule or ID, the file, and the change.

## Step 10 — Verify each feature

- [ ] Matches the feature inventory entry: every action, business rule, validation, edge case and error/empty state behaves as before
- [ ] Business logic compared line by line with the original source (calculations, conditions, formatting)
- [ ] Builds with no TypeScript or ESLint errors and no React warnings in the console (including StrictMode double-effect issues)
- [ ] Unit and e2e tests pass
- [ ] Works in mock mode, including error states
- [ ] Works against the real API (if the user can run it)
- [ ] Same URLs, same API calls and payloads, same rendering mode
- [ ] Auth: login, logout, token refresh, protected-route redirect and role gating behave the same
- [ ] Responsive at mobile and desktop widths
- [ ] No new custom CSS without a stated reason

## Done checklist

- [ ] On the latest stable React (versions recorded in the report)
- [ ] Feature inventory fully checked off; any deviations approved by the user
- [ ] Every endpoint mocked; the toggle works through env and through `?mock=`
- [ ] Function components and hooks wherever agreed (error boundaries excepted)
- [ ] Only frontend files changed (no backend, API contract, infra or CI/CD changes)
- [ ] Scan findings fixed or reported, and the scans re-run show no new issues
- [ ] Deploy changes (build command, output directory, env var names) listed for the user, not applied
- [ ] All work committed at logical save points; the working tree is clean
- [ ] Mocks and `legacy/` kept; they're removed only through the cleanup step, when the user asks

## Final step — Cleanup (only when the user asks)

Never do this automatically, and never as part of another step. Only start it when the user explicitly asks, after the done checklist is complete. The user may ask for both parts or just one.

1. Confirm the scope with the user (mocks, `legacy/`, or both) and list exactly what will be removed.
2. Tag the current commit first (e.g. `upgrade/pre-cleanup`) so everything can be restored.
3. **Remove the mocks** (its own commit, e.g. `chore(mocks): remove mock API layer`):
   - Delete `src/mocks/` (handlers and fixtures).
   - In `src/lib/api/client.ts`, remove `isMockEnabled`, `apiMockOn`, the dynamic mock import and the mock branch, so `api()` always calls the real client.
   - Remove `VITE_API_MOCK` / `NEXT_PUBLIC_API_MOCK` usage and the "MOCK API" badge.
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
