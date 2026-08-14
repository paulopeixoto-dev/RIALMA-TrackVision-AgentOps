# Vuestic Admin Frontend Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate the existing RIALMA TrackVision Vue frontend to use Vuestic UI/Vuestic Admin conventions while preserving authentication, permissions, routes, services, stores and domain pages.

**Architecture:** The current TrackVision app remains the domain source of truth. Vuestic UI is introduced as the visual system through a global plugin config and through existing `Base*` wrapper components, limiting page churn and keeping future visual changes centralized. Layout, login and administrative pages then adopt a Vuestic Admin-style shell without importing demo pages or unrelated template features.

**Tech Stack:** Vue 3, Vite, TypeScript, Vue Router, Pinia, Vuestic UI, lucide-vue-next, Vitest, Vue Test Utils, Playwright.

## Global Constraints

- Esta fase pertence ao repositorio `RIALMA-TrackVision-Frontend`.
- Seguir `docs/frontend-vue-guidelines.md`.
- Usar Vue 3 com Composition API e `<script setup>`.
- Manter Vue Router com meta fields para autenticacao e permissao.
- Manter Pinia para sessao e estado compartilhado.
- Manter services isolados para integracao com API.
- Instalar e configurar `vuestic-ui`.
- Registrar o plugin Vuestic no `src/main.ts`.
- Importar os estilos oficiais do Vuestic UI.
- Criar configuracao global de tema para identidade TrackVision.
- Adaptar o layout administrativo para uma estrutura inspirada no Vuestic Admin.
- Substituir os componentes base proprios por wrappers sobre Vuestic UI.
- Migrar login, dashboard e paginas administrativas para o novo padrao visual.
- Manter controles de permissao existentes.
- Manter integracao com backend Laravel existente.
- Nao trocar backend ou contratos de API.
- Nao adicionar paginas demo do Vuestic Admin.
- Nao adicionar analytics, i18n, Storybook, GTM ou graficos sem necessidade de dominio.
- Nao reescrever services que ja funcionam.
- Nao mudar estrategia de autenticacao.
- Nao implementar captura Intelbras, relatorios ou dashboard em tempo real nesta fase.
- Nao adicionar Tailwind nesta fase.
- `VITE_API_BASE_URL` continua sendo a URL base da API.
- Login continua em `POST /api/v1/auth/login`.
- Logout continua em `POST /api/v1/auth/logout`.
- Usuario atual continua vindo de `GET /api/v1/me`.
- Rotas existentes mantem os mesmos nomes.
- Meta permissions seguem iguais.
- Nenhum segredo novo pode ser exposto no frontend.
- Verificacoes finais obrigatorias: `npm run lint`, `npm run test`, `npm run build` e smoke visual em `http://127.0.0.1:5173`.

---

## File Structure

Create:

- `src/app/vuestic.ts`: exports the Vuestic global configuration and TrackVision theme tokens.
- `src/app/vuestic.test.ts`: verifies the Vuestic config exposes the TrackVision colors expected by the app.
- `src/components/base/BaseButton.test.ts`: verifies the wrapper preserves `variant`, `loading`, `disabled` and click behavior.
- `src/components/base/BaseModal.test.ts`: verifies the wrapper renders title/content and emits `close`.
- `src/components/base/BaseTable.test.ts`: verifies loading, empty state and row slot behavior.
- `src/test/vuestic.ts`: creates a Vue Test Utils helper that installs Vuestic UI consistently in component tests.

Modify:

- `package.json`: add `vuestic-ui`.
- `package-lock.json`: update dependency lock after `npm install`.
- `src/main.ts`: import Vuestic styles and install `createVuestic(vuesticGlobalConfig)`.
- `src/styles/main.css`: remove most custom widget styling and keep TrackVision layout helpers that complement Vuestic.
- `src/components/base/BaseAlert.vue`: wrap `VaAlert`.
- `src/components/base/BaseButton.vue`: wrap `VaButton`.
- `src/components/base/BaseInput.vue`: wrap `VaInput`.
- `src/components/base/BaseSelect.vue`: wrap `VaSelect`.
- `src/components/base/BaseModal.vue`: wrap `VaModal`.
- `src/components/base/BaseTable.vue`: provide a Vuestic-styled table wrapper while preserving the existing row slot contract.
- `src/layouts/AdminLayout.vue`: adopt a Vuestic Admin-style shell.
- `src/components/navigation/TheSidebar.vue`: adopt Vuestic navigation styling and keep permission filtering.
- `src/components/navigation/TheTopbar.vue`: adopt Vuestic topbar styling and keep logout.
- `src/pages/LoginPage.vue`: migrate login UI to Vuestic components through wrappers.
- `src/pages/DashboardPage.vue`: migrate dashboard cards to Vuestic-style surface.
- Administrative pages and forms under `src/pages` and `src/components/forms`: adjust markup/classes where needed after wrapper migration.
- Existing tests that assert CSS classes or raw HTML structure: update them to assert behavior and visible user-facing text.

