# Backend Foundation and Security Phase Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans for inline execution. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the Laravel backend foundation with runtime profiles, Passport authentication, and Spatie role-based permissions.

**Architecture:** Scaffold a Laravel API backend, add a TrackVision runtime profile resolver, then layer authentication and RBAC behind thin API controllers. Tests drive each behavior before implementation.

**Tech Stack:** Laravel 13, PHP 8.4, Composer, Laravel Passport, Spatie Laravel Permission, PHPUnit/Pest, SQLite for tests.

## Global Constraints

- Work in `RIALMA-TrackVision-Backend` on branch `codex/backend-foundation-security`.
- Follow `docs/backend-laravel-guidelines.md`.
- Do not implement Intelbras, edge sync, trips, reports, or Vue in this phase.
- Controllers stay thin.
- Use Form Requests for API input.
- Use Resources for API output.
- No secrets in Git.

---

## Task 1: Laravel Scaffold and Runtime Profiles

**Files:**

- Create/modify Laravel scaffold files in `RIALMA-TrackVision-Backend`.
- Create: `config/trackvision.php`
- Create: `app/Enums/NodeRole.php`
- Create: `app/Support/NodeRoleResolver.php`
- Create: `tests/Unit/Support/NodeRoleResolverTest.php`
- Modify: `.env.example`
- Modify: `README.md`

**Interfaces:**

- Produces: `App\Enums\NodeRole::Parent`
- Produces: `App\Enums\NodeRole::Edge`
- Produces: `App\Support\NodeRoleResolver::current(): NodeRole`
- Consumes: `TRACKVISION_NODE_ROLE=parent|edge`

**Steps:**

- [ ] Scaffold Laravel in the backend repo.
- [ ] Add `TRACKVISION_NODE_ROLE=parent` and TrackVision env keys to `.env.example`.
- [ ] Write `NodeRoleResolverTest` with tests for `parent`, `edge`, and invalid value.
- [ ] Run `php artisan test --filter=NodeRoleResolverTest` and confirm it fails because `NodeRoleResolver` does not exist.
- [ ] Create `NodeRole` enum.
- [ ] Create `NodeRoleResolver`.
- [ ] Create `config/trackvision.php`.
- [ ] Run `php artisan test --filter=NodeRoleResolverTest` and confirm it passes.
- [ ] Commit backend as `chore: scaffold laravel backend foundation`.

## Task 2: Laravel Passport Authentication

**Files:**

- Modify: `composer.json`
- Modify: `app/Models/User.php`
- Create: `app/Http/Controllers/Api/V1/Auth/LoginController.php`
- Create: `app/Http/Controllers/Api/V1/Auth/LogoutController.php`
- Create: `app/Http/Controllers/Api/V1/Auth/MeController.php`
- Create: `app/Http/Requests/Api/V1/Auth/LoginRequest.php`
- Create: `app/Http/Resources/Api/V1/UserResource.php`
- Modify: `routes/api.php`
- Create: `tests/Feature/Auth/LoginTest.php`
- Create: `tests/Feature/Auth/EdgeClientCredentialsTest.php`

**Interfaces:**

- Produces: `POST /api/v1/auth/login`
- Produces: `POST /api/v1/auth/logout`
- Produces: `GET /api/v1/me`
- Produces Passport scopes: `admin:read`, `admin:write`, `edge:read`, `edge:write`, `captures:write`, `reports:read`.

**Steps:**

- [ ] Install Laravel Passport.
- [ ] Run Passport migrations.
- [ ] Write `LoginTest` for successful login, invalid credentials, current user, and logout.
- [ ] Run `php artisan test --filter=LoginTest` and confirm it fails because endpoints do not exist.
- [ ] Configure Passport on `User`.
- [ ] Implement `LoginRequest`, `UserResource`, `LoginController`, `LogoutController`, and `MeController`.
- [ ] Register API routes.
- [ ] Run `php artisan test --filter=LoginTest` and confirm it passes.
- [ ] Write `EdgeClientCredentialsTest` for client credentials access to a protected minimal edge bootstrap route and denial on admin-only route.
- [ ] Run `php artisan test --filter=EdgeClientCredentialsTest` and confirm it fails until scope middleware is configured.
- [ ] Configure scopes and protected routes.
- [ ] Run `php artisan test --filter=Auth` and confirm it passes.
- [ ] Commit backend as `feat: add passport authentication`.

## Task 3: Spatie Roles and Permissions

**Files:**

- Modify: `composer.json`
- Modify: `app/Models/User.php`
- Create: `app/Support/Permissions/PermissionCatalog.php`
- Create: `database/seeders/PermissionSeeder.php`
- Create: `database/seeders/AdminUserSeeder.php`
- Create: `app/Http/Controllers/Api/V1/Admin/UserController.php`
- Create: `app/Http/Controllers/Api/V1/Admin/RoleController.php`
- Create: `app/Http/Controllers/Api/V1/Admin/PermissionController.php`
- Create: `tests/Feature/Admin/RbacTest.php`

**Interfaces:**

- Produces roles: `super_admin`, `admin`, `operator`, `auditor`, `edge_service`.
- Produces permissions: `users.manage`, `permissions.manage`, `vehicles.manage`, `cameras.manage`, `captures.view`, `trips.manage`, `reports.view`, `edge.sync`.
- Produces admin endpoints for listing users, roles, and permissions.

**Steps:**

- [ ] Install `spatie/laravel-permission`.
- [ ] Publish and run package migrations.
- [ ] Write `RbacTest` for role-permission mapping, forbidden permission management by `operator`, and allowed access by `super_admin`.
- [ ] Run `php artisan test --filter=RbacTest` and confirm it fails because permissions are not implemented.
- [ ] Add `HasRoles` to `User`.
- [ ] Implement `PermissionCatalog`.
- [ ] Implement `PermissionSeeder` and `AdminUserSeeder`.
- [ ] Implement admin controllers and routes protected by permissions.
- [ ] Run `php artisan test --filter=RbacTest` and confirm it passes.
- [ ] Run full backend test suite with `php artisan test`.
- [ ] Commit backend as `feat: add role based access control`.

## Verification

- `php artisan test`
- `composer validate`
- `git status --short --branch`

## Handoff

After this phase, the next detailed phase plan should cover parent admin domain: vehicles, locations, edge nodes, cameras, and camera pairs.
