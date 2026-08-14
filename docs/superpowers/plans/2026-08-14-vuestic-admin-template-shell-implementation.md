# Vuestic Admin Template Shell Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

## Nota De Atualizacao

Atualizado em 2026-08-14: a regra vigente do frontend e Vuestic Admin/Vuestic UI
first. Telas, layouts, formularios, navegacao, alerts, modais, tabelas e acoes devem
usar componentes `Va*` diretamente quando houver equivalente. Qualquer instrucao
abaixo que cite `Base*` deve ser lida como historico da primeira implementacao, nao
como padrao para novas mudancas.

**Goal:** Make the TrackVision login and every authenticated admin screen use a cohesive Vuestic Admin-inspired template shell.

**Architecture:** Keep the existing TrackVision Vue app as the domain source of truth. Use Vuestic UI layout primitives for the authenticated shell, keep `Base*` wrappers as the UI boundary, and preserve all router/store/service contracts.

**Tech Stack:** Vue 3, Vite, TypeScript, Vue Router, Pinia, Vuestic UI, lucide-vue-next, Vitest, Vue Test Utils.

## Global Constraints

- Frontend repository: `C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Frontend`.
- AgentOps repository: `C:\projetos\rialma\RIALMA-TrackVision-AgentOps`.
- Main frontend branch is `main`, even when the user says master.
- Do not replace the project with the Vuestic Admin scaffold.
- Do not import demo pages from Vuestic Admin.
- Do not add Tailwind in this phase.
- Preserve route names, permissions, services, stores, API payloads, and authentication flow.
- Keep `VITE_API_BASE_URL` unchanged.
- Use TDD for production code changes.
- Run `npm run lint`, `npm run test -- --reporter=dot`, and `npm run build` before completion.

---

## File Structure

- Modify `src/pages/LoginPage.vue`: Vuestic Admin-style auth screen with split desktop layout, compact mobile layout, `VaForm`, existing store login flow, and show/hide password.
- Create `src/pages/LoginPage.test.ts`: behavior tests for auth-template structure, login submit, redirect, and unauthorized errors.
- Modify `src/layouts/AdminLayout.vue`: wrap the authenticated application in `VaLayout`, manage sidebar open/minimized state, and pass state into navigation components.
- Create `src/layouts/AdminLayout.test.ts`: verifies the layout renders Vuestic shell landmarks and toggles the sidebar state.
- Modify `src/components/navigation/TheSidebar.vue`: make sidebar template-like, grouped, responsive, collapsible, permission-filtered, and route-preserving.
- Modify `src/components/navigation/TheSidebar.test.ts`: keep permission filtering test and add minimized/toggle behavior.
- Modify `src/components/navigation/TheTopbar.vue`: use `VaNavbar` style composition, expose menu toggle, user identity, operational context, and logout.
- Create `src/components/navigation/TheTopbar.test.ts`: verifies user context, toggle event, and logout redirect.
- Modify `src/pages/DashboardPage.vue`: make the dashboard look like a real admin landing view with Vuestic metric/action cards based on visible permissions.
- Modify `src/pages/ForbiddenPage.vue` and `src/pages/NotFoundPage.vue`: align utility/error pages with Vuestic card/result-style surfaces.
- Modify `src/pages/*Page.vue` where needed: ensure pages keep `page-section`, `page-header`, `VaCard`, `BaseTable`, `BaseModal`, `BaseAlert`, and consistent action placement.
- Modify `src/styles/main.css`: remove old custom shell assumptions, add Vuestic template shell classes, responsive layout, auth layout, page card density, and table/form polish.

---

### Task 1: Login Auth Template

**Files:**
- Create: `src/pages/LoginPage.test.ts`
- Modify: `src/pages/LoginPage.vue`
- Modify: `src/styles/main.css`

**Interfaces:**
- Consumes: `useAuthStore().login(credentials: { email: string; password: string }): Promise<void>`, `route.query.redirect`, `router.push(path: string)`.
- Produces: `LoginPage.vue` renders `[data-test="auth-template"]`, `[data-test="login-form"]`, `[data-test="password-visibility"]`, and keeps email/password submit behavior.