## Task 1: Add Vuestic UI Plugin And Theme

**Files:**
- Modify: `package.json`
- Modify: `package-lock.json`
- Create: `src/app/vuestic.ts`
- Create: `src/app/vuestic.test.ts`
- Modify: `src/main.ts`
- Modify: `src/App.test.ts`

**Interfaces:**
- Consumes: Vue app creation in `src/main.ts`.
- Produces: `vuesticGlobalConfig` exported from `src/app/vuestic.ts` and used by `createVuestic`.

- [ ] **Step 1: Write the failing config test**

Create `src/app/vuestic.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { vuesticGlobalConfig } from './vuestic'

describe('vuesticGlobalConfig', () => {
  it('defines the TrackVision theme colors used by Vuestic UI', () => {
    expect(vuesticGlobalConfig.config?.colors?.variables).toMatchObject({
      primary: '#1f6feb',
      secondary: '#1f2937',
      success: '#25855a',
      warning: '#b7791f',
      danger: '#c2410c',
    })
  })
})
```

- [ ] **Step 2: Run the new test to verify it fails**

Run:

```bash
npm run test -- src/app/vuestic.test.ts
```

Expected: FAIL because `src/app/vuestic.ts` does not exist.

- [ ] **Step 3: Install Vuestic UI**

Run:

```bash
npm install vuestic-ui
```

Expected: `package.json` and `package-lock.json` include `vuestic-ui`.

- [ ] **Step 4: Create the Vuestic config**

Create `src/app/vuestic.ts`:

```ts
import type { VuesticConfig } from 'vuestic-ui'

export const vuesticGlobalConfig = {
  config: {
    colors: {
      variables: {
        primary: '#1f6feb',
        secondary: '#1f2937',
        success: '#25855a',
        warning: '#b7791f',
        danger: '#c2410c',
        info: '#2563eb',
        backgroundPrimary: '#f5f7fa',
        backgroundSecondary: '#ffffff',
        backgroundElement: '#eef2f7',
        textPrimary: '#1f2933',
        textInverted: '#ffffff',
      },
    },
  },
} satisfies VuesticConfig
```

- [ ] **Step 5: Register Vuestic in `main.ts`**

Update `src/main.ts`:

```ts
import { createPinia } from 'pinia'
import { createApp } from 'vue'
import { createVuestic } from 'vuestic-ui'
import 'vuestic-ui/css'
import App from './App.vue'
import { vuesticGlobalConfig } from './app/vuestic'
import { createAppRouter } from './router'
import './styles/main.css'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia)
app.use(createAppRouter())
app.use(createVuestic(vuesticGlobalConfig))
app.mount('#app')
```

- [ ] **Step 6: Run the config test to verify it passes**

Run:

```bash
npm run test -- src/app/vuestic.test.ts
```

Expected: PASS.

- [ ] **Step 7: Run the existing app smoke unit test**

Run:

```bash
npm run test -- src/App.test.ts
```

Expected: PASS. If the test mounts components that need Vuestic, add the shared test plugin from Task 2 before updating assertions.

- [ ] **Step 8: Commit**

Run:

```bash
git add package.json package-lock.json src/main.ts src/app/vuestic.ts src/app/vuestic.test.ts src/App.test.ts
git commit -m "feat: configure vuestic ui theme"
```

## Task 2: Migrate Base Components To Vuestic Wrappers

**Files:**
- Create: `src/test/vuestic.ts`
- Modify: `src/components/base/BaseAlert.vue`
- Modify: `src/components/base/BaseButton.vue`
- Modify: `src/components/base/BaseInput.vue`
- Modify: `src/components/base/BaseSelect.vue`
- Modify: `src/components/base/BaseModal.vue`
- Modify: `src/components/base/BaseTable.vue`
- Create: `src/components/base/BaseButton.test.ts`
- Create: `src/components/base/BaseModal.test.ts`
- Create: `src/components/base/BaseTable.test.ts`

**Interfaces:**
- Consumes: `vuesticGlobalConfig` from Task 1.
- Produces: the same public wrapper APIs already used by forms and pages: `BaseButton`, `BaseInput`, `BaseSelect`, `BaseAlert`, `BaseModal`, `BaseTable`.

