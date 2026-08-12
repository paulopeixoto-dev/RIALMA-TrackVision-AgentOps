# Parent Admin Domain Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the parent-server administrative backend domain for vehicles, locations, edge nodes, cameras, and camera pairs.

**Architecture:** Add the domain in Laravel behind versioned `/api/v1/admin/*` routes. Keep controllers thin, validate input with Form Requests, serialize with API Resources, and isolate domain rules in small Actions or Support classes. Persist camera credentials encrypted and enforce camera-pair invariants before future capture/sync phases depend on them.

**Tech Stack:** Laravel 13, PHP 8.4, Laravel Passport, Spatie Laravel Permission, Eloquent, PHPUnit, SQLite test database, Laravel Pint.

## Global Constraints

- Backend code lives only in `RIALMA-TrackVision-Backend`.
- AgentOps documentation lives only in `RIALMA-TrackVision-AgentOps`.
- Work from backend branch `codex/backend-foundation-security`; create implementation branch `codex/parent-admin-domain` before code changes.
- Follow `docs/backend-laravel-guidelines.md`.
- Controllers stay thin.
- Form Requests validate API input.
- Actions isolate business rules and persistence decisions.
- API responses use Resources.
- Admin endpoints use Passport user tokens plus Spatie permissions.
- Use `vehicles.manage` for vehicle endpoints.
- Use `cameras.manage` for location, edge-node, camera, and camera-pair endpoints.
- Camera passwords are stored encrypted and are never returned by Resources.
- This phase does not implement Intelbras API calls, image capture, media storage, edge sync, trips, reports, or frontend.
- Run verification before each commit for the touched scope.

---

## File Structure Map

Backend files to create or modify:

- Create `app/Enums/CameraType.php`
- Create `app/Enums/CameraVendor.php`
- Create `app/Enums/CameraPairDirection.php`
- Create `app/Enums/EdgeNodeStatus.php`
- Create `app/Models/Concerns/HasPublicUuid.php`
- Create `app/Support/Vehicles/NormalizePlate.php`
- Create `app/Actions/Admin/Vehicles/CreateVehicleAction.php`
- Create `app/Actions/Admin/Vehicles/UpdateVehicleAction.php`
- Create `app/Actions/Admin/Locations/CreateLocationAction.php`
- Create `app/Actions/Admin/Locations/UpdateLocationAction.php`
- Create `app/Actions/Admin/EdgeNodes/CreateEdgeNodeAction.php`
- Create `app/Actions/Admin/EdgeNodes/UpdateEdgeNodeAction.php`
- Create `app/Actions/Admin/Cameras/CreateCameraAction.php`
- Create `app/Actions/Admin/Cameras/UpdateCameraAction.php`
- Create `app/Actions/Admin/CameraPairs/CreateCameraPairAction.php`
- Create `app/Actions/Admin/CameraPairs/UpdateCameraPairAction.php`
- Create `app/Models/Vehicle.php`
- Create `app/Models/Location.php`
- Create `app/Models/EdgeNode.php`
- Create `app/Models/Camera.php`
- Create `app/Models/CameraPair.php`
- Create migrations for `vehicles`, `locations`, `edge_nodes`, `cameras`, `camera_pairs`
- Create factories for the five models
- Create CRUD controllers under `app/Http/Controllers/Api/V1/Admin`
- Create Form Requests under `app/Http/Requests/Api/V1/Admin`
- Create Resources under `app/Http/Resources/Api/V1`
- Modify `routes/api.php`
- Create tests under `tests/Unit/Support` and `tests/Feature/Admin`
- Create backend API doc `docs/api-parent-admin.md`

---

## Task 1: Domain Enums And Plate Normalization

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Enums/CameraType.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/CameraVendor.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/CameraPairDirection.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/EdgeNodeStatus.php`
- Create: `RIALMA-TrackVision-Backend/app/Support/Vehicles/NormalizePlate.php`
- Test: `RIALMA-TrackVision-Backend/tests/Unit/Support/NormalizePlateTest.php`

**Interfaces:**

- Produces: `App\Enums\CameraType::Lpr`, `App\Enums\CameraType::Support`
- Produces: `App\Enums\CameraVendor::Intelbras`
- Produces: `App\Enums\CameraPairDirection::Outbound`, `Inbound`, `Unknown`
- Produces: `App\Enums\EdgeNodeStatus::Online`, `Offline`, `Degraded`
- Produces: `App\Support\Vehicles\NormalizePlate::__invoke(string $plate): string`

- [ ] **Step 1: Write the failing plate normalizer test**

Create `tests/Unit/Support/NormalizePlateTest.php`:

```php
<?php

namespace Tests\Unit\Support;

use App\Support\Vehicles\NormalizePlate;
use PHPUnit\Framework\TestCase;