- [ ] **Step 1: Write the failing login template test**

```ts
// src/pages/LoginPage.test.ts
import { flushPromises, mount } from '@vue/test-utils'
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { createRouter, createWebHistory } from 'vue-router'
import { ApiError } from '@/services/apiClient'
import { useAuthStore } from '@/stores/authStore'
import { createVuesticTestPlugin } from '@/test/vuestic'
import LoginPage from './LoginPage.vue'

vi.mock('@/services/authService', () => ({
  authService: {
    login: vi.fn(),
    logout: vi.fn(),
    me: vi.fn(),
  },
}))

vi.mock('@/services/permissionProbeService', () => ({
  permissionProbeService: {
    probeEffectivePermissions: vi.fn().mockResolvedValue(['users.manage']),
  },
}))

function createTestRouter() {
  return createRouter({
    history: createWebHistory(),
    routes: [
      { path: '/login', name: 'login', component: LoginPage },
      { path: '/dashboard', name: 'dashboard', component: { template: '<div>Dashboard</div>' } },
      { path: '/users', name: 'users', component: { template: '<div>Users</div>' } },
    ],
  })
}

async function mountLogin(path = '/login?redirect=/users') {
  const pinia = createPinia()
  setActivePinia(pinia)
  const router = createTestRouter()
  await router.push(path)
  await router.isReady()

  const wrapper = mount(LoginPage, {
    global: {
      plugins: [pinia, router, createVuesticTestPlugin()],
    },
  })

  return { wrapper, router, authStore: useAuthStore() }
}

describe('LoginPage', () => {
  beforeEach(() => {
    localStorage.clear()
    vi.clearAllMocks()
  })

  it('renders the Vuestic Admin auth template surface', async () => {
    const { wrapper } = await mountLogin()

    expect(wrapper.get('[data-test="auth-template"]').classes()).toContain('auth-template')
    expect(wrapper.get('[data-test="auth-brand-panel"]').text()).toContain('RIALMA')
    expect(wrapper.get('[data-test="login-form"]').text()).toContain('Acesso administrativo')
    expect(wrapper.find('input[name="email"]').exists()).toBe(true)
    expect(wrapper.find('input[name="password"]').exists()).toBe(true)
  })

  it('submits credentials and follows the redirect query', async () => {
    const { wrapper, router, authStore } = await mountLogin('/login?redirect=/users')
    const login = vi.spyOn(authStore, 'login').mockResolvedValue(undefined)

    await wrapper.find('input[name="email"]').setValue('admin@trackvision.local')
    await wrapper.find('input[name="password"]').setValue('Admin@123456')
    await wrapper.get('[data-test="login-form"]').trigger('submit')
    await flushPromises()

    expect(login).toHaveBeenCalledWith({
      email: 'admin@trackvision.local',
      password: 'Admin@123456',
    })
    expect(router.currentRoute.value.fullPath).toBe('/users')
  })

  it('shows an unauthorized message without leaving the login route', async () => {
    const { wrapper, authStore, router } = await mountLogin('/login')
    vi.spyOn(authStore, 'login').mockRejectedValue(new ApiError('Falha', 401, {}))

    await wrapper.find('input[name="email"]').setValue('wrong@example.com')
    await wrapper.find('input[name="password"]').setValue('wrong-password')
    await wrapper.get('[data-test="login-form"]').trigger('submit')
    await flushPromises()

    expect(wrapper.text()).toContain('Credenciais invalidas.')
    expect(router.currentRoute.value.name).toBe('login')
  })

  it('toggles password visibility', async () => {
    const { wrapper } = await mountLogin()
    const passwordInput = () => wrapper.find('input[name="password"]')

    expect(passwordInput().attributes('type')).toBe('password')
    await wrapper.get('[data-test="password-visibility"]').trigger('click')
    expect(passwordInput().attributes('type')).toBe('text')
  })
})
```

- [ ] **Step 2: Run the login test to verify it fails**

Run: `npm run test -- src/pages/LoginPage.test.ts --reporter=dot`

Expected: FAIL because `LoginPage.vue` does not render `data-test="auth-template"` or password visibility yet.

