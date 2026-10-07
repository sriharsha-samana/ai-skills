---
name: vue-nuxt-vuetify-upgrade
description: Detect an app's current Vue / Nuxt / Vuetify setup (Vue 2 or 3, plain Vue CLI/Vite or Nuxt 2/Bridge/3/4, Vuetify 1.5/2/3, Vuex or Pinia) and upgrade it to the latest stable Nuxt + Vue 3 + Vuetify + Pinia with full feature parity, a switchable mock API layer and frontend-only security fixes. Use for any Vue, Nuxt or Vuetify version upgrade or migration.
---

# Vue / Nuxt / Vuetify → Latest Upgrade

Goal: work out what the app runs on today, pick the right upgrade path, and bring it to the latest stable Nuxt + Vue 3 + Vuetify + Pinia, using the Composition API, without changing what the app does.

## Non-negotiables

These hold for the whole upgrade and override every default below, including the scaffold.

1. **Feature and functional parity.** Every feature, screen, flow, validation, permission check, edge case and error message in the current version must work the same afterwards. Nothing is dropped, merged or "simplified away".
2. **Preserve business logic.** Port calculations, rules, conditions, formatting and data transforms exactly, including quirks the code depends on. Change syntax, never the logic. If logic looks wrong, flag it to the user; don't fix it.
3. **Preserve APIs.** Same endpoints, methods, headers, query params, request bodies, response handling, error handling and call order or timing (e.g. debounce, polling, retries).
4. **Preserve application behaviour.** Same URLs, redirects, auth flow, session handling, stored keys (localStorage, cookies), rendering mode (SPA / SSR / static), loading and empty states, and side effects (analytics, downloads, notifications).
5. **Frontend only.** Change only frontend code in this repo. Never change the backend, API contracts, database, infrastructure, CI/CD or deployment. If parity seems to need a non-frontend change, stop and ask.
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
| Versions | Latest **stable** Nuxt, Vue, Vuetify, Pinia and `vuetify-nuxt-module`. Look them up when you start (`npm view <pkg> version`) and check each one's official upgrade guide. Never assume versions from memory. |
| Framework | Nuxt with its current recommended directory structure (`app/` source directory in Nuxt 4+), TypeScript |
| Component style | Composition API with `<script setup lang="ts">`. No mixins, filters or event buses. |
| UI | Vuetify through `vuetify-nuxt-module`. Built-in components and utility classes first. |
| Styling | As little custom CSS as possible. Colours, typography and component defaults come from the Vuetify theme and config. |
| State | Pinia setup stores via `@pinia/nuxt` |
| Routing | Nuxt file-based routing in `pages/`. All existing URLs stay the same. |
| Data | All HTTP goes through one `$api` layer, with a mock mode switched on or off by one setting |
| Auth | The existing auth model and flow, ported as-is |
| Rendering | Same mode as the current app: plain Vue SPA → `ssr: false`; Nuxt keeps its current `ssr` / `target` / `mode` |
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

- Detect and audit first. Report before changing code.
- Port or upgrade one feature, route or concern at a time. The app must build and run after each step.
- Don't add libraries beyond the target stack and what the scaffold uses. Ask first.
- When unsure about a library's API or version, check its docs or changelog. Do not guess.

---

## Step 0 — Detect the current setup

Read `package.json` and the lockfile (for the versions actually installed), plus the config files. Report a table like this:

| Check | Where to look | Possible results |
|---|---|---|
| Vue | `vue` version | 2.6 or older / 2.7 / 3.x |
| Nuxt | `nuxt`, `nuxt-edge`, `@nuxt/bridge`; `nuxt.config.*` | none / Nuxt 2 / Nuxt Bridge / Nuxt 3 / Nuxt 4+ |
| Build tool (no Nuxt) | `@vue/cli-service`, `vue.config.js`, `vite.config.*`, `webpack.config.*` | Vue CLI / Vite / custom webpack |
| Vuetify | `vuetify` version; `@nuxtjs/vuetify`, `vuetify-loader`, `vite-plugin-vuetify`, `vuetify-nuxt-module` | 1.5 / 2.x / 3.x (current or older minor) |
| Store | `vuex`, `pinia`, `store/` (Nuxt 2 auto-modules) | Vuex 3 / Vuex 4 / Pinia / none |
| Router | `vue-router` version, `src/router/*`, `pages/` | vue-router 3 / 4 / Nuxt pages |
| HTTP | `axios`, `@nuxtjs/axios`, `$fetch`/`ofetch`, `fetch` | — |
| Auth | `@nuxtjs/auth`, `@nuxtjs/auth-next`, `@sidebase/nuxt-auth`, `nuxt-auth-utils`, custom | — |
| Rendering | `ssr`, `target`, `mode` in `nuxt.config`; `generate` scripts | SPA / SSR / static |
| Component style | count of `export default {` vs `<script setup>` | Options / Composition / mixed |
| Language | `tsconfig.json`, `lang="ts"` | JS / TS / mixed |
| Other Vue libraries | e.g. `vue-i18n`, `@nuxtjs/i18n`, `vue-meta`, `@vue/test-utils`, chart and editor wrappers | Vue 3 support yes / no |

Then choose the path:

| Current setup | Path |
|---|---|
| Vue 2.x SPA (Vue CLI / webpack / Vite), any Vuetify | **R1: rebuild** as a new Nuxt app |
| Nuxt 2 or Nuxt Bridge (Vue 2) | **R2: rebuild** as a new Nuxt app |
| Vue 3 SPA (no Nuxt) | **R3: rebuild** as a Nuxt app (moves to file routing). Components mostly port as-is. |
| Nuxt 3 / older Nuxt 4+ with Vue 3 | **U: upgrade in place** |

Vuetify and Vuex/Pinia changes apply within whichever path is chosen.

- **Rebuild paths (R1–R3):** move the current code to `legacy/` and build the new app beside it, feature by feature.
- **In-place path (U):** upgrade on the existing code with no `legacy/` move, unless the user asks for one. Ask whether existing Options API components should be converted to the Composition API now. The default is to convert only the files you otherwise touch.

State which path you chose and why, and wait for the user to confirm.

## Step 1 — Audit (report only, no edits)

1. **Routes:** path, name, component, meta, guards, redirects, nesting. For Nuxt, also page-level `middleware`, `layout`, `validate` and `key`.
2. **API endpoints:** every call with its method, path, params, body, response shape and the file that makes it, plus the base URL and env vars.
3. **Auth:** where the token or session is stored (cookie name, localStorage key; for `@nuxtjs/auth` typically `auth._token.<strategy>`), how it's attached, the login, logout and refresh endpoints, 401/403 handling, guards, and role or permission checks.
4. **Store:** modules and what state, getters, mutations and actions each has, plus `nuxtServerInit` if present.
5. **Global pieces:** plugins (and what they inject), mixins, filters, directives, the event bus, `Vue.prototype` and `inject()` additions, global components and global styles.
6. **Third-party libraries** and their target-stack equivalents. Blockers go first.
7. **Deployment:** the build command, output directory, hosting, env files, CI config and rendering mode. Record these; don't change them.
8. Conventions taken from the scaffold, if one was provided.
9. **Feature inventory** (the parity baseline): every screen and its user actions, business rules (validations, calculations, conditional display, permission checks), side effects, and error and empty states, with the source file for each. Each item is later checked off when ported or upgraded.
10. **Security findings:** results of `npm audit` or whichever scanner the user names, plus manual checks from "Vulnerability fixes". Give each finding a severity and a proposed fix.
11. A proposed order of work, starting with the smallest and lowest-risk.

## Step 2 — Move existing code to `legacy/` (rebuild paths only)

1. Use `git mv` so file history is kept.
2. **Keep at the repo root:** `.git`, CI/CD config, deploy and hosting config (Dockerfile, `netlify.toml`, `vercel.json`, nginx config and similar), `.env*` files, `README`, `LICENSE`. Before moving, list what stays and what moves, and confirm with the user.
3. Move everything else into `legacy/`.
4. Make sure no tooling picks up `legacy/`: add it to the Nuxt `ignore` option, the tsconfig `exclude` and the ESLint ignores.
5. `legacy/` is a read-only reference. Never edit it, and don't delete it until the user approves. It can still run on its own (`cd legacy && npm install && npm run dev` or `npm run serve`) for side-by-side comparison.

