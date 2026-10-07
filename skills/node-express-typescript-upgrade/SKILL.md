---
name: node-express-typescript-upgrade
description: Backend-only. Detect a Node.js + Express + TypeScript service's current setup (Node version, Express 4/5, TypeScript and tsconfig, CJS/ESM, auth, DB, test tools), move it to legacy/ and rebuild it module by module on the latest Node LTS, Express and TypeScript, with full API and behaviour parity, contract tests recorded against legacy, a switchable mock layer for external integrations, and security fixes limited to this backend. Use for any Node, Express or TypeScript version upgrade or migration.
---

# Node.js + Express + TypeScript → Latest Upgrade

Goal: work out what the service runs on today and bring it to the latest Node.js LTS, the latest stable Express and the latest stable TypeScript, without changing what the service does.

## Non-negotiables

These hold for the whole upgrade and override every default below, including the scaffold.

1. **Functional parity.** Every route, middleware, background job, queue consumer and startup task must behave the same afterwards.
2. **Preserve business logic.** Change APIs and syntax required by the upgrade, never the logic. Keep calculations, rules, conditions and ordering exactly. If logic looks wrong, flag it to the user; don't fix it.
3. **Preserve the API contract.** Same routes and path patterns, methods, query and body parsing results, status codes, headers, cookies, JSON field names, null and empty handling, and error response bodies.
4. **Preserve application behaviour.** Same auth and session handling (cookie names, secrets, token validation), same DB schema and queries, same outbound calls (URLs, payloads, timeouts, retries), same env vars and config, same logging of business events.
5. **Backend only.** Change only this service's backend code, build and config. Frontend files the service serves (static files, server-rendered templates) are ported unchanged. Never change frontend apps, other services, the database schema, infrastructure, CI/CD or deployment. If parity seems to need such a change, stop and ask.
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
| Versions | Latest **Active LTS** Node.js, latest stable Express, latest stable TypeScript and `@types/*` matching them. Look them up when you start (`npm view express version`, `npm view typescript version`, nodejs.org release schedule) and never assume from memory. |
| Approach | Move the current code to `legacy/`, create a fresh project on the latest stack at the repo root, and port module by module, applying the Node, TypeScript and Express changes while porting. `legacy/` is the read-only reference and the parity baseline. |
| Module system | Same as legacy (CommonJS or ESM) unless the user asks to switch |
| TypeScript | Latest compiler with a tsconfig that matches the Node target, `strict` on in the new project. Typing fixes while porting must be behaviour-neutral; if strictness would force a logic change, keep the logic and flag it. |
| Code style | Modern syntax and async/await only in code you otherwise touch, and only when behaviour-neutral |
| Persistence | Keep the current DB library and ORM, upgraded to a compatible version. No schema changes. |
| External integrations | Behind a client module per integration, with a switchable mock layer (Step 5) |
| Auth | The existing model and flow, ported as-is |
| Deployment | Unchanged. If the runtime Node version, Docker base image or start command changes, tell the user; don't change CI/CD yourself. |

## Scaffold project

If the user provides a scaffold or reference project, read it first. Take from it the folder structure, tsconfig, lint and format setup, error-handling and validation patterns, test setup and code conventions. Where it differs from this skill, follow it (except for the non-negotiables). List the conventions you adopted in your Step 1 report.

## Rules