- [ ] **Step 3: Implement the minimal login template**

```vue
<!-- src/pages/LoginPage.vue template shape -->
<template>
  <main
    class="auth-template"
    data-test="auth-template"
  >
    <section
      class="auth-template__brand"
      data-test="auth-brand-panel"
    >
      <RouterLink
        class="auth-template__brand-mark"
        :to="{ name: 'login' }"
      >
        <span>RIALMA</span>
        <strong>TrackVision</strong>
      </RouterLink>
      <div class="auth-template__copy">
        <p class="page-eyebrow">Controle operacional</p>
        <h1>Monitoramento de acessos e viagens</h1>
        <p>Gestao segura para veiculos, cameras, viagens e relatorios operacionais.</p>
      </div>
    </section>

    <section class="auth-template__content">
      <VaCard class="auth-card">
        <VaCardContent class="auth-card__content">
          <VaForm
            class="auth-form"
            data-test="login-form"
            aria-labelledby="login-title"
            @submit.prevent="submitLogin"
          >
            <p class="page-eyebrow">Acesso administrativo</p>
            <h2 id="login-title">RIALMA TrackVision</h2>
            <p class="muted">Entre com seu usuario autorizado.</p>

            <BaseAlert v-if="formError" variant="error">
              {{ formError }}
            </BaseAlert>

            <BaseInput
              :error="fieldErrors.email"
              autocomplete="email"
              label="Email"
              :model-value="email"
              name="email"
              type="email"
              @update:model-value="email = $event"
            />

            <div class="password-field">
              <BaseInput
                :error="fieldErrors.password"
                autocomplete="current-password"
                label="Senha"
                :model-value="password"
                name="password"
                :type="passwordVisible ? 'text' : 'password'"
                @update:model-value="password = $event"
              />
              <button
                class="password-field__toggle"
                data-test="password-visibility"
                type="button"
                @click="passwordVisible = !passwordVisible"
              >
                {{ passwordVisible ? 'Ocultar' : 'Mostrar' }}
              </button>
            </div>

            <BaseButton
              class="auth-form__submit"
              :loading="isSubmitting"
              type="submit"
            >
              {{ isSubmitting ? 'Entrando...' : 'Entrar' }}
            </BaseButton>
          </VaForm>
        </VaCardContent>
      </VaCard>
    </section>
  </main>
</template>
```

Add to `<script setup>`:

```ts
const passwordVisible = ref(false)
```

Add CSS in `src/styles/main.css`:

```css
.auth-template {
  display: grid;
  min-height: 100vh;
  grid-template-columns: minmax(320px, 42vw) minmax(0, 1fr);
  background: var(--va-background-secondary);
}

.auth-template__brand {
  display: flex;
  min-height: 100%;
  flex-direction: column;
  justify-content: space-between;
  padding: 40px;
  background: #172033;
  color: #ffffff;
}

.auth-template__content {
  display: grid;
  place-items: center;
  padding: 24px;
}

.auth-card {
  width: min(440px, 100%);
}

.auth-card__content,
.auth-form {
  display: grid;
  gap: 16px;
}

.auth-form__submit {
  width: 100%;
}

.password-field {
  position: relative;
}

.password-field__toggle {
  position: absolute;
  right: 12px;
  top: 32px;
  border: 0;
  background: transparent;
  color: var(--va-primary);
  cursor: pointer;
  font-weight: 700;
}
```

- [ ] **Step 4: Run the login test to verify it passes**

Run: `npm run test -- src/pages/LoginPage.test.ts --reporter=dot`

Expected: PASS, four tests.

- [ ] **Step 5: Commit the login template**

```powershell
git add src/pages/LoginPage.vue src/pages/LoginPage.test.ts src/styles/main.css
git commit -m "feat: apply vuestic auth template"
```

---

### Task 2: Vuestic Admin Layout, Sidebar, And Topbar

**Files:**
- Create: `src/layouts/AdminLayout.test.ts`
- Create: `src/components/navigation/TheTopbar.test.ts`
- Modify: `src/layouts/AdminLayout.vue`
- Modify: `src/components/navigation/TheSidebar.vue`
- Modify: `src/components/navigation/TheSidebar.test.ts`
- Modify: `src/components/navigation/TheTopbar.vue`
- Modify: `src/styles/main.css`

