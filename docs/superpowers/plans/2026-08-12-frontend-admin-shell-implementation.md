# Frontend Admin Shell Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first operational Vue admin frontend for TrackVision with login, permission-aware navigation, read-only security lists, and CRUD screens for parent admin domain resources.

**Architecture:** Scaffold a Vue 3 + Vite + TypeScript application and keep HTTP access inside typed services. Pinia owns authentication state and probed effective permissions, Vue Router enforces route access through `meta.permission`, and pages compose small base components, domain forms, services, and loading/error/empty states. The UI is a dense operational admin tool, not a landing page.

**Tech Stack:** Vue 3, Vite, TypeScript, Vue Router, Pinia, Vitest, Vue Test Utils, Playwright, ESLint, CSS modules/global CSS, `lucide-vue-next`.

## Global Constraints

- Frontend code lives only in `RIALMA-TrackVision-Frontend`.
- AgentOps documentation lives only in `RIALMA-TrackVision-AgentOps`.
- Work from frontend `main`; create implementation branch `codex/frontend-admin-shell` before code changes.
- Follow `docs/frontend-vue-guidelines.md`.
- Use Vue 3 Composition API with `<script setup>`.
- Components stay small and named with two or more words, except `App`.
- Base components use prefix `Base`.
- Single-instance navigation components use prefix `The`.
- API calls stay in services or stores, never scattered directly in pages.
- Pinia stores own shared auth/session state.
- Vue Router guards enforce authentication and route permission metadata.
- No UI framework dependency in this phase.
- `lucide-vue-next` is allowed for icons.
- No secrets in frontend env variables.
- Use `VITE_API_BASE_URL` for backend base URL, ending at `/api/v1`.
- Runtime has no mocked data; mocks are only for tests.
- Users, roles, and permissions are read-only because backend currently exposes only list endpoints.
- CRUD is required for vehicles, locations, edge nodes, cameras, and camera pairs.
- Camera password is write-only and never displayed after save.
- Run verification before each commit for the touched scope.

---

## File Structure Map

Frontend files to create or modify:

- Create `package.json`
- Create `index.html`
- Create `vite.config.ts`
- Create `tsconfig.json`
- Create `tsconfig.node.json`
- Create `eslint.config.js`
- Create `playwright.config.ts`
- Create `.env.example`
- Create `src/main.ts`
- Create `src/App.vue`
- Create `src/app/config.ts`
- Create `src/styles/main.css`
- Create `src/types/api.ts`
- Create `src/types/auth.ts`
- Create `src/types/admin.ts`
- Create `src/services/apiClient.ts`
- Create `src/services/authService.ts`
- Create `src/services/permissionProbeService.ts`
- Create `src/services/usersService.ts`
- Create `src/services/rolesService.ts`
- Create `src/services/permissionsService.ts`
- Create `src/services/vehiclesService.ts`
- Create `src/services/locationsService.ts`
- Create `src/services/edgeNodesService.ts`
- Create `src/services/camerasService.ts`
- Create `src/services/cameraPairsService.ts`
- Create `src/stores/authStore.ts`
- Create `src/router/index.ts`
- Create `src/components/base/BaseAlert.vue`
- Create `src/components/base/BaseButton.vue`
- Create `src/components/base/BaseInput.vue`
- Create `src/components/base/BaseModal.vue`
- Create `src/components/base/BaseSelect.vue`
- Create `src/components/base/BaseTable.vue`
- Create `src/components/navigation/TheSidebar.vue`
- Create `src/components/navigation/TheTopbar.vue`
- Create `src/layouts/AdminLayout.vue`
- Create `src/components/forms/VehicleForm.vue`
- Create `src/components/forms/LocationForm.vue`
- Create `src/components/forms/EdgeNodeForm.vue`
- Create `src/components/forms/CameraForm.vue`
- Create `src/components/forms/CameraPairForm.vue`
- Create `src/pages/LoginPage.vue`
- Create `src/pages/DashboardPage.vue`
- Create `src/pages/UsersPage.vue`
- Create `src/pages/RolesPage.vue`
- Create `src/pages/PermissionsPage.vue`
- Create `src/pages/VehiclesPage.vue`
- Create `src/pages/LocationsPage.vue`
- Create `src/pages/EdgeNodesPage.vue`
- Create `src/pages/CamerasPage.vue`
- Create `src/pages/CameraPairsPage.vue`
- Create `src/pages/ForbiddenPage.vue`
- Create `src/pages/NotFoundPage.vue`
- Create tests under `src/**/*.test.ts`
- Create `tests/e2e/login.spec.ts`
- Modify `README.md`

---

## Task 1: Vue Vite Foundation And Tooling

**Files:**

- Create: `RIALMA-TrackVision-Frontend/package.json`
- Create: `RIALMA-TrackVision-Frontend/index.html`
- Create: `RIALMA-TrackVision-Frontend/vite.config.ts`
- Create: `RIALMA-TrackVision-Frontend/tsconfig.json`
- Create: `RIALMA-TrackVision-Frontend/tsconfig.node.json`
- Create: `RIALMA-TrackVision-Frontend/eslint.config.js`
- Create: `RIALMA-TrackVision-Frontend/playwright.config.ts`
- Create: `RIALMA-TrackVision-Frontend/.env.example`
- Create: `RIALMA-TrackVision-Frontend/src/main.ts`
- Create: `RIALMA-TrackVision-Frontend/src/App.vue`
- Create: `RIALMA-TrackVision-Frontend/src/app/config.ts`
- Create: `RIALMA-TrackVision-Frontend/src/styles/main.css`
- Create: `RIALMA-TrackVision-Frontend/src/App.test.ts`
- Modify: `RIALMA-TrackVision-Frontend/README.md`

**Interfaces:**

- Produces npm scripts: `dev`, `build`, `preview`, `test`, `lint`, `type-check`, `e2e`.
- Produces `getAppConfig(): { apiBaseUrl: string }`.
- Produces root app that renders `RIALMA TrackVision`.

- [ ] **Step 1: Create implementation branch**

Run:

```bash
git switch -c codex/frontend-admin-shell
```

Expected: branch `codex/frontend-admin-shell`.

- [ ] **Step 2: Add package manifest**

Create `package.json` with:

```json
{
  "name": "rialma-trackvision-frontend",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "lint": "eslint .",
    "type-check": "vue-tsc --noEmit",
    "e2e": "playwright test"
  },
  "dependencies": {
    "lucide-vue-next": "latest",
    "pinia": "latest",
    "vue": "latest",
    "vue-router": "latest"
  },
  "devDependencies": {
    "@playwright/test": "latest",
    "@tsconfig/node24": "latest",
    "@types/node": "latest",
    "@vitejs/plugin-vue": "latest",
    "@vue/eslint-config-typescript": "latest",
    "@vue/tsconfig": "latest",
    "@vue/test-utils": "latest",
    "eslint": "latest",
    "eslint-plugin-vue": "latest",
    "jsdom": "latest",
    "typescript": "latest",
    "typescript-eslint": "latest",
    "vitest": "latest",
    "vue-tsc": "latest"
  }
}
```

- [ ] **Step 3: Install dependencies**

Run:

```bash
npm install
```

Expected: `package-lock.json` created.

- [ ] **Step 4: Add Vite and TypeScript config**

Create `index.html`:

```html
<!doctype html>
<html lang="pt-BR">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>RIALMA TrackVision</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

Create `vite.config.ts`:

```ts
import { fileURLToPath, URL } from 'node:url'
import vue from '@vitejs/plugin-vue'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: [],
  },
})
```

Create `tsconfig.json`:

```json
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "include": ["env.d.ts", "src/**/*", "src/**/*.vue", "tests/**/*"],
  "exclude": ["src/**/__tests__/*"],
  "compilerOptions": {
    "composite": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

Create `tsconfig.node.json`:

```json
{
  "extends": "@tsconfig/node24/tsconfig.json",
  "include": ["vite.config.*", "vitest.config.*", "playwright.config.*", "eslint.config.*"],
  "compilerOptions": {
    "composite": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "types": ["node"]
  }
}
```

Create `env.d.ts`:

```ts
/// <reference types="vite/client" />
```

- [ ] **Step 5: Add lint and Playwright config**

Create `eslint.config.js`:

```js
import pluginVue from 'eslint-plugin-vue'
import tseslint from 'typescript-eslint'

export default [
  {
    ignores: ['dist', 'coverage', 'node_modules'],
  },
  ...pluginVue.configs['flat/recommended'],
  ...tseslint.configs.recommended,
  {
    files: ['**/*.vue', '**/*.ts'],
    rules: {
      'vue/multi-word-component-names': 'error',
    },
  },
]
```

Create `playwright.config.ts`:

```ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 30_000,
  use: {
    baseURL: 'http://127.0.0.1:5173',
    trace: 'on-first-retry',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
  webServer: {
    command: 'npm run dev -- --host 127.0.0.1',
    url: 'http://127.0.0.1:5173',
    reuseExistingServer: true,
  },
})
```

- [ ] **Step 6: Add env example and app config**

Create `.env.example`:

```env
VITE_API_BASE_URL=http://localhost:8000/api/v1
```

Create `src/app/config.ts`:

```ts
export interface AppConfig {
  apiBaseUrl: string
}

export function getAppConfig(): AppConfig {
  return {
    apiBaseUrl: import.meta.env.VITE_API_BASE_URL ?? 'http://localhost:8000/api/v1',
  }
}
```

- [ ] **Step 7: Add root Vue files and smoke test**

Create `src/App.vue`:

```vue
<script setup lang="ts">
</script>

<template>
  <RouterView />
</template>
```

Create `src/main.ts`:

```ts
import { createPinia } from 'pinia'
import { createApp } from 'vue'
import App from './App.vue'
import { createAppRouter } from './router'
import './styles/main.css'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia)
app.use(createAppRouter())
app.mount('#app')
```

Create `src/styles/main.css`:

```css
:root {
  color: #1f2933;
  background: #f5f7fa;
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  --color-surface: #ffffff;
  --color-border: #d9e2ec;
  --color-primary: #1f6feb;
  --color-success: #25855a;
  --color-warning: #b7791f;
  --color-danger: #c2410c;
  --color-muted: #66788a;
}

* {
  box-sizing: border-box;
}

body {
  min-width: 320px;
  margin: 0;
}

button,
input,
select,
textarea {
  font: inherit;
}

#app {
  min-height: 100vh;
}

.page {
  display: flex;
  min-height: 100vh;
  flex-direction: column;
}

.login-shell {
  display: grid;
  min-height: 100vh;
  place-items: center;
  padding: 24px;
}
```

Create `src/App.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { createPinia } from 'pinia'
import { describe, expect, it } from 'vitest'
import App from './App.vue'
import { createAppRouter } from './router'

describe('App', () => {
  it('mounts the router shell', async () => {
    const router = createAppRouter()
    router.push('/login')
    await router.isReady()

    const wrapper = mount(App, {
      global: {
        plugins: [createPinia(), router],
      },
    })

    expect(wrapper.text()).toContain('RIALMA TrackVision')
  })
})
```

- [ ] **Step 8: Add minimal router and login page for smoke test**

Create `src/router/index.ts`:

```ts
import { createRouter, createWebHistory } from 'vue-router'
import LoginPage from '@/pages/LoginPage.vue'

export function createAppRouter() {
  return createRouter({
    history: createWebHistory(),
    routes: [
      {
        path: '/',
        redirect: '/login',
      },
      {
        path: '/login',
        name: 'login',
        component: LoginPage,
      },
    ],
  })
}
```

Create `src/pages/LoginPage.vue`:

```vue
<template>
  <main class="login-shell">
    <section aria-labelledby="login-title">
      <p>Controle operacional</p>
      <h1 id="login-title">RIALMA TrackVision</h1>
      <p>Autenticacao administrativa</p>
    </section>
  </main>
</template>
```

- [ ] **Step 9: Run foundation verification**

Run:

```bash
npm run test
npm run lint
npm run build
```

Expected: all commands pass.

- [ ] **Step 10: Commit task**

Run:

```bash
git add .
git commit -m "chore: scaffold vue admin frontend"
```

---

## Task 2: Typed API Client And Auth Services

**Files:**

- Create: `src/types/api.ts`
- Create: `src/types/auth.ts`
- Create: `src/types/admin.ts`
- Create: `src/services/apiClient.ts`
- Create: `src/services/authService.ts`
- Create: `src/services/permissionProbeService.ts`
- Test: `src/services/authService.test.ts`
- Test: `src/services/apiClient.test.ts`

**Interfaces:**

- Produces `ApiError` class with `status`, `message`, `errors`, `isUnauthorized`, `isForbidden`.
- Produces `apiClient.get<T>()`, `post<T>()`, `patch<T>()`, `delete<T>()`, and `head()`.
- Produces `authService.login(credentials): Promise<LoginResponse>`.
- Produces `authService.me(): Promise<User>`.
- Produces `permissionProbeService.probeEffectivePermissions(): Promise<string[]>`.

- [ ] **Step 1: Write failing API client tests**

Create `src/services/apiClient.test.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { ApiError, createApiClient } from './apiClient'

describe('apiClient', () => {
  beforeEach(() => {
    vi.unstubAllGlobals()
  })

  it('normalizes validation errors from Laravel responses', async () => {
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
      ok: false,
      status: 422,
      json: async () => ({
        message: 'The plate field is required.',
        errors: { plate: ['The plate field is required.'] },
      }),
    }))

    const client = createApiClient({ apiBaseUrl: 'http://api.test', getToken: () => 'token' })

    await expect(client.post('/admin/vehicles', {})).rejects.toMatchObject({
      status: 422,
      isUnauthorized: false,
      isForbidden: false,
      errors: { plate: ['The plate field is required.'] },
    })
  })

  it('uses bearer token and handles head responses without JSON parsing', async () => {
    const fetchMock = vi.fn().mockResolvedValue({ ok: true, status: 204 })
    vi.stubGlobal('fetch', fetchMock)

    const client = createApiClient({ apiBaseUrl: 'http://api.test', getToken: () => 'abc' })

    await client.head('/admin/vehicles')

    const [, init] = fetchMock.mock.calls[0] as [string, RequestInit]
    const headers = init.headers as Headers

    expect(init.method).toBe('HEAD')
    expect(headers.get('Authorization')).toBe('Bearer abc')
  })
})
```

- [ ] **Step 2: Write failing auth service tests**

Create `src/services/authService.test.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { createAuthService } from './authService'

describe('authService', () => {
  beforeEach(() => {
    vi.unstubAllGlobals()
  })

  it('posts credentials and returns normalized login response', async () => {
    const fetchMock = vi.fn().mockResolvedValue({
      ok: true,
      status: 200,
      headers: new Headers({ 'content-type': 'application/json' }),
      json: async () => ({
        token_type: 'Bearer',
        access_token: 'token-123',
        expires_at: '2026-08-12T10:00:00.000000Z',
        user: { id: 1, name: 'Paulo', email: 'paulo@example.com' },
      }),
    })
    vi.stubGlobal('fetch', fetchMock)

    const service = createAuthService({ apiBaseUrl: 'http://api.test', getToken: () => null })

    const result = await service.login({ email: 'paulo@example.com', password: 'secret' })

    expect(result.accessToken).toBe('token-123')
    expect(result.user.email).toBe('paulo@example.com')
    expect(fetchMock).toHaveBeenCalledWith('http://api.test/auth/login', expect.objectContaining({
      method: 'POST',
    }))
  })
})
```

- [ ] **Step 3: Run tests to verify failure**

Run:

```bash
npm run test -- src/services/apiClient.test.ts src/services/authService.test.ts
```

Expected: FAIL because services and types do not exist.

- [ ] **Step 4: Implement API and auth types**

Create `src/types/api.ts`:

```ts
export interface LaravelResource<T> {
  data: T
}

export interface LaravelPaginated<T> {
  data: T[]
  links?: Record<string, string | null>
  meta?: Record<string, unknown>
}

export type FieldErrors = Record<string, string[]>
```

Create `src/types/auth.ts`:

```ts
export interface User {
  id: number
  name: string
  email: string
  roles?: string[]
  permissions?: string[]
}

export interface LoginCredentials {
  email: string
  password: string
}

export interface LoginResponse {
  tokenType: 'Bearer'
  accessToken: string
  expiresAt: string
  user: User
}
```

Create `src/types/admin.ts`:

```ts
export interface Permission {
  id: number
  name: string
}

export interface Role {
  id: number
  name: string
  permissions?: Permission[]
}

export interface Vehicle {
  id: number
  uuid: string
  plate: string
  plate_normalized: string
  fleet_code: string | null
  description: string | null
  is_active: boolean
}

export type VehicleInput = Pick<Vehicle, 'plate' | 'fleet_code' | 'description' | 'is_active'>

export interface Location {
  id: number
  uuid: string
  name: string
  description: string | null
  is_active: boolean
}

export type LocationInput = Omit<Location, 'id'>

export interface EdgeNode {
  id: number
  uuid: string
  name: string
  description: string | null
  status: 'online' | 'offline' | 'degraded'
  last_seen_at: string | null
  is_active: boolean
  location?: Location
}

export interface EdgeNodeInput {
  location_id: number
  name: string
  description: string | null
  is_active: boolean
}

export interface Camera {
  id: number
  uuid: string
  name: string
  type: 'lpr' | 'support'
  vendor: 'intelbras'
  host: string
  port: number
  channel: number | null
  username: string | null
  is_active: boolean
  location?: Location
  edge_node?: EdgeNode
}

export interface CameraInput {
  location_id: number
  edge_node_id: number
  name: string
  type: 'lpr' | 'support'
  vendor: 'intelbras'
  host: string
  port: number
  channel: number | null
  username: string | null
  password?: string
  is_active: boolean
}

export interface CameraPair {
  id: number
  uuid: string
  name: string
  direction: 'outbound' | 'inbound' | 'unknown'
  is_active: boolean
  location?: Location
  edge_node?: EdgeNode
  lpr_camera?: Camera
  support_camera?: Camera
}

export interface CameraPairInput {
  location_id: number
  edge_node_id: number
  name: string
  lpr_camera_id: number
  support_camera_id: number
  direction: 'outbound' | 'inbound' | 'unknown'
  is_active: boolean
}
```

- [ ] **Step 5: Implement API client**

Create `src/services/apiClient.ts`:

```ts
import type { FieldErrors } from '@/types/api'

export class ApiError extends Error {
  readonly isUnauthorized: boolean
  readonly isForbidden: boolean

  constructor(
    readonly status: number,
    message: string,
    readonly errors: FieldErrors = {},
  ) {
    super(message)
    this.name = 'ApiError'
    this.isUnauthorized = status === 401
    this.isForbidden = status === 403
  }
}

export interface ApiClientOptions {
  apiBaseUrl: string
  getToken: () => string | null
  onUnauthorized?: () => void
}

export function createApiClient(options: ApiClientOptions) {
  async function request<T>(path: string, init: RequestInit = {}): Promise<T> {
    const headers = new Headers(init.headers)
    headers.set('Accept', 'application/json')

    if (init.body && ! headers.has('Content-Type')) {
      headers.set('Content-Type', 'application/json')
    }

    const token = options.getToken()
    if (token) {
      headers.set('Authorization', `Bearer ${token}`)
    }

    const response = await fetch(`${options.apiBaseUrl}${path}`, { ...init, headers })

    if (! response.ok) {
      let payload: { message?: string; errors?: FieldErrors } = {}
      try {
        payload = await response.json()
      } catch {
        payload = {}
      }

      if (response.status === 401) {
        options.onUnauthorized?.()
      }

      throw new ApiError(response.status, payload.message ?? 'Erro ao comunicar com a API.', payload.errors ?? {})
    }

    if (init.method === 'HEAD' || response.status === 204) {
      return undefined as T
    }

    return response.json() as Promise<T>
  }

  return {
    get: <T>(path: string) => request<T>(path),
    post: <T>(path: string, body?: unknown) => request<T>(path, { method: 'POST', body: JSON.stringify(body ?? {}) }),
    patch: <T>(path: string, body?: unknown) => request<T>(path, { method: 'PATCH', body: JSON.stringify(body ?? {}) }),
    delete: <T>(path: string) => request<T>(path, { method: 'DELETE' }),
    head: (path: string) => request<void>(path, { method: 'HEAD' }),
  }
}
```

- [ ] **Step 6: Implement auth and permission probe services**

Create `src/services/authService.ts`:

```ts
import { getAppConfig } from '@/app/config'
import type { LoginCredentials, LoginResponse, User } from '@/types/auth'
import type { LaravelResource } from '@/types/api'
import { createApiClient, type ApiClientOptions } from './apiClient'

interface LoginPayload {
  token_type: 'Bearer'
  access_token: string
  expires_at: string
  user: User
}

function mapLogin(payload: LoginPayload): LoginResponse {
  return {
    tokenType: payload.token_type,
    accessToken: payload.access_token,
    expiresAt: payload.expires_at,
    user: payload.user,
  }
}

export function createAuthService(options: ApiClientOptions) {
  const client = createApiClient(options)

  return {
    async login(credentials: LoginCredentials): Promise<LoginResponse> {
      const payload = await client.post<LoginPayload>('/auth/login', credentials)
      return mapLogin(payload)
    },

    logout(): Promise<void> {
      return client.post<void>('/auth/logout')
    },

    async me(): Promise<User> {
      const response = await client.get<LaravelResource<User>>('/me')
      return response.data
    },
  }
}

export const authService = createAuthService({
  apiBaseUrl: getAppConfig().apiBaseUrl,
  getToken: () => localStorage.getItem('trackvision.token'),
})
```

Create `src/services/permissionProbeService.ts`:

```ts
import { getAppConfig } from '@/app/config'
import { ApiError, createApiClient } from './apiClient'

const probes = [
  { path: '/admin/users', permission: 'users.manage' },
  { path: '/admin/roles', permission: 'permissions.manage' },
  { path: '/admin/vehicles', permission: 'vehicles.manage' },
  { path: '/admin/locations', permission: 'cameras.manage' },
] as const

export const permissionProbeService = {
  async probeEffectivePermissions(token: string): Promise<string[]> {
    const client = createApiClient({
      apiBaseUrl: getAppConfig().apiBaseUrl,
      getToken: () => token,
    })

    const allowed: string[] = []

    for (const probe of probes) {
      try {
        await client.head(probe.path)
        allowed.push(probe.permission)
      } catch (error) {
        if (error instanceof ApiError && error.isForbidden) {
          continue
        }

        throw error
      }
    }

    return [...new Set(allowed)]
  },
}
```

`permissionProbeService.probeEffectivePermissions` uses `HEAD` requests:

- `/admin/users` maps to `users.manage`
- `/admin/roles` maps to `permissions.manage`
- `/admin/vehicles` maps to `vehicles.manage`
- `/admin/locations` maps to `cameras.manage`

The code catches `ApiError` with `isForbidden` and treats it as a missing permission. It re-throws `401` and network errors.

- [ ] **Step 7: Run service tests**

Run:

```bash
npm run test -- src/services/apiClient.test.ts src/services/authService.test.ts
npm run lint
npm run build
```

Expected: all commands pass.

- [ ] **Step 8: Commit task**

Run:

```bash
git add src/types src/services
git commit -m "feat: add typed api and auth services"
```

---

## Task 3: Auth Store And Router Guards

**Files:**

- Create: `src/stores/authStore.ts`
- Create: `src/router/index.ts`
- Modify: `src/main.ts`
- Modify: `src/pages/LoginPage.vue`
- Create: `src/stores/authStore.test.ts`
- Create: `src/router/routerGuard.test.ts`

**Interfaces:**

- Produces `useAuthStore()`.
- Produces auth state: `token`, `user`, `permissions`, `isAuthenticated`, `can(permission)`.
- Produces actions: `login(credentials)`, `logout()`, `restoreSession()`, `refreshUser()`.
- Produces routes with `meta.requiresAuth` and `meta.permission`.

- [ ] **Step 1: Write failing auth store test**

Create `src/stores/authStore.test.ts`:

```ts
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { useAuthStore } from './authStore'

vi.mock('@/services/authService', () => ({
  authService: {
    login: vi.fn().mockResolvedValue({
      tokenType: 'Bearer',
      accessToken: 'token-123',
      expiresAt: '2026-08-12T10:00:00.000000Z',
      user: { id: 1, name: 'Paulo', email: 'paulo@example.com' },
    }),
    me: vi.fn().mockResolvedValue({ id: 1, name: 'Paulo', email: 'paulo@example.com' }),
    logout: vi.fn().mockResolvedValue(undefined),
  },
}))

vi.mock('@/services/permissionProbeService', () => ({
  permissionProbeService: {
    probeEffectivePermissions: vi.fn().mockResolvedValue(['vehicles.manage']),
  },
}))

describe('authStore', () => {
  beforeEach(() => {
    localStorage.clear()
    setActivePinia(createPinia())
  })

  it('stores token, user and probed permissions after login', async () => {
    const store = useAuthStore()

    await store.login({ email: 'paulo@example.com', password: 'secret' })

    expect(store.isAuthenticated).toBe(true)
    expect(store.user?.email).toBe('paulo@example.com')
    expect(store.can('vehicles.manage')).toBe(true)
    expect(localStorage.getItem('trackvision.token')).toBe('token-123')
  })
})
```

- [ ] **Step 2: Write failing router guard test**

Create `src/router/routerGuard.test.ts`:

```ts
import { createPinia, setActivePinia } from 'pinia'
import { describe, expect, it } from 'vitest'
import { createAppRouter } from './index'
import { useAuthStore } from '@/stores/authStore'

describe('router guards', () => {
  it('redirects unauthenticated users to login', async () => {
    setActivePinia(createPinia())
    const router = createAppRouter()

    await router.push('/vehicles')

    expect(router.currentRoute.value.name).toBe('login')
  })

  it('blocks a route when permission is missing', async () => {
    setActivePinia(createPinia())
    const store = useAuthStore()
    store.token = 'token-123'
    store.user = { id: 1, name: 'Paulo', email: 'paulo@example.com' }
    store.permissions = []

    const router = createAppRouter()
    await router.push('/vehicles')

    expect(router.currentRoute.value.name).toBe('forbidden')
  })
})
```

- [ ] **Step 3: Run tests to verify failure**

Run:

```bash
npm run test -- src/stores/authStore.test.ts src/router/routerGuard.test.ts
```

Expected: FAIL because store and guard behavior do not exist.

- [ ] **Step 4: Implement auth store**

Use Pinia setup store or option store. Persist token under keys:

- `trackvision.token`
- `trackvision.user`
- `trackvision.permissions`

`can(permission)` returns true when `permission` is in effective permissions.

`logout()` calls backend logout when authenticated, clears local state, and removes localStorage keys.

- [ ] **Step 5: Implement router**

Routes:

- `/login` name `login`
- `/` redirects to `/dashboard`
- `/dashboard` permission-free but authenticated
- `/users` permission `users.manage`
- `/roles` permission `permissions.manage`
- `/permissions` permission `permissions.manage`
- `/vehicles` permission `vehicles.manage`
- `/locations` permission `cameras.manage`
- `/edge-nodes` permission `cameras.manage`
- `/cameras` permission `cameras.manage`
- `/camera-pairs` permission `cameras.manage`
- `/forbidden` name `forbidden`
- catch-all name `not-found`

Guard:

- unauthenticated protected route redirects to `login`;
- authenticated login route redirects to `dashboard`;
- missing route permission redirects to `forbidden`.

- [ ] **Step 6: Implement login page**

`LoginPage.vue` uses `authStore.login`, displays field errors from `ApiError`, loading state, and redirects to dashboard on success.

- [ ] **Step 7: Run auth and router tests**

Run:

```bash
npm run test -- src/stores/authStore.test.ts src/router/routerGuard.test.ts
npm run lint
npm run build
```

Expected: all commands pass.

- [ ] **Step 8: Commit task**

Run:

```bash
git add src/stores src/router src/pages/LoginPage.vue src/main.ts
git commit -m "feat: add auth store and route guards"
```

---

## Task 4: Base UI, Admin Layout, And Navigation

**Files:**

- Create: `src/components/base/BaseAlert.vue`
- Create: `src/components/base/BaseButton.vue`
- Create: `src/components/base/BaseInput.vue`
- Create: `src/components/base/BaseModal.vue`
- Create: `src/components/base/BaseSelect.vue`
- Create: `src/components/base/BaseTable.vue`
- Create: `src/components/navigation/TheSidebar.vue`
- Create: `src/components/navigation/TheTopbar.vue`
- Create: `src/layouts/AdminLayout.vue`
- Create: `src/components/navigation/TheSidebar.test.ts`
- Modify: `src/styles/main.css`
- Modify: `src/App.vue`

**Interfaces:**

- Produces permission-aware navigation items.
- Produces reusable base components for forms, tables, alerts, modals, and buttons.
- Produces layout slots for admin pages.

- [ ] **Step 1: Write failing sidebar test**

Create `src/components/navigation/TheSidebar.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { createPinia, setActivePinia } from 'pinia'
import { describe, expect, it } from 'vitest'
import { useAuthStore } from '@/stores/authStore'
import TheSidebar from './TheSidebar.vue'

describe('TheSidebar', () => {
  it('renders only links allowed by effective permissions', () => {
    setActivePinia(createPinia())
    const auth = useAuthStore()
    auth.permissions = ['vehicles.manage']

    const wrapper = mount(TheSidebar, {
      global: {
        stubs: ['RouterLink'],
      },
    })

    expect(wrapper.text()).toContain('Veiculos')
    expect(wrapper.text()).not.toContain('Cameras')
    expect(wrapper.text()).not.toContain('Usuarios')
  })
})
```

- [ ] **Step 2: Run test to verify failure**

Run:

```bash
npm run test -- src/components/navigation/TheSidebar.test.ts
```

Expected: FAIL because navigation components do not exist.

- [ ] **Step 3: Implement base components**

Implement:

- `BaseButton` with props `variant`, `type`, `disabled`, `loading`.
- `BaseInput` with props `modelValue`, `label`, `type`, `error`, emits `update:modelValue`.
- `BaseSelect` with props `modelValue`, `label`, `options`, `error`, emits `update:modelValue`.
- `BaseAlert` with props `variant`, slot content.
- `BaseModal` with props `open`, `title`, emits `close`.
- `BaseTable` with props `columns`, `rows`, `loading`, `emptyText`; exposes row slot.

Use shorthands `:`, `@`, `#` and stable `:key` values.

- [ ] **Step 4: Implement navigation and layout**

`TheSidebar` uses route metadata and auth store permissions. Labels:

- Dashboard
- Usuarios
- Roles
- Permissoes
- Veiculos
- Locais
- Edge Nodes
- Cameras
- Pares de Cameras

`TheTopbar` displays user name/email and logout action.

`AdminLayout` composes sidebar, topbar and main slot.

`App.vue` wraps protected pages through routed layout, not conditional global cards.

- [ ] **Step 5: Run layout verification**

Run:

```bash
npm run test -- src/components/navigation/TheSidebar.test.ts
npm run lint
npm run build
```

Expected: all commands pass.

- [ ] **Step 6: Commit task**

Run:

```bash
git add src/components src/layouts src/styles src/App.vue
git commit -m "feat: add admin layout and base components"
```

---

## Task 5: Read-Only Security Lists

**Files:**

- Create: `src/services/usersService.ts`
- Create: `src/services/rolesService.ts`
- Create: `src/services/permissionsService.ts`
- Create: `src/pages/UsersPage.vue`
- Create: `src/pages/RolesPage.vue`
- Create: `src/pages/PermissionsPage.vue`
- Create: `src/pages/ForbiddenPage.vue`
- Create: `src/pages/NotFoundPage.vue`
- Create: `src/pages/DashboardPage.vue`
- Create: `src/pages/UsersPage.test.ts`

**Interfaces:**

- Consumes `GET /admin/users`, `GET /admin/roles`, `GET /admin/permissions`.
- Produces read-only tables with loading, error, empty, and success states.

- [ ] **Step 1: Write failing users page test**

Create `src/pages/UsersPage.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it, vi } from 'vitest'
import UsersPage from './UsersPage.vue'

vi.mock('@/services/usersService', () => ({
  usersService: {
    list: vi.fn().mockResolvedValue({
      data: [{ id: 1, name: 'Paulo', email: 'paulo@example.com', roles: ['super_admin'] }],
    }),
  },
}))

describe('UsersPage', () => {
  it('renders users returned by the API', async () => {
    const wrapper = mount(UsersPage)
    await new Promise((resolve) => setTimeout(resolve, 0))

    expect(wrapper.text()).toContain('Paulo')
    expect(wrapper.text()).toContain('super_admin')
  })
})
```

- [ ] **Step 2: Run test to verify failure**

Run:

```bash
npm run test -- src/pages/UsersPage.test.ts
```

Expected: FAIL because page/service do not exist.

- [ ] **Step 3: Implement security list services**

Each service calls `apiClient.get<LaravelPaginated<T>>()`:

- `usersService.list()`
- `rolesService.list()`
- `permissionsService.list()`

- [ ] **Step 4: Implement read-only pages**

Each page:

- has title;
- fetches on mount;
- shows loading;
- shows API error;
- shows empty state;
- renders table on success.

Dashboard page shows shortcuts for permitted modules and static operational text. Forbidden and NotFound pages provide concise recovery links.

- [ ] **Step 5: Run tests**

Run:

```bash
npm run test -- src/pages/UsersPage.test.ts
npm run lint
npm run build
```

Expected: all commands pass.

- [ ] **Step 6: Commit task**

Run:

```bash
git add src/services/usersService.ts src/services/rolesService.ts src/services/permissionsService.ts src/pages
git commit -m "feat: add read only security admin pages"
```

---

## Task 6: Vehicle CRUD

**Files:**

- Create: `src/services/vehiclesService.ts`
- Create: `src/components/forms/VehicleForm.vue`
- Create: `src/pages/VehiclesPage.vue`
- Create: `src/pages/VehiclesPage.test.ts`

**Interfaces:**

- Consumes `/admin/vehicles` REST endpoints.
- Produces `vehiclesService.list/create/update/remove`.
- Produces CRUD UI for `plate`, `fleet_code`, `description`, `is_active`.

- [ ] **Step 1: Write failing vehicles page test**

Create `src/pages/VehiclesPage.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it, vi } from 'vitest'
import VehiclesPage from './VehiclesPage.vue'

vi.mock('@/services/vehiclesService', () => ({
  vehiclesService: {
    list: vi.fn().mockResolvedValue({
      data: [{
        id: 1,
        uuid: 'uuid-1',
        plate: 'ABC-1D23',
        plate_normalized: 'ABC1D23',
        fleet_code: 'TRUCK-01',
        description: 'Caminhao',
        is_active: true,
      }],
    }),
    create: vi.fn(),
    update: vi.fn(),
    remove: vi.fn(),
  },
}))

describe('VehiclesPage', () => {
  it('renders paginated vehicle data', async () => {
    const wrapper = mount(VehiclesPage)
    await new Promise((resolve) => setTimeout(resolve, 0))

    expect(wrapper.text()).toContain('ABC-1D23')
    expect(wrapper.text()).toContain('ABC1D23')
    expect(wrapper.text()).toContain('TRUCK-01')
  })
})
```

- [ ] **Step 2: Run test to verify failure**

Run:

```bash
npm run test -- src/pages/VehiclesPage.test.ts
```

Expected: FAIL because page/service do not exist.

- [ ] **Step 3: Implement vehicle service**

Methods:

- `list(): Promise<LaravelPaginated<Vehicle>>`
- `create(input: VehicleInput): Promise<Vehicle>`
- `update(vehicle: Vehicle, input: VehicleInput): Promise<Vehicle>`
- `remove(vehicle: Vehicle): Promise<void>`

Map Laravel item resources by reading `response.data`.

- [ ] **Step 4: Implement VehicleForm**

Props:

- `modelValue: VehicleInput`
- `errors: FieldErrors`
- `submitting: boolean`

Emits:

- `update:modelValue`
- `submit`
- `cancel`

Fields:

- plate;
- fleet_code;
- description;
- is_active checkbox.

- [ ] **Step 5: Implement VehiclesPage**

Page provides:

- table;
- create button;
- edit action;
- delete action with confirm;
- modal form;
- validation errors from `ApiError.errors`;
- success alert after create/update/delete.

- [ ] **Step 6: Run vehicle verification**

Run:

```bash
npm run test -- src/pages/VehiclesPage.test.ts
npm run lint
npm run build
```

Expected: all commands pass.

- [ ] **Step 7: Commit task**

Run:

```bash
git add src/services/vehiclesService.ts src/components/forms/VehicleForm.vue src/pages/VehiclesPage.vue src/pages/VehiclesPage.test.ts
git commit -m "feat: add vehicle management UI"
```

---

## Task 7: Locations And Edge Nodes CRUD

**Files:**

- Create: `src/services/locationsService.ts`
- Create: `src/services/edgeNodesService.ts`
- Create: `src/components/forms/LocationForm.vue`
- Create: `src/components/forms/EdgeNodeForm.vue`
- Create: `src/pages/LocationsPage.vue`
- Create: `src/pages/EdgeNodesPage.vue`
- Create: `src/pages/EdgeNodesPage.test.ts`

**Interfaces:**

- Consumes `/admin/locations` and `/admin/edge-nodes`.
- Produces CRUD UI for locations and edge nodes.

- [ ] **Step 1: Write failing edge nodes page test**

Create `src/pages/EdgeNodesPage.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it, vi } from 'vitest'
import EdgeNodesPage from './EdgeNodesPage.vue'

vi.mock('@/services/edgeNodesService', () => ({
  edgeNodesService: {
    list: vi.fn().mockResolvedValue({
      data: [{
        id: 1,
        uuid: 'edge-1',
        name: 'Edge Portaria 01',
        description: 'Servidor local',
        status: 'offline',
        last_seen_at: null,
        is_active: true,
        location: { id: 1, uuid: 'loc-1', name: 'Portaria', description: null, is_active: true },
      }],
    }),
    create: vi.fn(),
    update: vi.fn(),
    remove: vi.fn(),
  },
}))

vi.mock('@/services/locationsService', () => ({
  locationsService: {
    list: vi.fn().mockResolvedValue({ data: [{ id: 1, uuid: 'loc-1', name: 'Portaria', description: null, is_active: true }] }),
  },
}))

describe('EdgeNodesPage', () => {
  it('renders edge node status and location', async () => {
    const wrapper = mount(EdgeNodesPage)
    await new Promise((resolve) => setTimeout(resolve, 0))

    expect(wrapper.text()).toContain('Edge Portaria 01')
    expect(wrapper.text()).toContain('offline')
    expect(wrapper.text()).toContain('Portaria')
  })
})
```

- [ ] **Step 2: Run test to verify failure**

Run:

```bash
npm run test -- src/pages/EdgeNodesPage.test.ts
```

Expected: FAIL because page/service do not exist.

- [ ] **Step 3: Implement location service, form and page**

Location fields:

- `name`
- `description`
- `is_active`

Page behavior matches VehiclesPage pattern with domain-specific columns.

- [ ] **Step 4: Implement edge node service, form and page**

Edge node fields:

- `location_id`
- `name`
- `description`
- `is_active`

Read-only columns:

- `status`
- `last_seen_at`

The form loads active locations from `locationsService.list()`.

- [ ] **Step 5: Run location and edge-node verification**

Run:

```bash
npm run test -- src/pages/EdgeNodesPage.test.ts
npm run lint
npm run build
```

Expected: all commands pass.

- [ ] **Step 6: Commit task**

Run:

```bash
git add src/services/locationsService.ts src/services/edgeNodesService.ts src/components/forms/LocationForm.vue src/components/forms/EdgeNodeForm.vue src/pages/LocationsPage.vue src/pages/EdgeNodesPage.vue src/pages/EdgeNodesPage.test.ts
git commit -m "feat: add location and edge node management UI"
```

---

## Task 8: Cameras And Camera Pairs CRUD

**Files:**

- Create: `src/services/camerasService.ts`
- Create: `src/services/cameraPairsService.ts`
- Create: `src/components/forms/CameraForm.vue`
- Create: `src/components/forms/CameraPairForm.vue`
- Create: `src/pages/CamerasPage.vue`
- Create: `src/pages/CameraPairsPage.vue`
- Create: `src/components/forms/CameraForm.test.ts`
- Create: `src/pages/CameraPairsPage.test.ts`

**Interfaces:**

- Consumes `/admin/cameras` and `/admin/camera-pairs`.
- Produces camera CRUD with write-only password field.
- Produces camera-pair CRUD with filtered LPR/support camera selects.

- [ ] **Step 1: Write failing CameraForm test**

Create `src/components/forms/CameraForm.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import CameraForm from './CameraForm.vue'

describe('CameraForm', () => {
  it('does not display existing camera password and emits password only when typed', async () => {
    const wrapper = mount(CameraForm, {
      props: {
        modelValue: {
          location_id: 1,
          edge_node_id: 1,
          name: 'LPR Entrada',
          type: 'lpr',
          vendor: 'intelbras',
          host: '192.168.1.10',
          port: 80,
          channel: 1,
          username: 'admin',
          password: '',
          is_active: true,
        },
        locations: [{ id: 1, uuid: 'loc-1', name: 'Portaria', description: null, is_active: true }],
        edgeNodes: [{ id: 1, uuid: 'edge-1', name: 'Edge 01', description: null, status: 'offline', last_seen_at: null, is_active: true, location: { id: 1, uuid: 'loc-1', name: 'Portaria', description: null, is_active: true } }],
        errors: {},
        submitting: false,
      },
    })

    expect(wrapper.text()).not.toContain('camera-secret')

    await wrapper.find('input[name="password"]').setValue('new-secret')

    const emitted = wrapper.emitted('update:modelValue')?.at(-1)?.[0] as Record<string, unknown>
    expect(emitted.password).toBe('new-secret')
  })
})
```

- [ ] **Step 2: Write failing camera pairs page test**

Create `src/pages/CameraPairsPage.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it, vi } from 'vitest'
import CameraPairsPage from './CameraPairsPage.vue'

vi.mock('@/services/cameraPairsService', () => ({
  cameraPairsService: {
    list: vi.fn().mockResolvedValue({
      data: [{
        id: 1,
        uuid: 'pair-1',
        name: 'Entrada Principal',
        direction: 'outbound',
        is_active: true,
        lpr_camera: { id: 1, uuid: 'cam-1', name: 'LPR Entrada', type: 'lpr', vendor: 'intelbras', host: '192.168.1.10', port: 80, channel: 1, username: 'admin', is_active: true },
        support_camera: { id: 2, uuid: 'cam-2', name: 'Apoio Entrada', type: 'support', vendor: 'intelbras', host: '192.168.1.11', port: 80, channel: 1, username: 'admin', is_active: true },
      }],
    }),
    create: vi.fn(),
    update: vi.fn(),
    remove: vi.fn(),
  },
}))

vi.mock('@/services/locationsService', () => ({ locationsService: { list: vi.fn().mockResolvedValue({ data: [] }) } }))
vi.mock('@/services/edgeNodesService', () => ({ edgeNodesService: { list: vi.fn().mockResolvedValue({ data: [] }) } }))
vi.mock('@/services/camerasService', () => ({ camerasService: { list: vi.fn().mockResolvedValue({ data: [] }) } }))

describe('CameraPairsPage', () => {
  it('renders paired LPR and support cameras', async () => {
    const wrapper = mount(CameraPairsPage)
    await new Promise((resolve) => setTimeout(resolve, 0))

    expect(wrapper.text()).toContain('Entrada Principal')
    expect(wrapper.text()).toContain('LPR Entrada')
    expect(wrapper.text()).toContain('Apoio Entrada')
  })
})
```

- [ ] **Step 3: Run tests to verify failure**

Run:

```bash
npm run test -- src/components/forms/CameraForm.test.ts src/pages/CameraPairsPage.test.ts
```

Expected: FAIL because forms/pages do not exist.

- [ ] **Step 4: Implement cameras service, form and page**

Camera fields:

- `location_id`
- `edge_node_id`
- `name`
- `type`
- `vendor`
- `host`
- `port`
- `channel`
- `username`
- `password`
- `is_active`

When editing an existing camera, password input starts empty. The submit payload omits `password` when the input is empty.

- [ ] **Step 5: Implement camera pairs service, form and page**

Camera pair fields:

- `location_id`
- `edge_node_id`
- `name`
- `lpr_camera_id`
- `support_camera_id`
- `direction`
- `is_active`

Form filters camera options:

- LPR select shows active cameras with `type === 'lpr'`.
- Support select shows active cameras with `type === 'support'`.
- Both selects narrow by selected `location_id` and `edge_node_id`.

- [ ] **Step 6: Run camera verification**

Run:

```bash
npm run test -- src/components/forms/CameraForm.test.ts src/pages/CameraPairsPage.test.ts
npm run lint
npm run build
```

Expected: all commands pass.

- [ ] **Step 7: Commit task**

Run:

```bash
git add src/services/camerasService.ts src/services/cameraPairsService.ts src/components/forms/CameraForm.vue src/components/forms/CameraPairForm.vue src/pages/CamerasPage.vue src/pages/CameraPairsPage.vue src/components/forms/CameraForm.test.ts src/pages/CameraPairsPage.test.ts
git commit -m "feat: add camera management UI"
```

---

## Task 9: E2E Smoke, Documentation, And Final Verification

**Files:**

- Create: `tests/e2e/login.spec.ts`
- Modify: `README.md`
- Modify only files required by failed verification.

**Interfaces:**

- Produces local frontend usage documentation.
- Produces smoke test for login screen rendering.
- Produces clean branch ready for integration.

- [ ] **Step 1: Add Playwright smoke test**

Create `tests/e2e/login.spec.ts`:

```ts
import { expect, test } from '@playwright/test'

test('shows login screen', async ({ page }) => {
  await page.goto('/login')

  await expect(page.getByRole('heading', { name: 'RIALMA TrackVision' })).toBeVisible()
  await expect(page.getByLabel('Email')).toBeVisible()
  await expect(page.getByLabel('Senha')).toBeVisible()
})
```

- [ ] **Step 2: Update README**

Replace the README content with:

````markdown
# RIALMA TrackVision Frontend

Vue 3 administrative frontend for TrackVision.

## Stack

- Vue 3
- Vite
- TypeScript
- Vue Router
- Pinia
- Vitest
- Playwright

## Environment

Copy `.env.example` to `.env.local` and point the frontend to the Laravel API:

```env
VITE_API_BASE_URL=http://localhost:8000/api/v1
```

Do not put secrets in frontend environment variables.

## Local Commands

```bash
npm install
npm run dev
```

## Verification

```bash
npm run lint
npm run test
npm run build
npm run e2e
```

## Current Scope

This phase includes login, authenticated admin layout, permission-aware navigation,
read-only users/roles/permissions, and CRUD screens for vehicles, locations, edge
nodes, cameras, and camera pairs.

Users, roles, and permissions are read-only because the backend currently exposes
only list endpoints for these resources.
````

- [ ] **Step 3: Run full frontend verification**

Run:

```bash
npm run lint
npm run test
npm run build
npm run e2e
```

Expected: all commands pass.

- [ ] **Step 4: Inspect route/UI build**

Run:

```bash
npm run dev -- --host 127.0.0.1
```

Expected: dev server starts at `http://127.0.0.1:5173`. Keep it running only long enough to verify the app can be opened, then stop the server.

- [ ] **Step 5: Inspect Git status**

Run:

```bash
git status --short --branch
```

Expected: branch `codex/frontend-admin-shell` with only intended docs/e2e changes staged or ready.

- [ ] **Step 6: Commit task**

Run:

```bash
git add README.md tests/e2e/login.spec.ts
git commit -m "docs: add frontend admin usage and smoke test"
```

- [ ] **Step 7: Final branch verification**

Run:

```bash
npm run lint
npm run test
npm run build
git status --short --branch
```

Expected: lint, tests, and build pass; git status is clean on `codex/frontend-admin-shell`.

---

## Self-Review Checklist

- Spec coverage: login, logout, `/me`, permission-aware nav, read-only users/roles/permissions, CRUD vehicles, locations, edge nodes, cameras, camera pairs, error states, and tests all map to tasks.
- Placeholder scan: plan contains concrete file names, commands, test examples, route names, field names, and commit messages.
- Type consistency: `Vehicle`, `Location`, `EdgeNode`, `Camera`, `CameraPair`, `User`, `Role`, and `Permission` names match services, pages, forms, and tests.
- Scope control: capture, LPR images, support-camera images, trips, load review, reports, and backend changes remain outside this phase.
