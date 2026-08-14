# CI/CD Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Criar validacao automatica no GitHub Actions para backend e frontend do TrackVision.

**Architecture:** Cada repositorio recebe seu proprio workflow GitHub Actions, porque backend e frontend sao repositorios separados. O backend valida Composer, estilo PHP com Pint e testes Laravel. O frontend valida ESLint, Vitest e build Vite/TypeScript.

**Tech Stack:** GitHub Actions, Ubuntu runner, PHP 8.4, Composer v2, Laravel Pint, PHPUnit/Laravel Test Runner, Node.js 24, npm, ESLint, Vitest, Vite.

## Global Constraints

- Backend code lives only in `RIALMA-TrackVision-Backend`.
- Frontend code lives only in `RIALMA-TrackVision-Frontend`.
- AgentOps documentation lives only in `RIALMA-TrackVision-AgentOps`.
- Workflows run on `push` and `pull_request` to `main`.
- Workflows use `permissions: contents: read`.
- No workflow in this phase requires secrets.
- No deploy, Docker image publish, GitHub Environment, Dependabot or E2E Playwright in this phase.
- Backend CI must run `composer validate --strict`, `vendor/bin/pint --test`, and `php artisan test`.
- Frontend CI must run `npm run lint`, `npm test -- --run`, and `npm run build`.
- Correct existing backend Pint style drift before enabling Pint in CI.

---

## File Structure Map

Backend files:

```text
RIALMA-TrackVision-Backend/
|-- .github/
|   `-- workflows/
|       `-- backend-ci.yml
|-- README.md
`-- tests/Unit/Cameras/Intelbras/IntelbrasEventStreamClientTest.php
```

Frontend files:

```text
RIALMA-TrackVision-Frontend/
|-- .github/
|   `-- workflows/
|       `-- frontend-ci.yml
`-- README.md
```

AgentOps files:

```text
RIALMA-TrackVision-AgentOps/
|-- docs/project-context.md
`-- docs/superpowers/plans/2026-08-14-ci-cd-validation-implementation.md
```

---

## Task 1: Backend CI Workflow

**Files:**

- Create: `RIALMA-TrackVision-Backend/.github/workflows/backend-ci.yml`
- Modify: `RIALMA-TrackVision-Backend/tests/Unit/Cameras/Intelbras/IntelbrasEventStreamClientTest.php`
- Modify: `RIALMA-TrackVision-Backend/README.md`

**Interfaces:**

- Consumes: `composer.lock`, `.env.example`, `vendor/bin/pint`, `php artisan test`.
- Produces: GitHub Actions workflow named `Backend CI`.

- [ ] **Step 1: Confirm current backend style failure**

Run:

```bash
vendor/bin/pint --test
```

Expected: FAIL with `tests\Unit\Cameras\Intelbras\IntelbrasEventStreamClientTest.php` requiring `single_quote`.

- [ ] **Step 2: Fix the existing Pint drift**

Modify `tests/Unit/Cameras/Intelbras/IntelbrasEventStreamClientTest.php`:

```php
$body = '--boundary\r\nContent-Type: text/plain\r\n\r\nEvents[0].EventBaseInfo.Code=TrafficJunction\r\nEvents[0].EventBaseInfo.Action=Pulse\r\nEvents[0].TrafficCar.PlateNumber=ABC1D23\r\n--boundary--';
```

And:

```php
$onChunk('--boundary\r\nContent-Type: text/plain\r\n\r\nEvents[0].EventBaseInfo.Code=TrafficJunction\r\nEvents[0].EventBaseInfo.Action=Pulse\r\nEvents[0].TrafficCar.PlateNumber=ABC1D23\r\n');
$onChunk('--boundary--');
```

- [ ] **Step 3: Create backend workflow**

Create `.github/workflows/backend-ci.yml`:

```yaml
name: Backend CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  backend:
    name: Laravel
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: mbstring, dom, fileinfo, pdo_sqlite, sqlite3
          tools: composer:v2
          coverage: none

      - name: Cache Composer dependencies
        uses: actions/cache@v4
        with:
          path: ~/.composer/cache/files
          key: composer-${{ runner.os }}-${{ hashFiles('composer.lock') }}
          restore-keys: composer-${{ runner.os }}-

      - name: Validate Composer
        run: composer validate --strict

      - name: Install dependencies
        run: composer install --prefer-dist --no-progress --no-interaction

      - name: Prepare Laravel environment
        run: |
          cp .env.example .env
          php artisan key:generate --ansi

      - name: Check PHP style
        run: vendor/bin/pint --test

      - name: Run tests
        run: php artisan test
```

- [ ] **Step 4: Update backend README**

Append under `## Comandos`:

````markdown
### CI Local

O workflow `Backend CI` do GitHub Actions roda os comandos equivalentes:

```bash
composer validate --strict
vendor/bin/pint --test
php artisan test
```
````

- [ ] **Step 5: Verify backend CI commands locally**

