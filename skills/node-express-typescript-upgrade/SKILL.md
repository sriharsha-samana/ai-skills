---
name: node-express-typescript-upgrade
description: Detect a Node.js + Express + TypeScript service's current setup (Node version, Express 4/5, TypeScript and tsconfig, CJS/ESM, auth, DB, test tools) and upgrade it to the latest Node LTS, Express and TypeScript with full API and behaviour parity, contract tests as a safety net, a switchable mock layer for external integrations, and security fixes limited to this service. Use for any Node, Express or TypeScript version upgrade or migration.
---

# Node.js + Express + TypeScript → Latest Upgrade

Goal: work out what the service runs on today and bring it to the latest Node.js LTS, the latest stable Express and the latest stable TypeScript, without changing what the service does.

## Non-negotiables

These hold for the whole upgrade and override every default below, including the scaffold.

1. **Functional parity.** Every route, middleware, background job, queue consumer and startup task must behave the same afterwards.
2. **Preserve business logic.** Change APIs and syntax required by the upgrade, never the logic. Keep calculations, rules, conditions and ordering exactly. If logic looks wrong, flag it to the user; don't fix it.
3. **Preserve the API contract.** Same routes and path patterns, methods, query and body parsing results, status codes, headers, cookies, JSON field names, null and empty handling, and error response bodies.
4. **Preserve application behaviour.** Same auth and session handling (cookie names, secrets, token validation), same DB schema and queries, same outbound calls (URLs, payloads, timeouts, retries), same env vars and config, same logging of business events.
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
| Versions | Latest **Active LTS** Node.js, latest stable Express, latest stable TypeScript and `@types/*` matching them. Look them up when you start (`npm view express version`, `npm view typescript version`, nodejs.org release schedule) and never assume from memory. |
| Approach | Upgrade in place, one concern at a time (Node → TypeScript → Express → other dependencies). No rewrite. |
| Module system | Keep the current one (CommonJS or ESM). Switching is a separate project, only if the user asks. |
| TypeScript | Latest compiler with a tsconfig that matches the Node target. Turn on stricter flags only if the resulting fixes are behaviour-neutral; otherwise list them as follow-ups. |
| Code style | Modern syntax and async/await only in code you otherwise touch, and only when behaviour-neutral |
| Persistence | Keep the current DB library and ORM, upgraded to a compatible version. No schema changes. |
| External integrations | Behind a client module per integration, with a switchable mock layer (Step 4) |
| Auth | The existing model and flow, ported as-is |
| Deployment | Unchanged. If the runtime Node version, Docker base image or start command changes, tell the user; don't change CI/CD yourself. |

## Scaffold project

If the user provides a scaffold or reference project, read it first. Take from it the folder structure, tsconfig, lint and format setup, error-handling and validation patterns, test setup and code conventions. Where it differs from this skill, follow it (except for the non-negotiables). List the conventions you adopted in your Step 1 report.

## Rules

- Detect and audit first. Report before changing code.
- One concern per change. After each step the build, type check, tests and app startup must pass before moving on.
- Don't add runtime dependencies beyond what the upgrade requires and what the scaffold uses. Ask first.
- When unsure about an API or version, check the official migration guide or changelog. Do not guess.

---

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

Report the upgrade path (e.g. Node 16 → latest LTS, Express 4 → 5, TypeScript 4.x → latest) and any blockers (packages with no compatible version).

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
10. A proposed order of steps.

## Step 2 — Safety net before upgrading

- Run the existing tests and record the results. The baseline must be green, or failures must be listed as pre-existing.
- Add **contract (characterization) tests** with `supertest` (or the existing HTTP test tool) for each route in the feature inventory, written against the *current* version: status, relevant headers, exact JSON body (snapshots are fine), plus edge cases for path matching, query parsing (nested objects, arrays), empty bodies and error responses.
- Run them with mocks on (Step 4), so no external system is needed.
- These tests must pass unchanged after every step. Changing a snapshot means a deviation, which needs user approval.

## Step 3 — Upgrade step by step

### 3a. Node.js → latest LTS
- Update `engines`, `.nvmrc` and `@types/node`. Docker image and CI runtime changes are deployment, so list them for the user rather than changing them.
- Fix removed or deprecated APIs: `new Buffer()` → `Buffer.from`/`Buffer.alloc`, `url.parse` → `new URL()` (check behaviour on relative and odd URLs), `punycode` module, deprecated `crypto` and `fs` usages.
- Native modules (e.g. `bcrypt`, `sharp`) need versions built for the new Node. Rebuild and test.
- Keep existing HTTP client libraries. Moving to native `fetch` is optional and only if behaviour (timeouts, errors, redirects) is identical.

### 3b. TypeScript → latest
- Bump `typescript` and the `@types/*` packages, and set the tsconfig `target`/`lib` to what the Node version supports.
- Use `module` / `moduleResolution` that match the module system (`nodenext` for Node projects, or keep the current settings if changing them would alter output). Replace deprecated options flagged by the new compiler.
- Fix new type errors with behaviour-neutral changes only. No `any` sprinkling: use proper types or a narrow `// @ts-expect-error` with a reason.
- Compare compiled output for runtime-relevant differences (e.g. class field semantics with `useDefineForClassFields`, decorators used by TypeORM or class-validator need `experimentalDecorators` kept as-is).

### 3c. Express 4 → latest (5.x+)
Run the official codemod first (`npx @expressjs/codemod upgrade`), then review every change. Key changes (always confirm against the Express migration guide):

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

Upgrade middleware to versions compatible with the target Express (`@types/express`, `cors`, `helmet`, `express-session`, `passport`, `multer`, rate limiters). Don't change their configuration semantics. Check each library's changelog for changed defaults and pin the old values explicitly.

### 3d. Other dependencies
- Update the remaining dependencies to the latest versions compatible with the new stack, one group at a time, following each changelog.
- Replace abandoned packages (e.g. `request`) only when required for the upgrade or a security fix, with identical behaviour.
- ESLint: move to flat config if the new ESLint version requires it, keeping the same rules.

## Step 4 — Mock layer for external integrations

The goal is to run and test the service without any downstream system and without changing other services or deployment.

- Put each outbound integration behind a client module with an interface. If code calls `axios` directly in route handlers, extract a thin client that makes exactly the same calls (a behaviour-preserving refactor).
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

## Step 5 — Auth (port the existing flow, don't redesign it)

- Keep the same session cookie name and options, the secret source, the session store, Passport strategies and serialization, JWT algorithms, issuer/audience and expiry checks, and role checks.
- When upgrading `express-session`, `passport` or JWT libraries, set any option whose *default* changed to the old value explicitly. (Passport 0.6+ changed session handling around login and logout: check `req.logout` callbacks and session regeneration.)
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
- [ ] Only this service changed (no schema, other-service, infra or CI/CD changes)
- [ ] Scan findings fixed or reported; `npm audit` and other scans re-run show no new issues
- [ ] Deployment changes (Node runtime, base image, start command) listed for the user, not applied

## Report after each step

- Step and files changed
- What changed, and anything that deviates from current behaviour (with the reason)
- Feature inventory items verified
- Security fixes made (scanner rule or ID, file, change) and findings reported but not fixed
- Validation run and result
- Next step