- [ ] **Step 1: Create the shared Vuestic test plugin helper**

Create `src/test/vuestic.ts`:

```ts
import { createVuestic } from 'vuestic-ui'
import { vuesticGlobalConfig } from '@/app/vuestic'

export function createVuesticTestPlugin() {
  return createVuestic(vuesticGlobalConfig)
}
```

- [ ] **Step 2: Write the failing BaseButton test**

Create `src/components/base/BaseButton.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it, vi } from 'vitest'
import { createVuesticTestPlugin } from '@/test/vuestic'
import BaseButton from './BaseButton.vue'

function mountButton(props = {}) {
  return mount(BaseButton, {
    props,
    slots: { default: 'Salvar' },
    global: { plugins: [createVuesticTestPlugin()] },
  })
}

describe('BaseButton', () => {
  it('keeps the button disabled while loading', () => {
    const wrapper = mountButton({ loading: true })

    expect(wrapper.get('button').attributes('disabled')).toBeDefined()
    expect(wrapper.text()).toContain('Salvar')
  })

  it('emits click when enabled', async () => {
    const onClick = vi.fn()
    const wrapper = mount(BaseButton, {
      slots: { default: 'Entrar' },
      attrs: { onClick },
      global: { plugins: [createVuesticTestPlugin()] },
    })

    await wrapper.get('button').trigger('click')

    expect(onClick).toHaveBeenCalledTimes(1)
  })
})
```

- [ ] **Step 3: Write the failing BaseModal test**

Create `src/components/base/BaseModal.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import { createVuesticTestPlugin } from '@/test/vuestic'
import BaseModal from './BaseModal.vue'

describe('BaseModal', () => {
  it('renders title and content when open', () => {
    const wrapper = mount(BaseModal, {
      props: { open: true, title: 'Novo veiculo' },
      slots: { default: 'Formulario' },
      global: {
        plugins: [createVuesticTestPlugin()],
        stubs: { Teleport: true },
      },
    })

    expect(wrapper.text()).toContain('Novo veiculo')
    expect(wrapper.text()).toContain('Formulario')
  })

  it('emits close from the header action', async () => {
    const wrapper = mount(BaseModal, {
      props: { open: true, title: 'Editar usuario' },
      global: {
        plugins: [createVuesticTestPlugin()],
        stubs: { Teleport: true },
      },
    })

    await wrapper.get('[aria-label="Fechar"]').trigger('click')

    expect(wrapper.emitted('close')).toHaveLength(1)
  })
})
```

- [ ] **Step 4: Write the failing BaseTable test**

Create `src/components/base/BaseTable.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import BaseTable from './BaseTable.vue'

const columns = [
  { key: 'name', label: 'Nome' },
  { key: 'status', label: 'Status' },
]

describe('BaseTable', () => {
  it('renders a loading row', () => {
    const wrapper = mount(BaseTable, {
      props: { columns, rows: [], loading: true },
    })

    expect(wrapper.text()).toContain('Carregando...')
  })

  it('renders empty text when no rows exist', () => {
    const wrapper = mount(BaseTable, {
      props: { columns, rows: [], emptyText: 'Sem dados.' },
    })

    expect(wrapper.text()).toContain('Sem dados.')
  })

  it('keeps the custom row slot contract', () => {
    const wrapper = mount(BaseTable, {
      props: {
        columns,
        rows: [{ id: 1, name: 'Portaria', status: 'Ativo' }],
      },
      slots: {
        row: '<td>{{ row.name }}</td><td>{{ row.status }}</td>',
      },
    })

    expect(wrapper.text()).toContain('Portaria')
    expect(wrapper.text()).toContain('Ativo')
  })
})
```

- [ ] **Step 5: Run wrapper tests to verify they fail**

Run:

```bash
npm run test -- src/components/base/BaseButton.test.ts src/components/base/BaseModal.test.ts src/components/base/BaseTable.test.ts
```

Expected: FAIL before the wrappers are migrated to support the new behavior.

- [ ] **Step 6: Implement BaseButton with `VaButton`**

Update `src/components/base/BaseButton.vue`:

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = withDefaults(
  defineProps<{
    variant?: 'primary' | 'secondary' | 'ghost' | 'danger'
    type?: 'button' | 'submit' | 'reset'
    disabled?: boolean
    loading?: boolean
  }>(),
  {
    variant: 'primary',
    type: 'button',
    disabled: false,
    loading: false,
  },
)