Run:

```bash
composer validate --strict
vendor/bin/pint --test
php artisan test
git diff --check
```

Expected:

- Composer valid.
- Pint passes.
- Laravel tests pass.
- `git diff --check` has no output.

- [ ] **Step 6: Commit backend CI**

Run:

```bash
git add .github/workflows/backend-ci.yml README.md tests/Unit/Cameras/Intelbras/IntelbrasEventStreamClientTest.php
git commit -m "ci: add backend validation workflow"
```

---

## Task 2: Frontend CI Workflow

**Files:**

- Create: `RIALMA-TrackVision-Frontend/.github/workflows/frontend-ci.yml`
- Modify: `RIALMA-TrackVision-Frontend/README.md`

**Interfaces:**

- Consumes: `package-lock.json`, `npm run lint`, `npm test -- --run`, `npm run build`.
- Produces: GitHub Actions workflow named `Frontend CI`.

- [ ] **Step 1: Confirm frontend quality commands pass locally**

Run:

```bash
npm run lint
npm test -- --run
npm run build
```

Expected: all commands PASS before adding them to CI.

- [ ] **Step 2: Create frontend workflow**

Create `.github/workflows/frontend-ci.yml`:

```yaml
name: Frontend CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  frontend:
    name: Vue
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Setup Node
        uses: actions/setup-node@v5
        with:
          node-version: '24'
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Run unit tests
        run: npm test -- --run

      - name: Build
        run: npm run build
```

- [ ] **Step 3: Update frontend README**

Append under `## Verification`:

````markdown
### CI Local

O workflow `Frontend CI` do GitHub Actions roda os comandos equivalentes:

```bash
npm run lint
npm test -- --run
npm run build
```
````

- [ ] **Step 4: Verify frontend CI commands locally**

Run:

```bash
npm run lint
npm test -- --run
npm run build
git diff --check
```

Expected:

- ESLint passes.
- Vitest passes.
- Production build passes.
- `git diff --check` has no output.

- [ ] **Step 5: Commit frontend CI**

Run:

```bash
git add .github/workflows/frontend-ci.yml README.md
git commit -m "ci: add frontend validation workflow"
```

---

## Task 3: AgentOps Handoff And Final Publication

**Files:**

- Modify: `RIALMA-TrackVision-AgentOps/docs/project-context.md`
- Verify: backend, frontend and AgentOps git status.

**Interfaces:**

- Consumes: backend and frontend workflow commits from Tasks 1 and 2.
- Produces: project documentation note that CI validation exists.

- [ ] **Step 1: Update AgentOps project context**

Append to `docs/project-context.md`:

```markdown
## CI/CD

Backend e frontend possuem workflows GitHub Actions para validacao em `push` e
`pull_request` para `main`.

- Backend: Composer validate, Pint e testes Laravel.
- Frontend: ESLint, Vitest e build Vite.

Deploy automatico ainda nao faz parte do escopo; sera definido junto com a
infraestrutura de producao e homologacao de campo.
```

- [ ] **Step 2: Verify repository states**

Run:

```bash
git status -sb
```

In:

- `RIALMA-TrackVision-AgentOps`
- `RIALMA-TrackVision-Backend`
- `RIALMA-TrackVision-Frontend`

Expected:

- AgentOps has only the `docs/project-context.md` change before commit.
- Backend is clean after Task 1 commit.
- Frontend is clean after Task 2 commit.

- [ ] **Step 3: Commit AgentOps handoff note**

Run:

```bash
git add docs/project-context.md
git commit -m "docs: record ci validation workflows"
```

- [ ] **Step 4: Push all repositories**

Run:

```bash
git push origin main
```

In:

- `RIALMA-TrackVision-AgentOps`
- `RIALMA-TrackVision-Backend`
- `RIALMA-TrackVision-Frontend`

- [ ] **Step 5: Final verification after push**

Run:

```bash
git status -sb
git log --oneline -3
```

In each repository.

Expected:

- Each repository shows `main...origin/main` with no ahead/behind.
- Recent logs include the CI-related commit(s).

---

## Acceptance Checklist

- [ ] Backend workflow exists at `.github/workflows/backend-ci.yml`.
- [ ] Backend workflow runs on `push` and `pull_request` to `main`.
- [ ] Backend workflow uses `permissions: contents: read`.
- [ ] Backend workflow runs Composer validate, Pint and Laravel tests.
- [ ] Existing backend Pint drift is fixed.
- [ ] Frontend workflow exists at `.github/workflows/frontend-ci.yml`.
- [ ] Frontend workflow runs on `push` and `pull_request` to `main`.
- [ ] Frontend workflow uses `permissions: contents: read`.
- [ ] Frontend workflow runs ESLint, Vitest and build.
- [ ] Backend README documents local CI-equivalent commands.
- [ ] Frontend README documents local CI-equivalent commands.
- [ ] AgentOps project context records that CI validation exists.
- [ ] All local verification commands pass before push.
