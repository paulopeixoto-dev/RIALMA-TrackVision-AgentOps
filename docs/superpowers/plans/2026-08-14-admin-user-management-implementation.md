# Admin User Management Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Completar a gestao administrativa de usuarios com criacao, edicao, ativacao/desativacao, reset de senha e atribuicao de roles existentes.

**Architecture:** O backend Laravel adiciona `is_active` em usuarios, bloqueia login de usuario inativo e expoe endpoints administrativos de escrita com Form Requests, Actions e Resources. O frontend Vue transforma `UsersPage` em uma tela CRUD usando services, formularios pequenos e modais, mantendo roles e permissoes como catalogos somente leitura.

**Tech Stack:** Laravel 13, PHP 8.4, Laravel Passport, Spatie Laravel Permission, Eloquent, Vue 3, Vite, TypeScript, Pinia, Vue Router, Vitest, Vue Test Utils.

## Global Constraints

- Backend must follow `docs/backend-laravel-guidelines.md`.
- Frontend must follow `docs/frontend-vue-guidelines.md`.
- Backend code lives only in `RIALMA-TrackVision-Backend`.
- Frontend code lives only in `RIALMA-TrackVision-Frontend`.
- AgentOps documentation lives only in `RIALMA-TrackVision-AgentOps`.
- Controllers devem ficar magros.
- Validacao deve ficar em Form Requests.
- Regras de negocio devem ficar em Actions/Services.
- Eloquent deve carregar relacoes necessarias para evitar N+1.
- Toda escrita de usuario exige `users.manage`.
- Listagem de usuarios exige `users.manage`.
- Listagem de roles deve permitir `users.manage` ou `permissions.manage`.
- Listagem de permissoes continua exigindo `permissions.manage`.
- Roles e permissoes continuam controladas por seed/catalogo nesta fase.
- Senha nunca deve ser retornada em Resource.
- Usuario inativo nao pode autenticar.
- Usuario autenticado nao pode desativar a si mesmo.
- Usuario autenticado nao pode remover de si mesmo acesso efetivo a `users.manage`.

---

## File Structure Map

Backend files:

```text
RIALMA-TrackVision-Backend/
|-- app/
|   |-- Actions/Admin/Users/
|   |   |-- CreateUserAction.php
|   |   |-- DeactivateUserAction.php
|   |   |-- ResetUserPasswordAction.php
|   |   |-- SyncUserRolesAction.php
|   |   `-- UpdateUserAction.php
|   |-- Http/
|   |   |-- Controllers/Api/V1/Admin/UserController.php
|   |   |-- Controllers/Api/V1/Auth/LoginController.php
|   |   |-- Controllers/Api/V1/Auth/MeController.php
|   |   |-- Requests/Api/V1/Admin/ResetUserPasswordRequest.php
|   |   |-- Requests/Api/V1/Admin/StoreUserRequest.php
|   |   |-- Requests/Api/V1/Admin/UpdateUserRequest.php
|   |   `-- Resources/Api/V1/UserResource.php
|   |-- Models/User.php
|   `-- Support/Permissions/PermissionCatalog.php
|-- database/
|   |-- factories/UserFactory.php
|   `-- migrations/2026_08_14_090001_add_is_active_to_users_table.php
|-- docs/api-parent-admin.md
|-- routes/api.php
`-- tests/Feature/
    |-- Admin/UserAdminTest.php
    `-- Auth/LoginTest.php
```

Frontend files:

```text
RIALMA-TrackVision-Frontend/
|-- README.md
`-- src/
    |-- components/forms/
    |   |-- UserForm.test.ts
    |   |-- UserForm.vue
    |   |-- UserPasswordForm.test.ts
    |   `-- UserPasswordForm.vue
    |-- pages/
    |   |-- UsersPage.test.ts
    |   `-- UsersPage.vue
    |-- services/
    |   |-- usersService.test.ts
    |   `-- usersService.ts
    `-- types/
        |-- admin.ts
        `-- auth.ts
```

## Execution Setup

- Backend execution branch: `codex/admin-user-management-backend`.
- Frontend execution branch: `codex/admin-user-management-frontend`.
- Use isolated worktrees through `superpowers:using-git-worktrees` before implementation.
- Backend baseline before Task 1:

```bash
composer validate
php artisan test
```

- Frontend baseline before Task 3:

```bash
npm test -- --run
npm run build
```

- If baseline fails, stop and report the exact command, exit code, and failing output.

---

## Task 1: Backend Active Users, Login Guard, And User Resource Contract

**Files:**

- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_14_090001_add_is_active_to_users_table.php`
- Modify: `RIALMA-TrackVision-Backend/app/Models/User.php`
- Modify: `RIALMA-TrackVision-Backend/database/factories/UserFactory.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Auth/LoginController.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Auth/MeController.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/UserResource.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Auth/LoginTest.php`

**Interfaces:**

- Produces: `users.is_active` boolean with default `true`.
- Produces: `UserResource` fields `uuid`, `is_active`, `roles`, `permissions`, `created_at`, `updated_at`.
- Produces: inactive users rejected by `POST /api/v1/auth/login`.
- Consumes: existing `PermissionSeeder`, Spatie roles, Passport personal access client.

- [ ] **Step 1: Write failing tests for inactive login and enriched user response**

Modify `tests/Feature/Auth/LoginTest.php` imports:

```php
use Database\Seeders\PermissionSeeder;
```

Append these tests:

```php
public function test_inactive_user_cannot_login(): void
{
    $this->createPersonalAccessClient();

    User::factory()->create([
        'email' => 'inactive@trackvision.test',
        'password' => 'secret-password',
        'is_active' => false,
    ]);

    $this->postJson('/api/v1/auth/login', [
        'email' => 'inactive@trackvision.test',
        'password' => 'secret-password',
    ])
        ->assertUnprocessable()
        ->assertJsonValidationErrors(['email']);
}

public function test_login_and_me_return_user_status_roles_and_permissions(): void
{
    $this->seed(PermissionSeeder::class);
    $this->createPersonalAccessClient();

    $user = User::factory()->create([
        'email' => 'admin@trackvision.test',
        'password' => 'secret-password',
        'is_active' => true,
    ]);
    $user->assignRole('admin');

    $response = $this->postJson('/api/v1/auth/login', [
        'email' => 'admin@trackvision.test',
        'password' => 'secret-password',
    ])
        ->assertOk()
        ->assertJsonPath('user.is_active', true)
        ->assertJsonPath('user.roles.0', 'admin');

    $this->assertContains('users.manage', $response->json('user.permissions'));

    $token = $response->json('access_token');

    $meResponse = $this->withToken($token)
        ->getJson('/api/v1/me')
        ->assertOk()
        ->assertJsonPath('data.is_active', true)
        ->assertJsonPath('data.roles.0', 'admin');

    $this->assertContains('users.manage', $meResponse->json('data.permissions'));
}
```

- [ ] **Step 2: Run tests and verify RED**

Run:

```bash
php artisan test tests/Feature/Auth/LoginTest.php --filter='inactive_user_cannot_login|login_and_me_return_user_status_roles_and_permissions'
```

Expected: FAIL because `is_active` does not exist, login does not check status, and `UserResource` does not expose status/permissions.

- [ ] **Step 3: Add user active migration**

Create `database/migrations/2026_08_14_090001_add_is_active_to_users_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table): void {
            $table->boolean('is_active')->default(true)->after('password')->index();
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table): void {
            $table->dropColumn('is_active');
        });
    }
};
```

- [ ] **Step 4: Update User model and factory**

Modify `app/Models/User.php` fillable:

```php
#[Fillable(['uuid', 'name', 'email', 'password', 'is_active'])]
```

Modify `app/Models/User.php` casts:

```php
protected function casts(): array
{
    return [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
        'is_active' => 'boolean',
    ];
}
```

Modify `database/factories/UserFactory.php` default state:

```php
return [
    'name' => fake()->name(),
    'email' => fake()->unique()->safeEmail(),
    'email_verified_at' => now(),
    'password' => static::$password ??= Hash::make('password'),
    'is_active' => true,
    'remember_token' => Str::random(10),
];
```

- [ ] **Step 5: Update login and current user controllers**

Modify `app/Http/Controllers/Api/V1/Auth/LoginController.php` credential check:

```php
if (! $user || ! $user->is_active || ! Hash::check($credentials['password'], $user->password)) {
    throw ValidationException::withMessages([
        'email' => ['The provided credentials are incorrect.'],
    ]);
}