const color = computed(() => (props.variant === 'danger' ? 'danger' : props.variant === 'secondary' ? 'secondary' : 'primary'))
const preset = computed(() => (props.variant === 'ghost' ? 'plain' : undefined))
</script>

<template>
  <VaButton
    class="base-button"
    :color="color"
    :disabled="disabled || loading"
    :loading="loading"
    :preset="preset"
    :type="type"
  >
    <slot />
  </VaButton>
</template>
```

- [ ] **Step 7: Implement BaseInput with `VaInput`**

Update `src/components/base/BaseInput.vue`:

```vue
<script setup lang="ts">
const props = withDefaults(
  defineProps<{
    modelValue: string | number | null
    label: string
    name?: string
    type?: string
    error?: string | string[]
    autocomplete?: string
  }>(),
  {
    name: undefined,
    type: 'text',
    error: undefined,
    autocomplete: undefined,
  },
)

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

function firstError(error?: string | string[]): string {
  return Array.isArray(error) ? error[0] : (error ?? '')
}
</script>

<template>
  <VaInput
    class="base-field"
    :autocomplete="autocomplete"
    :error="Boolean(firstError(props.error))"
    :error-messages="firstError(props.error)"
    :label="label"
    :model-value="modelValue ?? ''"
    :name="name"
    :type="type"
    @update:model-value="emit('update:modelValue', String($event))"
  />
</template>
```

- [ ] **Step 8: Implement BaseSelect with `VaSelect`**

Update `src/components/base/BaseSelect.vue`:

```vue
<script setup lang="ts">
import { computed } from 'vue'

export interface BaseSelectOption {
  label: string
  value: string | number
  disabled?: boolean
}

const props = withDefaults(
  defineProps<{
    modelValue: string | number | null
    label: string
    name?: string
    options: BaseSelectOption[]
    error?: string | string[]
  }>(),
  {
    name: undefined,
    error: undefined,
  },
)

const emit = defineEmits<{
  'update:modelValue': [value: string | number]
}>()

const selectedOption = computed(() => props.options.find((option) => option.value === props.modelValue) ?? null)

function firstError(error?: string | string[]): string {
  return Array.isArray(error) ? error[0] : (error ?? '')
}

function updateValue(option: BaseSelectOption | null): void {
  emit('update:modelValue', option?.value ?? '')
}
</script>

<template>
  <VaSelect
    class="base-field"
    :error="Boolean(firstError(error))"
    :error-messages="firstError(error)"
    :label="label"
    :model-value="selectedOption"
    :name="name"
    :options="options"
    text-by="label"
    track-by="value"
    value-by="value"
    @update:model-value="updateValue"
  />
</template>
```

- [ ] **Step 9: Implement BaseAlert with `VaAlert`**

Update `src/components/base/BaseAlert.vue`:

```vue
<script setup lang="ts">
const props = withDefaults(
  defineProps<{
    variant?: 'info' | 'success' | 'warning' | 'error'
  }>(),
  {
    variant: 'info',
  },
)

const colorByVariant = {
  info: 'info',
  success: 'success',
  warning: 'warning',
  error: 'danger',
} as const
</script>

<template>
  <VaAlert
    class="base-alert"
    :color="colorByVariant[props.variant]"
    role="status"
  >
    <slot />
  </VaAlert>
</template>
```

- [ ] **Step 10: Implement BaseModal with `VaModal`**

Update `src/components/base/BaseModal.vue`:

```vue
<script setup lang="ts">
defineProps<{
  open: boolean
  title: string
}>()

const emit = defineEmits<{
  close: []
}>()
</script>

<template>
  <VaModal
    class="base-modal"
    :model-value="open"
    hide-default-actions
    max-width="760px"
    mobile-fullscreen
    @update:model-value="!$event && emit('close')"
  >
    <template #header>
      <div class="base-modal__header">
        <h2>{{ title }}</h2>
        <VaButton
          aria-label="Fechar"
          icon="close"
          preset="plain"
          @click="emit('close')"
        />
      </div>
    </template>

    <slot />
  </VaModal>
</template>
```

- [ ] **Step 11: Implement BaseTable with Vuestic-styled markup**

Update `src/components/base/BaseTable.vue` while preserving the existing slot contract:

```vue
<script setup lang="ts">
export interface BaseTableColumn {
  key: string
  label: string
}

defineProps<{
  columns: BaseTableColumn[]
  rows: unknown[]
  loading?: boolean
  emptyText?: string
}>()

function asRecord(row: unknown): Record<string, unknown> {
  return typeof row === 'object' && row !== null ? (row as Record<string, unknown>) : {}
}