- Detect and audit first. Report before changing code.
- Port one module or feature per change. After each step the build, type check, tests and app startup must pass before moving on.
- Never edit `legacy/`. Read it to port from.
- Don't add runtime dependencies beyond what the upgrade requires and what the scaffold uses. Ask first.
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
| Node version | `engines` in `package.json`, `.nvmrc`, `.node-version`, Dockerfile `FROM`, CI config |
| Package manager | lockfile (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`) |
| Express | `express` and `@types/express` versions |
| TypeScript | `typescript` version; `tsconfig.json` `target`, `module`, `moduleResolution`, `strict`, `paths`, `outDir` |
| Module system | `"type": "module"` in `package.json`, `import`/`require` usage |
| Build and run | `tsc`, `ts-node`, `tsx`, `ts-node-dev`, `nodemon`, `swc`, `esbuild`; the start script |
| Middleware | `body-parser`, `cors`, `helmet`, `cookie-parser`, `express-session`, `morgan`, `multer`, `express-async-errors`, rate limiters |
| Auth | `passport` (+ strategies), `express-session` (+ store), `jsonwebtoken`, `jose`, custom middleware |
| DB | Prisma, TypeORM, Sequelize, Knex, Mongoose, `pg`, `mysql2`, etc. |
| HTTP clients | `axios`, `node-fetch`, `got`, `request` (deprecated), native `fetch` |
| Validation | `joi`, `zod`, `express-validator`, `class-validator` |
| Tests | Jest, Mocha, Vitest, `supertest`, coverage level |
| Lint and format | ESLint config format (legacy `.eslintrc` vs flat `eslint.config.*`), Prettier |

Report the version gap (e.g. Node 16 → latest LTS, Express 4 → 5, TypeScript 4.x → latest), which porting notes from Step 4 apply, and any blockers (packages with no compatible version).

## Step 1 — Audit (report only, no edits)

1. **Routes:** every route with its method, path pattern (note wildcards, optional params and regex), middleware chain, request parsing, response shape and status codes.
2. **Outbound integrations:** each HTTP API, queue, email or SMS provider, storage or other external service, with its client code, env vars, timeouts and retries.
3. **Auth:** session cookie name, secret and store; JWT library, algorithms, verify options and claims checked; roles and permissions.
4. **Background work:** cron jobs, workers, queue consumers, startup tasks.
5. **Config:** env vars and config files.
6. **Deployment:** build command, start command, runtime Node version, Docker image. Record these; don't change them.
7. **Feature inventory** (the parity baseline): every route and job, its business rules, side effects (DB writes, outbound calls, emails) and error cases.
8. **Security findings:** `npm audit` (or the user's scanner), plus manual checks from "Vulnerability fixes". Give each finding a severity and a proposed fix.
9. Conventions taken from the scaffold, if provided.
10. A proposed porting order, starting with the smallest and lowest-risk features.

## Step 2 — Move existing code to `legacy/`

1. Use `git mv` so file history is kept.
2. **Keep at the repo root:** `.git`, CI/CD config, deploy and hosting config (Dockerfile, k8s manifests, Procfile and similar), `.env*` files, `README`, `LICENSE`. Before moving, list what stays and what moves, and confirm with the user.
3. Move everything else (`src/`, `test/`, `package.json`, the lockfile, `tsconfig.json`, build and lint config and so on) into `legacy/`.
4. Make sure the new project's tooling ignores `legacy/`: tsconfig `include: ["src"]` (or `exclude`), ESLint ignores, and test runner roots or ignore patterns.
5. `legacy/` is a read-only reference. Never edit it, and don't delete it until the user approves. It can still run on its own (`cd legacy && npm install && npm start`, on its original Node version) for side-by-side comparison.
6. Root deploy config (e.g. the Dockerfile) still describes the legacy build. List what will need to change at cut-over for the user; don't change it yourself.

## Step 3 — Contract tests (the parity safety net)

- Run the legacy tests and record the results as the baseline (pre-existing failures listed).
- In the new project, add **black-box HTTP contract tests** (e.g. `test/contract/`) for each route in the feature inventory: status, relevant headers, cookies, and the exact JSON body (snapshots are fine). Include edge cases for path matching, query parsing (nested objects, arrays), empty bodies and error responses. The tests take the target from `CONTRACT_BASE_URL`.
- **Record the snapshots against the running legacy app.** Don't add a mock toggle to `legacy/`. If its downstream URLs come from env, point them at a local stub server that serves the same fixtures as the new app's mocks. Otherwise record against the dev environment.
- Run the same tests against the new app (mocks on) for every ported feature. They must pass unchanged. Changing a snapshot means a deviation, which needs user approval.

## Step 4 — New project and porting

1. Create the new project at the root from the scaffold if one was provided. Otherwise use a fresh `package.json` (same name, scripts and module type as legacy), the latest Node LTS in `engines` and `.nvmrc`, the latest TypeScript with a fresh `tsconfig.json`, and the same folder layout as legacy.
2. Dependencies: the same libraries as legacy, at their latest versions compatible with the new stack. Swap abandoned packages (e.g. `request`) only when required for the upgrade or a security fix, with identical behaviour.
3. Port in this order, one change at a time:
   1. Config and env loading, with the same variable names and defaults.
   2. App bootstrap and middleware: the same middleware, **in the same order**, with the same options. Set explicitly any option whose default changed.
   3. Error handler, with the same error response format and status codes.
   4. Auth (Step 6).
   5. Outbound clients and mocks (Step 5).
   6. DB layer: the same schema, models and queries. Copy migrations unchanged; no new migrations.
   7. Routes, feature by feature, with the same paths and handlers.
   8. Static files and templates, copied unchanged.
   9. Background jobs, workers and consumers.
4. While porting, apply the porting notes below. After porting a group of routes, the official Express codemod (`npx @expressjs/codemod upgrade`) can be run on the new code, but review every change it makes.
5. Run the contract tests for each ported feature.

### Porting notes: Node.js → latest LTS
- Docker image and CI runtime changes are deployment, so list them for the user rather than changing them.
- Replace removed or deprecated APIs: `new Buffer()` → `Buffer.from`/`Buffer.alloc`, `url.parse` → `new URL()` (check behaviour on relative and odd URLs), `punycode` module, deprecated `crypto` and `fs` usages.
- Native modules (e.g. `bcrypt`, `sharp`) need versions built for the new Node.
- Keep the existing HTTP client libraries. Moving to native `fetch` is only allowed if behaviour (timeouts, errors, redirects) is identical.

### Porting notes: TypeScript → latest
- Set the tsconfig `target`/`lib` to what the Node version supports, and `module`/`moduleResolution` to match the module system (`nodenext` for Node projects).
- Don't carry over deprecated compiler options; replace them with their current equivalents.
- Type fixes must be behaviour-neutral. No `any` sprinkling: use proper types or a narrow `// @ts-expect-error` with a reason.
- Watch for runtime-relevant compiler semantics: class fields under `useDefineForClassFields`, and decorators used by TypeORM or class-validator (keep `experimentalDecorators` / `emitDecoratorMetadata` if legacy used them).

### Porting notes: Express 4 → latest (5.x+)
Key changes (always confirm against the Express migration guide):

| Express 4 | Express 5 | Parity note |
|---|---|---|
| `app.get('*', ...)`, `'/files/*'` | named wildcards: `'/*splat'`, `'/files/*splat'` | `'/*splat'` doesn't match `/`. Use `'/{*splat}'` to include root. |
| optional `'/:id?'` | `'/{:id}'` | |
| regex characters in string paths (`'/user(s)?'`) | not supported. Use explicit routes or a RegExp. | Test each in the contract tests. |
| `req.param('x')` | `req.params.x` / `req.body.x` / `req.query.x` | |
| `res.send(status, body)`, `res.json(obj, status)` | `res.status(status).send(body)` / `.json(obj)` | |
| `res.sendfile` | `res.sendFile` | |
| `app.del` | `app.delete` | |
| `res.redirect('back')`, `res.location('back')` | `res.redirect(req.get('Referrer') \|\| '/')` | |
| query parser defaulted to `extended` (nested objects) | default `simple` | Keep parity with `app.set('query parser', 'extended')` if any code reads nested or array query params |
| `express.urlencoded()` default `extended: true` | default `extended: false` | Set `extended` explicitly to the old value |
| `req.body` is `{}` when no parser ran | `req.body` is `undefined` | Check code that reads `req.body` without a parser |
| rejected promises in async handlers not caught | forwarded to the error handler automatically | `express-async-errors` can be removed. Unhandled rejections now yield the error handler's response instead of a hang or crash. Report this as an (improving) deviation. |
| `req.host` without port | includes port | Check code using `req.host` |
| `express.static` served dotfiles by default in some setups | `dotfiles` defaults to `'ignore'` | Set `dotfiles` explicitly if `.well-known` or similar must be served |

Use middleware versions compatible with the target Express (`@types/express`, `cors`, `helmet`, `express-session`, `passport`, `multer`, rate limiters). Don't change their configuration semantics. Check each library's changelog for changed defaults and pin the old values explicitly.

### Porting notes: other tooling
- For every other library, check its changelog between the legacy and new versions for changed defaults, and set the old values explicitly.
- ESLint: use flat config (`eslint.config.*`) in the new project, with the same rules as legacy.

## Step 5 — Mock layer for external integrations

The goal is to run and test the service without any downstream system and without changing other services or deployment.

- Put each outbound integration behind a client module with an interface. Where legacy calls `axios` directly in route handlers, port it as a thin client that makes exactly the same calls.
- Choose the real or mock implementation in one place, driven by one env var.

```ts
// src/config.ts
export const config = {
  mockExternals: process.env.MOCK_EXTERNALS === 'true',
  // ...existing config
}
```

```ts
// src/clients/payments/index.ts
import { config } from '../../config'
import type { PaymentsClient } from './types'
import { httpPaymentsClient } from './http'
import { mockPaymentsClient } from './mock'

export const paymentsClient: PaymentsClient =
  config.mockExternals ? mockPaymentsClient : httpPaymentsClient
```

```ts
// at startup (e.g. src/server.ts)
if (config.mockExternals) logger.warn('MOCK EXTERNALS ENABLED — no calls go to real downstream systems')
```

Mock rules:
- Toggle: `MOCK_EXTERNALS=true npm run dev`, or set it in `.env`. The default is off.
- Mock every outbound integration from the audit. Mocks return fixtures (e.g. `src/clients/<name>/fixtures/*.json`) with the real response shapes, and can simulate error cases (timeouts, 4xx/5xx).
- Fake data only: no real customer data, tokens or secrets.
- The database stays real (local or dev). Swapping in an in-memory DB changes query behaviour, so only do it if the user asks.
- Contract tests run with mocks on.

## Step 6 — Auth (port the existing flow, don't redesign it)

- Keep the same session cookie name and options, the secret source, the session store, Passport strategies and serialization, JWT algorithms, issuer/audience and expiry checks, and role checks.
- When moving to newer `express-session`, `passport` or JWT library versions, set any option whose *default* changed to the old value explicitly. (Passport 0.6+ changed session handling around login and logout: check `req.logout` callbacks and session regeneration.)
- If a new version rejects the current setup (e.g. a weak secret or a missing algorithm), stop and tell the user. Changing secrets invalidates sessions and tokens.
- Don't introduce a new auth library or model unless the user asks.

## Vulnerability fixes

Fix issues reported by common scans (`npm audit`, Snyk, SonarQube, Semgrep, CodeQL) when the fix stays inside this service and keeps parity.

| Finding | Fix |
|---|---|
| Vulnerable dependencies | Patched version with the same API (the upgrade covers most) |
| SQL or NoSQL injection | Parameterized queries / ORM bindings; reject operator objects (`$where`, `$ne`) in user input for Mongo |
| Command injection (`exec` with input) | `execFile`/`spawn` with argument arrays, no shell |
| Path traversal (`sendFile`/`fs` with request input) | `root` option or normalize and check against the base directory |
| Prototype pollution (deep-merging `req.body`/`req.query`) | Skip `__proto__`, `constructor` and `prototype` keys, or use safe merge utilities |
| SSRF (user-controlled outbound URLs) | Restrict to the hosts the feature legitimately uses |
| ReDoS (catastrophic regexes on input) | Rewrite as an equivalent safe regex, or limit input length |
| Open redirect | Allow only relative or allow-listed targets; fall back to the current default |
| Hardcoded secrets | Move to env or config. Tell the user to rotate anything that was committed. |
| Secrets or PII in logs | Remove or mask them |
| JWT verify without pinned `algorithms`, or `jwt.decode` used for auth | Pin the algorithm in use and use `verify` with the existing options |
| `eval` / `new Function` / `vm` on input | Replace with equivalent non-dynamic code |

Rules for security fixes:
- A fix must not change the API contract or business logic, except to block the exploit itself.
- **Report but don't change** findings whose fix changes behaviour clients rely on, or that live outside this service: adding `helmet` or security headers, tightening CORS, adding CSRF protection or rate limiting, password hashing changes (they break existing hashes), cookie flag changes, TLS and infrastructure. Describe the risk and the recommended fix for the user to decide.
- List every fix with the scanner rule or ID, the file, and the change.

## Verify after each step

- [ ] `tsc --noEmit` and lint clean, with no new warnings
- [ ] Existing tests and contract tests pass unchanged (any snapshot change approved)
- [ ] Builds and starts with the production start command
- [ ] Works with mocks on and against real dev dependencies (if the user can run them)
- [ ] Auth: login, session, protected routes, token validation and roles behave the same
- [ ] Background jobs and consumers start and run

## Done checklist

- [ ] On the latest Node LTS, Express and TypeScript (versions recorded in the report)
- [ ] Feature inventory fully checked off; any deviations approved
- [ ] Every outbound integration mockable; the toggle works through `MOCK_EXTERNALS`
- [ ] Only backend code changed (no frontend apps, schema, other-service, infra or CI/CD changes)
- [ ] `legacy/` untouched; deletion only on user approval
- [ ] Scan findings fixed or reported; `npm audit` and other scans re-run show no new issues
- [ ] Deployment changes (Node runtime, base image, start command) listed for the user, not applied
- [ ] All work committed at logical save points; the working tree is clean
- [ ] Mocks and `legacy/` kept; they're removed only through the cleanup step, when the user asks

## Final step — Cleanup (only when the user asks)

Never do this automatically, and never as part of another step. Only start it when the user explicitly asks, after the done checklist is complete. The user may ask for both parts or just one.

1. Confirm the scope with the user (mocks, `legacy/`, or both) and list exactly what will be removed.
2. Tag the current commit first (e.g. `upgrade/pre-cleanup`) so everything can be restored.
3. **Remove the mocks** (its own commit, e.g. `chore(mocks): remove mock API layer`):
   - Delete each client's `mock.ts` and `fixtures/`.
   - In each client's `index.ts`, export the real client directly (remove the ternary and the mock import).
   - Remove `mockExternals` from `src/config.ts` and the startup warning.
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