$user->load('roles.permissions');
```

Keep token creation unchanged. The JSON response still uses:

```php
'user' => UserResource::make($user)->resolve(),
```

Modify `app/Http/Controllers/Api/V1/Auth/MeController.php`:

```php
public function __invoke(Request $request): UserResource
{
    return UserResource::make($request->user()->load('roles.permissions'));
}
```

- [ ] **Step 6: Update UserResource contract**

Modify `app/Http/Resources/Api/V1/UserResource.php` response array:

```php
return [
    'id' => $this->id,
    'uuid' => $this->uuid,
    'name' => $this->name,
    'email' => $this->email,
    'is_active' => $this->is_active,
    'roles' => $this->whenLoaded('roles', fn () => $this->roles->pluck('name')->values()),
    'permissions' => $this->when(
        $this->relationLoaded('roles'),
        fn () => $this->getAllPermissions()->pluck('name')->unique()->values(),
    ),
    'created_at' => $this->created_at?->toJSON(),
    'updated_at' => $this->updated_at?->toJSON(),
];
```

- [ ] **Step 7: Run tests and verify GREEN**

Run:

```bash
php artisan test tests/Feature/Auth/LoginTest.php --filter='inactive_user_cannot_login|login_and_me_return_user_status_roles_and_permissions'
```

Expected: PASS.

- [ ] **Step 8: Run full auth tests**

Run:

```bash
php artisan test tests/Feature/Auth/LoginTest.php
```

Expected: PASS.

- [ ] **Step 9: Commit backend active user contract**

Run:

```bash
git add app/Models/User.php database/factories/UserFactory.php database/migrations/2026_08_14_090001_add_is_active_to_users_table.php app/Http/Controllers/Api/V1/Auth/LoginController.php app/Http/Controllers/Api/V1/Auth/MeController.php app/Http/Resources/Api/V1/UserResource.php tests/Feature/Auth/LoginTest.php
git commit -m "feat: guard inactive admin users"
```

---

## Task 2: Backend Admin User Management API

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Actions/Admin/Users/CreateUserAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Admin/Users/DeactivateUserAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Admin/Users/ResetUserPasswordAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Admin/Users/SyncUserRolesAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Admin/Users/UpdateUserAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/ResetUserPasswordRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/StoreUserRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/UpdateUserRequest.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Admin/UserAdminTest.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/UserController.php`
- Modify: `RIALMA-TrackVision-Backend/routes/api.php`
- Modify: `RIALMA-TrackVision-Backend/tests/Feature/Admin/RbacTest.php`

**Interfaces:**

- Consumes: `UserResource` from Task 1.
- Produces: `POST /api/v1/admin/users`.
- Produces: `GET /api/v1/admin/users/{user}`.
- Produces: `PATCH /api/v1/admin/users/{user}`.
- Produces: `PATCH /api/v1/admin/users/{user}/password`.
- Produces: `DELETE /api/v1/admin/users/{user}` as logical deactivation.
- Produces: `GET /api/v1/admin/roles` accessible by `users.manage` or `permissions.manage`.

- [ ] **Step 1: Write failing feature tests**

Create `tests/Feature/Admin/UserAdminTest.php`:

```php
<?php

namespace Tests\Feature\Admin;

use App\Models\User;
use Database\Seeders\PermissionSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Hash;
use Laravel\Passport\Passport;
use Tests\TestCase;

class UserAdminTest extends TestCase
{
    use RefreshDatabase;

    public function test_admin_can_create_user_with_roles(): void
    {
        $this->actingAdmin();

        $this->postJson('/api/v1/admin/users', [
            'name' => 'Operador Patio',
            'email' => 'operador@trackvision.test',
            'password' => 'secret-password',
            'password_confirmation' => 'secret-password',
            'is_active' => true,
            'roles' => ['operator'],
        ])
            ->assertCreated()
            ->assertJsonPath('data.name', 'Operador Patio')
            ->assertJsonPath('data.email', 'operador@trackvision.test')
            ->assertJsonPath('data.is_active', true)
            ->assertJsonPath('data.roles.0', 'operator');

        $user = User::query()->where('email', 'operador@trackvision.test')->firstOrFail();
        $this->assertTrue(Hash::check('secret-password', $user->password));
        $this->assertTrue($user->hasRole('operator'));
    }

    public function test_create_user_rejects_duplicate_email(): void
    {
        $this->actingAdmin();
        User::factory()->create(['email' => 'duplicado@trackvision.test']);

        $this->postJson('/api/v1/admin/users', [
            'name' => 'Duplicado',
            'email' => 'duplicado@trackvision.test',
            'password' => 'secret-password',
            'password_confirmation' => 'secret-password',
            'roles' => ['operator'],
        ])
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['email']);
    }

    public function test_create_user_rejects_unknown_role(): void
    {
        $this->actingAdmin();

        $this->postJson('/api/v1/admin/users', [
            'name' => 'Operador',
            'email' => 'operador@trackvision.test',
            'password' => 'secret-password',
            'password_confirmation' => 'secret-password',
            'roles' => ['role_inexistente'],
        ])
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['roles.0']);
    }

    public function test_admin_can_update_user_and_roles(): void
    {
        $this->actingAdmin();
        $user = User::factory()->create(['name' => 'Nome Antigo', 'email' => 'antigo@trackvision.test']);
        $user->assignRole('auditor');

        $response = $this->patchJson("/api/v1/admin/users/{$user->id}", [
            'name' => 'Nome Novo',
            'email' => 'novo@trackvision.test',
            'is_active' => true,
            'roles' => ['operator', 'auditor'],
        ])
            ->assertOk()
            ->assertJsonPath('data.name', 'Nome Novo')
            ->assertJsonPath('data.email', 'novo@trackvision.test')
            ->assertJsonPath('data.is_active', true);

        $this->assertContains('operator', $response->json('data.roles'));
        $this->assertContains('auditor', $response->json('data.roles'));
        $this->assertTrue($user->refresh()->hasAllRoles(['operator', 'auditor']));
    }

    public function test_admin_can_reset_user_password(): void
    {
        $this->actingAdmin();
        $user = User::factory()->create(['password' => 'old-password']);

        $this->patchJson("/api/v1/admin/users/{$user->id}/password", [
            'password' => 'new-password',
            'password_confirmation' => 'new-password',
        ])->assertOk();

        $this->assertTrue(Hash::check('new-password', $user->refresh()->password));
        $this->assertFalse(Hash::check('old-password', $user->password));
    }

    public function test_admin_can_deactivate_user_without_deleting_record(): void
    {
        $this->actingAdmin();
        $user = User::factory()->create(['is_active' => true]);

        $this->deleteJson("/api/v1/admin/users/{$user->id}")
            ->assertNoContent();

        $this->assertFalse($user->refresh()->is_active);
        $this->assertDatabaseHas('users', ['id' => $user->id]);
    }

    public function test_admin_cannot_deactivate_self(): void
    {
        $admin = $this->actingAdmin();

        $this->deleteJson("/api/v1/admin/users/{$admin->id}")
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['is_active']);

        $this->assertTrue($admin->refresh()->is_active);
    }

    public function test_admin_cannot_remove_own_users_manage_access(): void
    {
        $admin = $this->actingAdmin();

        $this->patchJson("/api/v1/admin/users/{$admin->id}", [
            'name' => $admin->name,
            'email' => $admin->email,
            'is_active' => true,
            'roles' => ['auditor'],
        ])
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['roles']);

        $this->assertTrue($admin->refresh()->can('users.manage'));
    }

    public function test_user_without_users_manage_cannot_write_users(): void
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('auditor');
        Passport::actingAs($user, ['admin:read']);

        $this->postJson('/api/v1/admin/users', [
            'name' => 'Bloqueado',
            'email' => 'bloqueado@trackvision.test',
            'password' => 'secret-password',
            'password_confirmation' => 'secret-password',
            'roles' => ['auditor'],
        ])->assertForbidden();
    }

    public function test_user_manager_can_list_roles_for_assignment(): void
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('admin');
        Passport::actingAs($user, ['admin:read']);

        $this->getJson('/api/v1/admin/roles')
            ->assertOk()
            ->assertJsonFragment(['name' => 'operator']);
    }

    private function actingAdmin(): User
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('super_admin');
        Passport::actingAs($user, ['admin:read']);

        return $user;
    }
}
```