function rowKey(row: unknown, index: number): string {
  const record = asRecord(row)
  return String(record.id ?? record.uuid ?? index)
}

function cellValue(row: unknown, key: string): unknown {
  return asRecord(row)[key]
}
</script>

<template>
  <div class="table-wrap va-table-responsive">
    <table class="va-table va-table--hoverable base-table">
      <thead>
        <tr>
          <th
            v-for="column in columns"
            :key="column.key"
          >
            {{ column.label }}
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="loading">
          <td :colspan="columns.length">
            Carregando...
          </td>
        </tr>
        <tr v-else-if="rows.length === 0">
          <td :colspan="columns.length">
            {{ emptyText ?? 'Nenhum registro encontrado.' }}
          </td>
        </tr>
        <template v-else>
          <tr
            v-for="(row, index) in rows"
            :key="rowKey(row, index)"
          >
            <slot
              name="row"
              :row="row"
            >
              <td
                v-for="column in columns"
                :key="column.key"
              >
                {{ cellValue(row, column.key) }}
              </td>
            </slot>
          </tr>
        </template>
      </tbody>
    </table>
  </div>
</template>
```

- [ ] **Step 12: Run wrapper tests to verify they pass**

Run:

```bash
npm run test -- src/components/base/BaseButton.test.ts src/components/base/BaseModal.test.ts src/components/base/BaseTable.test.ts
```

Expected: PASS.

- [ ] **Step 13: Commit**

Run:

```bash
git add src/test/vuestic.ts src/components/base
git commit -m "feat: migrate base components to vuestic"
```

## Task 3: Migrate Admin Shell, Sidebar, Topbar And Login

**Files:**
- Modify: `src/layouts/AdminLayout.vue`
- Modify: `src/components/navigation/TheSidebar.vue`
- Modify: `src/components/navigation/TheTopbar.vue`
- Modify: `src/pages/LoginPage.vue`
- Modify: `src/components/navigation/TheSidebar.test.ts`
- Modify: `src/App.test.ts`
- Modify: `tests/e2e/login.spec.ts`

**Interfaces:**
- Consumes: Vuestic plugin from Task 1 and Base wrappers from Task 2.
- Produces: Vuestic Admin-style shell while preserving route names, permission filtering and auth flow.

- [ ] **Step 1: Run current sidebar and auth tests for baseline**

Run:

```bash
npm run test -- src/components/navigation/TheSidebar.test.ts src/router/routerGuard.test.ts src/stores/authStore.test.ts
```

Expected: PASS before visual migration. If baseline fails, inspect and fix existing failing setup before changing shell files.

- [ ] **Step 2: Update sidebar test to assert permissions by visible link text**

Keep the test focused on behavior:

```ts
expect(wrapper.text()).toContain('Dashboard')
expect(wrapper.text()).toContain('Usuarios')
expect(wrapper.text()).not.toContain('Veiculos')
```

Do not assert internal Vuestic classes.

- [ ] **Step 3: Migrate `AdminLayout.vue`**

Use a shell with classes that complement Vuestic:

```vue
<script setup lang="ts">
import TheSidebar from '@/components/navigation/TheSidebar.vue'
import TheTopbar from '@/components/navigation/TheTopbar.vue'
</script>

<template>
  <div class="admin-shell">
    <TheSidebar />
    <div class="admin-main">
      <TheTopbar />
      <main class="admin-content">
        <RouterView />
      </main>
    </div>
  </div>
</template>
```

- [ ] **Step 4: Migrate `TheSidebar.vue` styling**

Preserve `navigationItems`, `visibleItems` and `authStore.can`. Change the template to Vuestic-compatible markup:

```vue
<template>
  <aside class="sidebar">
    <RouterLink
      class="sidebar__brand"
      :to="{ name: 'dashboard' }"
    >
      <span>RIALMA</span>
      <strong>TrackVision</strong>
    </RouterLink>

    <nav
      class="sidebar__nav"
      aria-label="Principal"
    >
      <RouterLink
        v-for="item in visibleItems"
        :key="item.route"
        class="sidebar__link"
        active-class="sidebar__link--active"
        :to="{ name: item.route }"
      >
        <component
          :is="item.icon"
          :size="18"
          aria-hidden="true"
        />
        <span>{{ item.label }}</span>
      </RouterLink>
    </nav>
  </aside>
</template>
```

- [ ] **Step 5: Migrate `TheTopbar.vue`**

Keep logout behavior and use `BaseButton`:

```vue
<script setup lang="ts">
import { LogOut } from 'lucide-vue-next'
import { useRouter } from 'vue-router'
import BaseButton from '@/components/base/BaseButton.vue'
import { useAuthStore } from '@/stores/authStore'