**Interfaces:**
- Consumes: `TheSidebar` visible route list from `authStore.can(permission?: string): boolean`.
- Produces: `AdminLayout.vue` renders `[data-test="admin-layout"]`, `[data-test="sidebar-toggle"]`, `[data-test="admin-sidebar"]`, and `[data-test="admin-topbar"]`.
- Produces: `TheTopbar.vue` emits `toggle-sidebar` and keeps `logout(): Promise<void>` route to `{ name: 'login' }`.
- Produces: `TheSidebar.vue` accepts `minimized?: boolean` and emits `toggle-minimized` and `navigate`.

- [ ] **Step 1: Write failing layout and navigation tests**

```ts
// src/layouts/AdminLayout.test.ts
import { mount } from '@vue/test-utils'
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, describe, expect, it } from 'vitest'
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/authStore'
import { createVuesticTestPlugin } from '@/test/vuestic'
import AdminLayout from './AdminLayout.vue'

function router() {
  return createRouter({
    history: createWebHistory(),
    routes: [
      {
        path: '/',
        component: AdminLayout,
        children: [{ path: 'dashboard', name: 'dashboard', component: { template: '<div>Dashboard</div>' } }],
      },
    ],
  })
}

describe('AdminLayout', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    const authStore = useAuthStore()
    authStore.user = { id: 1, name: 'Paulo Peixoto', email: 'admin@trackvision.local', permissions: [] }
    authStore.permissions = ['users.manage', 'vehicles.manage', 'captures.view', 'cameras.manage', 'permissions.manage']
  })

  it('renders a Vuestic layout shell with sidebar, topbar and content', async () => {
    const appRouter = router()
    await appRouter.push('/dashboard')
    await appRouter.isReady()

    const wrapper = mount(AdminLayout, {
      global: { plugins: [appRouter, createVuesticTestPlugin()] },
    })

    expect(wrapper.get('[data-test="admin-layout"]').classes()).toContain('admin-layout')
    expect(wrapper.get('[data-test="admin-sidebar"]').text()).toContain('Dashboard')
    expect(wrapper.get('[data-test="admin-topbar"]').text()).toContain('Paulo Peixoto')
    expect(wrapper.text()).toContain('Dashboard')
  })

  it('toggles the minimized sidebar state from the topbar control', async () => {
    const appRouter = router()
    await appRouter.push('/dashboard')
    await appRouter.isReady()

    const wrapper = mount(AdminLayout, {
      global: { plugins: [appRouter, createVuesticTestPlugin()] },
    })

    expect(wrapper.get('[data-test="admin-sidebar"]').classes()).not.toContain('sidebar--minimized')
    await wrapper.get('[data-test="sidebar-toggle"]').trigger('click')
    expect(wrapper.get('[data-test="admin-sidebar"]').classes()).toContain('sidebar--minimized')
  })
})
```

```ts
// src/components/navigation/TheTopbar.test.ts
import { mount } from '@vue/test-utils'
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/authStore'
import { createVuesticTestPlugin } from '@/test/vuestic'
import TheTopbar from './TheTopbar.vue'

function router() {
  return createRouter({
    history: createWebHistory(),
    routes: [
      { path: '/login', name: 'login', component: { template: '<div>Login</div>' } },
      { path: '/dashboard', name: 'dashboard', component: { template: '<div>Dashboard</div>' } },
    ],
  })
}

describe('TheTopbar', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    const authStore = useAuthStore()
    authStore.user = { id: 1, name: 'Paulo Peixoto', email: 'admin@trackvision.local', permissions: [] }
  })

  it('shows operational context and emits sidebar toggles', () => {
    const wrapper = mount(TheTopbar, {
      global: { plugins: [router(), createVuesticTestPlugin()] },
    })

    expect(wrapper.get('[data-test="admin-topbar"]').text()).toContain('Painel administrativo')
    expect(wrapper.get('[data-test="admin-topbar"]').text()).toContain('Paulo Peixoto')
    wrapper.get('[data-test="sidebar-toggle"]').trigger('click')
    expect(wrapper.emitted('toggle-sidebar')).toHaveLength(1)
  })

  it('logs out and routes to login', async () => {
    const appRouter = router()
    await appRouter.push('/dashboard')
    await appRouter.isReady()
    const authStore = useAuthStore()
    const logout = vi.spyOn(authStore, 'logout').mockResolvedValue(undefined)

    const wrapper = mount(TheTopbar, {
      global: { plugins: [appRouter, createVuesticTestPlugin()] },
    })

    await wrapper.get('[data-test="logout-button"]').trigger('click')

    expect(logout).toHaveBeenCalledOnce()
    expect(appRouter.currentRoute.value.name).toBe('login')
  })
})
```