class NormalizePlateTest extends TestCase
{
    public function test_normalizes_plate_to_uppercase_alphanumeric_value(): void
    {
        $normalize = new NormalizePlate;

        $this->assertSame('ABC1D23', $normalize(' abc-1d23 '));
        $this->assertSame('BRA2E19', $normalize('BRA 2E-19'));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
php artisan test --filter=NormalizePlateTest
```

Expected: FAIL because `App\Support\Vehicles\NormalizePlate` does not exist.

- [ ] **Step 3: Create enums**

Create `app/Enums/CameraType.php`:

```php
<?php

namespace App\Enums;

enum CameraType: string
{
    case Lpr = 'lpr';
    case Support = 'support';
}
```

Create `app/Enums/CameraVendor.php`:

```php
<?php

namespace App\Enums;

enum CameraVendor: string
{
    case Intelbras = 'intelbras';
}
```

Create `app/Enums/CameraPairDirection.php`:

```php
<?php

namespace App\Enums;

enum CameraPairDirection: string
{
    case Outbound = 'outbound';
    case Inbound = 'inbound';
    case Unknown = 'unknown';
}
```

Create `app/Enums/EdgeNodeStatus.php`:

```php
<?php

namespace App\Enums;

enum EdgeNodeStatus: string
{
    case Online = 'online';
    case Offline = 'offline';
    case Degraded = 'degraded';
}
```

- [ ] **Step 4: Create plate normalizer**

Create `app/Support/Vehicles/NormalizePlate.php`:

```php
<?php

namespace App\Support\Vehicles;

class NormalizePlate
{
    public function __invoke(string $plate): string
    {
        return strtoupper((string) preg_replace('/[^A-Za-z0-9]/', '', $plate));
    }
}
```

- [ ] **Step 5: Run normalization test**

Run:

```bash
php artisan test --filter=NormalizePlateTest
```

Expected: PASS.

- [ ] **Step 6: Commit task**

Run:

```bash
git add app/Enums app/Support/Vehicles tests/Unit/Support/NormalizePlateTest.php
git commit -m "feat: add parent admin domain enums"
```

---

## Task 2: Schema, Models, Relationships, And Factories

**Files:**

- Create: `database/migrations/*_create_vehicles_table.php`
- Create: `database/migrations/*_create_locations_table.php`
- Create: `database/migrations/*_create_edge_nodes_table.php`
- Create: `database/migrations/*_create_cameras_table.php`
- Create: `database/migrations/*_create_camera_pairs_table.php`
- Create: `app/Models/Vehicle.php`
- Create: `app/Models/Location.php`
- Create: `app/Models/EdgeNode.php`
- Create: `app/Models/Camera.php`
- Create: `app/Models/CameraPair.php`
- Create: `app/Models/Concerns/HasPublicUuid.php`
- Create: `database/factories/VehicleFactory.php`
- Create: `database/factories/LocationFactory.php`
- Create: `database/factories/EdgeNodeFactory.php`
- Create: `database/factories/CameraFactory.php`
- Create: `database/factories/CameraPairFactory.php`
- Test: `tests/Feature/Admin/ParentAdminSchemaTest.php`

**Interfaces:**

- Consumes: enums from Task 1.
- Produces Eloquent models with numeric `id`, public `uuid`, soft deletes, casts, and relationships.
- Produces encrypted camera password cast through `Camera::$casts['password_encrypted']`.

- [ ] **Step 1: Write failing schema test**

Create `tests/Feature/Admin/ParentAdminSchemaTest.php`:

```php
<?php

namespace Tests\Feature\Admin;

use App\Enums\CameraPairDirection;
use App\Enums\CameraType;
use App\Enums\CameraVendor;
use App\Enums\EdgeNodeStatus;
use App\Models\Camera;
use App\Models\CameraPair;
use App\Models\EdgeNode;
use App\Models\Location;
use App\Models\Vehicle;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ParentAdminSchemaTest extends TestCase
{
    use RefreshDatabase;

    public function test_domain_models_persist_relationships_and_casts(): void
    {
        $vehicle = Vehicle::factory()->create([
            'plate' => 'ABC-1D23',
            'plate_normalized' => 'ABC1D23',
            'is_active' => true,
        ]);
        $location = Location::factory()->create();
        $edgeNode = EdgeNode::factory()->for($location)->create([
            'status' => EdgeNodeStatus::Offline,
        ]);
        $lpr = Camera::factory()->for($location)->for($edgeNode)->create([
            'type' => CameraType::Lpr,
            'vendor' => CameraVendor::Intelbras,
            'password_encrypted' => 'secret',
        ]);
        $support = Camera::factory()->for($location)->for($edgeNode)->create([
            'type' => CameraType::Support,
            'vendor' => CameraVendor::Intelbras,
        ]);
        $pair = CameraPair::factory()
            ->for($location)
            ->for($edgeNode)
            ->for($lpr, 'lprCamera')
            ->for($support, 'supportCamera')
            ->create(['direction' => CameraPairDirection::Outbound]);

        $this->assertSame('ABC1D23', $vehicle->plate_normalized);
        $this->assertNotEmpty($vehicle->uuid);
        $this->assertSame($location->id, $edgeNode->location->id);
        $this->assertSame(CameraType::Lpr, $lpr->type);
        $this->assertNotSame('secret', $lpr->getRawOriginal('password_encrypted'));
        $this->assertSame($lpr->id, $pair->lprCamera->id);
        $this->assertSame($support->id, $pair->supportCamera->id);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
php artisan test --filter=ParentAdminSchemaTest
```

Expected: FAIL because domain models and migrations do not exist.

- [ ] **Step 3: Create migrations**

Use Artisan names:

```bash
php artisan make:migration create_vehicles_table
php artisan make:migration create_locations_table
php artisan make:migration create_edge_nodes_table
php artisan make:migration create_cameras_table
php artisan make:migration create_camera_pairs_table
```

Implement schema:

```php
Schema::create('vehicles', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->string('plate', 20);
    $table->string('plate_normalized', 20)->unique();
    $table->string('fleet_code', 80)->nullable();
    $table->text('description')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
});
```

```php
Schema::create('locations', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->string('name');
    $table->text('description')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
});
```

```php
Schema::create('edge_nodes', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->foreignId('location_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
    $table->string('name');
    $table->text('description')->nullable();
    $table->string('status', 30)->default('offline');
    $table->timestamp('last_seen_at')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
});
```

```php
Schema::create('cameras', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->foreignId('location_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
    $table->foreignId('edge_node_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
    $table->string('name');
    $table->string('type', 30);
    $table->string('vendor', 30)->default('intelbras');
    $table->string('host');
    $table->unsignedInteger('port')->default(80);
    $table->unsignedInteger('channel')->default(1);
    $table->string('username')->nullable();
    $table->text('password_encrypted')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();

    $table->index(['location_id', 'edge_node_id', 'type']);
});
```

```php
Schema::create('camera_pairs', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->foreignId('location_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
    $table->foreignId('edge_node_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
    $table->string('name');
    $table->foreignId('lpr_camera_id')->constrained('cameras')->cascadeOnUpdate()->restrictOnDelete();
    $table->foreignId('support_camera_id')->constrained('cameras')->cascadeOnUpdate()->restrictOnDelete();
    $table->string('direction', 30)->default('unknown');
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();

    $table->unique(['edge_node_id', 'lpr_camera_id', 'support_camera_id']);
});
```

Each migration `down()` drops tables in reverse dependency order for that migration.

- [ ] **Step 4: Create public UUID trait**

Create `app/Models/Concerns/HasPublicUuid.php`:

```php
<?php

namespace App\Models\Concerns;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

trait HasPublicUuid
{
    protected static function bootHasPublicUuid(): void
    {
        static::creating(function (Model $model): void {
            if (! $model->getAttribute('uuid')) {
                $model->setAttribute('uuid', (string) Str::uuid());
            }
        });
    }
}
```

- [ ] **Step 5: Create models**

Create models with `HasFactory`, `HasPublicUuid`, `SoftDeletes`, fillable fields, casts, and relationships. Keep the database primary key as the default auto-incrementing integer `id`; `uuid` is a separate public identifier used by APIs and future sync:

```php
protected function casts(): array
{
    return [
        'is_active' => 'boolean',
    ];
}
```

Use enum casts where applicable:

```php
'type' => CameraType::class,
'vendor' => CameraVendor::class,
'password_encrypted' => 'encrypted',
```

Relationships:

- `Location::edgeNodes(): HasMany`
- `Location::cameras(): HasMany`
- `Location::cameraPairs(): HasMany`
- `EdgeNode::location(): BelongsTo`
- `EdgeNode::cameras(): HasMany`
- `EdgeNode::cameraPairs(): HasMany`
- `Camera::location(): BelongsTo`
- `Camera::edgeNode(): BelongsTo`
- `CameraPair::location(): BelongsTo`
- `CameraPair::edgeNode(): BelongsTo`
- `CameraPair::lprCamera(): BelongsTo`
- `CameraPair::supportCamera(): BelongsTo`

- [ ] **Step 6: Create factories**

Factories produce valid records:

- `VehicleFactory`: plate `ABC-1D23`, plate_normalized `ABC1D23`, active true.
- `LocationFactory`: name from faker, active true.
- `EdgeNodeFactory`: `status => EdgeNodeStatus::Offline`.
- `CameraFactory`: valid Intelbras camera with nullable username and encrypted password.
- `CameraPairFactory`: direction `CameraPairDirection::Unknown`.

- [ ] **Step 7: Run schema test**

Run:

```bash
php artisan test --filter=ParentAdminSchemaTest
```

Expected: PASS.

- [ ] **Step 8: Commit task**

Run:

```bash
git add app/Models database/migrations database/factories tests/Feature/Admin/ParentAdminSchemaTest.php
git commit -m "feat: add parent admin domain schema"
```

---

## Task 3: Vehicle Admin API

**Files:**

- Create: `app/Actions/Admin/Vehicles/CreateVehicleAction.php`
- Create: `app/Actions/Admin/Vehicles/UpdateVehicleAction.php`
- Create: `app/Http/Controllers/Api/V1/Admin/VehicleController.php`
- Create: `app/Http/Requests/Api/V1/Admin/StoreVehicleRequest.php`
- Create: `app/Http/Requests/Api/V1/Admin/UpdateVehicleRequest.php`
- Create: `app/Http/Resources/Api/V1/VehicleResource.php`
- Modify: `routes/api.php`
- Test: `tests/Feature/Admin/VehicleAdminTest.php`

**Interfaces:**

- Consumes: `NormalizePlate::__invoke(string): string`
- Produces REST endpoints under `/api/v1/admin/vehicles`
- Produces JSON fields: `id`, `uuid`, `plate`, `plate_normalized`, `fleet_code`, `description`, `is_active`

- [ ] **Step 1: Write failing vehicle API tests**

Create `tests/Feature/Admin/VehicleAdminTest.php`:

```php
<?php

namespace Tests\Feature\Admin;

use App\Models\User;
use App\Models\Vehicle;
use Database\Seeders\PermissionSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Passport\Passport;
use Tests\TestCase;

class VehicleAdminTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_with_permission_can_create_vehicle_with_normalized_plate(): void
    {
        $user = $this->actingAdmin();

        $this->postJson('/api/v1/admin/vehicles', [
            'plate' => 'abc-1d23',
            'fleet_code' => 'TRUCK-01',
            'description' => 'Caminhao de teste',
            'is_active' => true,
        ])
            ->assertCreated()
            ->assertJsonPath('data.plate', 'abc-1d23')
            ->assertJsonPath('data.plate_normalized', 'ABC1D23')
            ->assertJsonPath('data.fleet_code', 'TRUCK-01');

        $this->assertTrue($user->can('vehicles.manage'));
        $this->assertDatabaseHas('vehicles', ['plate_normalized' => 'ABC1D23']);
    }

    public function test_duplicate_vehicle_plate_is_rejected_after_normalization(): void
    {
        $this->actingAdmin();
        Vehicle::factory()->create(['plate' => 'ABC-1D23', 'plate_normalized' => 'ABC1D23']);

        $this->postJson('/api/v1/admin/vehicles', ['plate' => 'abc 1d23'])
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['plate']);
    }

    public function test_user_without_vehicle_permission_cannot_create_vehicle(): void
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('auditor');
        Passport::actingAs($user, ['admin:read']);

        $this->postJson('/api/v1/admin/vehicles', ['plate' => 'ABC-1D23'])
            ->assertForbidden();
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

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
php artisan test --filter=VehicleAdminTest
```

Expected: FAIL with 404 or missing controller/request classes.

- [ ] **Step 3: Implement Form Requests**

`StoreVehicleRequest`:

```php
public function authorize(): bool
{
    return $this->user()?->can('vehicles.manage') === true;
}

public function rules(): array
{
    return [
        'plate' => ['required', 'string', 'max:20'],
        'fleet_code' => ['nullable', 'string', 'max:80'],
        'description' => ['nullable', 'string'],
        'is_active' => ['sometimes', 'boolean'],
    ];
}
```

`UpdateVehicleRequest` uses the same rules with `sometimes` on `plate`.

- [ ] **Step 4: Implement Actions**

`CreateVehicleAction::execute(array $data): Vehicle` normalizes plate, checks duplicate by `plate_normalized`, throws `ValidationException::withMessages(['plate' => ['The plate has already been registered.']])`, and creates the model.

`UpdateVehicleAction::execute(Vehicle $vehicle, array $data): Vehicle` normalizes plate when present, rejects duplicate normalized plate excluding current vehicle id, updates the model, and returns `refresh()`.

- [ ] **Step 5: Implement Resource and Controller**

`VehicleResource` returns only public fields:

```php
return [
    'id' => $this->id,
    'uuid' => $this->uuid,
    'plate' => $this->plate,
    'plate_normalized' => $this->plate_normalized,
    'fleet_code' => $this->fleet_code,
    'description' => $this->description,
    'is_active' => $this->is_active,
];
```

`VehicleController` methods:

- `index()` returns `VehicleResource::collection(Vehicle::query()->orderBy('plate_normalized')->paginate())`
- `store(StoreVehicleRequest $request, CreateVehicleAction $action)` returns created resource with `201`
- `show(Vehicle $vehicle)` returns resource
- `update(UpdateVehicleRequest $request, Vehicle $vehicle, UpdateVehicleAction $action)` returns resource
- `destroy(Vehicle $vehicle)` soft deletes and returns `204`

- [ ] **Step 6: Register routes**

Inside the existing `/api/v1/admin` group guarded by `EnsureUserAccessToken::using('admin:read')`:

```php
Route::apiResource('vehicles', VehicleController::class)
    ->middleware('permission:vehicles.manage,api');
```

- [ ] **Step 7: Run vehicle tests**

Run:

```bash
php artisan test --filter=VehicleAdminTest
```

Expected: PASS.

- [ ] **Step 8: Commit task**

Run:

```bash
git add app/Actions/Admin/Vehicles app/Http/Controllers/Api/V1/Admin/VehicleController.php app/Http/Requests/Api/V1/Admin/StoreVehicleRequest.php app/Http/Requests/Api/V1/Admin/UpdateVehicleRequest.php app/Http/Resources/Api/V1/VehicleResource.php routes/api.php tests/Feature/Admin/VehicleAdminTest.php
git commit -m "feat: add vehicle admin api"
```

---

## Task 4: Location And Edge Node Admin API

**Files:**

- Create: `app/Actions/Admin/Locations/CreateLocationAction.php`
- Create: `app/Actions/Admin/Locations/UpdateLocationAction.php`
- Create: `app/Actions/Admin/EdgeNodes/CreateEdgeNodeAction.php`
- Create: `app/Actions/Admin/EdgeNodes/UpdateEdgeNodeAction.php`
- Create: `app/Http/Controllers/Api/V1/Admin/LocationController.php`
- Create: `app/Http/Controllers/Api/V1/Admin/EdgeNodeController.php`
- Create: `app/Http/Requests/Api/V1/Admin/StoreLocationRequest.php`
- Create: `app/Http/Requests/Api/V1/Admin/UpdateLocationRequest.php`
- Create: `app/Http/Requests/Api/V1/Admin/StoreEdgeNodeRequest.php`
- Create: `app/Http/Requests/Api/V1/Admin/UpdateEdgeNodeRequest.php`
- Create: `app/Http/Resources/Api/V1/LocationResource.php`
- Create: `app/Http/Resources/Api/V1/EdgeNodeResource.php`
- Modify: `routes/api.php`
- Test: `tests/Feature/Admin/LocationEdgeNodeAdminTest.php`

**Interfaces:**

- Produces REST endpoints under `/api/v1/admin/locations`
- Produces REST endpoints under `/api/v1/admin/edge-nodes`
- Produces `EdgeNode.status` enum default `offline`

- [ ] **Step 1: Write failing location and edge-node API tests**

Create `tests/Feature/Admin/LocationEdgeNodeAdminTest.php`:

```php
<?php

namespace Tests\Feature\Admin;

use App\Enums\EdgeNodeStatus;
use App\Models\Location;
use App\Models\User;
use Database\Seeders\PermissionSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Passport\Passport;
use Tests\TestCase;

class LocationEdgeNodeAdminTest extends TestCase
{
    use RefreshDatabase;

    public function test_admin_can_create_location_and_edge_node(): void
    {
        $this->actingCameraAdmin();

        $locationId = $this->postJson('/api/v1/admin/locations', [
            'name' => 'Portaria Principal',
            'description' => 'Entrada dos caminhoes',
            'is_active' => true,
        ])
            ->assertCreated()
            ->assertJsonPath('data.name', 'Portaria Principal')
            ->json('data.id');

        $this->postJson('/api/v1/admin/edge-nodes', [
            'location_id' => $locationId,
            'name' => 'Edge Portaria 01',
            'description' => 'Servidor local',
            'is_active' => true,
        ])
            ->assertCreated()
            ->assertJsonPath('data.location.id', $locationId)
            ->assertJsonPath('data.status', EdgeNodeStatus::Offline->value);
    }

    public function test_user_without_camera_permission_cannot_create_location(): void
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('auditor');
        Passport::actingAs($user, ['admin:read']);

        $this->postJson('/api/v1/admin/locations', ['name' => 'Portaria'])
            ->assertForbidden();
    }

    public function test_edge_node_requires_existing_location(): void
    {
        $this->actingCameraAdmin();

        $this->postJson('/api/v1/admin/edge-nodes', [
            'location_id' => 999,
            'name' => 'Edge Portaria 01',
        ])
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['location_id']);
    }

    private function actingCameraAdmin(): void
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('super_admin');
        Passport::actingAs($user, ['admin:read']);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
php artisan test --filter=LocationEdgeNodeAdminTest
```

Expected: FAIL with 404 or missing classes.

- [ ] **Step 3: Implement location requests, actions, resource, controller**

Validation:

```php
'name' => ['required', 'string', 'max:255'],
'description' => ['nullable', 'string'],
'is_active' => ['sometimes', 'boolean'],
```

Authorization: `$this->user()?->can('cameras.manage') === true`.

`LocationResource` returns `id`, `uuid`, `name`, `description`, `is_active`.

Controller uses paginated `index`, resource `show`, action-backed `store` and `update`, and soft-delete `destroy`.

- [ ] **Step 4: Implement edge-node requests, actions, resource, controller**

Validation:

```php
'location_id' => ['required', 'integer', 'exists:locations,id'],
'name' => ['required', 'string', 'max:255'],
'description' => ['nullable', 'string'],
'is_active' => ['sometimes', 'boolean'],
```

`CreateEdgeNodeAction` sets `status` to `EdgeNodeStatus::Offline`.

`EdgeNodeResource` returns `id`, `uuid`, `name`, `description`, `status`, `last_seen_at`, `is_active`, and loaded `location`.

Controller eager-loads `location` in list/detail responses.

- [ ] **Step 5: Register routes**

```php
Route::apiResource('locations', LocationController::class)
    ->middleware('permission:cameras.manage,api');

Route::apiResource('edge-nodes', EdgeNodeController::class)
    ->middleware('permission:cameras.manage,api');
```

- [ ] **Step 6: Run location and edge-node tests**

Run:

```bash
php artisan test --filter=LocationEdgeNodeAdminTest
```

Expected: PASS.

- [ ] **Step 7: Commit task**

Run:

```bash
git add app/Actions/Admin/Locations app/Actions/Admin/EdgeNodes app/Http/Controllers/Api/V1/Admin/LocationController.php app/Http/Controllers/Api/V1/Admin/EdgeNodeController.php app/Http/Requests/Api/V1/Admin app/Http/Resources/Api/V1/LocationResource.php app/Http/Resources/Api/V1/EdgeNodeResource.php routes/api.php tests/Feature/Admin/LocationEdgeNodeAdminTest.php
git commit -m "feat: add location and edge node admin api"
```

---

## Task 5: Camera Admin API

**Files:**

- Create: `app/Actions/Admin/Cameras/CreateCameraAction.php`
- Create: `app/Actions/Admin/Cameras/UpdateCameraAction.php`
- Create: `app/Http/Controllers/Api/V1/Admin/CameraController.php`
- Create: `app/Http/Requests/Api/V1/Admin/StoreCameraRequest.php`
- Create: `app/Http/Requests/Api/V1/Admin/UpdateCameraRequest.php`
- Create: `app/Http/Resources/Api/V1/CameraResource.php`
- Modify: `routes/api.php`
- Test: `tests/Feature/Admin/CameraAdminTest.php`

**Interfaces:**

- Consumes: `CameraType`, `CameraVendor`
- Produces REST endpoints under `/api/v1/admin/cameras`
- Produces camera JSON with no `password_encrypted`, no `password`, and no secret values

- [ ] **Step 1: Write failing camera API tests**

Create `tests/Feature/Admin/CameraAdminTest.php`:

```php
<?php

namespace Tests\Feature\Admin;

use App\Enums\CameraType;
use App\Enums\CameraVendor;
use App\Models\Camera;
use App\Models\EdgeNode;
use App\Models\Location;
use App\Models\User;
use Database\Seeders\PermissionSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Passport\Passport;
use Tests\TestCase;

class CameraAdminTest extends TestCase
{
    use RefreshDatabase;

    public function test_admin_can_create_camera_with_encrypted_password_hidden_from_response(): void
    {
        $this->actingCameraAdmin();
        $location = Location::factory()->create();
        $edgeNode = EdgeNode::factory()->for($location)->create();

        $cameraId = $this->postJson('/api/v1/admin/cameras', [
            'location_id' => $location->id,
            'edge_node_id' => $edgeNode->id,
            'name' => 'LPR Entrada',
            'type' => CameraType::Lpr->value,
            'vendor' => CameraVendor::Intelbras->value,
            'host' => '192.168.1.10',
            'port' => 80,
            'channel' => 1,
            'username' => 'admin',
            'password' => 'camera-secret',
            'is_active' => true,
        ])
            ->assertCreated()
            ->assertJsonMissingPath('data.password')
            ->assertJsonMissingPath('data.password_encrypted')
            ->assertJsonPath('data.type', CameraType::Lpr->value)
            ->json('data.id');

        $camera = Camera::query()->findOrFail($cameraId);
        $this->assertNotSame('camera-secret', $camera->getRawOriginal('password_encrypted'));
        $this->assertSame('camera-secret', $camera->password_encrypted);
    }

    public function test_camera_requires_edge_node_from_same_location(): void
    {
        $this->actingCameraAdmin();
        $location = Location::factory()->create();
        $otherLocation = Location::factory()->create();
        $edgeNode = EdgeNode::factory()->for($otherLocation)->create();

        $this->postJson('/api/v1/admin/cameras', [
            'location_id' => $location->id,
            'edge_node_id' => $edgeNode->id,
            'name' => 'LPR Entrada',
            'type' => CameraType::Lpr->value,
            'vendor' => CameraVendor::Intelbras->value,
            'host' => '192.168.1.10',
        ])
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['edge_node_id']);
    }

    public function test_user_without_camera_permission_cannot_list_cameras(): void
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('auditor');
        Passport::actingAs($user, ['admin:read']);

        $this->getJson('/api/v1/admin/cameras')
            ->assertForbidden();
    }

    private function actingCameraAdmin(): void
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('super_admin');
        Passport::actingAs($user, ['admin:read']);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
php artisan test --filter=CameraAdminTest
```

Expected: FAIL with 404 or missing camera classes.

- [ ] **Step 3: Implement camera requests**

Store rules:

```php
'location_id' => ['required', 'integer', 'exists:locations,id'],
'edge_node_id' => ['required', 'integer', 'exists:edge_nodes,id'],
'name' => ['required', 'string', 'max:255'],
'type' => ['required', Rule::enum(CameraType::class)],
'vendor' => ['required', Rule::enum(CameraVendor::class)],
'host' => ['required', 'string', 'max:255'],
'port' => ['sometimes', 'integer', 'min:1', 'max:65535'],
'channel' => ['sometimes', 'integer', 'min:1'],
'username' => ['nullable', 'string', 'max:255'],
'password' => ['nullable', 'string', 'max:255'],
'is_active' => ['sometimes', 'boolean'],
```

Use `withValidator()` or `after()` to add error on `edge_node_id` when the selected edge node is not from `location_id`.

Update rules are the same but use `sometimes` for mutable fields.

- [ ] **Step 4: Implement camera actions**

`CreateCameraAction::execute(array $data): Camera` maps validated `password` to `password_encrypted`, removes `password`, creates camera, and returns `load(['location', 'edgeNode'])`.

`UpdateCameraAction::execute(Camera $camera, array $data): Camera` maps `password` only when present, preserves existing credential when absent, updates camera, and returns `load(['location', 'edgeNode'])`.

- [ ] **Step 5: Implement camera resource and controller**

`CameraResource` returns:

```php
'id', 'uuid', 'name', 'type', 'vendor', 'host', 'port', 'channel', 'username', 'is_active'
```

It includes loaded `location` and `edge_node`, and never returns `password` or `password_encrypted`.

Controller:

- paginates with `with(['location', 'edgeNode'])`;
- uses Store/Update requests and Actions;
- soft deletes with `204`.

- [ ] **Step 6: Register camera routes**

```php
Route::apiResource('cameras', CameraController::class)
    ->middleware('permission:cameras.manage,api');
```

- [ ] **Step 7: Run camera tests**

Run:

```bash
php artisan test --filter=CameraAdminTest
```

Expected: PASS.

- [ ] **Step 8: Commit task**

Run:

```bash
git add app/Actions/Admin/Cameras app/Http/Controllers/Api/V1/Admin/CameraController.php app/Http/Requests/Api/V1/Admin/StoreCameraRequest.php app/Http/Requests/Api/V1/Admin/UpdateCameraRequest.php app/Http/Resources/Api/V1/CameraResource.php routes/api.php tests/Feature/Admin/CameraAdminTest.php
git commit -m "feat: add camera admin api"
```

---

## Task 6: Camera Pair Admin API

**Files:**

- Create: `app/Actions/Admin/CameraPairs/CreateCameraPairAction.php`
- Create: `app/Actions/Admin/CameraPairs/UpdateCameraPairAction.php`
- Create: `app/Http/Controllers/Api/V1/Admin/CameraPairController.php`
- Create: `app/Http/Requests/Api/V1/Admin/StoreCameraPairRequest.php`
- Create: `app/Http/Requests/Api/V1/Admin/UpdateCameraPairRequest.php`
- Create: `app/Http/Resources/Api/V1/CameraPairResource.php`
- Modify: `routes/api.php`
- Test: `tests/Feature/Admin/CameraPairAdminTest.php`

**Interfaces:**

- Consumes: active `Camera` records with type `lpr` and `support`
- Produces REST endpoints under `/api/v1/admin/camera-pairs`
- Produces invariant: pair cameras match requested `location_id` and `edge_node_id`

- [ ] **Step 1: Write failing camera-pair API tests**

Create `tests/Feature/Admin/CameraPairAdminTest.php`:

```php
<?php

namespace Tests\Feature\Admin;

use App\Enums\CameraPairDirection;
use App\Enums\CameraType;
use App\Models\Camera;
use App\Models\EdgeNode;
use App\Models\Location;
use App\Models\User;
use Database\Seeders\PermissionSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Passport\Passport;
use Tests\TestCase;

class CameraPairAdminTest extends TestCase
{
    use RefreshDatabase;

    public function test_admin_can_create_camera_pair_with_lpr_and_support_cameras(): void
    {
        $this->actingCameraAdmin();
        [$location, $edgeNode, $lpr, $support] = $this->cameraSet();

        $this->postJson('/api/v1/admin/camera-pairs', [
            'location_id' => $location->id,
            'edge_node_id' => $edgeNode->id,
            'name' => 'Entrada Principal',
            'lpr_camera_id' => $lpr->id,
            'support_camera_id' => $support->id,
            'direction' => CameraPairDirection::Outbound->value,
            'is_active' => true,
        ])
            ->assertCreated()
            ->assertJsonPath('data.name', 'Entrada Principal')
            ->assertJsonPath('data.lpr_camera.id', $lpr->id)
            ->assertJsonPath('data.support_camera.id', $support->id)
            ->assertJsonPath('data.direction', CameraPairDirection::Outbound->value);
    }

    public function test_camera_pair_rejects_two_lpr_cameras(): void
    {
        $this->actingCameraAdmin();
        [$location, $edgeNode, $lpr] = $this->cameraSet();
        $secondLpr = Camera::factory()->for($location)->for($edgeNode)->create([
            'type' => CameraType::Lpr,
        ]);

        $this->postJson('/api/v1/admin/camera-pairs', [
            'location_id' => $location->id,
            'edge_node_id' => $edgeNode->id,
            'name' => 'Par invalido',
            'lpr_camera_id' => $lpr->id,
            'support_camera_id' => $secondLpr->id,
            'direction' => CameraPairDirection::Unknown->value,
        ])
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['support_camera_id']);
    }

    public function test_camera_pair_rejects_cameras_from_different_locations(): void
    {
        $this->actingCameraAdmin();
        [$location, $edgeNode, $lpr] = $this->cameraSet();
        $otherLocation = Location::factory()->create();
        $otherEdgeNode = EdgeNode::factory()->for($otherLocation)->create();
        $support = Camera::factory()->for($otherLocation)->for($otherEdgeNode)->create([
            'type' => CameraType::Support,
        ]);

        $this->postJson('/api/v1/admin/camera-pairs', [
            'location_id' => $location->id,
            'edge_node_id' => $edgeNode->id,
            'name' => 'Par invalido',
            'lpr_camera_id' => $lpr->id,
            'support_camera_id' => $support->id,
            'direction' => CameraPairDirection::Unknown->value,
        ])
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['support_camera_id']);
    }

    public function test_camera_pair_rejects_inactive_camera(): void
    {
        $this->actingCameraAdmin();
        [$location, $edgeNode, $lpr, $support] = $this->cameraSet();
        $support->update(['is_active' => false]);

        $this->postJson('/api/v1/admin/camera-pairs', [
            'location_id' => $location->id,
            'edge_node_id' => $edgeNode->id,
            'name' => 'Par invalido',
            'lpr_camera_id' => $lpr->id,
            'support_camera_id' => $support->id,
        ])
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['support_camera_id']);
    }

    public function test_user_without_camera_permission_cannot_list_camera_pairs(): void
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('auditor');
        Passport::actingAs($user, ['admin:read']);

        $this->getJson('/api/v1/admin/camera-pairs')
            ->assertForbidden();
    }

    private function cameraSet(): array
    {
        $location = Location::factory()->create();
        $edgeNode = EdgeNode::factory()->for($location)->create();
        $lpr = Camera::factory()->for($location)->for($edgeNode)->create(['type' => CameraType::Lpr]);
        $support = Camera::factory()->for($location)->for($edgeNode)->create(['type' => CameraType::Support]);

        return [$location, $edgeNode, $lpr, $support];
    }

    private function actingCameraAdmin(): void
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole('super_admin');
        Passport::actingAs($user, ['admin:read']);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
php artisan test --filter=CameraPairAdminTest
```

Expected: FAIL with 404 or missing camera-pair classes.

- [ ] **Step 3: Implement camera-pair requests**

Store rules:

```php
'location_id' => ['required', 'integer', 'exists:locations,id'],
'edge_node_id' => ['required', 'integer', 'exists:edge_nodes,id'],
'name' => ['required', 'string', 'max:255'],
'lpr_camera_id' => ['required', 'integer', 'exists:cameras,id'],
'support_camera_id' => ['required', 'integer', 'different:lpr_camera_id', 'exists:cameras,id'],
'direction' => ['sometimes', Rule::enum(CameraPairDirection::class)],
'is_active' => ['sometimes', 'boolean'],
```

Update rules are the same with `sometimes` for mutable fields.

- [ ] **Step 4: Implement camera-pair actions**

`CreateCameraPairAction::execute(array $data): CameraPair` and `UpdateCameraPairAction::execute(CameraPair $pair, array $data): CameraPair` must load selected cameras and validate:

- `lpr_camera_id` points to active type `CameraType::Lpr`;
- `support_camera_id` points to active type `CameraType::Support`;
- both cameras have matching `location_id`;
- both cameras have matching `edge_node_id`;
- matching ids equal the pair payload `location_id` and `edge_node_id`.

On validation failure throw:

```php
ValidationException::withMessages([
    'support_camera_id' => ['The support camera must be active, type support, and belong to the same location and edge node.'],
]);
```

Use `lpr_camera_id` for LPR-specific failures.

- [ ] **Step 5: Implement resource and controller**

`CameraPairResource` returns:

- `id`
- `uuid`
- `name`
- `direction`
- `is_active`
- loaded `location`
- loaded `edge_node`
- loaded `lpr_camera`
- loaded `support_camera`

Controller eager-loads `location`, `edgeNode`, `lprCamera`, and `supportCamera`; uses Store/Update requests and Actions; soft deletes with `204`.

- [ ] **Step 6: Register camera-pair routes**

```php
Route::apiResource('camera-pairs', CameraPairController::class)
    ->middleware('permission:cameras.manage,api');
```

- [ ] **Step 7: Run camera-pair tests**

Run:

```bash
php artisan test --filter=CameraPairAdminTest
```

Expected: PASS.

- [ ] **Step 8: Commit task**

Run:

```bash
git add app/Actions/Admin/CameraPairs app/Http/Controllers/Api/V1/Admin/CameraPairController.php app/Http/Requests/Api/V1/Admin/StoreCameraPairRequest.php app/Http/Requests/Api/V1/Admin/UpdateCameraPairRequest.php app/Http/Resources/Api/V1/CameraPairResource.php routes/api.php tests/Feature/Admin/CameraPairAdminTest.php
git commit -m "feat: add camera pair admin api"
```

---

## Task 7: API Contract Documentation

**Files:**

- Create: `RIALMA-TrackVision-Backend/docs/api-parent-admin.md`
- Modify: `RIALMA-TrackVision-Backend/README.md`

**Interfaces:**

- Documents public admin endpoints created in Tasks 3 through 6.
- Documents that camera password is write-only.

- [ ] **Step 1: Create backend docs directory**

Run:

```bash
New-Item -ItemType Directory -Force -Path docs | Out-Null
```

- [ ] **Step 2: Write API contract doc**

Create `docs/api-parent-admin.md` with:

```markdown
# Parent Admin API

Base path: `/api/v1/admin`

Authentication: Passport bearer token for a user.

Authorization:

- `vehicles.manage` for `/vehicles`.
- `cameras.manage` for `/locations`, `/edge-nodes`, `/cameras`, and `/camera-pairs`.

## Vehicles

- `GET /vehicles`
- `POST /vehicles`
- `GET /vehicles/{vehicle}`
- `PATCH /vehicles/{vehicle}`
- `DELETE /vehicles/{vehicle}`

Writable fields: `plate`, `fleet_code`, `description`, `is_active`.
Response fields: `id`, `uuid`, `plate`, `plate_normalized`, `fleet_code`, `description`, `is_active`.

## Locations

Writable fields: `name`, `description`, `is_active`.

## Edge Nodes

Writable fields: `location_id`, `name`, `description`, `is_active`.
`status` starts as `offline` and heartbeat updates it in a future phase.

## Cameras

Writable fields: `location_id`, `edge_node_id`, `name`, `type`, `vendor`, `host`, `port`, `channel`, `username`, `password`, `is_active`.
`password` is write-only. API responses never include `password` or `password_encrypted`.

## Camera Pairs

Writable fields: `location_id`, `edge_node_id`, `name`, `lpr_camera_id`, `support_camera_id`, `direction`, `is_active`.
A valid pair requires one active `lpr` camera and one active `support` camera from the same location and edge node.
```

- [ ] **Step 3: Link doc from README**

Add one bullet to `README.md`:

```markdown
- Parent Admin API: `docs/api-parent-admin.md`
```

- [ ] **Step 4: Commit documentation**

Run:

```bash
git add docs/api-parent-admin.md README.md
git commit -m "docs: add parent admin api contract"
```

---

## Task 8: Full Verification And Final Cleanup

**Files:**

- Modify only files needed by Pint or failing tests.

**Interfaces:**

- Produces a clean backend branch ready for review.

- [ ] **Step 1: Run scoped admin test suite**

Run:

```bash
php artisan test --filter=Admin
```

Expected: PASS.

- [ ] **Step 2: Run full backend test suite**

Run:

```bash
php artisan test
```

Expected: PASS.

- [ ] **Step 3: Validate Composer**

Run:

```bash
composer validate
```

Expected: `./composer.json is valid`.

- [ ] **Step 4: Check formatting**

Run:

```bash
vendor\bin\pint --test
```

Expected: PASS. If it fails, run `vendor\bin\pint`, review the diff, rerun this step and the affected tests.

- [ ] **Step 5: Inspect route list**

Run:

```bash
php artisan route:list --path=api/v1/admin
```

Expected: routes for users, roles, permissions, vehicles, locations, edge-nodes, cameras, and camera-pairs.

- [ ] **Step 6: Inspect Git status**

Run:

```bash
git status --short --branch
```

Expected: clean branch `codex/parent-admin-domain`.

---

## Self-Review Checklist

- Spec coverage: all approved entities, routes, permissions, validations, and secret-handling requirements map to tasks.
- Placeholder scan: the plan contains concrete file names, commands, rules, and expected outcomes.
- Type consistency: enum case names, route names, model names, request names, and resource names match across tasks.
- Scope control: Intelbras runtime integration, media, edge sync, trips, reports, and frontend remain outside this phase.
