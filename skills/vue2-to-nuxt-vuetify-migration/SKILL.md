---
name: vue2-to-nuxt-vuetify-migration
description: Migrate a Vue 2 (+ Vuetify 2, Vuex, vue-router) app to Nuxt + Vue 3 + Vuetify 3 + Pinia using the Composition API. Moves the old code into legacy/, rebuilds feature by feature, keeps the existing auth flow and URLs, and adds a switchable mock API layer. Use for Vue 2 → Vue 3 / Nuxt upgrades, Vuetify 2 → 3 ports, Vuex → Pinia, or vue-router → Nuxt pages.
---

# Vue 2 → Nuxt + Vuetify 3 Migration

Goal: rebuild a working Vue 2 app as a Nuxt app (Vue 3, Vuetify 3, Pinia, Composition API) next to the old code, porting one feature at a time until the new app does everything the old one did.

## Non-negotiables

These hold for the whole migration and override every default below, including the scaffold.

1. **Feature and functional parity.** Every feature, screen, flow, validation, permission check, edge case and error message in the current version must work the same in the new app. Nothing is dropped, merged or "simplified away".
2. **Preserve business logic.** Port calculations, rules, conditions, formatting and data transforms exactly, including quirks the code depends on. Change syntax (Options → Composition API, Vuex → Pinia) but never the logic. If legacy logic looks wrong, flag it to the user; don't fix it.
3. **Preserve APIs.** Same endpoints, methods, headers, query params, request bodies, response handling, error handling and call order or timing (e.g. debounce, polling, retries).
4. **Preserve application behaviour.** Same URLs, redirects, auth flow, session handling, stored keys (localStorage, cookies), loading and empty states, and side effects (analytics, downloads, notifications).
5. **Frontend only.** Change only frontend code in this repo. Never change the backend, API contracts, database, infrastructure, CI/CD or deployment. If parity seems to need a non-frontend change, stop and ask.
6. **Security fixes are allowed** (see "Vulnerability fixes" below), but only within the frontend and without breaking rules 1–5.

Any deviation, however small, must be listed in the step report with its reason and approved by the user.

## Precedence

1. The non-negotiables above
2. Explicit user instructions
3. The scaffold project, if the user provides one (see below)
4. The defaults in this skill

## Preferences (defaults unless the user says otherwise)

| Area | Default |
|---|---|
| Old code | Moved to `legacy/`. Read-only reference. Never edited, never deleted until the user approves. |
| Framework | Latest stable Nuxt with the `app/` source directory, Vue 3, TypeScript |
| Component style | Composition API with `<script setup lang="ts">`. No Options API, mixins, filters or event buses. |
| UI | Vuetify 3 (latest stable) through `vuetify-nuxt-module`. Built-in components and utility classes first. |
| Styling | As little custom CSS as possible. Colours, typography and component defaults come from the Vuetify theme and config. |
| State | Pinia setup stores via `@pinia/nuxt` |
| Routing | Nuxt file-based routing in `app/pages/`. All legacy URLs stay the same. |
| Data | All HTTP goes through one `$api` layer, with a mock mode switched on or off by one setting |
| Auth | The existing auth model and flow, ported as-is |
| Rendering | `ssr: false` (SPA), matching the legacy app's runtime and hosting, unless the user asks for SSR |
| Deployment, backend, other services | Unchanged. Ask before touching CI/CD, hosting or env config. |

## Scaffold project

If the user provides a scaffold project (path, repo or files), read it before writing any code. Take from it:

- folder structure and file naming
- `nuxt.config`, Vuetify config, theme, layouts
- the patterns for the API layer, stores, composables and middleware
- lint, format and TypeScript settings
- component and page examples to copy

Where the scaffold differs from this skill, follow the scaffold. List the conventions you adopted in your Step 1 report. If no scaffold is provided, use the defaults in this skill.

## Rules

- Audit first. Report before changing code.
- Port one feature or route at a time. The app must build and run after each step.
- Behaviour parity: same URLs, same API calls and payloads, same auth behaviour, same validation and the same user-visible flows. Visual polish may change toward stock Vuetify; behaviour may not.
- Never edit `legacy/`. Read it to port from.
- Don't add libraries beyond Nuxt, Vuetify, Pinia and what the scaffold uses. Ask first.
- When unsure about a library's API or version, check its docs. Do not guess.

---

## Step 1 — Audit (report only, no edits)