## Step 3 — Set up the target app

**Rebuild paths:** copy the scaffold if one was provided. Otherwise:

```
app/
  app.vue                 # <v-app> + <NuxtLayout><NuxtPage/></NuxtLayout>
  error.vue               # error page
  layouts/default.vue     # v-app-bar, v-navigation-drawer, v-main
  pages/                  # file-based routes
  components/             # auto-imported
  composables/            # useXxx() — shared logic, replaces mixins
  stores/                 # Pinia setup stores
  middleware/             # route guards
  plugins/                # api.ts, etc.
  mocks/                  # mock handlers + fixtures (loaded only in mock mode)
  utils/                  # pure helpers (replaces filters)
  types/                  # API/domain types
public/
server/                   # only if Nuxt 2 serverMiddleware existed — port it 1:1
nuxt.config.ts
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  ssr: false, // set to match the current app's rendering mode
  modules: ['@pinia/nuxt', 'vuetify-nuxt-module'],
  ignore: ['legacy/**'],
  runtimeConfig: {
    public: {
      apiBase: '',     // NUXT_PUBLIC_API_BASE — same value the current app uses
      apiMock: false,  // NUXT_PUBLIC_API_MOCK=true to turn mocks on
    },
  },
  vuetify: {
    vuetifyOptions: {
      theme: { themes: { light: { colors: { /* current brand colours */ } } } },
      defaults: { /* e.g. VBtn: { variant: 'flat' } — set once here, not per component */ },
    },
  },
})
```

**In-place path:** bump the packages one major version at a time (Nuxt, then Vuetify, then the rest). Follow each official upgrade guide and its codemods, then fix the build and run the tests before the next bump.

If the build command or output directory changes (for example `.output/public` from `nuxi generate`, or `.output/server` for SSR), tell the user the deploy pipeline will need updating at cut-over. Do not change it yourself.

## Step 4 — API layer and mock mode

Every HTTP call goes through `$api`. Components never call `fetch`, `axios`, `this.$axios` or `$fetch` directly.

### Toggle

- **Default:** `NUXT_PUBLIC_API_MOCK=true` or `false` in `.env`.
- **Dev-only override:** add `?mock=on` or `?mock=off` to any URL. The choice is remembered in a cookie (SSR-safe) and ignored in production builds.
- When mocks are on, show a `<v-chip color="warning" size="small">MOCK API</v-chip>` in the app bar.