Extend `src/components/navigation/TheSidebar.test.ts` with:

```ts
it('emits toggle and hides labels when minimized', async () => {
  const authStore = useAuthStore()
  authStore.permissions = ['captures.view']
  const router = createRouter({
    history: createWebHistory(),
    routes: [
      { path: '/dashboard', name: 'dashboard', component: { template: '<div />' } },
      { path: '/trips', name: 'trips', component: { template: '<div />' } },
    ],
  })

  const wrapper = mount(TheSidebar, {
    props: { minimized: true },
    global: { plugins: [router] },
  })

  expect(wrapper.get('[data-test="admin-sidebar"]').classes()).toContain('sidebar--minimized')
  await wrapper.get('[data-test="sidebar-minimize"]').trigger('click')
  expect(wrapper.emitted('toggle-minimized')).toHaveLength(1)
})
```

- [ ] **Step 2: Run navigation tests to verify they fail**

Run:

```powershell
npm run test -- src/layouts/AdminLayout.test.ts src/components/navigation/TheSidebar.test.ts src/components/navigation/TheTopbar.test.ts --reporter=dot
```

Expected: FAIL because data-test hooks, `VaLayout`, and emitted toggle contracts are missing.

- [ ] **Step 3: Implement `AdminLayout.vue` with `VaLayout`**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import TheSidebar from '@/components/navigation/TheSidebar.vue'
import TheTopbar from '@/components/navigation/TheTopbar.vue'

const sidebarMinimized = ref(false)

function toggleSidebar(): void {
  sidebarMinimized.value = !sidebarMinimized.value
}
</script>

<template>
  <VaLayout
    class="admin-layout"
    data-test="admin-layout"
    :top="{ fixed: true, order: 2 }"
    :left="{ fixed: true, order: 1 }"
  >
    <template #top>
      <TheTopbar @toggle-sidebar="toggleSidebar" />
    </template>

    <template #left>
      <TheSidebar
        :minimized="sidebarMinimized"
        @toggle-minimized="toggleSidebar"
      />
    </template>

    <template #content>
      <main class="admin-content">
        <RouterView />
      </main>
    </template>
  </VaLayout>
</template>
```

- [ ] **Step 4: Implement `TheSidebar.vue` contract**

```vue
<aside
  class="sidebar"
  :class="{ 'sidebar--minimized': minimized }"
  data-test="admin-sidebar"
>
  <RouterLink class="sidebar__brand" :to="{ name: 'dashboard' }">
    <span>RIALMA</span>
    <strong>TrackVision</strong>
  </RouterLink>

  <button
    class="sidebar__minimize"
    data-test="sidebar-minimize"
    type="button"
    @click="$emit('toggle-minimized')"
  >
    {{ minimized ? 'Expandir' : 'Recolher' }}
  </button>

  <nav class="sidebar__nav" aria-label="Principal">
    <p class="sidebar__group">Operacao</p>
    <RouterLink
      v-for="item in visibleItems"
      :key="item.route"
      class="sidebar__link"
      active-class="sidebar__link--active"
      :to="{ name: item.route }"
      @click="$emit('navigate')"
    >
      <component :is="item.icon" :size="18" aria-hidden="true" />
      <span>{{ item.label }}</span>
    </RouterLink>
  </nav>