1. Versions: `vue`, `vuetify`, `vue-router`, `vuex`, HTTP client, `vue-i18n`, build tool, test tools.
2. **Routes:** every route with its path, name, component, meta, guards, redirects and nesting.
3. **API endpoints:** every call with its method, path, params, body, response shape and the file that makes it. Include the base URL and env vars.
4. **Auth:** where the token or session is stored (cookie, localStorage key), how it's attached (header name, interceptor), the login, logout and refresh endpoints, 401/403 handling, route guards, and role or permission checks.
5. **Store:** Vuex modules and what state, getters, mutations and actions each has.
6. **Global pieces:** plugins, mixins, filters, directives, the event bus, `Vue.prototype` additions, global components and global styles.
7. **Third-party Vue 2 libraries** and their Vue 3 replacements. Blockers go first.
8. **Deployment:** the build command, output directory, hosting, env files and CI config. Record these; don't change them.
9. Conventions taken from the scaffold, if one was provided.
10. **Feature inventory** (the parity baseline): every screen and its user actions, business rules (validations, calculations, conditional display, permission checks), side effects, and error and empty states, with the legacy file for each. Each item is later checked off when ported.
11. **Security findings:** results of the scans the user provides or that can be run on the frontend (`npm audit`, or whichever scanner the user names), plus manual checks from "Vulnerability fixes". Give each finding a severity and a proposed fix.
12. A proposed order for porting features, starting with the smallest and lowest-risk.

## Step 2 — Move existing code to `legacy/`

1. Use `git mv` so file history is kept.
2. **Keep at the repo root:** `.git`, CI/CD config (`.github/`, `.gitlab-ci.yml` and similar), deploy and hosting config (Dockerfile, `netlify.toml`, `vercel.json`, nginx config and similar), `.env*` files, `README`, `LICENSE`. Before moving, list what stays and what moves, and confirm with the user.
3. Move everything else, including `src/`, `public/`, `package.json`, the lockfile and build config, into `legacy/`.
4. Make sure no tooling picks up `legacy/`: add it to the Nuxt `ignore` option, the tsconfig `exclude` and the ESLint ignores.
5. Optional: the legacy app can still run on its own (`cd legacy && npm install && npm run serve`) for side-by-side comparison.

## Step 3 — Set up the new Nuxt app

Copy the scaffold if one was provided. Otherwise:

```
app/
  app.vue                 # <v-app> + <NuxtLayout><NuxtPage/></NuxtLayout>
  layouts/default.vue     # v-app-bar, v-navigation-drawer, v-main
  pages/                  # file-based routes
  components/             # auto-imported
  composables/            # useXxx() — shared logic, replaces mixins
  stores/                 # Pinia setup stores
  middleware/             # route guards (auth etc.)
  plugins/                # api.ts, etc.
  mocks/                  # mock handlers + fixtures (loaded only in mock mode)
  utils/                  # pure helpers (replaces filters)
  types/                  # API/domain types
public/
nuxt.config.ts
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  ssr: false,
  modules: ['@pinia/nuxt', 'vuetify-nuxt-module'],
  ignore: ['legacy/**'],
  runtimeConfig: {
    public: {
      apiBase: '',     // NUXT_PUBLIC_API_BASE — same value the legacy app used
      apiMock: false,  // NUXT_PUBLIC_API_MOCK=true to turn mocks on
    },
  },
  vuetify: {
    vuetifyOptions: {
      theme: { themes: { light: { colors: { /* legacy brand colours */ } } } },
      defaults: { /* e.g. VBtn: { variant: 'flat' } — set once here, not per component */ },
    },
  },
})
```

The build output differs from the legacy app (`nuxi generate` writes to `.output/public`). Tell the user the deploy build command and output directory will need updating when they cut over. Do not change them yourself.

## Step 4 — API layer and mock mode

Every HTTP call goes through `$api`. Components never call `fetch` or `axios` directly.

### Toggle

- **Default:** `NUXT_PUBLIC_API_MOCK=true` or `false` in `.env`.
- **Dev-only override:** add `?mock=on` or `?mock=off` to any URL. The choice is remembered in localStorage and ignored in production builds.
- When mocks are on, show a `<v-chip color="warning" size="small">MOCK API</v-chip>` in the app bar.