const authStore = useAuthStore()
const router = useRouter()

async function logout(): Promise<void> {
  await authStore.logout()
  await router.push({ name: 'login' })
}
</script>

<template>
  <header class="topbar">
    <div>
      <p class="topbar__eyebrow">
        Painel administrativo
      </p>
      <strong>{{ authStore.user?.name ?? 'Operador' }}</strong>
    </div>
    <BaseButton
      type="button"
      variant="secondary"
      title="Sair"
      @click="logout"
    >
      <LogOut
        :size="18"
        aria-hidden="true"
      />
      <span>Sair</span>
    </BaseButton>
  </header>
</template>
```

- [ ] **Step 6: Migrate `LoginPage.vue`**

Keep existing script logic. Replace the template with a Vuestic-style login:

```vue
<template>
  <main class="login-shell">
    <VaCard
      class="login-panel"
      tag="form"
      aria-labelledby="login-title"
      @submit.prevent="submitLogin"
    >
      <VaCardContent class="login-panel__content">
        <p class="page-eyebrow">
          Controle operacional
        </p>
        <h1 id="login-title">
          RIALMA TrackVision
        </h1>
        <p class="muted">
          Acesso administrativo
        </p>

        <BaseAlert
          v-if="formError"
          variant="error"
        >
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
        <BaseInput
          :error="fieldErrors.password"
          autocomplete="current-password"
          label="Senha"
          :model-value="password"
          name="password"
          type="password"
          @update:model-value="password = $event"
        />

        <BaseButton
          :loading="isSubmitting"
          type="submit"
        >
          {{ isSubmitting ? 'Entrando...' : 'Entrar' }}
        </BaseButton>
      </VaCardContent>
    </VaCard>
  </main>