</aside>
```

Add props/emits:

```ts
withDefaults(defineProps<{ minimized?: boolean }>(), {
  minimized: false,
})

defineEmits<{
  'toggle-minimized': []
  navigate: []
}>()
```

- [ ] **Step 5: Implement `TheTopbar.vue` as a Vuestic navbar**

```vue
<VaNavbar
  class="topbar"
  data-test="admin-topbar"
>
  <template #left>
    <BaseButton
      data-test="sidebar-toggle"
      type="button"
      variant="ghost"
      title="Alternar menu"
      @click="$emit('toggle-sidebar')"
    >
      <Menu :size="18" aria-hidden="true" />
    </BaseButton>
    <div>
      <p class="topbar__eyebrow">Painel administrativo</p>
      <strong>Operacao TrackVision</strong>
    </div>
  </template>

  <template #right>
    <div class="topbar__user">
      <span>{{ authStore.user?.name ? authStore.user.name : 'Operador' }}</span>
      <small>{{ authStore.user?.email ? authStore.user.email : 'Sessao ativa' }}</small>
    </div>
    <BaseButton
      data-test="logout-button"
      type="button"
      variant="secondary"
      title="Sair"
      @click="logout"
    >
      <LogOut :size="18" aria-hidden="true" />
      <span>Sair</span>
    </BaseButton>
  </template>
</VaNavbar>
```

Add emits and icon import:

```ts
import { LogOut, Menu } from 'lucide-vue-next'

defineEmits<{
  'toggle-sidebar': []
}>()
```

- [ ] **Step 6: Add shell CSS**

```css
.admin-layout {
  min-height: 100vh;
  background: var(--va-background-primary);
}

.sidebar {
  display: flex;
  width: 270px;
  min-height: 100vh;
  flex-direction: column;
  border-right: 1px solid var(--va-background-border);
  background: #172033;
  color: #ffffff;
  transition: width 160ms ease;
}

.sidebar--minimized {
  width: 82px;
}

.sidebar--minimized .sidebar__brand strong,
.sidebar--minimized .sidebar__link span,
.sidebar--minimized .sidebar__group {
  display: none;
}

.sidebar__minimize {
  margin: 0 12px 12px;
  min-height: 34px;
  border: 1px solid rgb(255 255 255 / 16%);
  border-radius: 6px;
  background: rgb(255 255 255 / 8%);
  color: #ffffff;
  cursor: pointer;
}

.topbar {
  min-height: 64px;
  border-bottom: 1px solid var(--va-background-border);
  background: var(--va-background-secondary);
}

.topbar__user {
  display: grid;
  justify-items: end;
  line-height: 1.2;
}

.admin-content {
  min-height: calc(100vh - 64px);
  padding: 24px;
}
```

- [ ] **Step 7: Run navigation tests to verify they pass**

Run:

```powershell
npm run test -- src/layouts/AdminLayout.test.ts src/components/navigation/TheSidebar.test.ts src/components/navigation/TheTopbar.test.ts --reporter=dot
```

Expected: PASS.

- [ ] **Step 8: Commit shell changes**

```powershell
git add src/layouts/AdminLayout.vue src/layouts/AdminLayout.test.ts src/components/navigation/TheSidebar.vue src/components/navigation/TheSidebar.test.ts src/components/navigation/TheTopbar.vue src/components/navigation/TheTopbar.test.ts src/styles/main.css
git commit -m "feat: apply vuestic admin shell layout"
```

---

### Task 3: Page Template Surfaces And Dashboard Polish

**Files:**
- Modify: `src/pages/DashboardPage.vue`
- Modify: `src/pages/ForbiddenPage.vue`
- Modify: `src/pages/NotFoundPage.vue`
- Modify: `src/pages/UsersPage.vue`
- Modify: `src/pages/RolesPage.vue`
- Modify: `src/pages/PermissionsPage.vue`
- Modify: `src/pages/VehiclesPage.vue`
- Modify: `src/pages/TripsPage.vue`
- Modify: `src/pages/LocationsPage.vue`
- Modify: `src/pages/EdgeNodesPage.vue`
- Modify: `src/pages/CamerasPage.vue`
- Modify: `src/pages/CameraPairsPage.vue`
- Modify: `src/styles/main.css`
- Modify tests only when assertions depend on changed visible text or markup.

**Interfaces:**
- Consumes: Existing `BaseAlert`, `BaseButton`, `BaseModal`, `BaseTable`, `BaseInput`, `BaseSelect`, and page services.
- Produces: Every admin page has a visible `page-section`, `page-header`, and at least one Vuestic content surface; dashboard has `[data-test="dashboard-module-grid"]`.

- [ ] **Step 1: Write the failing dashboard template test**

Extend `src/App.test.ts` or create `src/pages/DashboardPage.test.ts` with:

```ts
import { mount } from '@vue/test-utils'
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, describe, expect, it } from 'vitest'
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/authStore'
import { createVuesticTestPlugin } from '@/test/vuestic'
import DashboardPage from './DashboardPage.vue'