- [ ] **Step 2: Run user admin tests and verify RED**

Run:

```bash
php artisan test tests/Feature/Admin/UserAdminTest.php
```

Expected: FAIL because routes, requests, actions and user write endpoints do not exist.

- [ ] **Step 3: Create admin user Form Requests**

Create `app/Http/Requests/Api/V1/Admin/StoreUserRequest.php`:

```php
<?php

namespace App\Http\Requests\Api\V1\Admin;

use App\Support\Permissions\PermissionCatalog;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('users.manage') === true;
    }

    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'max:255', Rule::unique('users', 'email')],
            'password' => ['required', 'string', 'confirmed', 'min:8'],
            'is_active' => ['sometimes', 'boolean'],
            'roles' => ['sometimes', 'array'],
            'roles.*' => [
                'string',
                'distinct',
                Rule::exists('roles', 'name')->where('guard_name', PermissionCatalog::GUARD),
            ],
        ];
    }
}
```

Create `app/Http/Requests/Api/V1/Admin/UpdateUserRequest.php`:

```php
<?php

namespace App\Http\Requests\Api\V1\Admin;

use App\Support\Permissions\PermissionCatalog;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdateUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('users.manage') === true;
    }

    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        $userId = $this->route('user')?->getKey();

        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'max:255', Rule::unique('users', 'email')->ignore($userId)],
            'is_active' => ['required', 'boolean'],
            'roles' => ['required', 'array'],
            'roles.*' => [
                'string',
                'distinct',
                Rule::exists('roles', 'name')->where('guard_name', PermissionCatalog::GUARD),
            ],
        ];
    }
}
```

Create `app/Http/Requests/Api/V1/Admin/ResetUserPasswordRequest.php`:

```php
<?php

namespace App\Http\Requests\Api\V1\Admin;

use Illuminate\Foundation\Http\FormRequest;

class ResetUserPasswordRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('users.manage') === true;
    }

    /**
     * @return array<string, list<string>>
     */
    public function rules(): array
    {
        return [
            'password' => ['required', 'string', 'confirmed', 'min:8'],
        ];
    }
}
```

- [ ] **Step 4: Create user management Actions**

Create `app/Actions/Admin/Users/SyncUserRolesAction.php`:

```php
<?php

namespace App\Actions\Admin\Users;

use App\Models\User;
use App\Support\Permissions\PermissionCatalog;
use Illuminate\Support\Collection;
use Illuminate\Validation\ValidationException;
use Spatie\Permission\Models\Role;

class SyncUserRolesAction
{
    /**
     * @param list<string> $roleNames
     */
    public function execute(User $target, array $roleNames, User $actor): void
    {
        $roles = Role::query()
            ->where('guard_name', PermissionCatalog::GUARD)
            ->whereIn('name', $roleNames)
            ->with('permissions')
            ->get();

        if ($roles->count() !== count(array_unique($roleNames))) {
            throw ValidationException::withMessages([
                'roles' => ['Uma ou mais roles informadas nao existem.'],
            ]);
        }

        if ($actor->is($target) && ! $this->rolesGrantUsersManage($roles)) {
            throw ValidationException::withMessages([
                'roles' => ['Voce nao pode remover seu proprio acesso a gestao de usuarios.'],
            ]);
        }

        $target->syncRoles($roles);
    }

    /**
     * @param Collection<int, Role> $roles
     */
    private function rolesGrantUsersManage(Collection $roles): bool
    {
        return $roles->contains(
            fn (Role $role): bool => $role->name === PermissionCatalog::SUPER_ADMIN
                || $role->permissions->contains('name', 'users.manage'),
        );
    }
}
```

Create `app/Actions/Admin/Users/CreateUserAction.php`:

```php
<?php

namespace App\Actions\Admin\Users;

use App\Models\User;
use Illuminate\Support\Facades\DB;

class CreateUserAction
{
    public function __construct(private readonly SyncUserRolesAction $syncRoles) {}

    /**
     * @param array<string, mixed> $data
     */
    public function execute(array $data, User $actor): User
    {
        return DB::transaction(function () use ($data, $actor): User {
            $roles = $data['roles'] ?? [];

            $user = User::query()->create([
                'name' => $data['name'],
                'email' => $data['email'],
                'password' => $data['password'],
                'is_active' => $data['is_active'] ?? true,
            ]);

            $this->syncRoles->execute($user, $roles, $actor);

            return $user->refresh()->load('roles.permissions');
        });
    }
}
```

Create `app/Actions/Admin/Users/UpdateUserAction.php`:

```php
<?php

namespace App\Actions\Admin\Users;

use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Validation\ValidationException;

class UpdateUserAction
{
    public function __construct(private readonly SyncUserRolesAction $syncRoles) {}

    /**
     * @param array<string, mixed> $data
     */
    public function execute(User $target, array $data, User $actor): User
    {
        return DB::transaction(function () use ($target, $data, $actor): User {
            if ($actor->is($target) && $data['is_active'] === false) {
                throw ValidationException::withMessages([
                    'is_active' => ['Voce nao pode desativar seu proprio usuario.'],
                ]);
            }

            $target->update([
                'name' => $data['name'],
                'email' => $data['email'],
                'is_active' => $data['is_active'],
            ]);

            $this->syncRoles->execute($target, $data['roles'], $actor);

            return $target->refresh()->load('roles.permissions');
        });
    }
}
```

Create `app/Actions/Admin/Users/ResetUserPasswordAction.php`:

```php
<?php

namespace App\Actions\Admin\Users;

use App\Models\User;

class ResetUserPasswordAction
{
    public function execute(User $target, string $password): User
    {
        $target->update(['password' => $password]);

        return $target->refresh()->load('roles.permissions');
    }
}
```

Create `app/Actions/Admin/Users/DeactivateUserAction.php`:

```php
<?php

namespace App\Actions\Admin\Users;

use App\Models\User;
use Illuminate\Validation\ValidationException;

class DeactivateUserAction
{
    public function execute(User $target, User $actor): void
    {
        if ($actor->is($target)) {
            throw ValidationException::withMessages([
                'is_active' => ['Voce nao pode desativar seu proprio usuario.'],
            ]);
        }

        $target->update(['is_active' => false]);
    }
}
```

- [ ] **Step 5: Expand UserController**

Replace `app/Http/Controllers/Api/V1/Admin/UserController.php` with:

```php
<?php

namespace App\Http\Controllers\Api\V1\Admin;

use App\Actions\Admin\Users\CreateUserAction;
use App\Actions\Admin\Users\DeactivateUserAction;
use App\Actions\Admin\Users\ResetUserPasswordAction;
use App\Actions\Admin\Users\UpdateUserAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Api\V1\Admin\ResetUserPasswordRequest;
use App\Http\Requests\Api\V1\Admin\StoreUserRequest;
use App\Http\Requests\Api\V1\Admin\UpdateUserRequest;
use App\Http\Resources\Api\V1\UserResource;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class UserController extends Controller
{
    public function index(): AnonymousResourceCollection
    {
        return UserResource::collection(
            User::query()
                ->with('roles.permissions')
                ->orderBy('name')
                ->paginate(),
        );
    }

    public function store(StoreUserRequest $request, CreateUserAction $action): JsonResponse
    {
        $user = $action->execute($request->validated(), $request->user());

        return UserResource::make($user)
            ->response()
            ->setStatusCode(Response::HTTP_CREATED);
    }

    public function show(User $user): UserResource
    {
        return UserResource::make($user->load('roles.permissions'));
    }

    public function update(UpdateUserRequest $request, User $user, UpdateUserAction $action): UserResource
    {
        return UserResource::make(
            $action->execute($user, $request->validated(), $request->user()),
        );
    }

    public function password(ResetUserPasswordRequest $request, User $user, ResetUserPasswordAction $action): UserResource
    {
        return UserResource::make(
            $action->execute($user, $request->validated('password')),
        );
    }

    public function destroy(Request $request, User $user, DeactivateUserAction $action): JsonResponse
    {
        $action->execute($user, $request->user());

        return response()->noContent();
    }
}
```

- [ ] **Step 6: Update admin routes**

Modify `routes/api.php` admin group.

Replace the existing users route:

```php
Route::get('/users', [UserController::class, 'index'])
    ->middleware('permission:users.manage,api');
```

with:

```php
Route::patch('/users/{user}/password', [UserController::class, 'password'])
    ->middleware('permission:users.manage,api');

Route::apiResource('users', UserController::class)
    ->middleware('permission:users.manage,api');
```

Replace the existing roles route middleware:

```php
Route::get('/roles', [RoleController::class, 'index'])
    ->middleware('permission:permissions.manage,api');
```

with:

```php
Route::get('/roles', [RoleController::class, 'index'])
    ->middleware('permission:permissions.manage|users.manage,api');
```

- [ ] **Step 7: Update existing RBAC test expectation**

Modify `tests/Feature/Admin/RbacTest.php` by appending:

```php
public function test_user_manager_can_list_roles_but_not_permissions(): void
{
    $this->seed(PermissionSeeder::class);
    $user = User::factory()->create();
    $user->assignRole('admin');

    Passport::actingAs($user, ['admin:read']);

    $this->getJson('/api/v1/admin/roles')
        ->assertOk()
        ->assertJsonFragment(['name' => 'operator']);

    $this->getJson('/api/v1/admin/permissions')
        ->assertForbidden();
}
```

- [ ] **Step 8: Run user admin tests and verify GREEN**

Run:

```bash
php artisan test tests/Feature/Admin/UserAdminTest.php tests/Feature/Admin/RbacTest.php
```

Expected: PASS.

- [ ] **Step 9: Run backend admin/auth focused tests**

Run:

```bash
php artisan test tests/Feature/Admin/UserAdminTest.php tests/Feature/Admin/RbacTest.php tests/Feature/Auth/LoginTest.php
```

Expected: PASS.

- [ ] **Step 10: Commit backend admin user API**

Run:

```bash
git add app/Actions/Admin/Users app/Http/Requests/Api/V1/Admin/StoreUserRequest.php app/Http/Requests/Api/V1/Admin/UpdateUserRequest.php app/Http/Requests/Api/V1/Admin/ResetUserPasswordRequest.php app/Http/Controllers/Api/V1/Admin/UserController.php routes/api.php tests/Feature/Admin/UserAdminTest.php tests/Feature/Admin/RbacTest.php
git commit -m "feat: manage admin users"
```

---

## Task 3: Frontend User Services, Types, And Forms

**Files:**

- Create: `RIALMA-TrackVision-Frontend/src/components/forms/UserForm.vue`
- Create: `RIALMA-TrackVision-Frontend/src/components/forms/UserForm.test.ts`
- Create: `RIALMA-TrackVision-Frontend/src/components/forms/UserPasswordForm.vue`
- Create: `RIALMA-TrackVision-Frontend/src/components/forms/UserPasswordForm.test.ts`
- Create: `RIALMA-TrackVision-Frontend/src/services/usersService.test.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/services/usersService.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/types/auth.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/types/admin.ts`

**Interfaces:**

- Consumes: backend user endpoints from Task 2.
- Produces: `createUsersService(options)`.
- Produces: `usersService.create`, `show`, `update`, `resetPassword`, `deactivate`.
- Produces: `CreateUserInput`, `UpdateUserInput`, `ResetUserPasswordInput`, `UserFormModel`.
- Produces: `UserForm` and `UserPasswordForm`.

- [ ] **Step 1: Write failing service tests**

Create `src/services/usersService.test.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { createUsersService } from './usersService'

describe('usersService', () => {
  const fetchMock = vi.fn()

  beforeEach(() => {
    fetchMock.mockReset()
    vi.stubGlobal('fetch', fetchMock)
    localStorage.clear()
  })

  it('creates a user with roles and active status', async () => {
    fetchMock.mockResolvedValueOnce(new Response(JSON.stringify({
      data: { id: 2, uuid: 'user-uuid', name: 'Operador', email: 'op@test.local', is_active: true, roles: ['operator'] },
    }), { status: 201 }))

    const service = createUsersService({ apiBaseUrl: 'http://api.test', getToken: () => 'token-123' })

    await service.create({
      name: 'Operador',
      email: 'op@test.local',
      password: 'secret-password',
      password_confirmation: 'secret-password',
      is_active: true,
      roles: ['operator'],
    })

    const [url, init] = fetchMock.mock.calls[0]
    expect(url).toBe('http://api.test/admin/users')
    expect(init.method).toBe('POST')
    expect(init.body).toBe(JSON.stringify({
      name: 'Operador',
      email: 'op@test.local',
      password: 'secret-password',
      password_confirmation: 'secret-password',
      is_active: true,
      roles: ['operator'],
    }))
    expect((init.headers as Headers).get('Authorization')).toBe('Bearer token-123')
  })

  it('updates a user with roles as names', async () => {
    fetchMock.mockResolvedValueOnce(new Response(JSON.stringify({
      data: { id: 2, uuid: 'user-uuid', name: 'Operador Editado', email: 'op@test.local', is_active: false, roles: ['auditor'] },
    }), { status: 200 }))

    const service = createUsersService({ apiBaseUrl: 'http://api.test', getToken: () => null })

    await service.update({ id: 2 } as never, {
      name: 'Operador Editado',
      email: 'op@test.local',
      is_active: false,
      roles: ['auditor'],
    })

    const [url, init] = fetchMock.mock.calls[0]
    expect(url).toBe('http://api.test/admin/users/2')
    expect(init.method).toBe('PATCH')
    expect(init.body).toBe(JSON.stringify({
      name: 'Operador Editado',
      email: 'op@test.local',
      is_active: false,
      roles: ['auditor'],
    }))
  })

  it('resets a user password and deactivates a user', async () => {
    fetchMock
      .mockResolvedValueOnce(new Response(JSON.stringify({
        data: { id: 2, uuid: 'user-uuid', name: 'Operador', email: 'op@test.local', is_active: true, roles: ['operator'] },
      }), { status: 200 }))
      .mockResolvedValueOnce(new Response(null, { status: 204 }))

    const service = createUsersService({ apiBaseUrl: 'http://api.test', getToken: () => null })

    await service.resetPassword({ id: 2 } as never, {
      password: 'new-password',
      password_confirmation: 'new-password',
    })
    await service.deactivate({ id: 2 } as never)

    expect(fetchMock.mock.calls[0][0]).toBe('http://api.test/admin/users/2/password')
    expect(fetchMock.mock.calls[0][1].method).toBe('PATCH')
    expect(fetchMock.mock.calls[1][0]).toBe('http://api.test/admin/users/2')
    expect(fetchMock.mock.calls[1][1].method).toBe('DELETE')
  })
})
```