```ts
// app/plugins/api.ts
export default defineNuxtPlugin(async () => {
  const { apiBase, apiMock } = useRuntimeConfig().public
  const mockOn = isMockEnabled(apiMock)

  const http = $fetch.create({
    baseURL: apiBase,
    onRequest({ options }) {
      // Port the legacy request interceptor exactly (same header, same token source)
    },
    async onResponseError({ response }) {
      // Port legacy 401/403/refresh handling exactly
    },
  })

  const mock = mockOn ? (await import('~/mocks')).handleMock : null

  function api<T>(url: string, opts: Parameters<typeof http>[1] = {}): Promise<T> {
    return mock ? mock<T>(url, opts) : http<T>(url, opts)
  }

  return { provide: { api, apiMockOn: mockOn } }
})

function isMockEnabled(envValue: unknown): boolean {
  if (import.meta.dev && import.meta.client) {
    try {
      const q = new URLSearchParams(location.search).get('mock')
      if (q === 'on' || q === 'off') localStorage.setItem('apiMock', q)
      const saved = localStorage.getItem('apiMock')
      if (saved) return saved === 'on'
    } catch {}
  }
  return String(envValue) === 'true'
}
```

```ts
// app/mocks/index.ts
import users from './fixtures/users.json'

type Ctx = { params: Record<string, string>; query: any; body: any }
const routes: Record<string, (ctx: Ctx) => unknown> = {
  // One entry per endpoint found in the Step 1 audit — same paths, same response shapes
  'POST /auth/login': () => ({ token: 'mock-token', user: users[0] }),
  'GET /users': () => users,
  'GET /users/:id': ({ params }) =>
    users.find(u => String(u.id) === params.id)
    ?? Promise.reject(createError({ statusCode: 404, statusMessage: 'Not found' })),
}

export async function handleMock<T>(url: string, opts: any = {}): Promise<T> {
  const method = String(opts.method ?? 'GET').toUpperCase()
  const path = url.split('?')[0]
  for (const [key, handler] of Object.entries(routes)) {
    const [m, pattern] = key.split(' ')
    const params = m === method ? match(pattern, path) : null
    if (params) {
      await new Promise(r => setTimeout(r, 200)) // simulate latency
      return (await handler({ params, query: opts.query, body: opts.body })) as T
    }
  }
  throw createError({ statusCode: 501, statusMessage: `No mock for ${method} ${path}` })
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
- Build fixtures from the response shapes found in the legacy code and its types. Use fake data only: no real user data, tokens or secrets.
- Cover error states too (401, 404, validation errors) so the UI's error paths can be tested.
- Mocks are loaded with a dynamic import, so with the toggle off they never run, and the API layer, backend, deploy and dependencies stay unchanged.

### Using the API

- Calls are grouped by domain in composables or store actions, e.g. `useUsersApi()` → `$api<User[]>('/users')`.
- Page data: `const { data, pending, error } = await useAsyncData('users', () => $api<User[]>('/users'))`.
- Type every response in `app/types/`.

## Step 5 — Auth (port the existing flow, don't redesign it)

- Use the same storage (same cookie name or localStorage key), header, endpoints, refresh logic and logout behaviour as the legacy code.
- Auth state lives in a Pinia store, `useAuthStore`, holding the token/user, `login()`, `logout()`, `refresh()` and role checks.
- Legacy `router.beforeEach` guards become `app/middleware/auth.global.ts`, or named middleware applied with `definePageMeta({ middleware: 'auth' })`. Keep the same redirect targets and query params (e.g. `?redirect=`).
- Legacy `meta: { requiresAuth, roles }` becomes `definePageMeta({ requiresAuth: true, roles: [...] })`.
- Do not introduce an auth library, OAuth provider or session-model change unless the user asks.

## Step 6 — Store: Vuex → Pinia

Use one setup store per Vuex module:

```ts
// app/stores/users.ts
export const useUsersStore = defineStore('users', () => {
  const list = ref<User[]>([])                                  // state
  const active = computed(() => list.value.filter(u => u.active)) // getters
  async function fetchAll() {                                   // actions (+ mutations merged in)
    list.value = await useNuxtApp().$api<User[]>('/users')
  }
  return { list, active, fetchAll }
})
```

- Mutations go away: assign state directly inside actions.
- `mapState` / `mapGetters` → `const { list } = storeToRefs(useUsersStore())`. `mapActions` → call `store.fetchAll()` directly.
- `rootState` / `rootGetters` → call the other store inside the action.
- Keep state local (`ref` in the component or a composable) if only one page uses it. Not everything belongs in a store.

## Step 7 — Routes: vue-router → Nuxt pages

| Legacy route | Nuxt file |
|---|---|
| `/` | `pages/index.vue` |
| `/users` | `pages/users/index.vue` |
| `/users/:id` | `pages/users/[id].vue` |
| `/users/:id?` | `pages/users/[[id]].vue` |
| nested `children` | `pages/users.vue` (with `<NuxtPage/>`) + `pages/users/*.vue` |
| `*` / catch-all | `pages/[...slug].vue` |
| `redirect` | `definePageMeta({ redirect })` or `routeRules` |
| route `name` | keep via `definePageMeta({ name: 'legacy-name' })` if code references it |
| `meta` | `definePageMeta({ ... })` |
| `this.$router.push` | `navigateTo()` / `useRouter()` |
| `this.$route` | `useRoute()` |
| layouts by wrapper component | `app/layouts/*.vue` + `definePageMeta({ layout })` |

Every legacy URL must resolve to the same screen. Check this against the route list from the audit.

## Step 8 — Port components (Composition API)

| Vue 2 Options API | Vue 3 `<script setup>` |
|---|---|
| `props: {...}` | `const props = defineProps<{...}>()` (+ `withDefaults`) |
| `$emit('x')` | `const emit = defineEmits<{ x: [payload: T] }>()` |
| `value` prop + `input` event / `.sync` | `const model = defineModel<T>()` / `defineModel('name')` |
| `data()` | `ref()` / `reactive()` |
| `computed` | `computed()` |
| `watch` | `watch()` / `watchEffect()` (add `{ deep: true }` for array mutation) |
| `methods` | plain functions |
| `created` | top-level code in `<script setup>` |
| `mounted`, `beforeDestroy`, `destroyed` | `onMounted`, `onBeforeUnmount`, `onUnmounted` |
| mixins | composables in `app/composables/` |
| filters | functions in `app/utils/`, called in templates |
| event bus (`$on`/`$emit` on a shared Vue) | Pinia store or props/emits. Use `mitt` only if truly global. |
| `this.$refs.x` | `const x = useTemplateRef('x')` |
| `this.$set` / `$delete` | direct assignment / `delete` |
| `$listeners` | included in `$attrs` |
| `$scopedSlots` | `$slots` / `useSlots()` |
| `this.$vuetify.breakpoint` | `const { mdAndUp } = useDisplay()` |
| `Vue.prototype.$x` | provide from a Nuxt plugin → `useNuxtApp().$x` |
| global components | `app/components/` (auto-imported) |

Template breaking changes to fix while porting:
- `v-if` now takes precedence over `v-for` on the same element. Move one of them to a `<template>` or filter in a `computed`.
- `.native` modifier removed. `@keyup.13` → `@keyup.enter`.
- `slot="x"` / `slot-scope` → `#x="props"`.
- `/deep/`, `>>>`, `::v-deep` → `:deep()`.
- Transition classes: `-enter` → `-enter-from`, `-leave` → `-leave-from`.
- `:key` on `<template v-for>` goes on the `<template>`.

## Step 9 — Vuetify 2 → 3, minimal custom CSS

Styling order of preference:
1. A Vuetify component with props (`variant`, `density`, `color`, `elevation`, `rounded`).
2. Global defaults or theme in the Vuetify config, for anything repeated.
3. Vuetify utility classes: spacing `pa-4`, `mt-2`, `ga-2`; flex `d-flex`, `align-center`, `justify-space-between`; text `text-h5`, `text-medium-emphasis`, `font-weight-bold`; colour `bg-surface`, `text-primary`; display `d-none d-md-flex`.
4. Only then a minimal `<style scoped>` that uses theme variables (`rgb(var(--v-theme-primary))`), never hard-coded colours. Explain why it was needed.

Layout uses `v-app`, `v-app-bar`, `v-navigation-drawer`, `v-main`, `v-container`, `v-row`, `v-col`, `v-card`, `v-sheet`. Drop legacy custom CSS when a Vuetify equivalent exists, and don't port global stylesheets wholesale.

Common Vuetify 2 → 3 changes:

| Vuetify 2 | Vuetify 3 |
|---|---|
| `dense` | `density="compact"` |
| `outlined`, `text`, `depressed`, `plain` | `variant="outlined" \| "text" \| "flat" \| "plain"` |
| `small`, `large`, `x-small` | `size="small"` etc. |
| `dark` / `light` on components | `theme="dark"` or the global theme |
| `value` / `input` on inputs, dialogs, menus | `v-model` / `model-value` / `update:model-value` |
| `item-text` | `item-title` |
| `v-list-item-content`, `v-list-item-group` | removed. Use `title`/`subtitle` props; `v-list` with `v-model:selected`. |
| `v-list-item-icon`, `v-list-item-avatar` | `prepend-icon`, `prepend-avatar` or `#prepend` |
| `v-subheader` | `v-list-subheader` |
| `v-tabs-items` / `v-tab-item` | `v-window` / `v-window-item` |
| `<v-btn icon><v-icon>mdi-x</v-icon></v-btn>` | `<v-btn icon="mdi-x" />` |
| `v-data-table` headers `{ text, value }` | `{ title, key }`; slots `item.<key>`; `:items-per-page`; server-side → `v-data-table-server` |
| `v-menu` `#activator="{ on }"` + `v-on="on"` | `#activator="{ props }"` + `v-bind="props"` |
| `v-overflow-btn`, `v-simple-table` | `v-select`, `v-table` |
| `v-layout` / `v-flex` (grid) | `v-row` / `v-col` |
| `subtitle-1`, `headline` classes | `text-subtitle-1`, `text-h5` |
| `$vuetify.breakpoint` | `useDisplay()` |
| `$vuetify.goTo` | `useGoTo()` |

## Vulnerability fixes

Fix issues reported by common code and dependency scans (e.g. `npm audit`, Snyk, SonarQube, Semgrep, CodeQL, OWASP ZAP findings that trace to frontend code), as long as the fix stays in the frontend and keeps parity.

Typical frontend fixes:

| Finding | Fix |
|---|---|
| Vulnerable or outdated packages | The new stack replaces most of them. For the rest, use a patched version with the same API. |
| XSS through `v-html` / `innerHTML` | Render as text, or sanitize first if HTML is genuinely needed (adding a sanitizer such as DOMPurify is the one allowed new dependency; tell the user) |
| `eval`, `new Function`, string `setTimeout` | Replace with equivalent non-dynamic code |
| Open redirect (e.g. unvalidated `?redirect=`) | Allow only same-origin relative paths; otherwise fall back to the legacy default page |
| Secrets, API keys or credentials in frontend code | Move them to runtime config or env. If a secret was ever shipped to the browser, tell the user it must be rotated (that's outside the frontend). |
| `target="_blank"` without `rel` | Add `rel="noopener noreferrer"` |
| Sensitive data in URLs, logs or `console.*` | Remove it from logs; leave URLs alone if the API contract needs them, and flag it |
| Unvalidated user input passed to URLs or DOM APIs | Encode or validate it |
| Insecure `postMessage` handlers | Check `event.origin` |

Rules for security fixes:
- A fix must not change visible behaviour, API calls or business logic, except to block the exploit itself (e.g. rejecting a malicious redirect).
- **Report but don't change** findings whose fix would alter the auth model, an API contract, the backend or infrastructure. Examples: the token is stored in localStorage, missing CSP or security headers set by the server, CORS, server-side validation. Describe the risk and the recommended fix for the user to decide.
- List every security fix in the step report with the scanner rule or ID, the file, and the change.

## Step 10 — Verify each ported feature

- [ ] Matches the feature inventory entry: every action, business rule, validation, edge case and error/empty state behaves as in `legacy/`
- [ ] Business logic compared line by line with the legacy source (calculations, conditions, formatting)
- [ ] Builds with no TypeScript, ESLint or Vue warnings
- [ ] Works in mock mode, including error states
- [ ] Works against the real API (if the user can run it)
- [ ] Same URLs, same API calls and payloads as `legacy/`
- [ ] Auth: login, logout, token refresh, protected-route redirect and role gating behave the same
- [ ] Responsive at mobile and desktop widths
- [ ] No new custom CSS without a stated reason

## Done checklist

- [ ] Every legacy route and feature ported, checked against the audit lists
- [ ] Feature inventory fully checked off; any deviations approved by the user
- [ ] Only frontend files changed (no backend, API contract, infra or CI/CD changes)
- [ ] Scan findings fixed or reported, and the scans re-run on the new app show no new issues
- [ ] Every endpoint mocked; the toggle works through env and through `?mock=`
- [ ] No Options API, mixins, filters, Vuex or event buses in `app/`
- [ ] `legacy/` untouched; deletion only on user approval
- [ ] Deploy changes (build command, output directory) listed for the user, not applied

## Report after each step

- Files changed
- What was ported or changed, and anything that deviates from legacy behaviour (with the reason)
- Feature inventory items completed
- Security fixes made (scanner rule or ID, file, change) and findings reported but not fixed
- Validation run and result
- Next step