```ts
// app/plugins/api.ts
export default defineNuxtPlugin(async () => {
  const { apiBase, apiMock } = useRuntimeConfig().public
  const mockOn = isMockEnabled(apiMock)

  const http = $fetch.create({
    baseURL: apiBase,
    onRequest({ options }) {
      // Port the current request interceptor exactly (same header, same token source)
    },
    async onResponseError({ response }) {
      // Port the current 401/403/refresh handling exactly
    },
  })

  const mock = mockOn ? (await import('~/mocks')).handleMock : null

  function api<T>(url: string, opts: Parameters<typeof http>[1] = {}): Promise<T> {
    return mock ? mock<T>(url, opts) : http<T>(url, opts)
  }

  return { provide: { api, apiMockOn: mockOn } }
})

function isMockEnabled(envValue: unknown): boolean {
  if (import.meta.dev) {
    const cookie = useCookie<'on' | 'off' | null>('api-mock')
    const q = useRequestURL().searchParams.get('mock')
    if (q === 'on' || q === 'off') cookie.value = q
    if (cookie.value) return cookie.value === 'on'
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
- Build fixtures from the response shapes found in the current code and its types. Use fake data only: no real user data, tokens or secrets.
- Cover error states too (401, 404, validation errors) so the UI's error paths can be tested.
- Mocks are loaded with a dynamic import, so with the toggle off they never run, and the backend, deploy and dependencies stay unchanged.

### Using the API

- Calls are grouped by domain in composables or store actions, e.g. `useUsersApi()` → `$api<User[]>('/users')`.
- Page data: `const { data, status, error } = await useAsyncData('users', () => $api<User[]>('/users'))`.
- Type every response in `types/`.

## Step 5 — Auth (port the existing flow, don't redesign it)

- Use the same storage (same cookie name or localStorage key, including `@nuxtjs/auth` key names), header, endpoints, refresh logic and logout behaviour.
- Auth state lives in a Pinia store, `useAuthStore`, holding the token/user, `login()`, `logout()`, `refresh()` and role checks.
- In SSR apps the token must be readable on the server: if the current app uses cookies, use `useCookie` with the same name. If it uses localStorage in an SSR app, keep that behaviour and flag it.
- Guards become route middleware: `defineNuxtRouteMiddleware((to, from) => { ... return navigateTo(...) })`. Keep the same redirect targets and query params.
- Replacing an auth module (`@nuxtjs/auth`, `@nuxtjs/auth-next`) means reimplementing the same strategy in the store, plugin and middleware. Don't switch to a different auth library or session model unless the user asks.

## Step 6 — Store → Pinia

Use one setup store per Vuex module or Nuxt 2 `store/` file:

```ts
// app/stores/users.ts
export const useUsersStore = defineStore('users', () => {
  const list = ref<User[]>([])                                    // state
  const active = computed(() => list.value.filter(u => u.active)) // getters
  async function fetchAll() {                                     // actions (+ mutations merged in)
    list.value = await useNuxtApp().$api<User[]>('/users')
  }
  return { list, active, fetchAll }
})
```

- Mutations go away: assign state directly inside actions.
- `mapState` / `mapGetters` → `storeToRefs(store)`. `mapActions` → call the action directly.
- `rootState` / `rootGetters` → call the other store inside the action.
- `nuxtServerInit` → a server-only plugin (`plugins/init.server.ts`) that calls the same actions. Pinia state is passed to the client automatically.
- Keep state local (`ref` in the component or a composable) if only one page uses it.

## Step 7 — Routing

### From vue-router (R1, R3)

| Current route | Nuxt file |
|---|---|
| `/` | `pages/index.vue` |
| `/users/:id` | `pages/users/[id].vue` |
| `/users/:id?` | `pages/users/[[id]].vue` |
| nested `children` | `pages/users.vue` (with `<NuxtPage/>`) + `pages/users/*.vue` |
| `*` / catch-all | `pages/[...slug].vue` |
| `redirect` | `definePageMeta({ redirect })` or `routeRules` |
| route `name` | keep via `definePageMeta({ name })` if code references it |
| `meta`, `beforeEnter` | `definePageMeta({ ...meta, middleware })` |
| `this.$router` / `this.$route` | `navigateTo()`, `useRouter()` / `useRoute()` |

### From Nuxt 2 pages (R2)

| Nuxt 2 | Latest Nuxt |
|---|---|
| `pages/_id.vue` | `pages/[id].vue` |
| `pages/_.vue` | `pages/[...slug].vue` |
| `<Nuxt/>` in layouts | `<slot/>` |
| `<NuxtChild/>` | `<NuxtPage/>` |
| `layout: 'x'`, `middleware`, `validate`, `key`, `transition` | `definePageMeta({ layout, middleware, validate, key, pageTransition })` |
| `asyncData(ctx)` | `await useAsyncData(key, () => ...)` |
| `fetch()` + `$fetchState` | `useAsyncData` / `useLazyAsyncData` + `status` / `error` |
| `head()` | `useHead()` / `useSeoMeta()` |
| `watchQuery` | `watch(() => route.query, refresh)` or `useAsyncData(..., { watch })` |
| middleware `({ store, redirect, route })` | `defineNuxtRouteMiddleware((to, from) => ...)` |
| plugin `(ctx, inject)` | `defineNuxtPlugin(() => ({ provide: { x } }))` |
| `context.error()` / `this.$nuxt.error()` | `showError()` / `throw createError()` |
| `layouts/error.vue` | `error.vue` at the source root |
| `static/` | `public/` |
| `serverMiddleware` | `server/middleware/` or `server/api/`, ported 1:1 |
| `env`, `publicRuntimeConfig`, `privateRuntimeConfig` | `runtimeConfig` / `runtimeConfig.public` |
| `loading` option / `$nuxt.$loading` | `<NuxtLoadingIndicator/>` / `useLoadingIndicator()` |
| `@nuxtjs/axios` | `$api` (Step 4) |
| `@nuxtjs/vuetify` | `vuetify-nuxt-module` |

Every current URL must resolve to the same screen. Check this against the route list from the audit.

### Nuxt 3 → latest (U)

- Follow the official upgrade guide for each major, and run its codemods where provided. For Nuxt 4 that includes the `app/` directory move, `useAsyncData`/`useFetch` `data` becoming a shallow ref (mutating nested data no longer triggers reactivity — use `deep: true` or replace the value), and `data`/`error` defaulting to `undefined` instead of `null` (check `=== null` comparisons).
- Check modules for compatibility with the new major before bumping.

## Step 8 — Components (Composition API)

Rebuild paths convert every component. On the in-place path, convert the components the user approved (Step 0).

| Vue 2 / Options API | `<script setup>` |
|---|---|
| `props: {...}` | `const props = defineProps<{...}>()` (+ `withDefaults`) |
| `$emit('x')` | `const emit = defineEmits<{ x: [payload: T] }>()` |
| `value` prop + `input` / `.sync` / `modelValue` + `update:modelValue` | `const model = defineModel<T>()` / `defineModel('name')` |
| `data()` | `ref()` / `reactive()` |
| `computed`, `watch` | `computed()`, `watch()` (add `{ deep: true }` for array mutation) |
| `methods` | plain functions |
| `created` | top-level code |
| `mounted`, `beforeDestroy` / `beforeUnmount`, `destroyed` / `unmounted` | `onMounted`, `onBeforeUnmount`, `onUnmounted` |
| mixins | composables |
| filters | functions in `utils/` |
| event bus | Pinia store or props/emits. Use `mitt` only if truly global. |
| `this.$refs.x` | `useTemplateRef('x')` |
| `this.$set` / `$delete` | direct assignment / `delete` |
| `$listeners`, `$scopedSlots` | `$attrs`, `$slots` / `useSlots()` |
| `this.$vuetify.breakpoint` / `.display` | `useDisplay()` |
| `Vue.prototype.$x` / Nuxt 2 `inject('x')` | plugin `provide` → `useNuxtApp().$x` |

Vue 2 template changes to fix: `v-if` takes precedence over `v-for` on the same element; the `.native` modifier is removed; `@keyup.13` → `@keyup.enter`; `slot` / `slot-scope` → `#name="props"`; `/deep/`, `>>>`, `::v-deep` → `:deep()`; `-enter` / `-leave` transition classes → `-enter-from` / `-leave-from`; `:key` on `<template v-for>` goes on the `<template>`.

## Step 9 — Vuetify → latest, minimal custom CSS

Styling order of preference:
1. A Vuetify component with props (`variant`, `density`, `color`, `elevation`, `rounded`).
2. Global defaults or theme in the Vuetify config, for anything repeated.
3. Vuetify utility classes: spacing `pa-4`, `ga-2`; flex `d-flex`, `align-center`; text `text-h5`, `text-medium-emphasis`; colour `bg-surface`, `text-primary`; display `d-none d-md-flex`.
4. Only then a minimal `<style scoped>` that uses theme variables (`rgb(var(--v-theme-primary))`), never hard-coded colours. Explain why it was needed.

Layout uses `v-app`, `v-app-bar`, `v-navigation-drawer`, `v-main`, `v-container`, `v-row`, `v-col`, `v-card`, `v-sheet`. Drop custom CSS when a Vuetify equivalent exists, and don't port global stylesheets wholesale. Visual output may move toward stock Vuetify; behaviour may not.

### Vuetify 1.5 / 2 → 3+

| Vuetify 1.5 / 2 | Vuetify 3+ |
|---|---|
| `v-layout row wrap` / `v-flex xs12 md6` (1.5) | `v-row` / `v-col cols="12" md="6"` |
| `v-content` (1.5), `v-toolbar app` | `v-main`, `v-app-bar` |
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
| `v-data-table` headers `{ text, value }` | `{ title, key }`; slots `item.<key>`; server-side → `v-data-table-server` |
| `#activator="{ on }"` + `v-on="on"` | `#activator="{ props }"` + `v-bind="props"` |
| `v-overflow-btn`, `v-simple-table` | `v-select`, `v-table` |
| `subtitle-1`, `headline` classes | `text-subtitle-1`, `text-h5` |
| `$vuetify.breakpoint`, `$vuetify.goTo` | `useDisplay()`, `useGoTo()` |

### Vuetify 3.x → latest

- Read the release notes for every minor (and major, if one is newer) between the current and target versions.
- Components that have graduated from labs move their imports from `vuetify/labs/*` to the stable components. With `vuetify-nuxt-module` auto-import, just remove the labs imports.
- Check deprecation warnings in the console and fix them all.

## Vulnerability fixes

Fix issues reported by common code and dependency scans (e.g. `npm audit`, Snyk, SonarQube, Semgrep, CodeQL, OWASP ZAP findings that trace to frontend code), as long as the fix stays in the frontend and keeps parity.

| Finding | Fix |
|---|---|
| Vulnerable or outdated packages | The upgrade replaces most of them. For the rest, use a patched version with the same API. |
| XSS through `v-html` / `innerHTML` | Render as text, or sanitize first if HTML is genuinely needed (adding a sanitizer such as DOMPurify is the one allowed new dependency; tell the user) |
| `eval`, `new Function`, string `setTimeout` | Replace with equivalent non-dynamic code |
| Open redirect (e.g. unvalidated `?redirect=`) | Allow only same-origin relative paths; otherwise fall back to the current default page |
| Secrets, API keys or credentials in frontend code | Move them to private `runtimeConfig` (SSR) or env. If a secret was ever shipped to the browser, tell the user it must be rotated. |
| `target="_blank"` without `rel` | Add `rel="noopener noreferrer"` |
| Sensitive data in URLs, logs or `console.*` | Remove it from logs; leave URLs alone if the API contract needs them, and flag it |
| Unvalidated user input passed to URLs or DOM APIs | Encode or validate it |
| Insecure `postMessage` handlers | Check `event.origin` |

Rules for security fixes:
- A fix must not change visible behaviour, API calls or business logic, except to block the exploit itself (e.g. rejecting a malicious redirect).
- **Report but don't change** findings whose fix would alter the auth model, an API contract, the backend or infrastructure. Examples: the token is stored in localStorage, missing CSP or security headers set by the server, CORS, server-side validation. Describe the risk and the recommended fix for the user to decide.
- List every security fix in the step report with the scanner rule or ID, the file, and the change.

## Step 10 — Verify each feature

- [ ] Matches the feature inventory entry: every action, business rule, validation, edge case and error/empty state behaves as before
- [ ] Business logic compared line by line with the original source (calculations, conditions, formatting)
- [ ] Builds with no TypeScript, ESLint, Vue, Nuxt or Vuetify warnings
- [ ] Works in mock mode, including error states
- [ ] Works against the real API (if the user can run it)
- [ ] Same URLs, same API calls and payloads, same rendering mode
- [ ] Auth: login, logout, token refresh, protected-route redirect and role gating behave the same
- [ ] Responsive at mobile and desktop widths
- [ ] No new custom CSS without a stated reason

## Done checklist

- [ ] On the latest stable Nuxt, Vue, Vuetify and Pinia (versions recorded in the report)
- [ ] Feature inventory fully checked off; any deviations approved by the user
- [ ] Every endpoint mocked; the toggle works through env and through `?mock=`
- [ ] No Vuex, mixins, filters or event buses; Composition API wherever agreed
- [ ] Only frontend files changed (no backend, API contract, infra or CI/CD changes)
- [ ] Scan findings fixed or reported, and the scans re-run show no new issues
- [ ] `legacy/` (rebuild paths) untouched; deletion only on user approval
- [ ] Deploy changes (build command, output directory) listed for the user, not applied

## Report after each step

- Files changed
- What was ported or changed, and anything that deviates from current behaviour (with the reason)
- Feature inventory items completed
- Security fixes made (scanner rule or ID, file, change) and findings reported but not fixed
- Validation run and result
- Next step