- [ ] **Step 2: Write failing form tests**

Create `src/components/forms/UserForm.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import UserForm from './UserForm.vue'

const roles = [
  { id: 1, name: 'operator', permissions: ['trips.manage'] },
  { id: 2, name: 'auditor', permissions: ['reports.view'] },
]

describe('UserForm', () => {
  it('emits create payload with selected roles and password fields', async () => {
    const wrapper = mount(UserForm, {
      props: {
        mode: 'create',
        modelValue: {
          name: '',
          email: '',
          password: '',
          password_confirmation: '',
          is_active: true,
          roles: [],
        },
        roles,
        errors: {},
        submitting: false,
      },
    })

    await wrapper.get('input[name="name"]').setValue('Operador')
    await wrapper.get('input[name="email"]').setValue('op@test.local')
    await wrapper.get('input[name="password"]').setValue('secret-password')
    await wrapper.get('input[name="password_confirmation"]').setValue('secret-password')
    await wrapper.get('input[value="operator"]').setValue(true)

    expect(wrapper.emitted('update:modelValue')?.at(-1)?.[0]).toMatchObject({
      roles: ['operator'],
    })
  })

  it('hides password fields in edit mode and shows validation errors', () => {
    const wrapper = mount(UserForm, {
      props: {
        mode: 'edit',
        modelValue: {
          name: 'Paulo',
          email: 'paulo@test.local',
          is_active: true,
          roles: ['auditor'],
        },
        roles,
        errors: { email: ['Email invalido.'] },
        submitting: false,
      },
    })

    expect(wrapper.find('input[name="password"]').exists()).toBe(false)
    expect(wrapper.text()).toContain('Email invalido.')
    expect((wrapper.get('input[value="auditor"]').element as HTMLInputElement).checked).toBe(true)
  })
})
```

Create `src/components/forms/UserPasswordForm.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import UserPasswordForm from './UserPasswordForm.vue'

describe('UserPasswordForm', () => {
  it('emits password fields and submit event', async () => {
    const wrapper = mount(UserPasswordForm, {
      props: {
        modelValue: { password: '', password_confirmation: '' },
        errors: {},
        submitting: false,
      },
    })

    await wrapper.get('input[name="password"]').setValue('new-password')
    await wrapper.setProps({
      modelValue: { password: 'new-password', password_confirmation: '' },
    })
    await wrapper.get('input[name="password_confirmation"]').setValue('new-password')
    await wrapper.get('form').trigger('submit')

    expect(wrapper.emitted('update:modelValue')?.at(-1)?.[0]).toEqual({
      password: 'new-password',
      password_confirmation: 'new-password',
    })
    expect(wrapper.emitted('submit')).toHaveLength(1)
  })
})
```

- [ ] **Step 3: Run frontend focused tests and verify RED**

Run:

```bash
npm test -- --run src/services/usersService.test.ts src/components/forms/UserForm.test.ts src/components/forms/UserPasswordForm.test.ts
```

Expected: FAIL because service factory and forms do not exist.

- [ ] **Step 4: Extend frontend user types**

Modify `src/types/auth.ts` `User`:

```ts
export interface User {
  id: number
  uuid?: string
  name: string
  email: string
  is_active?: boolean
  roles?: string[]
  permissions?: string[]
  created_at?: string | null
  updated_at?: string | null
}
```

Modify `src/types/admin.ts` imports:

```ts
import type { User } from './auth'
```

Add user input types near `Role`:

```ts
export interface CreateUserInput {
  name: string
  email: string
  password: string
  password_confirmation: string
  is_active: boolean
  roles: string[]
}

export interface UpdateUserInput {
  name: string
  email: string
  is_active: boolean
  roles: string[]
}

export interface ResetUserPasswordInput {
  password: string
  password_confirmation: string
}

export type UserFormModel = CreateUserInput | UpdateUserInput
export type AdminUser = User
```

- [ ] **Step 5: Expand usersService**

Replace `src/services/usersService.ts` with:

```ts
import { getAppConfig } from '@/app/config'
import type { CreateUserInput, ResetUserPasswordInput, UpdateUserInput } from '@/types/admin'
import type { LaravelPaginated, LaravelResource } from '@/types/api'
import type { User } from '@/types/auth'
import { createApiClient, type ApiClientOptions } from './apiClient'

export function createUsersService(options: ApiClientOptions) {
  const client = createApiClient(options)

  return {
    list(): Promise<LaravelPaginated<User>> {
      return client.get<LaravelPaginated<User>>('/admin/users')
    },

    async create(input: CreateUserInput): Promise<User> {
      const response = await client.post<LaravelResource<User>>('/admin/users', input)
      return response.data
    },

    async show(user: Pick<User, 'id'>): Promise<User> {
      const response = await client.get<LaravelResource<User>>(`/admin/users/${user.id}`)
      return response.data
    },

    async update(user: Pick<User, 'id'>, input: UpdateUserInput): Promise<User> {
      const response = await client.patch<LaravelResource<User>>(`/admin/users/${user.id}`, input)
      return response.data
    },

    async resetPassword(user: Pick<User, 'id'>, input: ResetUserPasswordInput): Promise<User> {
      const response = await client.patch<LaravelResource<User>>(`/admin/users/${user.id}/password`, input)
      return response.data
    },

    deactivate(user: Pick<User, 'id'>): Promise<void> {
      return client.delete<void>(`/admin/users/${user.id}`)
    },
  }
}

export const usersService = createUsersService({
  apiBaseUrl: getAppConfig().apiBaseUrl,
  getToken: () => localStorage.getItem('trackvision.token'),
})
```

- [ ] **Step 6: Create UserForm**

Create `src/components/forms/UserForm.vue`:

```vue
<script setup lang="ts">
import BaseButton from '@/components/base/BaseButton.vue'
import BaseInput from '@/components/base/BaseInput.vue'
import type { Role, UserFormModel } from '@/types/admin'
import type { FieldErrors } from '@/types/api'

const props = defineProps<{
  mode: 'create' | 'edit'
  modelValue: UserFormModel
  roles: Role[]
  errors: FieldErrors
  submitting: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: UserFormModel]
  submit: []
  cancel: []
}>()

function updateField(key: string, value: string | boolean | string[]): void {
  emit('update:modelValue', { ...props.modelValue, [key]: value } as UserFormModel)
}

function updateActive(event: Event): void {
  updateField('is_active', (event.target as HTMLInputElement).checked)
}

function toggleRole(roleName: string, event: Event): void {
  const checked = (event.target as HTMLInputElement).checked
  const currentRoles = props.modelValue.roles
  const roles = checked
    ? [...new Set([...currentRoles, roleName])]
    : currentRoles.filter((name) => name !== roleName)

  updateField('roles', roles)
}
</script>

<template>
  <form
    class="entity-form"
    @submit.prevent="$emit('submit')"
  >
    <BaseInput
      :error="errors.name"
      label="Nome"
      :model-value="modelValue.name"
      name="name"
      @update:model-value="updateField('name', $event)"
    />
    <BaseInput
      :error="errors.email"
      label="Email"
      :model-value="modelValue.email"
      name="email"
      type="email"
      autocomplete="email"
      @update:model-value="updateField('email', $event)"
    />
    <template v-if="mode === 'create'">
      <BaseInput
        :error="errors.password"
        label="Senha inicial"
        :model-value="'password' in modelValue ? modelValue.password : ''"
        name="password"
        type="password"
        autocomplete="new-password"
        @update:model-value="updateField('password', $event)"
      />
      <BaseInput
        :error="errors.password_confirmation"
        label="Confirmar senha"
        :model-value="'password_confirmation' in modelValue ? modelValue.password_confirmation : ''"
        name="password_confirmation"
        type="password"
        autocomplete="new-password"
        @update:model-value="updateField('password_confirmation', $event)"
      />
    </template>

    <label class="checkbox-field">
      <input
        :checked="modelValue.is_active"
        name="is_active"
        type="checkbox"
        @change="updateActive"
      >
      <span>Ativo</span>
    </label>

    <fieldset class="entity-fieldset">
      <legend>Roles</legend>
      <p
        v-if="errors.roles"
        class="field-error"
      >
        {{ Array.isArray(errors.roles) ? errors.roles[0] : errors.roles }}
      </p>
      <label
        v-for="role in roles"
        :key="role.id"
        class="checkbox-field"
      >
        <input
          :checked="modelValue.roles.includes(role.name)"
          :value="role.name"
          type="checkbox"
          @change="toggleRole(role.name, $event)"
        >
        <span>{{ role.name }}</span>
      </label>
    </fieldset>

    <div class="form-actions">
      <BaseButton
        :loading="submitting"
        type="submit"
      >
        Salvar
      </BaseButton>
      <BaseButton
        type="button"
        variant="ghost"
        @click="$emit('cancel')"
      >
        Cancelar
      </BaseButton>
    </div>
  </form>
</template>
```

- [ ] **Step 7: Create UserPasswordForm**

Create `src/components/forms/UserPasswordForm.vue`:

```vue
<script setup lang="ts">
import BaseButton from '@/components/base/BaseButton.vue'
import BaseInput from '@/components/base/BaseInput.vue'
import type { ResetUserPasswordInput } from '@/types/admin'
import type { FieldErrors } from '@/types/api'

const props = defineProps<{
  modelValue: ResetUserPasswordInput
  errors: FieldErrors
  submitting: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: ResetUserPasswordInput]
  submit: []
  cancel: []
}>()

function updateField<K extends keyof ResetUserPasswordInput>(key: K, value: ResetUserPasswordInput[K]): void {
  emit('update:modelValue', { ...props.modelValue, [key]: value })
}
</script>

<template>
  <form
    class="entity-form"
    @submit.prevent="$emit('submit')"
  >
    <BaseInput
      :error="errors.password"
      label="Nova senha"
      :model-value="modelValue.password"
      name="password"
      type="password"
      autocomplete="new-password"
      @update:model-value="updateField('password', $event)"
    />
    <BaseInput
      :error="errors.password_confirmation"
      label="Confirmar senha"
      :model-value="modelValue.password_confirmation"
      name="password_confirmation"
      type="password"
      autocomplete="new-password"
      @update:model-value="updateField('password_confirmation', $event)"
    />
    <div class="form-actions">
      <BaseButton
        :loading="submitting"
        type="submit"
      >
        Redefinir senha
      </BaseButton>
      <BaseButton
        type="button"
        variant="ghost"
        @click="$emit('cancel')"
      >
        Cancelar
      </BaseButton>
    </div>
  </form>
</template>
```

- [ ] **Step 8: Run focused frontend tests and verify GREEN**

Run:

```bash
npm test -- --run src/services/usersService.test.ts src/components/forms/UserForm.test.ts src/components/forms/UserPasswordForm.test.ts
```

Expected: PASS.

- [ ] **Step 9: Commit frontend user service and forms**

Run:

```bash
git add src/types/auth.ts src/types/admin.ts src/services/usersService.ts src/services/usersService.test.ts src/components/forms/UserForm.vue src/components/forms/UserForm.test.ts src/components/forms/UserPasswordForm.vue src/components/forms/UserPasswordForm.test.ts
git commit -m "feat: add admin user forms and service"
```

---

## Task 4: Frontend Users Page Management Flow

**Files:**

- Modify: `RIALMA-TrackVision-Frontend/src/pages/UsersPage.vue`
- Modify: `RIALMA-TrackVision-Frontend/src/pages/UsersPage.test.ts`

**Interfaces:**

- Consumes: `usersService` from Task 3.
- Consumes: `rolesService.list()`.
- Consumes: `UserForm` and `UserPasswordForm`.
- Produces: user create/edit/deactivate/reset password UI.

- [ ] **Step 1: Replace UsersPage tests with management coverage**

Replace `src/pages/UsersPage.test.ts` with:

```ts
import { mount } from '@vue/test-utils'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { ApiError } from '@/services/apiClient'
import { rolesService } from '@/services/rolesService'
import { usersService } from '@/services/usersService'
import UsersPage from './UsersPage.vue'

vi.mock('@/services/usersService', () => ({
  usersService: {
    list: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    resetPassword: vi.fn(),
    deactivate: vi.fn(),
  },
}))

vi.mock('@/services/rolesService', () => ({
  rolesService: {
    list: vi.fn(),
  },
}))

const user = {
  id: 1,
  uuid: 'user-uuid',
  name: 'Paulo',
  email: 'paulo@example.com',
  is_active: true,
  roles: ['super_admin'],
}

const roles = [
  { id: 1, name: 'super_admin', permissions: ['users.manage'] },
  { id: 2, name: 'operator', permissions: ['trips.manage'] },
]

async function flush(): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, 0))
}

async function setBodyInput(name: string, value: string): Promise<void> {
  const input = document.querySelector<HTMLInputElement>(`input[name="${name}"]`)
  if (!input) {
    throw new Error(`Input ${name} not found`)
  }

  input.value = value
  input.dispatchEvent(new Event('input', { bubbles: true }))
  await flush()
}

async function clickBodyInputValue(value: string): Promise<void> {
  const input = document.querySelector<HTMLInputElement>(`input[value="${value}"]`)
  if (!input) {
    throw new Error(`Input value ${value} not found`)
  }

  input.click()
  await flush()
}

async function submitBodyForm(): Promise<void> {
  const form = document.querySelector<HTMLFormElement>('form')
  if (!form) {
    throw new Error('Form not found')
  }

  form.dispatchEvent(new Event('submit', { bubbles: true, cancelable: true }))
  await flush()
}

describe('UsersPage', () => {
  beforeEach(() => {
    vi.mocked(usersService.list).mockResolvedValue({ data: [user] } as never)
    vi.mocked(usersService.create).mockResolvedValue(user as never)
    vi.mocked(usersService.update).mockResolvedValue(user as never)
    vi.mocked(usersService.resetPassword).mockResolvedValue(user as never)
    vi.mocked(usersService.deactivate).mockResolvedValue(undefined)
    vi.mocked(rolesService.list).mockResolvedValue({ data: roles } as never)
    vi.stubGlobal('confirm', vi.fn().mockReturnValue(true))
  })

  it('renders users with status and roles', async () => {
    const wrapper = mount(UsersPage)
    await flush()

    expect(wrapper.text()).toContain('Paulo')
    expect(wrapper.text()).toContain('super_admin')
    expect(wrapper.text()).toContain('Ativo')
  })

  it('opens create modal and creates user', async () => {
    const wrapper = mount(UsersPage, { attachTo: document.body })
    await flush()

    await wrapper.get('[data-test="create-user"]').trigger('click')
    await setBodyInput('name', 'Operador')
    await setBodyInput('email', 'op@test.local')
    await setBodyInput('password', 'secret-password')
    await setBodyInput('password_confirmation', 'secret-password')
    await clickBodyInputValue('operator')
    await submitBodyForm()

    expect(usersService.create).toHaveBeenCalledWith(expect.objectContaining({
      name: 'Operador',
      email: 'op@test.local',
      roles: ['operator'],
    }))
    expect(wrapper.text()).toContain('Usuario criado.')
  })

  it('opens edit modal and updates roles', async () => {
    const wrapper = mount(UsersPage, { attachTo: document.body })
    await flush()

    await wrapper.get('[data-test="edit-user"]').trigger('click')
    await clickBodyInputValue('operator')
    await submitBodyForm()

    expect(usersService.update).toHaveBeenCalledWith(user, expect.objectContaining({
      roles: expect.arrayContaining(['super_admin', 'operator']),
    }))
  })

  it('shows validation errors without closing modal', async () => {
    vi.mocked(usersService.create).mockRejectedValueOnce(new ApiError(422, 'Dados invalidos.', {
      email: ['Email ja cadastrado.'],
    }))

    const wrapper = mount(UsersPage, { attachTo: document.body })
    await flush()

    await wrapper.get('[data-test="create-user"]').trigger('click')
    await submitBodyForm()

    expect(wrapper.text()).toContain('Dados invalidos.')
    expect(document.body.textContent).toContain('Email ja cadastrado.')
  })

  it('resets password and deactivates user', async () => {
    const wrapper = mount(UsersPage, { attachTo: document.body })
    await flush()

    await wrapper.get('[data-test="reset-password"]').trigger('click')
    await setBodyInput('password', 'new-password')
    await setBodyInput('password_confirmation', 'new-password')
    await submitBodyForm()

    await wrapper.get('[data-test="deactivate-user"]').trigger('click')
    await flush()

    expect(usersService.resetPassword).toHaveBeenCalledWith(user, {
      password: 'new-password',
      password_confirmation: 'new-password',
    })
    expect(usersService.deactivate).toHaveBeenCalledWith(user)
  })
})
```