</template>
```

- [ ] **Step 7: Run shell and auth tests**

Run:

```bash
npm run test -- src/components/navigation/TheSidebar.test.ts src/router/routerGuard.test.ts src/stores/authStore.test.ts src/App.test.ts
```

Expected: PASS.

- [ ] **Step 8: Commit**

Run:

```bash
git add src/layouts/AdminLayout.vue src/components/navigation src/pages/LoginPage.vue src/App.test.ts tests/e2e/login.spec.ts
git commit -m "feat: apply vuestic admin shell"
```

## Task 4: Migrate Dashboard, Pages And Forms To Vuestic Surfaces

**Files:**
- Modify: `src/pages/DashboardPage.vue`
- Modify: `src/pages/UsersPage.vue`
- Modify: `src/pages/RolesPage.vue`
- Modify: `src/pages/PermissionsPage.vue`
- Modify: `src/pages/VehiclesPage.vue`
- Modify: `src/pages/TripsPage.vue`
- Modify: `src/pages/LocationsPage.vue`
- Modify: `src/pages/EdgeNodesPage.vue`
- Modify: `src/pages/CamerasPage.vue`
- Modify: `src/pages/CameraPairsPage.vue`
- Modify: `src/pages/ForbiddenPage.vue`
- Modify: `src/pages/NotFoundPage.vue`
- Modify: `src/components/forms/*.vue`
- Modify: page and form tests touched by markup changes.

**Interfaces:**
- Consumes: Base wrappers from Task 2 and admin shell from Task 3.
- Produces: all current TrackVision pages rendered inside Vuestic-style content surfaces without changing service calls or payloads.

- [ ] **Step 1: Run representative page and form tests for baseline**

Run:

```bash
npm run test -- src/pages/UsersPage.test.ts src/pages/VehiclesPage.test.ts src/pages/CameraPairsPage.test.ts src/components/forms/CameraForm.test.ts src/components/forms/UserForm.test.ts
```

Expected: PASS before markup migration. If a baseline test fails because Vuestic is not installed in its mount helper, update that test to use `createVuesticTestPlugin()` and keep the same behavioral assertions.

- [ ] **Step 2: Migrate page headers to Vuestic card/surface pattern**

For each page, keep the existing `<section class="page-section">` and header structure. Wrap table/filter content in `VaCard` with `VaCardContent`:

```vue
<VaCard class="content-panel">
  <VaCardContent>
    <BaseTable
      :columns="columns"
      :loading="loading"
      :rows="rows"
    />
  </VaCardContent>
</VaCard>
```

Use the actual rows variable for each page, for example `users`, `vehicles`, `locations`, `edgeNodes`, `cameras`, `cameraPairs` or `trips`.

- [ ] **Step 3: Keep CRUD methods unchanged**

Do not change functions such as:

```ts
async function saveVehicle(): Promise<void>
async function deleteVehicle(vehicle: Vehicle): Promise<void>
async function saveUser(): Promise<void>
async function savePassword(): Promise<void>
```

Only adjust template classes and component wrappers.

- [ ] **Step 4: Migrate boolean controls in forms to `VaCheckbox`**

For each form currently using `.checkbox-field`, replace the raw checkbox with:

```vue
<VaCheckbox
  :model-value="modelValue.is_active"
  label="Ativo"
  name="is_active"
  @update:model-value="updateField('is_active', Boolean($event))"
/>
```

For camera form use label `"Ativa"`.

- [ ] **Step 5: Preserve password write-only behavior**

In `src/components/forms/CameraForm.vue`, keep:

```vue
<BaseInput
  :error="errors.password"
  autocomplete="new-password"
  label="Senha"
  :model-value="modelValue.password ?? ''"
  name="password"
  type="password"
  @update:model-value="updateField('password', $event)"
/>
```

The test `src/components/forms/CameraForm.test.ts` must continue proving that an existing password is not displayed.

- [ ] **Step 6: Update page/form tests to use Vuestic plugin where needed**

When a test mounts a component that renders `VaCard`, `VaCheckbox`, `VaInput`, `VaSelect`, `VaButton` or `VaModal`, include:

```ts
global: {
  plugins: [createVuesticTestPlugin()],
}
```

If the test already stubs base components, keep the stubs and do not assert Vuestic internals.

- [ ] **Step 7: Run representative tests**

Run:

```bash
npm run test -- src/pages/UsersPage.test.ts src/pages/VehiclesPage.test.ts src/pages/CameraPairsPage.test.ts src/components/forms/CameraForm.test.ts src/components/forms/UserForm.test.ts
```

Expected: PASS.

- [ ] **Step 8: Commit**

Run:

```bash
git add src/pages src/components/forms
git commit -m "feat: migrate admin pages to vuestic surfaces"
```

## Task 5: Rework Global Styles For Vuestic Admin Look

**Files:**
- Modify: `src/styles/main.css`
- Modify: `index.html`

**Interfaces:**
- Consumes: Vuestic CSS and markup/classes from Tasks 2-4.
- Produces: a TrackVision admin skin that uses Vuestic tokens, has responsive layout and does not fight Vuestic component styling.

- [ ] **Step 1: Write a CSS intent checklist in the commit message draft**

Use this checklist while editing:

```text
- body and #app establish full-height app shell
- login uses centered panel and no marketing hero
- admin shell uses sidebar + topbar + content layout
- mobile layout keeps navigation usable under 800px
- forms use consistent gaps
- table wrapper scrolls horizontally on small screens
- no orb, bokeh, landing-page or demo-template decoration
```

- [ ] **Step 2: Update `src/styles/main.css`**

Keep only project-level layout and spacing styles. Use Vuestic variables where possible:

```css
:root {
  color: var(--va-text-primary);
  background: var(--va-background-primary);
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
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

.login-shell {
  display: grid;
  min-height: 100vh;
  place-items: center;
  padding: 24px;
  background: linear-gradient(135deg, #eef2f7 0%, #f8fafc 55%, #e8eef7 100%);
}

.login-panel {
  width: min(430px, 100%);
}

.login-panel__content,
.entity-form,
.page-section,
.content-panel__body {
  display: grid;
  gap: 16px;
}

.admin-shell {
  display: grid;
  min-height: 100vh;
  grid-template-columns: 270px minmax(0, 1fr);
  background: var(--va-background-primary);
}

.sidebar {
  display: flex;
  min-height: 100vh;
  flex-direction: column;
  border-right: 1px solid var(--va-background-border);
  background: #172033;
  color: #ffffff;
}

.sidebar__brand {
  display: grid;
  padding: 22px;
  color: #ffffff;
  text-decoration: none;
}

.sidebar__brand span {
  color: #9fb3c8;
  font-size: 12px;
  font-weight: 700;
}

.sidebar__nav {
  display: grid;
  gap: 4px;
  padding: 12px;
}

.sidebar__link {
  display: flex;
  min-height: 42px;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 6px;
  color: #d9e2ec;
  text-decoration: none;
}

.sidebar__link--active,
.sidebar__link:hover {
  background: rgb(255 255 255 / 12%);
  color: #ffffff;
}

.admin-main {
  display: flex;
  min-width: 0;
  flex-direction: column;
}

.topbar {
  display: flex;
  min-height: 64px;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 0 24px;
  border-bottom: 1px solid var(--va-background-border);
  background: var(--va-background-secondary);
}

.topbar__eyebrow,
.page-eyebrow,
.muted {
  color: var(--va-text-secondary);
}

.admin-content {
  width: min(1240px, 100%);
  padding: 24px;
}

.page-header,
.form-actions,
.row-actions {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 10px;
}

.page-header {
  justify-content: space-between;
}

.page-header h1,
.login-panel h1,
.base-modal__header h2 {
  margin: 0;
  letter-spacing: 0;
}

.table-wrap {
  overflow-x: auto;
}

.base-table {
  min-width: 100%;
}

.base-modal__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  width: 100%;
}

@media (max-width: 800px) {
  .admin-shell {
    grid-template-columns: 1fr;
  }

  .sidebar {
    min-height: auto;
  }

  .sidebar__nav {
    grid-auto-flow: column;
    overflow-x: auto;
  }

  .topbar,
  .admin-content {
    padding-inline: 16px;
  }
}
```

- [ ] **Step 3: Update `index.html` to include Vuestic-friendly assets**

Add Material Icons and Source Sans Pro links inside `<head>` if they are not already present:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Source+Sans+Pro:wght@400;600;700&display=swap" rel="stylesheet">
<link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">
```

- [ ] **Step 4: Run style-sensitive tests**

Run:

```bash
npm run test -- src/App.test.ts src/pages/VehiclesPage.test.ts src/components/navigation/TheSidebar.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit**

Run:

```bash
git add src/styles/main.css index.html
git commit -m "style: align frontend with vuestic admin"
```

## Task 6: Full Verification, Browser Smoke And Remote Sync

**Files:**
- Modify only files needed to fix verification failures discovered in this task.

**Interfaces:**
- Consumes: all prior tasks.
- Produces: verified frontend branch with Vuestic migration committed and pushed.

- [ ] **Step 1: Run lint**

Run:

```bash
npm run lint
```

Expected: exit code 0.

- [ ] **Step 2: Run unit/component tests**

Run:

```bash
npm run test
```

Expected: all Vitest suites pass.

- [ ] **Step 3: Run production build**

Run:

```bash
npm run build
```

Expected: `vue-tsc --noEmit` and `vite build` exit with code 0.

- [ ] **Step 4: Start or reuse backend and frontend dev servers**

Backend:

```powershell
Start-Process -FilePath php -ArgumentList @('artisan','serve','--host=127.0.0.1','--port=8000') -WorkingDirectory 'C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Backend' -WindowStyle Hidden
```

Frontend:

```powershell
Start-Process -FilePath npm.cmd -ArgumentList @('run','dev','--','--host','127.0.0.1','--port','5173') -WorkingDirectory 'C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Frontend' -WindowStyle Hidden
```

- [ ] **Step 5: Verify admin API login still works**

Run:

```powershell
$body = @{ email = 'admin@trackvision.local'; password = 'Admin@123456' } | ConvertTo-Json
$response = Invoke-RestMethod -Uri 'http://127.0.0.1:8000/api/v1/auth/login' -Method Post -ContentType 'application/json' -Body $body
[pscustomobject]@{ token_present = [bool]$response.access_token; token_type = $response.token_type; user_email = $response.user.email }
```

Expected: `token_present` is `True`, `token_type` is `Bearer`, and `user_email` is `admin@trackvision.local`.

- [ ] **Step 6: Run browser smoke**

Open:

```text
http://127.0.0.1:5173
```

Expected:

- the page title is `RIALMA TrackVision`;
- unauthenticated users land on `/login?redirect=/dashboard`;
- login form shows `Email`, `Senha` and `Entrar`;
- after logging in with `admin@trackvision.local` / `Admin@123456`, the dashboard renders;
- sidebar includes TrackVision domain links allowed for the admin user.

- [ ] **Step 7: Inspect git status**

Run:

```bash
git status --short --branch
```

Expected: clean working tree on the migration branch after commits.

- [ ] **Step 8: Push branch**

Run:

```bash
git push -u origin codex/vuestic-admin-migration
```

Expected: branch pushed to GitHub.

## Self-Review

- Spec coverage: every spec requirement is covered by Tasks 1-6.
- Placeholder scan: no placeholder, incomplete task or vague implementation step is intentionally left in this plan.
- Type consistency: `vuesticGlobalConfig`, `createVuesticTestPlugin`, `BaseButton`, `BaseInput`, `BaseSelect`, `BaseAlert`, `BaseModal` and `BaseTable` names match across tasks.
- Scope check: the plan is one migration project and excludes backend/API/auth changes.