describe('DashboardPage', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    const authStore = useAuthStore()
    authStore.permissions = ['users.manage', 'vehicles.manage', 'cameras.manage']
  })

  it('renders template dashboard cards for allowed modules', async () => {
    const router = createRouter({
      history: createWebHistory(),
      routes: [
        { path: '/dashboard', name: 'dashboard', component: DashboardPage },
        { path: '/users', name: 'users', component: { template: '<div />' } },
        { path: '/vehicles', name: 'vehicles', component: { template: '<div />' } },
        { path: '/locations', name: 'locations', component: { template: '<div />' } },
      ],
    })

    const wrapper = mount(DashboardPage, {
      global: { plugins: [router, createVuesticTestPlugin()] },
    })

    expect(wrapper.get('[data-test="dashboard-module-grid"]').text()).toContain('Usuarios')
    expect(wrapper.get('[data-test="dashboard-module-grid"]').text()).toContain('Veiculos')
    expect(wrapper.get('[data-test="dashboard-module-grid"]').text()).toContain('Locais e cameras')
  })
})
```

- [ ] **Step 2: Run dashboard test to verify it fails**

Run: `npm run test -- src/pages/DashboardPage.test.ts --reporter=dot`

Expected: FAIL because `data-test="dashboard-module-grid"` is missing.

- [ ] **Step 3: Implement dashboard and page surface polish**

Change `DashboardPage.vue` module grid shape:

```vue
<div
  class="module-grid"
  data-test="dashboard-module-grid"
>
  <RouterLink
    v-for="module in modules"
    :key="module.route"
    class="module-card"
    :to="{ name: module.route }"
  >
    <span>{{ module.label }}</span>
    <small>{{ module.description }}</small>
  </RouterLink>
</div>
```

Use module metadata:

```ts
const modules = computed(() => [
  { label: 'Usuarios', route: 'users', permission: 'users.manage', description: 'Controle de acesso administrativo' },
  { label: 'Veiculos', route: 'vehicles', permission: 'vehicles.manage', description: 'Cadastro de caminhoes monitorados' },
  { label: 'Locais e cameras', route: 'locations', permission: 'cameras.manage', description: 'Topologia de portarias e equipamentos' },
].filter((module) => authStore.can(module.permission)))
```

For `ForbiddenPage.vue` and `NotFoundPage.vue`, use:

```vue
<section class="page-section utility-page">
  <VaCard class="utility-card">
    <VaCardContent class="utility-card__body">
      <p class="page-eyebrow">Acesso</p>
      <h1>Acesso negado</h1>
      <p class="muted">Sua sessao nao possui permissao para esta area.</p>
      <BaseButton :to="{ name: 'dashboard' }">Voltar ao dashboard</BaseButton>
    </VaCardContent>
  </VaCard>
</section>
```

Keep existing titles and meanings for each page. Do not change services or payloads.

- [ ] **Step 4: Add page surface CSS**

```css
.page-section {
  display: grid;
  gap: 16px;
}

.page-header {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.content-panel,
.trip-detail,
.utility-card {
  overflow: hidden;
  border-radius: 8px;
}

.module-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 12px;
}