- [ ] **Step 2: Run page test and verify RED**

Run:

```bash
npm test -- --run src/pages/UsersPage.test.ts
```

Expected: FAIL because page lacks modals, actions, forms and service calls.

- [ ] **Step 3: Update UsersPage script**

Modify `src/pages/UsersPage.vue` imports:

```ts
import { computed, onMounted, ref } from 'vue'
import BaseAlert from '@/components/base/BaseAlert.vue'
import BaseButton from '@/components/base/BaseButton.vue'
import BaseModal from '@/components/base/BaseModal.vue'
import BaseTable from '@/components/base/BaseTable.vue'
import UserForm from '@/components/forms/UserForm.vue'
import UserPasswordForm from '@/components/forms/UserPasswordForm.vue'
import { ApiError } from '@/services/apiClient'
import { rolesService } from '@/services/rolesService'
import { usersService } from '@/services/usersService'
import type { CreateUserInput, ResetUserPasswordInput, Role, UpdateUserInput, UserFormModel } from '@/types/admin'
import type { FieldErrors } from '@/types/api'
import type { User } from '@/types/auth'
```

Use this state and helpers:

```ts
const columns = [
  { key: 'name', label: 'Nome' },
  { key: 'email', label: 'Email' },
  { key: 'roles', label: 'Roles' },
  { key: 'is_active', label: 'Status' },
  { key: 'actions', label: 'Acoes' },
]

const emptyCreateForm: CreateUserInput = {
  name: '',
  email: '',
  password: '',
  password_confirmation: '',
  is_active: true,
  roles: [],
}

const users = ref<User[]>([])
const roles = ref<Role[]>([])
const loading = ref(true)
const submitting = ref(false)
const userModalOpen = ref(false)
const passwordModalOpen = ref(false)
const editingUser = ref<User | null>(null)
const passwordUser = ref<User | null>(null)
const userForm = ref<UserFormModel>({ ...emptyCreateForm })
const passwordForm = ref<ResetUserPasswordInput>({ password: '', password_confirmation: '' })
const fieldErrors = ref<FieldErrors>({})
const passwordErrors = ref<FieldErrors>({})
const error = ref('')
const success = ref('')
const userFormMode = computed(() => editingUser.value ? 'edit' : 'create')

function userFrom(row: unknown): User {
  return row as User
}

function rolesFor(user: User): string {
  return user.roles?.join(', ') || '-'
}

function normalizeCreateForm(input: UserFormModel): CreateUserInput {
  return {
    name: input.name.trim(),
    email: input.email.trim(),
    password: 'password' in input ? input.password : '',
    password_confirmation: 'password_confirmation' in input ? input.password_confirmation : '',
    is_active: input.is_active,
    roles: input.roles,
  }
}

function normalizeUpdateForm(input: UserFormModel): UpdateUserInput {
  return {
    name: input.name.trim(),
    email: input.email.trim(),
    is_active: input.is_active,
    roles: input.roles,
  }
}
```

Use these async methods:

```ts
async function loadUsers(): Promise<void> {
  loading.value = true
  error.value = ''

  try {
    const [usersResponse, rolesResponse] = await Promise.all([
      usersService.list(),
      rolesService.list(),
    ])
    users.value = usersResponse.data
    roles.value = rolesResponse.data
  } catch {
    error.value = 'Nao foi possivel carregar usuarios.'
  } finally {
    loading.value = false
  }
}

function openCreate(): void {
  editingUser.value = null
  userForm.value = { ...emptyCreateForm }
  fieldErrors.value = {}
  userModalOpen.value = true
}

function openEdit(user: User): void {
  editingUser.value = user
  userForm.value = {
    name: user.name,
    email: user.email,
    is_active: user.is_active ?? true,
    roles: [...(user.roles ?? [])],
  }
  fieldErrors.value = {}
  userModalOpen.value = true
}

function closeUserModal(): void {
  userModalOpen.value = false
  submitting.value = false
}

function openPassword(user: User): void {
  passwordUser.value = user
  passwordForm.value = { password: '', password_confirmation: '' }
  passwordErrors.value = {}
  passwordModalOpen.value = true
}

function closePasswordModal(): void {
  passwordModalOpen.value = false
  submitting.value = false
}
```

Use these save methods:

```ts
async function saveUser(): Promise<void> {
  submitting.value = true
  fieldErrors.value = {}
  success.value = ''

  try {
    if (editingUser.value) {
      await usersService.update(editingUser.value, normalizeUpdateForm(userForm.value))
      success.value = 'Usuario atualizado.'
    } else {
      await usersService.create(normalizeCreateForm(userForm.value))
      success.value = 'Usuario criado.'
    }

    closeUserModal()
    await loadUsers()
  } catch (apiError) {
    if (apiError instanceof ApiError) {
      fieldErrors.value = apiError.errors
      error.value = apiError.message
    } else {
      error.value = 'Nao foi possivel salvar o usuario.'
    }
  } finally {
    submitting.value = false
  }
}

async function resetPassword(): Promise<void> {
  if (!passwordUser.value) {
    return
  }

  submitting.value = true
  passwordErrors.value = {}
  success.value = ''

  try {
    await usersService.resetPassword(passwordUser.value, passwordForm.value)
    success.value = 'Senha redefinida.'
    closePasswordModal()
  } catch (apiError) {
    if (apiError instanceof ApiError) {
      passwordErrors.value = apiError.errors
      error.value = apiError.message
    } else {
      error.value = 'Nao foi possivel redefinir a senha.'
    }
  } finally {
    submitting.value = false
  }
}

async function deactivateUser(user: User): Promise<void> {
  if (!window.confirm(`Desativar usuario ${user.email}?`)) {
    return
  }

  try {
    await usersService.deactivate(user)
    success.value = 'Usuario desativado.'
    await loadUsers()
  } catch (apiError) {
    error.value = apiError instanceof ApiError ? apiError.message : 'Nao foi possivel desativar o usuario.'
  }
}

onMounted(loadUsers)
```