.module-card {
  display: grid;
  min-height: 96px;
  align-content: center;
  gap: 6px;
  padding: 16px;
  border: 1px solid var(--va-background-border);
  border-radius: 8px;
  color: var(--va-text-primary);
  text-decoration: none;
}

.module-card:hover {
  border-color: var(--va-primary);
  background: var(--va-background-element);
}

.module-card span {
  color: var(--va-primary);
  font-weight: 700;
}

.module-card small {
  color: var(--va-text-secondary);
}
```

- [ ] **Step 5: Run page tests**

Run:

```powershell
npm run test -- src/pages/DashboardPage.test.ts src/pages/UsersPage.test.ts src/pages/VehiclesPage.test.ts src/pages/TripsPage.test.ts src/pages/EdgeNodesPage.test.ts src/pages/CameraPairsPage.test.ts --reporter=dot
```

Expected: PASS.

- [ ] **Step 6: Commit page polish**

```powershell
git add src/pages src/styles/main.css
git commit -m "feat: align pages with vuestic template surfaces"
```

---

### Task 4: Full Verification, Runtime Smoke, And Push

**Files:**
- No production file changes expected.
- Commit any test-only fixes required by verification failures.

**Interfaces:**
- Consumes: Frontend local dev server on `http://127.0.0.1:5173`.
- Consumes: Backend API on `http://127.0.0.1:8000/api/v1`.
- Produces: Main branch pushed to GitHub after validation.

- [ ] **Step 1: Run full frontend verification**

Run:

```powershell
npm run lint
npm run test -- --reporter=dot
npm run build
```

Expected:

- lint exits `0`;
- all Vitest files pass;
- build exits `0`;
- existing Vuestic bundle-size warning is acceptable.

- [ ] **Step 2: Verify local services**

Run from AgentOps:

```powershell
docker ps --filter "name=rialma-trackvision-mysql" --format "{{.Names}} {{.Status}}"
Get-NetTCPConnection -LocalPort 5173,8000 -State Listen | Select-Object LocalAddress,LocalPort,OwningProcess
```

Expected:

- `rialma-trackvision-mysql` is running and healthy;
- ports `5173` and `8000` are listening.

- [ ] **Step 3: Restart frontend if needed**

If port `5173` is not listening, start Vite:

```powershell
Start-Process -FilePath 'npm.cmd' -ArgumentList 'run','dev','--','--host','127.0.0.1','--port','5173' -WorkingDirectory 'C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Frontend' -WindowStyle Hidden
```

- [ ] **Step 4: Smoke HTTP login and backend auth**

Run:

```powershell
$front = Invoke-WebRequest -Uri 'http://127.0.0.1:5173/login?redirect=/dashboard' -UseBasicParsing
$body = @{ email = 'admin@trackvision.local'; password = 'Admin@123456' } | ConvertTo-Json
$login = Invoke-RestMethod -Uri 'http://127.0.0.1:8000/api/v1/auth/login' -Method Post -Body $body -ContentType 'application/json'
[PSCustomObject]@{
  frontend_status = $front.StatusCode
  frontend_title = if ($front.Content -match '<title>(.*?)</title>') { $Matches[1] } else { '' }
  token_present = [bool]$login.access_token
  user_email = $login.user.email
}
```

Expected:

- `frontend_status` is `200`;
- `frontend_title` is `RIALMA TrackVision`;
- `token_present` is `True`;
- `user_email` is `admin@trackvision.local`.

- [ ] **Step 5: Open the local dashboard for review**

Use Codex browser panel:

```text
open http://127.0.0.1:5173/dashboard
```

Expected: the user can see the updated Vuestic Admin-style dashboard.

- [ ] **Step 6: Push main**

Run:

```powershell
git status --short --branch
git push origin main
```

Expected: frontend `main` is pushed to `origin/main` with a clean worktree.

---

## Self-Review

- Spec coverage: login, authenticated shell, sidebar, topbar, page surfaces, CSS, tests, smoke, and push are covered by Tasks 1-4.
- Marker scan: no pending markers are intentionally left in this plan.
- Type consistency: emitted events are `toggle-sidebar`, `toggle-minimized`, and `navigate`; test hooks are stable across tasks.