- [ ] **Step 4: Update UsersPage template**

Replace the template with:

```vue
<template>
  <section class="page-section">
    <header class="page-header">
      <div>
        <p class="page-eyebrow">
          Seguranca
        </p>
        <h1>Usuarios</h1>
      </div>
      <BaseButton
        data-test="create-user"
        type="button"
        @click="openCreate"
      >
        Novo usuario
      </BaseButton>
    </header>

    <BaseAlert
      v-if="error"
      variant="error"
    >
      {{ error }}
    </BaseAlert>
    <BaseAlert
      v-if="success"
      variant="success"
    >
      {{ success }}
    </BaseAlert>

    <BaseTable
      :columns="columns"
      empty-text="Nenhum usuario encontrado."
      :loading="loading"
      :rows="users"
    >
      <template #row="{ row }">
        <td>{{ userFrom(row).name }}</td>
        <td>{{ userFrom(row).email }}</td>
        <td>{{ rolesFor(userFrom(row)) }}</td>
        <td>{{ userFrom(row).is_active === false ? 'Inativo' : 'Ativo' }}</td>
        <td>
          <div class="row-actions">
            <BaseButton
              data-test="edit-user"
              type="button"
              variant="secondary"
              @click="openEdit(userFrom(row))"
            >
              Editar
            </BaseButton>
            <BaseButton
              data-test="reset-password"
              type="button"
              variant="secondary"
              @click="openPassword(userFrom(row))"
            >
              Redefinir senha
            </BaseButton>
            <BaseButton
              v-if="userFrom(row).is_active !== false"
              data-test="deactivate-user"
              type="button"
              variant="danger"
              @click="deactivateUser(userFrom(row))"
            >
              Desativar
            </BaseButton>
          </div>
        </td>
      </template>
    </BaseTable>

    <BaseModal
      :open="userModalOpen"
      :title="editingUser ? 'Editar usuario' : 'Novo usuario'"
      @close="closeUserModal"
    >
      <UserForm
        v-model="userForm"
        :errors="fieldErrors"
        :mode="userFormMode"
        :roles="roles"
        :submitting="submitting"
        @cancel="closeUserModal"
        @submit="saveUser"
      />
    </BaseModal>

    <BaseModal
      :open="passwordModalOpen"
      title="Redefinir senha"
      @close="closePasswordModal"
    >
      <UserPasswordForm
        v-model="passwordForm"
        :errors="passwordErrors"
        :submitting="submitting"
        @cancel="closePasswordModal"
        @submit="resetPassword"
      />
    </BaseModal>
  </section>
</template>
```

- [ ] **Step 5: Run page test and verify GREEN**

Run:

```bash
npm test -- --run src/pages/UsersPage.test.ts
```

Expected: PASS.

- [ ] **Step 6: Run focused frontend security/user tests**

Run:

```bash
npm test -- --run src/pages/UsersPage.test.ts src/services/usersService.test.ts src/components/forms/UserForm.test.ts src/components/forms/UserPasswordForm.test.ts
```

Expected: PASS.

- [ ] **Step 7: Commit frontend UsersPage management flow**

Run:

```bash
git add src/pages/UsersPage.vue src/pages/UsersPage.test.ts
git commit -m "feat: manage users from admin page"
```

---

## Task 5: Documentation, Final Verification, And Handoff

**Files:**

- Modify: `RIALMA-TrackVision-Backend/docs/api-parent-admin.md`
- Modify: `RIALMA-TrackVision-Backend/README.md`
- Modify: `RIALMA-TrackVision-Frontend/README.md`

**Interfaces:**

- Documents: admin user management endpoints, payloads, permissions and self-lock protections.
- Produces: final verification evidence for backend and frontend branches.

- [ ] **Step 1: Update backend API documentation**

Append to `RIALMA-TrackVision-Backend/docs/api-parent-admin.md`:

````markdown
## Admin User Management

Endpoints:

- `GET /users`
- `POST /users`
- `GET /users/{user}`
- `PATCH /users/{user}`
- `PATCH /users/{user}/password`
- `DELETE /users/{user}`

Authorization:

- User read/write endpoints require `users.manage`.
- `GET /roles` can be used by users with `users.manage` or `permissions.manage`.
- `GET /permissions` remains restricted to `permissions.manage`.

Create payload:

```json
{
  "name": "Operador Patio",
  "email": "operador@example.com",
  "password": "secret-password",
  "password_confirmation": "secret-password",
  "is_active": true,
  "roles": ["operator"]
}
```

Update payload:

```json
{
  "name": "Operador Patio 01",
  "email": "operador01@example.com",
  "is_active": true,
  "roles": ["operator", "auditor"]
}
```

Password payload:

```json
{
  "password": "new-password",
  "password_confirmation": "new-password"
}
```

`DELETE /users/{user}` performs logical deactivation by setting `is_active = false`.
Inactive users cannot authenticate. The API rejects self-deactivation and rejects
self role changes that would remove effective `users.manage` access.
````

- [ ] **Step 2: Update backend README**

Append to `RIALMA-TrackVision-Backend/README.md`:

```markdown
### Admin User Management

Administrative users are managed through `/api/v1/admin/users`.

Roles and permissions remain seeded through `PermissionSeeder`; the admin UI assigns
existing roles to users but does not create or edit the role catalog.

Inactive users cannot login. The backend prevents an authenticated admin from
deactivating their own account or removing their own effective `users.manage` access.
```

- [ ] **Step 3: Update frontend README**

Append to `RIALMA-TrackVision-Frontend/README.md`:

```markdown
### User Management

The `/users` admin screen supports creating users, editing profile/status/roles,
resetting passwords and deactivating users.

Roles are loaded from the backend catalog and sent as role names. The frontend hides
no backend security rule; validation and authorization errors are shown from API
responses.
```

- [ ] **Step 4: Run backend final verification**

Run:

```bash
composer validate
php artisan test
php artisan route:list --path=api/v1/admin/users
php artisan route:list --path=api/v1/admin/roles
git diff --check
```

Expected:

- Composer valid.
- Full Laravel test suite passes.
- User and role routes are registered.
- `git diff --check` has no output.

- [ ] **Step 5: Run frontend final verification**

Run:

```bash
npm test -- --run
npm run build
git diff --check
```

Expected:

- Vitest passes.
- Production build passes.
- `git diff --check` has no output.

- [ ] **Step 6: Commit documentation updates**

Backend:

```bash
git add docs/api-parent-admin.md README.md
git commit -m "docs: add admin user management guide"
```

Frontend:

```bash
git add README.md
git commit -m "docs: add user management guide"
```

- [ ] **Step 7: Prepare integration summary**

Run in each implementation worktree:

```bash
git log --oneline main..HEAD
git diff --stat main..HEAD
```

Include in the handoff:

- commits created;
- verification commands and results;
- backend route list entries for user management;
- frontend build result;
- any skipped backend test and reason.

---

## Acceptance Checklist

- [ ] Admin com `users.manage` cria usuario com roles existentes.
- [ ] Admin com `users.manage` edita nome, email, status e roles.
- [ ] Admin com `users.manage` redefine senha de outro usuario.
- [ ] Usuario inativo nao autentica.
- [ ] API impede auto-desativacao.
- [ ] API impede que o proprio admin remova seu acesso efetivo a `users.manage`.
- [ ] Roles podem ser listadas por `users.manage` para atribuicao.
- [ ] Permissoes continuam restritas a `permissions.manage`.
- [ ] Roles/permissoes permanecem somente leitura.
- [ ] Frontend mostra status, roles e feedback de validacao.
- [ ] Backend segue Controllers magros, Form Requests, Actions/Services e Resources.
- [ ] Frontend segue services, componentes pequenos e estados de loading/erro/sucesso.
