# Trip Reports And Audit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build audited load-status review plus CSV/PDF trip exports with private evidence images.

**Architecture:** The Laravel backend records immutable audit rows inside the existing load-status transaction, then exposes report endpoints that reuse validated trip filters and eager-loaded report data. CSV generation streams rows directly, while PDF generation uses a dedicated renderer that embeds private storage images as data URIs. The Vue frontend adds authenticated report downloads, date/direction filters, and a compact audit timeline in the existing `/trips` workflow.

**Tech Stack:** Laravel 13, PHP 8.4, Laravel Passport, Spatie Laravel Permission, Eloquent, streamed responses, dompdf/dompdf, Vue 3, Vite, TypeScript, Pinia, Vue Router, Vitest, Vue Test Utils.

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
- Auditoria e criada pelo backend, nao pelo frontend.
- Auditoria nao pode ser editada ou apagada por endpoint admin v1.
- `reports.view` e obrigatoria para CSV/PDF.
- `trips.manage` continua obrigatoria para alterar carga.
- Exportacao nao deve revelar paths internos de storage.
- PDF nao deve carregar assets remotos.
- CSV nao deve incluir tokens ou URLs assinadas.
- A API deve validar periodo e volume antes de gerar PDF pesado.
- Frontend baixa relatorios com Authorization header.
- Token nunca deve ir em query string.

---

## File Structure Map

Backend files:

```text
RIALMA-TrackVision-Backend/
|-- app/
|   |-- Actions/
|   |   |-- Reports/
|   |   |   |-- BuildTripReportCsvAction.php
|   |   |   |-- BuildTripReportPdfAction.php
|   |   |   `-- ListTripReportEventsAction.php
|   |   `-- Trips/
|   |       `-- UpdateLoadStatusAction.php
|   |-- Data/
|   |   `-- Reports/
|   |       `-- TripReportFiltersData.php
|   |-- Http/
|   |   |-- Controllers/Api/V1/Admin/
|   |   |   |-- TripCsvReportController.php
|   |   |   |-- TripPdfReportController.php
|   |   |   |-- TripController.php
|   |   |   `-- TripLoadReviewController.php
|   |   |-- Requests/Api/V1/Admin/
|   |   |   |-- ExportTripCsvReportRequest.php
|   |   |   `-- ExportTripPdfReportRequest.php
|   |   `-- Resources/Api/V1/
|   |       |-- TripEventLoadStatusAuditResource.php
|   |       `-- TripEventResource.php
|   |-- Models/
|   |   |-- TripEvent.php
|   |   |-- TripEventLoadStatusAudit.php
|   |   `-- User.php
|   `-- Services/Reports/
|       |-- TripReportImageEncoder.php
|       `-- TripReportPdfRenderer.php
|-- config/trackvision.php
|-- database/
|   |-- factories/
|   |   `-- TripEventLoadStatusAuditFactory.php
|   `-- migrations/
|       |-- 2026_08_13_210001_add_uuid_to_users_table.php
|       `-- 2026_08_13_210002_create_trip_event_load_status_audits_table.php
|-- docs/api-parent-admin.md
|-- resources/views/reports/trips-pdf.blade.php
|-- routes/api.php
`-- tests/Feature/
    `-- Admin/
        |-- TripAdminTest.php
        `-- TripReportAdminTest.php
```

Frontend files:

```text
RIALMA-TrackVision-Frontend/
|-- src/
|   |-- pages/
|   |   |-- TripsPage.vue
|   |   `-- TripsPage.test.ts
|   |-- services/
|   |   |-- reportsService.test.ts
|   |   |-- reportsService.ts
|   |   |-- tripsService.test.ts
|   |   `-- tripsService.ts
|   `-- types/admin.ts
`-- README.md
```

## Execution Setup

- Backend execution branch: `codex/trip-reports-audit-backend`.
- Frontend execution branch: `codex/trip-reports-audit-frontend`.
- Use isolated worktrees through `superpowers:using-git-worktrees` before implementation.
- Backend baseline before Task 1:

```bash
composer validate
php artisan test
```

- Frontend baseline before Task 5:

```bash
npm test -- --run
npm run build
```

- If baseline fails, stop and report the exact command, exit code, and failing output.

---

## Task 1: Backend Audit Schema, User UUID, And Relationships

**Files:**

- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_13_210001_add_uuid_to_users_table.php`
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_13_210002_create_trip_event_load_status_audits_table.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/TripEventLoadStatusAudit.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/TripEventLoadStatusAuditFactory.php`
- Modify: `RIALMA-TrackVision-Backend/app/Models/User.php`
- Modify: `RIALMA-TrackVision-Backend/app/Models/TripEvent.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/UserResource.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Admin/TripAdminTest.php`

**Interfaces:**

- Consumes: existing `TripEvent`, `CaptureEvent`, `User`, `LoadStatus`, `HasPublicUuid`.
- Produces: `TripEventLoadStatusAudit` model with `tripEvent()`, `captureEvent()`, and `user()` relations.
- Produces: `TripEvent::loadStatusAudits(): HasMany`.
- Produces: `User` public `uuid` generated by `HasPublicUuid`.
- Produces: `UserResource` field `uuid`.

- [ ] **Step 1: Write failing audit schema test**

Append this test to `tests/Feature/Admin/TripAdminTest.php`:

```php
public function test_trip_event_load_status_audit_model_links_trip_event_capture_and_user(): void
{
    $user = $this->actingWithRole('operator');
    [, $event, $capture] = $this->tripFixture();

    $audit = \App\Models\TripEventLoadStatusAudit::factory()->create([
        'trip_event_id' => $event->id,
        'capture_event_id' => $capture->id,
        'user_id' => $user->id,
        'old_load_status' => LoadStatus::Unknown,
        'new_load_status' => LoadStatus::Loaded,
        'changed_at' => '2026-08-13T15:00:00Z',
    ]);

    $this->assertNotNull($user->refresh()->uuid);
    $this->assertSame(LoadStatus::Unknown, $audit->refresh()->old_load_status);
    $this->assertSame(LoadStatus::Loaded, $audit->new_load_status);
    $this->assertTrue($audit->tripEvent->is($event));
    $this->assertTrue($audit->captureEvent->is($capture));
    $this->assertTrue($audit->user->is($user));
    $this->assertTrue($event->loadStatusAudits->first()->is($audit));
}
```

- [ ] **Step 2: Run the audit schema test and verify RED**

Run:

```bash
php artisan test --filter=trip_event_load_status_audit_model_links_trip_event_capture_and_user
```

Expected: FAIL because `TripEventLoadStatusAudit`, factory, table, user `uuid`, and `TripEvent::loadStatusAudits()` do not exist.

- [ ] **Step 3: Add UUID support to users**

Create `database/migrations/2026_08_13_210001_add_uuid_to_users_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Str;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table): void {
            $table->uuid('uuid')->nullable()->unique()->after('id');
        });

        DB::table('users')
            ->whereNull('uuid')
            ->orderBy('id')
            ->lazyById()
            ->each(fn (object $user): int => DB::table('users')
                ->where('id', $user->id)
                ->update(['uuid' => (string) Str::uuid()]));
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table): void {
            $table->dropUnique(['uuid']);
            $table->dropColumn('uuid');
        });
    }
};
```

Modify `app/Models/User.php`:

```php
use App\Models\Concerns\HasPublicUuid;
```

Change the class body traits:

```php
use HasApiTokens, HasFactory, HasPublicUuid, HasRoles, Notifiable;
```

Change fillable:

```php
#[Fillable(['uuid', 'name', 'email', 'password'])]
```

Modify `app/Http/Resources/Api/V1/UserResource.php`:

```php
'uuid' => $this->uuid,
```

Place `uuid` immediately after `id`.

- [ ] **Step 4: Create audit table migration**

Create `database/migrations/2026_08_13_210002_create_trip_event_load_status_audits_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('trip_event_load_status_audits', function (Blueprint $table): void {
            $table->id();
            $table->uuid('uuid')->unique();
            $table->foreignId('trip_event_id')->constrained()->cascadeOnUpdate()->cascadeOnDelete();
            $table->foreignId('capture_event_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
            $table->foreignId('user_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
            $table->string('old_load_status', 40);
            $table->string('new_load_status', 40);
            $table->timestamp('changed_at')->index();
            $table->timestamps();

            $table->index(['trip_event_id', 'changed_at']);
            $table->index('capture_event_id');
            $table->index('user_id');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('trip_event_load_status_audits');
    }
};
```

- [ ] **Step 5: Create audit model and relationship**

Create `app/Models/TripEventLoadStatusAudit.php`:

```php
<?php

namespace App\Models;

use App\Enums\LoadStatus;
use App\Models\Concerns\HasPublicUuid;
use Database\Factories\TripEventLoadStatusAuditFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Fillable(['uuid', 'trip_event_id', 'capture_event_id', 'user_id', 'old_load_status', 'new_load_status', 'changed_at'])]
class TripEventLoadStatusAudit extends Model
{
    /** @use HasFactory<TripEventLoadStatusAuditFactory> */
    use HasFactory, HasPublicUuid;

    protected static function booted(): void
    {
        static::updating(fn (): bool => false);
        static::deleting(fn (): bool => false);
    }

    protected function casts(): array
    {
        return [
            'old_load_status' => LoadStatus::class,
            'new_load_status' => LoadStatus::class,
            'changed_at' => 'immutable_datetime',
        ];
    }

    public function tripEvent(): BelongsTo
    {
        return $this->belongsTo(TripEvent::class);
    }

    public function captureEvent(): BelongsTo
    {
        return $this->belongsTo(CaptureEvent::class);
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

Modify `app/Models/TripEvent.php` imports:

```php
use Illuminate\Database\Eloquent\Relations\HasMany;
```

Add this relation:

```php
public function loadStatusAudits(): HasMany
{
    return $this->hasMany(TripEventLoadStatusAudit::class)->latest('changed_at');
}
```

- [ ] **Step 6: Create audit factory**

Create `database/factories/TripEventLoadStatusAuditFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Enums\LoadStatus;
use App\Models\TripEvent;
use App\Models\TripEventLoadStatusAudit;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<TripEventLoadStatusAudit>
 */
class TripEventLoadStatusAuditFactory extends Factory
{
    public function definition(): array
    {
        $event = TripEvent::factory()->create();

        return [
            'trip_event_id' => $event->id,
            'capture_event_id' => $event->capture_event_id,
            'user_id' => User::factory(),
            'old_load_status' => LoadStatus::Unknown,
            'new_load_status' => LoadStatus::Loaded,
            'changed_at' => now(),
        ];
    }
}
```

- [ ] **Step 7: Run audit schema test and verify GREEN**

Run:

```bash
php artisan test --filter=trip_event_load_status_audit_model_links_trip_event_capture_and_user
```

Expected: PASS.

- [ ] **Step 8: Commit backend audit schema**

Run:

```bash
git add app/Models/User.php app/Models/TripEvent.php app/Models/TripEventLoadStatusAudit.php app/Http/Resources/Api/V1/UserResource.php database/factories/TripEventLoadStatusAuditFactory.php database/migrations/2026_08_13_210001_add_uuid_to_users_table.php database/migrations/2026_08_13_210002_create_trip_event_load_status_audits_table.php tests/Feature/Admin/TripAdminTest.php
git commit -m "feat: add load status audit schema"
```

---

## Task 2: Backend Audited Load Status Updates

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/TripEventLoadStatusAuditResource.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Trips/UpdateLoadStatusAction.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/TripLoadReviewController.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/TripController.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/TripEventResource.php`
- Modify: `RIALMA-TrackVision-Backend/tests/Feature/Admin/TripAdminTest.php`

**Interfaces:**

- Consumes: `TripEventLoadStatusAudit`.
- Produces: `UpdateLoadStatusAction::execute(TripEvent $tripEvent, LoadStatus $loadStatus, User $actor): TripEvent`.
- Produces: `TripEventLoadStatusAuditResource`.
- Produces: `TripEventResource` field `load_status_audits` when relation is loaded.

- [ ] **Step 1: Write failing audited update tests**

Append to `tests/Feature/Admin/TripAdminTest.php`:

```php
public function test_load_status_update_creates_audit_with_actor_and_old_new_values(): void
{
    $user = $this->actingWithRole('operator');
    [, $event, $capture] = $this->tripFixture();

    $this->patchJson("/api/v1/admin/trip-events/{$event->id}/load-status", [
        'load_status' => 'loaded',
    ])
        ->assertOk()
        ->assertJsonPath('data.load_status', 'loaded')
        ->assertJsonPath('data.load_status_audits.0.old_load_status', 'unknown')
        ->assertJsonPath('data.load_status_audits.0.new_load_status', 'loaded')
        ->assertJsonPath('data.load_status_audits.0.user.id', $user->id)
        ->assertJsonPath('data.load_status_audits.0.user.uuid', $user->uuid);

    $this->assertDatabaseHas('trip_event_load_status_audits', [
        'trip_event_id' => $event->id,
        'capture_event_id' => $capture->id,
        'user_id' => $user->id,
        'old_load_status' => 'unknown',
        'new_load_status' => 'loaded',
    ]);
}

public function test_repeating_same_load_status_does_not_create_duplicate_audit(): void
{
    $this->actingWithRole('operator');
    [, $event] = $this->tripFixture();

    $this->patchJson("/api/v1/admin/trip-events/{$event->id}/load-status", ['load_status' => 'loaded'])->assertOk();
    $this->patchJson("/api/v1/admin/trip-events/{$event->id}/load-status", ['load_status' => 'loaded'])->assertOk();

    $this->assertSame(1, \App\Models\TripEventLoadStatusAudit::query()->where('trip_event_id', $event->id)->count());
}

public function test_forbidden_load_status_update_does_not_create_audit(): void
{
    $this->actingWithRole('auditor');
    [, $event] = $this->tripFixture();

    $this->patchJson("/api/v1/admin/trip-events/{$event->id}/load-status", [
        'load_status' => 'loaded',
    ])->assertForbidden();

    $this->assertDatabaseCount('trip_event_load_status_audits', 0);
}

public function test_trip_detail_returns_load_status_audits_ordered_newest_first(): void
{
    $user = $this->actingWithRole('auditor');
    [$trip, $event, $capture] = $this->tripFixture();

    \App\Models\TripEventLoadStatusAudit::factory()->create([
        'trip_event_id' => $event->id,
        'capture_event_id' => $capture->id,
        'user_id' => $user->id,
        'old_load_status' => LoadStatus::Unknown,
        'new_load_status' => LoadStatus::NeedsReview,
        'changed_at' => '2026-08-13T14:00:00Z',
    ]);
    \App\Models\TripEventLoadStatusAudit::factory()->create([
        'trip_event_id' => $event->id,
        'capture_event_id' => $capture->id,
        'user_id' => $user->id,
        'old_load_status' => LoadStatus::NeedsReview,
        'new_load_status' => LoadStatus::Loaded,
        'changed_at' => '2026-08-13T15:00:00Z',
    ]);

    $this->getJson("/api/v1/admin/trips/{$trip->id}")
        ->assertOk()
        ->assertJsonPath('data.events.0.load_status_audits.0.new_load_status', 'loaded')
        ->assertJsonPath('data.events.0.load_status_audits.1.new_load_status', 'needs_review');
}
```

- [ ] **Step 2: Run audited update tests and verify RED**

Run:

```bash
php artisan test --filter='load_status_update_creates_audit|repeating_same_load_status|forbidden_load_status_update|trip_detail_returns_load_status_audits'
```

Expected: FAIL because the action signature, audit creation, resource, and eager loading are not implemented.

- [ ] **Step 3: Create audit resource**

Create `app/Http/Resources/Api/V1/TripEventLoadStatusAuditResource.php`:

```php
<?php

namespace App\Http\Resources\Api\V1;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class TripEventLoadStatusAuditResource extends JsonResource
{
    /**
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        $this->resource->loadMissing('user');

        return [
            'id' => $this->id,
            'uuid' => $this->uuid,
            'old_load_status' => $this->old_load_status?->value,
            'new_load_status' => $this->new_load_status?->value,
            'changed_at' => $this->changed_at?->toJSON(),
            'user' => [
                'id' => $this->user?->id,
                'uuid' => $this->user?->uuid,
                'name' => $this->user?->name,
                'email' => $this->user?->email,
            ],
        ];
    }
}
```

- [ ] **Step 4: Make the load update action transactional and audited**

Modify `app/Actions/Trips/UpdateLoadStatusAction.php` to this shape:

```php
<?php

namespace App\Actions\Trips;

use App\Enums\LoadStatus;
use App\Models\TripEvent;
use App\Models\TripEventLoadStatusAudit;
use App\Models\User;
use Illuminate\Support\Facades\DB;

class UpdateLoadStatusAction
{
    public function execute(TripEvent $tripEvent, LoadStatus $loadStatus, User $actor): TripEvent
    {
        return DB::transaction(function () use ($tripEvent, $loadStatus, $actor): TripEvent {
            /** @var TripEvent $lockedEvent */
            $lockedEvent = TripEvent::query()
                ->with('captureEvent')
                ->whereKey($tripEvent->getKey())
                ->lockForUpdate()
                ->firstOrFail();

            $oldLoadStatus = $lockedEvent->load_status;

            if ($oldLoadStatus !== $loadStatus) {
                TripEventLoadStatusAudit::query()->create([
                    'trip_event_id' => $lockedEvent->id,
                    'capture_event_id' => $lockedEvent->capture_event_id,
                    'user_id' => $actor->id,
                    'old_load_status' => $oldLoadStatus,
                    'new_load_status' => $loadStatus,
                    'changed_at' => now(),
                ]);

                $lockedEvent->update(['load_status' => $loadStatus]);
                $lockedEvent->captureEvent()->update(['load_status' => $loadStatus]);
            }

            return $lockedEvent->refresh()->load([
                'captureEvent.mediaAssets',
                'captureEvent.cameraPair',
                'loadStatusAudits.user',
            ]);
        });
    }
}
```

- [ ] **Step 5: Pass actor from controller**

Modify `app/Http/Controllers/Api/V1/Admin/TripLoadReviewController.php`:

```php
return TripEventResource::make(
    $action->execute(
        $tripEvent,
        LoadStatus::from($request->validated('load_status')),
        $request->user(),
    ),
);
```

- [ ] **Step 6: Eager load audits in trip detail and serialize them**

Modify `app/Http/Controllers/Api/V1/Admin/TripController.php` show load list:

```php
$trip->load([
    'vehicle',
    'location',
    'events' => fn ($query) => $query->orderBy('occurred_at'),
    'events.captureEvent.cameraPair',
    'events.captureEvent.mediaAssets',
    'events.loadStatusAudits.user',
])
```

Modify `app/Http/Resources/Api/V1/TripEventResource.php` return array:

```php
'load_status_audits' => TripEventLoadStatusAuditResource::collection($this->whenLoaded('loadStatusAudits')),
```

Place it after `load_status`.

- [ ] **Step 7: Run audited update tests and verify GREEN**

Run:

```bash
php artisan test --filter='load_status_update_creates_audit|repeating_same_load_status|forbidden_load_status_update|trip_detail_returns_load_status_audits'
```

Expected: PASS.

- [ ] **Step 8: Run existing trip/admin regression tests**

Run:

```bash
php artisan test tests/Feature/Admin/TripAdminTest.php tests/Feature/Trips/TripClassificationTest.php
```

Expected: PASS.

- [ ] **Step 9: Commit audited update task**

Run:

```bash
git add app/Actions/Trips/UpdateLoadStatusAction.php app/Http/Controllers/Api/V1/Admin/TripController.php app/Http/Controllers/Api/V1/Admin/TripLoadReviewController.php app/Http/Resources/Api/V1/TripEventLoadStatusAuditResource.php app/Http/Resources/Api/V1/TripEventResource.php tests/Feature/Admin/TripAdminTest.php
git commit -m "feat: audit trip load status changes"
```

---

## Task 3: Backend CSV Trip Report

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Data/Reports/TripReportFiltersData.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/ExportTripCsvReportRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Reports/ListTripReportEventsAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Reports/BuildTripReportCsvAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/TripCsvReportController.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Admin/TripReportAdminTest.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Trips/ListTripsAction.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/IndexTripRequest.php`
- Modify: `RIALMA-TrackVision-Backend/tests/Feature/Admin/TripAdminTest.php`
- Modify: `RIALMA-TrackVision-Backend/config/trackvision.php`
- Modify: `RIALMA-TrackVision-Backend/routes/api.php`

**Interfaces:**

- Produces: `TripReportFiltersData::fromArray(array $data): self`.
- Produces: `ListTripReportEventsAction::execute(TripReportFiltersData $filters, int $maxItems, bool $limitByTrips = false): Collection`.
- Produces: `BuildTripReportCsvAction::execute(Collection $events): string`.
- Produces: `GET /api/v1/admin/reports/trips.csv`.

- [ ] **Step 1: Write failing direction filter and CSV report tests**

Append this list filter test to `tests/Feature/Admin/TripAdminTest.php`:

```php
public function test_trip_list_filters_by_direction(): void
{
    $this->actingWithRole('auditor');
    [$trip] = $this->tripFixture();
    $inboundCapture = CaptureEvent::factory()->create([
        'direction' => CameraPairDirection::Inbound,
    ]);
    $otherTrip = Trip::factory()->create([
        'vehicle_id' => $inboundCapture->vehicle_id,
        'location_id' => $inboundCapture->cameraPair->location_id,
        'status' => TripStatus::Closed,
    ]);
    TripEvent::factory()->create([
        'trip_id' => $otherTrip->id,
        'capture_event_id' => $inboundCapture->id,
        'direction' => CameraPairDirection::Inbound,
    ]);

    $this->getJson('/api/v1/admin/trips?direction=outbound')
        ->assertOk()
        ->assertJsonCount(1, 'data')
        ->assertJsonPath('data.0.id', $trip->id);
}
```

Create `tests/Feature/Admin/TripReportAdminTest.php`:

```php
<?php

namespace Tests\Feature\Admin;

use App\Enums\CameraPairDirection;
use App\Enums\LoadStatus;
use App\Enums\MediaAssetKind;
use App\Enums\TripStatus;
use App\Models\CaptureEvent;
use App\Models\MediaAsset;
use App\Models\Trip;
use App\Models\TripEvent;
use App\Models\TripEventLoadStatusAudit;
use App\Models\User;
use Database\Seeders\PermissionSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Passport\Passport;
use Tests\TestCase;

class TripReportAdminTest extends TestCase
{
    use RefreshDatabase;

    public function test_csv_report_requires_reports_view_permission(): void
    {
        $this->actingWithoutReportPermission();

        $this->get('/api/v1/admin/reports/trips.csv?date_from=2026-08-01&date_to=2026-08-31')
            ->assertForbidden();
    }

    public function test_csv_report_requires_date_range(): void
    {
        $this->actingWithRole('auditor');

        $this->get('/api/v1/admin/reports/trips.csv')
            ->assertUnprocessable()
            ->assertJsonValidationErrors(['date_from', 'date_to']);
    }

    public function test_csv_report_streams_one_row_per_trip_event(): void
    {
        $user = $this->actingWithRole('auditor');
        [$trip, $event, $capture] = $this->tripFixture();
        MediaAsset::factory()->create([
            'capture_event_id' => $capture->id,
            'camera_id' => $capture->lpr_camera_id,
            'kind' => MediaAssetKind::LprImage,
            'disk' => 'local',
            'path' => 'private/lpr.jpg',
            'content_type' => 'image/jpeg',
            'byte_size' => 10,
        ]);
        TripEventLoadStatusAudit::factory()->create([
            'trip_event_id' => $event->id,
            'capture_event_id' => $capture->id,
            'user_id' => $user->id,
            'old_load_status' => LoadStatus::Unknown,
            'new_load_status' => LoadStatus::Loaded,
            'changed_at' => '2026-08-13T15:00:00Z',
        ]);

        $response = $this->get('/api/v1/admin/reports/trips.csv?date_from=2026-08-01&date_to=2026-08-31');

        $response->assertOk();
        $response->assertHeader('content-type', 'text/csv; charset=UTF-8');
        $csv = $response->streamedContent();

        $this->assertStringContainsString('trip_uuid,vehicle_plate,vehicle_fleet_code,location_name', $csv);
        $this->assertStringContainsString($trip->uuid, $csv);
        $this->assertStringContainsString($event->uuid, $csv);
        $this->assertStringContainsString('true,false', $csv);
        $this->assertStringContainsString($user->email, $csv);
        $this->assertStringNotContainsString('private/lpr.jpg', $csv);
    }

    public function test_csv_report_rejects_ranges_longer_than_180_days(): void
    {
        $this->actingWithRole('auditor');

        $this->get('/api/v1/admin/reports/trips.csv?date_from=2026-01-01&date_to=2026-08-01')
            ->assertUnprocessable()
            ->assertJsonValidationErrors('date_to');
    }

    private function actingWithRole(string $role): User
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole($role);
        Passport::actingAs($user, ['admin:read']);

        return $user;
    }

    private function actingWithoutReportPermission(): User
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        Passport::actingAs($user, ['admin:read']);

        return $user;
    }

    private function tripFixture(): array
    {
        $capture = CaptureEvent::factory()->create([
            'direction' => CameraPairDirection::Outbound,
            'load_status' => LoadStatus::Loaded,
            'event_time' => '2026-08-13T10:00:00Z',
        ]);
        $trip = Trip::factory()->create([
            'vehicle_id' => $capture->vehicle_id,
            'location_id' => $capture->cameraPair->location_id,
            'status' => TripStatus::Closed,
            'opened_at' => '2026-08-13T10:00:00Z',
            'closed_at' => '2026-08-13T12:00:00Z',
        ]);
        $event = TripEvent::factory()->create([
            'trip_id' => $trip->id,
            'capture_event_id' => $capture->id,
            'direction' => CameraPairDirection::Outbound,
            'load_status' => LoadStatus::Loaded,
            'occurred_at' => '2026-08-13T10:00:00Z',
        ]);

        return [$trip, $event, $capture];
    }
}
```

- [ ] **Step 2: Run direction filter and CSV report tests and verify RED**

Run:

```bash
php artisan test --filter=trip_list_filters_by_direction
php artisan test tests/Feature/Admin/TripReportAdminTest.php --filter=csv
```

Expected: the direction filter test FAILS because `direction` is not validated or applied yet, and CSV tests FAIL because report route, request, actions, and controller do not exist.

- [ ] **Step 3: Extend trip list direction filter**

Modify `app/Http/Requests/Api/V1/Admin/IndexTripRequest.php` imports:

```php
use App\Enums\CameraPairDirection;
```

Add to rules:

```php
'direction' => ['sometimes', Rule::enum(CameraPairDirection::class)],
```

Modify `app/Actions/Trips/ListTripsAction.php` before date filters:

```php
->when($filters['direction'] ?? null, fn ($query, $direction) => $query->whereHas(
    'events',
    fn ($eventQuery) => $eventQuery->where('direction', $direction),
))
```

- [ ] **Step 4: Add report config limits**

Modify `config/trackvision.php` before `admin`:

```php
'reports_pdf_max_trips' => (int) env('TRACKVISION_REPORTS_PDF_MAX_TRIPS', 100),

'reports_csv_max_rows' => (int) env('TRACKVISION_REPORTS_CSV_MAX_ROWS', 5000),
```

- [ ] **Step 5: Create filter DTO and CSV Form Request**

Create `app/Data/Reports/TripReportFiltersData.php`:

```php
<?php

namespace App\Data\Reports;

use Carbon\CarbonImmutable;

final readonly class TripReportFiltersData
{
    public function __construct(
        public CarbonImmutable $dateFrom,
        public CarbonImmutable $dateTo,
        public ?string $status,
        public ?string $plate,
        public ?int $vehicleId,
        public ?int $locationId,
        public ?string $loadStatus,
        public ?string $direction,
    ) {}

    /**
     * @param array<string, mixed> $data
     */
    public static function fromArray(array $data): self
    {
        return new self(
            dateFrom: CarbonImmutable::parse($data['date_from'])->startOfDay(),
            dateTo: CarbonImmutable::parse($data['date_to'])->endOfDay(),
            status: $data['status'] ?? null,
            plate: $data['plate'] ?? null,
            vehicleId: isset($data['vehicle_id']) ? (int) $data['vehicle_id'] : null,
            locationId: isset($data['location_id']) ? (int) $data['location_id'] : null,
            loadStatus: $data['load_status'] ?? null,
            direction: $data['direction'] ?? null,
        );
    }
}
```

Create `app/Http/Requests/Api/V1/Admin/ExportTripCsvReportRequest.php`:

```php
<?php

namespace App\Http\Requests\Api\V1\Admin;

use App\Data\Reports\TripReportFiltersData;
use App\Enums\CameraPairDirection;
use App\Enums\LoadStatus;
use App\Enums\TripStatus;
use Carbon\CarbonImmutable;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Validator;

class ExportTripCsvReportRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('reports.view') === true;
    }

    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        return [
            'date_from' => ['required', 'date'],
            'date_to' => ['required', 'date', 'after_or_equal:date_from'],
            'status' => ['sometimes', Rule::enum(TripStatus::class)],
            'plate' => ['sometimes', 'string', 'max:20'],
            'vehicle_id' => ['sometimes', 'integer', 'exists:vehicles,id'],
            'location_id' => ['sometimes', 'integer', 'exists:locations,id'],
            'load_status' => ['sometimes', Rule::enum(LoadStatus::class)],
            'direction' => ['sometimes', Rule::enum(CameraPairDirection::class)],
        ];
    }

    public function after(): array
    {
        return [
            function (Validator $validator): void {
                if ($validator->errors()->isNotEmpty()) {
                    return;
                }

                $dateFrom = CarbonImmutable::parse($this->input('date_from'));
                $dateTo = CarbonImmutable::parse($this->input('date_to'));

                if ($dateFrom->diffInDays($dateTo) > 180) {
                    $validator->errors()->add('date_to', 'O periodo maximo para CSV e de 180 dias.');
                }
            },
        ];
    }

    public function filters(): TripReportFiltersData
    {
        return TripReportFiltersData::fromArray($this->validated());
    }
}
```

- [ ] **Step 6: Create report query action**

Create `app/Actions/Reports/ListTripReportEventsAction.php`:

```php
<?php

namespace App\Actions\Reports;

use App\Data\Reports\TripReportFiltersData;
use App\Models\TripEvent;
use App\Support\Vehicles\NormalizePlate;
use Illuminate\Support\Collection;
use Symfony\Component\HttpKernel\Exception\UnprocessableEntityHttpException;

class ListTripReportEventsAction
{
    public function __construct(private readonly NormalizePlate $normalizePlate) {}

    /**
     * @return Collection<int, TripEvent>
     */
    public function execute(TripReportFiltersData $filters, int $maxItems, bool $limitByTrips = false): Collection
    {
        $query = TripEvent::query()
            ->with([
                'trip.vehicle',
                'trip.location',
                'captureEvent.mediaAssets',
                'loadStatusAudits.user',
            ])
            ->whereHas('trip', fn ($tripQuery) => $tripQuery
                ->whereBetween('opened_at', [$filters->dateFrom, $filters->dateTo])
                ->when($filters->status, fn ($q, $status) => $q->where('status', $status))
                ->when($filters->vehicleId, fn ($q, $vehicleId) => $q->where('vehicle_id', $vehicleId))
                ->when($filters->locationId, fn ($q, $locationId) => $q->where('location_id', $locationId))
                ->when($filters->plate, fn ($q, $plate) => $q->whereHas(
                    'vehicle',
                    fn ($vehicleQuery) => $vehicleQuery->where('plate_normalized', 'like', '%'.($this->normalizePlate)($plate).'%'),
                )))
            ->when($filters->loadStatus, fn ($q, $loadStatus) => $q->where('load_status', $loadStatus))
            ->when($filters->direction, fn ($q, $direction) => $q->where('direction', $direction))
            ->orderBy('occurred_at');

        $count = $limitByTrips
            ? (clone $query)->select('trip_id')->distinct()->count('trip_id')
            : (clone $query)->count();

        if ($count > $maxItems) {
            $unit = $limitByTrips ? 'viagens' : 'linhas';

            throw new UnprocessableEntityHttpException("O relatorio encontrou {$count} {$unit}; refine os filtros.");
        }

        return $query->get();
    }
}
```

- [ ] **Step 7: Create CSV build action**

Create `app/Actions/Reports/BuildTripReportCsvAction.php`:

```php
<?php

namespace App\Actions\Reports;

use App\Enums\MediaAssetKind;
use App\Models\TripEvent;
use Illuminate\Support\Collection;

class BuildTripReportCsvAction
{
    /**
     * @param Collection<int, TripEvent> $events
     */
    public function execute(Collection $events): string
    {
        $handle = fopen('php://temp', 'r+');

        fputcsv($handle, [
            'trip_uuid',
            'vehicle_plate',
            'vehicle_fleet_code',
            'location_name',
            'trip_status',
            'trip_opened_at',
            'trip_closed_at',
            'review_required_reason',
            'event_uuid',
            'event_direction',
            'event_occurred_at',
            'event_load_status',
            'capture_plate',
            'capture_plate_normalized',
            'lpr_image_available',
            'support_image_available',
            'last_load_reviewed_by',
            'last_load_reviewed_at',
        ]);

        foreach ($events as $event) {
            $trip = $event->trip;
            $capture = $event->captureEvent;
            $lastAudit = $event->loadStatusAudits->sortByDesc('changed_at')->first();

            fputcsv($handle, [
                $trip->uuid,
                $trip->vehicle?->plate,
                $trip->vehicle?->fleet_code,
                $trip->location?->name,
                $trip->status?->value,
                $trip->opened_at?->toJSON(),
                $trip->closed_at?->toJSON(),
                $trip->review_required_reason,
                $event->uuid,
                $event->direction?->value,
                $event->occurred_at?->toJSON(),
                $event->load_status?->value,
                $capture->plate,
                $capture->plate_normalized,
                $capture->mediaAssets->contains(fn ($asset): bool => $asset->kind === MediaAssetKind::LprImage) ? 'true' : 'false',
                $capture->mediaAssets->contains(fn ($asset): bool => $asset->kind === MediaAssetKind::SupportImage) ? 'true' : 'false',
                $lastAudit?->user?->email,
                $lastAudit?->changed_at?->toJSON(),
            ]);
        }

        rewind($handle);

        return stream_get_contents($handle);
    }
}
```

- [ ] **Step 8: Create CSV controller and route**

Create `app/Http/Controllers/Api/V1/Admin/TripCsvReportController.php`:

```php
<?php

namespace App\Http\Controllers\Api\V1\Admin;

use App\Actions\Reports\BuildTripReportCsvAction;
use App\Actions\Reports\ListTripReportEventsAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Api\V1\Admin\ExportTripCsvReportRequest;
use Symfony\Component\HttpFoundation\StreamedResponse;

class TripCsvReportController extends Controller
{
    public function __invoke(
        ExportTripCsvReportRequest $request,
        ListTripReportEventsAction $listEvents,
        BuildTripReportCsvAction $buildCsv,
    ): StreamedResponse {
        $events = $listEvents->execute($request->filters(), config('trackvision.reports_csv_max_rows'));
        $csv = $buildCsv->execute($events);
        $filename = 'trackvision-trips-'.now()->format('Ymd-His').'.csv';

        return response()->streamDownload(
            fn (): int|false => print $csv,
            $filename,
            ['Content-Type' => 'text/csv; charset=UTF-8'],
        );
    }
}
```

Modify `routes/api.php` imports:

```php
use App\Http\Controllers\Api\V1\Admin\TripCsvReportController;
```

Add inside the admin group:

```php
Route::get('/reports/trips.csv', TripCsvReportController::class)
    ->middleware('permission:reports.view,api');
```

- [ ] **Step 9: Run direction filter and CSV report tests and verify GREEN**

Run:

```bash
php artisan test --filter=trip_list_filters_by_direction
php artisan test tests/Feature/Admin/TripReportAdminTest.php --filter=csv
```

Expected: PASS.

- [ ] **Step 10: Commit CSV report task**

Run:

```bash
git add app/Data/Reports/TripReportFiltersData.php app/Http/Requests/Api/V1/Admin/ExportTripCsvReportRequest.php app/Actions/Reports/ListTripReportEventsAction.php app/Actions/Reports/BuildTripReportCsvAction.php app/Http/Controllers/Api/V1/Admin/TripCsvReportController.php app/Actions/Trips/ListTripsAction.php app/Http/Requests/Api/V1/Admin/IndexTripRequest.php config/trackvision.php routes/api.php tests/Feature/Admin/TripAdminTest.php tests/Feature/Admin/TripReportAdminTest.php
git commit -m "feat: export trip report csv"
```

---

## Task 4: Backend PDF Trip Report

**Files:**

- Modify: `RIALMA-TrackVision-Backend/composer.json`
- Modify: `RIALMA-TrackVision-Backend/composer.lock`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/ExportTripPdfReportRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/Reports/TripReportImageEncoder.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/Reports/TripReportPdfRenderer.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Reports/BuildTripReportPdfAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/TripPdfReportController.php`
- Create: `RIALMA-TrackVision-Backend/resources/views/reports/trips-pdf.blade.php`
- Modify: `RIALMA-TrackVision-Backend/routes/api.php`
- Modify: `RIALMA-TrackVision-Backend/tests/Feature/Admin/TripReportAdminTest.php`

**Interfaces:**

- Consumes: `ListTripReportEventsAction`.
- Produces: `TripReportImageEncoder::dataUri(?MediaAsset $asset): ?string`.
- Produces: `BuildTripReportPdfAction::execute(Collection $events): string`.
- Produces: `GET /api/v1/admin/reports/trips.pdf`.

- [ ] **Step 1: Write failing PDF report tests**

Append to `tests/Feature/Admin/TripReportAdminTest.php`:

```php
public function test_pdf_report_requires_reports_view_permission(): void
{
    $this->actingWithoutReportPermission();

    $this->get('/api/v1/admin/reports/trips.pdf?date_from=2026-08-01&date_to=2026-08-31')
        ->assertForbidden();
}

public function test_pdf_report_returns_pdf_download_without_storage_paths(): void
{
    \Illuminate\Support\Facades\Storage::fake('local');
    $this->actingWithRole('auditor');
    [, , $capture] = $this->tripFixture();
    $media = MediaAsset::factory()->create([
        'capture_event_id' => $capture->id,
        'camera_id' => $capture->lpr_camera_id,
        'kind' => MediaAssetKind::LprImage,
        'disk' => 'local',
        'path' => 'private/lpr.jpg',
        'content_type' => 'image/jpeg',
        'byte_size' => 10,
    ]);
    \Illuminate\Support\Facades\Storage::disk('local')->put($media->path, $this->jpegBytes());

    $response = $this->get('/api/v1/admin/reports/trips.pdf?date_from=2026-08-01&date_to=2026-08-31');

    $response->assertOk();
    $response->assertHeader('content-type', 'application/pdf');
    $pdf = $response->streamedContent();

    $this->assertStringStartsWith('%PDF', $pdf);
    $this->assertStringNotContainsString('private/lpr.jpg', $pdf);
}

public function test_pdf_report_rejects_ranges_longer_than_31_days(): void
{
    $this->actingWithRole('auditor');

    $this->get('/api/v1/admin/reports/trips.pdf?date_from=2026-08-01&date_to=2026-09-15')
        ->assertUnprocessable()
        ->assertJsonValidationErrors('date_to');
}

public function test_pdf_report_rejects_more_than_configured_trip_limit(): void
{
    config(['trackvision.reports_pdf_max_trips' => 1]);
    $this->actingWithRole('auditor');
    $this->tripFixture();
    $this->tripFixture();

    $this->get('/api/v1/admin/reports/trips.pdf?date_from=2026-08-01&date_to=2026-08-31')
        ->assertUnprocessable();
}

private function jpegBytes(): string
{
    return base64_decode('/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAP//////////////////////////////////////////////////////////////////////////////////////2wBDAf//////////////////////////////////////////////////////////////////////////////////////wAARCAABAAEDASIAAhEBAxEB/8QAFQABAQAAAAAAAAAAAAAAAAAAAAX/xAAUEAEAAAAAAAAAAAAAAAAAAAAA/9oADAMBAAIQAxAAAAH/xAAUEAEAAAAAAAAAAAAAAAAAAAAA/9oACAEBAAEFAqf/xAAUEQEAAAAAAAAAAAAAAAAAAAAA/9oACAEDAQE/ASP/xAAUEQEAAAAAAAAAAAAAAAAAAAAA/9oACAECAQE/ASP/xAAUEAEAAAAAAAAAAAAAAAAAAAAA/9oACAEBAAY/Al//xAAUEAEAAAAAAAAAAAAAAAAAAAAA/9oACAEBAAE/IV//2gAMAwEAAgADAAAAEP/EFBQRAQAAAAAAAAAAAAAAAAAAABD/2gAIAQMBAT8QH//EFBQRAQAAAAAAAAAAAAAAAAAAABD/2gAIAQIBAT8QH//EFBABAQAAAAAAAAAAAAAAAAAAARD/2gAIAQEAAT8QH//Z');
}
```

- [ ] **Step 2: Run PDF tests and verify RED**

Run:

```bash
php artisan test tests/Feature/Admin/TripReportAdminTest.php --filter=pdf
```

Expected: FAIL because PDF request, controller, renderer, and route do not exist.

- [ ] **Step 3: Add dompdf dependency**

Run:

```bash
composer require dompdf/dompdf
```

Expected: `composer.json` and `composer.lock` include `dompdf/dompdf`.

- [ ] **Step 4: Create PDF Form Request**

Create `app/Http/Requests/Api/V1/Admin/ExportTripPdfReportRequest.php`:

```php
<?php

namespace App\Http\Requests\Api\V1\Admin;

use App\Data\Reports\TripReportFiltersData;
use App\Enums\CameraPairDirection;
use App\Enums\LoadStatus;
use App\Enums\TripStatus;
use Carbon\CarbonImmutable;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Validator;

class ExportTripPdfReportRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('reports.view') === true;
    }

    public function rules(): array
    {
        return [
            'date_from' => ['required', 'date'],
            'date_to' => ['required', 'date', 'after_or_equal:date_from'],
            'status' => ['sometimes', Rule::enum(TripStatus::class)],
            'plate' => ['sometimes', 'string', 'max:20'],
            'vehicle_id' => ['sometimes', 'integer', 'exists:vehicles,id'],
            'location_id' => ['sometimes', 'integer', 'exists:locations,id'],
            'load_status' => ['sometimes', Rule::enum(LoadStatus::class)],
            'direction' => ['sometimes', Rule::enum(CameraPairDirection::class)],
        ];
    }

    public function after(): array
    {
        return [
            function (Validator $validator): void {
                if ($validator->errors()->isNotEmpty()) {
                    return;
                }

                $dateFrom = CarbonImmutable::parse($this->input('date_from'));
                $dateTo = CarbonImmutable::parse($this->input('date_to'));

                if ($dateFrom->diffInDays($dateTo) > 31) {
                    $validator->errors()->add('date_to', 'O periodo maximo para PDF e de 31 dias.');
                }
            },
        ];
    }

    public function filters(): TripReportFiltersData
    {
        return TripReportFiltersData::fromArray($this->validated());
    }
}
```

- [ ] **Step 5: Create image encoder**

Create `app/Services/Reports/TripReportImageEncoder.php`:

```php
<?php

namespace App\Services\Reports;

use App\Models\MediaAsset;
use Illuminate\Support\Facades\Storage;

class TripReportImageEncoder
{
    public function dataUri(?MediaAsset $asset): ?string
    {
        if (! $asset || ! str_starts_with($asset->content_type, 'image/')) {
            return null;
        }

        if (! Storage::disk($asset->disk)->exists($asset->path)) {
            return null;
        }

        $bytes = Storage::disk($asset->disk)->get($asset->path);

        return 'data:'.$asset->content_type.';base64,'.base64_encode($bytes);
    }
}
```

- [ ] **Step 6: Create PDF renderer and Blade view**

Create `app/Services/Reports/TripReportPdfRenderer.php`:

```php
<?php

namespace App\Services\Reports;

use Dompdf\Dompdf;
use Dompdf\Options;

class TripReportPdfRenderer
{
    public function render(string $html): string
    {
        $options = new Options;
        $options->set('isRemoteEnabled', false);

        $dompdf = new Dompdf($options);
        $dompdf->loadHtml($html, 'UTF-8');
        $dompdf->setPaper('A4', 'portrait');
        $dompdf->render();

        return $dompdf->output();
    }
}
```

Create `resources/views/reports/trips-pdf.blade.php`:

```blade
<!doctype html>
<html lang="pt-BR">
<head>
    <meta charset="utf-8">
    <style>
        body { color: #111827; font-family: DejaVu Sans, sans-serif; font-size: 11px; }
        h1 { font-size: 20px; margin: 0 0 12px; }
        h2 { border-bottom: 1px solid #d1d5db; font-size: 15px; margin: 18px 0 8px; padding-bottom: 4px; }
        table { border-collapse: collapse; margin-bottom: 10px; width: 100%; }
        th, td { border: 1px solid #d1d5db; padding: 5px; text-align: left; vertical-align: top; }
        th { background: #f3f4f6; }
        .muted { color: #6b7280; }
        .images { display: table; table-layout: fixed; width: 100%; }
        .image-cell { display: table-cell; padding: 4px; width: 50%; }
        .image-cell img { max-height: 180px; max-width: 100%; }
        .missing { border: 1px dashed #9ca3af; color: #6b7280; padding: 28px; text-align: center; }
        .page-break { page-break-after: always; }
    </style>
</head>
<body>
<h1>Relatorio de Viagens TrackVision</h1>
<p class="muted">Gerado em {{ now()->format('d/m/Y H:i:s') }}</p>

@foreach ($trips as $trip)
    <section>
        <h2>{{ $trip['vehicle_plate'] ?? 'Sem placa' }} - {{ $trip['location_name'] ?? 'Sem local' }}</h2>
        <table>
            <tr><th>Status</th><td>{{ $trip['status'] }}</td><th>Abertura</th><td>{{ $trip['opened_at'] }}</td></tr>
            <tr><th>Fechamento</th><td>{{ $trip['closed_at'] ?? '-' }}</td><th>Revisao</th><td>{{ $trip['review_required_reason'] ?? '-' }}</td></tr>
        </table>

        @foreach ($trip['events'] as $event)
            <table>
                <tr><th>Direcao</th><td>{{ $event['direction'] }}</td><th>Horario</th><td>{{ $event['occurred_at'] }}</td></tr>
                <tr><th>Carga</th><td>{{ $event['load_status'] }}</td><th>Placa capturada</th><td>{{ $event['capture_plate'] ?? '-' }}</td></tr>
                <tr><th>Ultima revisao</th><td colspan="3">{{ $event['last_review'] ?? '-' }}</td></tr>
            </table>
            <div class="images">
                <div class="image-cell">
                    <strong>LPR</strong>
                    @if ($event['lpr_image'])
                        <img src="{{ $event['lpr_image'] }}" alt="Imagem LPR">
                    @else
                        <div class="missing">Sem imagem LPR</div>
                    @endif
                </div>
                <div class="image-cell">
                    <strong>Apoio</strong>
                    @if ($event['support_image'])
                        <img src="{{ $event['support_image'] }}" alt="Imagem de apoio">
                    @else
                        <div class="missing">Sem imagem de apoio</div>
                    @endif
                </div>
            </div>
        @endforeach
    </section>
    @if (! $loop->last)
        <div class="page-break"></div>
    @endif
@endforeach
</body>
</html>
```

- [ ] **Step 7: Create PDF build action**

Create `app/Actions/Reports/BuildTripReportPdfAction.php`:

```php
<?php

namespace App\Actions\Reports;

use App\Enums\MediaAssetKind;
use App\Models\TripEvent;
use App\Services\Reports\TripReportImageEncoder;
use App\Services\Reports\TripReportPdfRenderer;
use Illuminate\Support\Collection;

class BuildTripReportPdfAction
{
    public function __construct(
        private readonly TripReportImageEncoder $imageEncoder,
        private readonly TripReportPdfRenderer $renderer,
    ) {}

    /**
     * @param Collection<int, TripEvent> $events
     */
    public function execute(Collection $events): string
    {
        $trips = $events
            ->groupBy('trip_id')
            ->map(function (Collection $tripEvents): array {
                /** @var TripEvent $firstEvent */
                $firstEvent = $tripEvents->first();
                $trip = $firstEvent->trip;

                return [
                    'uuid' => $trip->uuid,
                    'vehicle_plate' => $trip->vehicle?->plate,
                    'location_name' => $trip->location?->name,
                    'status' => $trip->status?->value,
                    'opened_at' => $trip->opened_at?->toJSON(),
                    'closed_at' => $trip->closed_at?->toJSON(),
                    'review_required_reason' => $trip->review_required_reason,
                    'events' => $tripEvents->map(fn (TripEvent $event): array => $this->eventPayload($event))->values()->all(),
                ];
            })
            ->values()
            ->all();

        return $this->renderer->render(view('reports.trips-pdf', ['trips' => $trips])->render());
    }

    private function eventPayload(TripEvent $event): array
    {
        $capture = $event->captureEvent;
        $media = $capture->mediaAssets->keyBy(fn ($asset): string => $asset->kind->value);
        $lastAudit = $event->loadStatusAudits->sortByDesc('changed_at')->first();

        return [
            'uuid' => $event->uuid,
            'direction' => $event->direction?->value,
            'occurred_at' => $event->occurred_at?->toJSON(),
            'load_status' => $event->load_status?->value,
            'capture_plate' => $capture->plate,
            'last_review' => $lastAudit
                ? $lastAudit->user?->email.' em '.$lastAudit->changed_at?->toJSON()
                : null,
            'lpr_image' => $this->imageEncoder->dataUri($media->get(MediaAssetKind::LprImage->value)),
            'support_image' => $this->imageEncoder->dataUri($media->get(MediaAssetKind::SupportImage->value)),
        ];
    }
}
```

- [ ] **Step 8: Create PDF controller and route**

Create `app/Http/Controllers/Api/V1/Admin/TripPdfReportController.php`:

```php
<?php

namespace App\Http\Controllers\Api\V1\Admin;

use App\Actions\Reports\BuildTripReportPdfAction;
use App\Actions\Reports\ListTripReportEventsAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Api\V1\Admin\ExportTripPdfReportRequest;
use Symfony\Component\HttpFoundation\StreamedResponse;

class TripPdfReportController extends Controller
{
    public function __invoke(
        ExportTripPdfReportRequest $request,
        ListTripReportEventsAction $listEvents,
        BuildTripReportPdfAction $buildPdf,
    ): StreamedResponse {
        $events = $listEvents->execute(
            $request->filters(),
            config('trackvision.reports_pdf_max_trips'),
            limitByTrips: true,
        );
        $pdf = $buildPdf->execute($events);
        $filename = 'trackvision-trips-'.now()->format('Ymd-His').'.pdf';

        return response()->streamDownload(
            fn (): int|false => print $pdf,
            $filename,
            ['Content-Type' => 'application/pdf'],
        );
    }
}
```

Modify `routes/api.php` imports:

```php
use App\Http\Controllers\Api\V1\Admin\TripPdfReportController;
```

Add inside the admin group:

```php
Route::get('/reports/trips.pdf', TripPdfReportController::class)
    ->middleware('permission:reports.view,api');
```

- [ ] **Step 9: Run PDF tests and verify GREEN**

Run:

```bash
php artisan test tests/Feature/Admin/TripReportAdminTest.php --filter=pdf
```

Expected: PASS.

- [ ] **Step 10: Run backend report regression suite**

Run:

```bash
composer validate
php artisan test tests/Feature/Admin/TripReportAdminTest.php tests/Feature/Admin/TripAdminTest.php
```

Expected: Composer valid and tests pass.

- [ ] **Step 11: Commit PDF report task**

Run:

```bash
git add composer.json composer.lock app/Http/Requests/Api/V1/Admin/ExportTripPdfReportRequest.php app/Services/Reports/TripReportImageEncoder.php app/Services/Reports/TripReportPdfRenderer.php app/Actions/Reports/BuildTripReportPdfAction.php app/Http/Controllers/Api/V1/Admin/TripPdfReportController.php resources/views/reports/trips-pdf.blade.php routes/api.php tests/Feature/Admin/TripReportAdminTest.php
git commit -m "feat: export trip report pdf"
```

---

## Task 5: Frontend Report Service And Types

**Files:**

- Create: `RIALMA-TrackVision-Frontend/src/services/reportsService.ts`
- Create: `RIALMA-TrackVision-Frontend/src/services/reportsService.test.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/types/admin.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/services/tripsService.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/services/tripsService.test.ts`

**Interfaces:**

- Produces: `TripEventLoadStatusAudit` type.
- Extends: `TripFilters` with `date_from`, `date_to`, `vehicle_id`, `location_id`, and `direction`.
- Produces: `reportsService.downloadCsv(filters: TripFilters): Promise<void>`.
- Produces: `reportsService.downloadPdf(filters: TripFilters): Promise<void>`.

- [ ] **Step 1: Write failing report service tests**

Create `src/services/reportsService.test.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { reportsService } from './reportsService'

const fetchMock = vi.fn()
const clickMock = vi.fn()
vi.stubGlobal('fetch', fetchMock)

describe('reportsService', () => {
  beforeEach(() => {
    localStorage.setItem('trackvision.token', 'token-123')
    fetchMock.mockReset()
    clickMock.mockReset()
    vi.stubGlobal('URL', {
      createObjectURL: vi.fn().mockReturnValue('blob:report-url'),
      revokeObjectURL: vi.fn(),
    })
    vi.spyOn(document, 'createElement').mockReturnValue({
      href: '',
      download: '',
      click: clickMock,
    } as unknown as HTMLAnchorElement)
  })

  it('downloads CSV with current filters and Authorization header', async () => {
    fetchMock.mockResolvedValueOnce(new Response(new Blob(['csv'], { type: 'text/csv' }), { status: 200 }))

    await reportsService.downloadCsv({
      date_from: '2026-08-01',
      date_to: '2026-08-31',
      status: 'closed',
      plate: 'ABC-1D23',
      load_status: 'loaded',
      direction: 'outbound',
    })

    const [url, init] = fetchMock.mock.calls[0]
    expect(url).toContain('/admin/reports/trips.csv?')
    expect(url).toContain('date_from=2026-08-01')
    expect(url).toContain('date_to=2026-08-31')
    expect(url).toContain('status=closed')
    expect(url).toContain('plate=ABC-1D23')
    expect(url).toContain('load_status=loaded')
    expect(url).toContain('direction=outbound')
    expect(url).not.toContain('token-123')
    expect((init.headers as Headers).get('Authorization')).toBe('Bearer token-123')
    expect(clickMock).toHaveBeenCalledOnce()
    expect(URL.revokeObjectURL).toHaveBeenCalledWith('blob:report-url')
  })

  it('downloads PDF as an authenticated blob', async () => {
    fetchMock.mockResolvedValueOnce(new Response(new Blob(['pdf'], { type: 'application/pdf' }), { status: 200 }))

    await reportsService.downloadPdf({ date_from: '2026-08-01', date_to: '2026-08-31' })

    const [url, init] = fetchMock.mock.calls[0]
    expect(url).toContain('/admin/reports/trips.pdf?')
    expect((init.headers as Headers).get('Accept')).toBe('application/pdf')
    expect((init.headers as Headers).get('Authorization')).toBe('Bearer token-123')
  })
})
```

- [ ] **Step 2: Update trips service test with extended filters**

Modify `src/services/tripsService.test.ts` list test call:

```ts
await tripsService.list({
  status: 'open',
  plate: 'ABC-1D23',
  load_status: 'unknown',
  date_from: '2026-08-01',
  date_to: '2026-08-31',
  direction: 'outbound',
}, 2)
```

Add assertions:

```ts
expect(url).toContain('date_from=2026-08-01')
expect(url).toContain('date_to=2026-08-31')
expect(url).toContain('direction=outbound')
```

- [ ] **Step 3: Run service tests and verify RED**

Run:

```bash
npm test -- --run src/services/reportsService.test.ts src/services/tripsService.test.ts
```

Expected: FAIL because `reportsService` does not exist and `tripsService` does not serialize the new filters.

- [ ] **Step 4: Extend admin types**

Modify `src/types/admin.ts`:

```ts
export interface TripEventLoadStatusAudit {
  id: number
  uuid: string
  old_load_status: LoadStatus
  new_load_status: LoadStatus
  changed_at: string | null
  user: {
    id: number | null
    uuid: string | null
    name: string | null
    email: string | null
  }
}
```

Add to `TripEvent`:

```ts
load_status_audits?: TripEventLoadStatusAudit[]
```

Replace `TripFilters`:

```ts
export interface TripFilters {
  status?: TripStatus | ''
  plate?: string
  load_status?: LoadStatus | ''
  date_from?: string
  date_to?: string
  vehicle_id?: number | ''
  location_id?: number | ''
  direction?: TripEventDirection | ''
}
```

- [ ] **Step 5: Extend tripsService query serialization**

Modify `src/services/tripsService.ts` `queryFrom`:

```ts
if (filters.date_from) params.set('date_from', filters.date_from)
if (filters.date_to) params.set('date_to', filters.date_to)
if (filters.vehicle_id) params.set('vehicle_id', String(filters.vehicle_id))
if (filters.location_id) params.set('location_id', String(filters.location_id))
if (filters.direction) params.set('direction', filters.direction)
```

Place these before `params.set('page', String(page))`.

- [ ] **Step 6: Create reports service**

Create `src/services/reportsService.ts`:

```ts
import { getAppConfig } from '@/app/config'
import type { TripFilters } from '@/types/admin'
import { ApiError } from './apiClient'

function joinUrl(baseUrl: string, path: string): string {
  return `${baseUrl.replace(/\/$/, '')}/${path.replace(/^\//, '')}`
}

function queryFrom(filters: TripFilters): string {
  const params = new URLSearchParams()

  if (filters.date_from) params.set('date_from', filters.date_from)
  if (filters.date_to) params.set('date_to', filters.date_to)
  if (filters.status) params.set('status', filters.status)
  if (filters.plate?.trim()) params.set('plate', filters.plate.trim())
  if (filters.vehicle_id) params.set('vehicle_id', String(filters.vehicle_id))
  if (filters.location_id) params.set('location_id', String(filters.location_id))
  if (filters.load_status) params.set('load_status', filters.load_status)
  if (filters.direction) params.set('direction', filters.direction)

  const query = params.toString()
  return query ? `?${query}` : ''
}

async function downloadReport(path: string, filters: TripFilters, accept: string, filename: string): Promise<void> {
  const headers = new Headers()
  headers.set('Accept', accept)

  const token = localStorage.getItem('trackvision.token')
  if (token) {
    headers.set('Authorization', `Bearer ${token}`)
  }

  const response = await fetch(joinUrl(getAppConfig().apiBaseUrl, `${path}${queryFrom(filters)}`), { headers })

  if (!response.ok) {
    throw new ApiError(response.status, 'Nao foi possivel baixar o relatorio.')
  }

  const url = URL.createObjectURL(await response.blob())
  const anchor = document.createElement('a')
  anchor.href = url
  anchor.download = filename
  anchor.click()
  URL.revokeObjectURL(url)
}

export const reportsService = {
  downloadCsv(filters: TripFilters): Promise<void> {
    return downloadReport('/admin/reports/trips.csv', filters, 'text/csv', 'trackvision-trips.csv')
  },

  downloadPdf(filters: TripFilters): Promise<void> {
    return downloadReport('/admin/reports/trips.pdf', filters, 'application/pdf', 'trackvision-trips.pdf')
  },
}
```

- [ ] **Step 7: Run service tests and verify GREEN**

Run:

```bash
npm test -- --run src/services/reportsService.test.ts src/services/tripsService.test.ts
```

Expected: PASS.

- [ ] **Step 8: Commit frontend report service**

Run:

```bash
git add src/types/admin.ts src/services/reportsService.ts src/services/reportsService.test.ts src/services/tripsService.ts src/services/tripsService.test.ts
git commit -m "feat: add trip report downloads"
```

---

## Task 6: Frontend Trips Page Exports And Audit Timeline

**Files:**

- Modify: `RIALMA-TrackVision-Frontend/src/pages/TripsPage.vue`
- Modify: `RIALMA-TrackVision-Frontend/src/pages/TripsPage.test.ts`

**Interfaces:**

- Consumes: `reportsService.downloadCsv(filters)` and `reportsService.downloadPdf(filters)`.
- Consumes: `TripEvent.load_status_audits`.
- Produces: CSV/PDF buttons visible with `reports.view`.
- Produces: date and direction filters in `/trips`.
- Produces: audit timeline in selected trip details.

- [ ] **Step 1: Write failing page tests for exports and audit timeline**

Modify `src/pages/TripsPage.test.ts` imports:

```ts
import { reportsService } from '@/services/reportsService'
```

Add mock:

```ts
vi.mock('@/services/reportsService', () => ({
  reportsService: {
    downloadCsv: vi.fn().mockResolvedValue(undefined),
    downloadPdf: vi.fn().mockResolvedValue(undefined),
  },
}))
```

Modify `detailedTrip.events[0]` with:

```ts
load_status_audits: [{
  id: 99,
  uuid: 'audit-uuid',
  old_load_status: 'unknown',
  new_load_status: 'loaded',
  changed_at: '2026-08-13T15:00:00Z',
  user: { id: 5, uuid: 'user-uuid', name: 'Paulo Peixoto', email: 'paulo@example.com' },
}],
```

Add beforeEach resets:

```ts
vi.mocked(reportsService.downloadCsv).mockReset().mockResolvedValue(undefined)
vi.mocked(reportsService.downloadPdf).mockReset().mockResolvedValue(undefined)
```

Add tests:

```ts
it('shows report buttons only when user can view reports', async () => {
  const authStore = useAuthStore()
  authStore.permissions = ['captures.view']
  const wrapper = mount(TripsPage)
  await waitForPromises()

  expect(wrapper.findAll('button').map((button) => button.text())).not.toContain('CSV')
  expect(wrapper.findAll('button').map((button) => button.text())).not.toContain('PDF')

  authStore.permissions = ['captures.view', 'reports.view']
  const allowedWrapper = mount(TripsPage)
  await waitForPromises()

  expect(allowedWrapper.findAll('button').map((button) => button.text())).toContain('CSV')
  expect(allowedWrapper.findAll('button').map((button) => button.text())).toContain('PDF')
})

it('downloads CSV and PDF with the current filters', async () => {
  const authStore = useAuthStore()
  authStore.permissions = ['captures.view', 'reports.view']
  const wrapper = mount(TripsPage)
  await waitForPromises()

  await wrapper.get('[data-test="export-csv"]').trigger('click')
  await wrapper.get('[data-test="export-pdf"]').trigger('click')

  expect(reportsService.downloadCsv).toHaveBeenCalledWith(expect.objectContaining({
    date_from: expect.any(String),
    date_to: expect.any(String),
    status: '',
    plate: '',
    load_status: '',
    direction: '',
  }))
  expect(reportsService.downloadPdf).toHaveBeenCalledWith(expect.objectContaining({
    date_from: expect.any(String),
    date_to: expect.any(String),
  }))
})

it('renders load status audit timeline in selected trip detail', async () => {
  const wrapper = mount(TripsPage)
  await waitForPromises()
  await wrapper.get('[data-test="select-trip"]').trigger('click')
  await waitForPromises()

  expect(wrapper.text()).toContain('Historico de carga')
  expect(wrapper.text()).toContain('Paulo Peixoto')
  expect(wrapper.text()).toContain('Nao revisada')
  expect(wrapper.text()).toContain('Carregado')
})

it('shows export error without clearing selected trip', async () => {
  const authStore = useAuthStore()
  authStore.permissions = ['captures.view', 'reports.view']
  vi.mocked(reportsService.downloadCsv).mockRejectedValueOnce(new Error('download failed'))
  const wrapper = mount(TripsPage)
  await waitForPromises()
  await wrapper.get('[data-test="select-trip"]').trigger('click')
  await waitForPromises()

  await wrapper.get('[data-test="export-csv"]').trigger('click')
  await waitForPromises()

  expect(wrapper.text()).toContain('Nao foi possivel baixar o relatorio.')
  expect(wrapper.text()).toContain('Entrada 01')
})
```

Update the existing pagination test assertions from exact filter objects to object matching:

```ts
expect(tripsService.list).toHaveBeenLastCalledWith(expect.objectContaining({
  status: '',
  plate: '',
  load_status: '',
  direction: '',
  date_from: expect.any(String),
  date_to: expect.any(String),
}), 2)
```

Use the same `expect.objectContaining(...)` shape for the previous-page assertion with page `1`.

- [ ] **Step 2: Run page tests and verify RED**

Run:

```bash
npm test -- --run src/pages/TripsPage.test.ts
```

Expected: FAIL because page does not import reports service, does not render export buttons, and does not render audit timeline.

- [ ] **Step 3: Update TripsPage script**

Modify `src/pages/TripsPage.vue` imports:

```ts
import { reportsService } from '@/services/reportsService'
import type { LoadStatus, Trip, TripEvent, TripEventLoadStatusAudit, TripFilters } from '@/types/admin'
```

Add direction options:

```ts
const directionOptions = [
  { label: 'Todas', value: '' },
  { label: 'Ida', value: 'outbound' },
  { label: 'Volta', value: 'inbound' },
  { label: 'Indefinida', value: 'unknown' },
]
```

Add date helpers:

```ts
function dateInputValue(date: Date): string {
  return date.toISOString().slice(0, 10)
}

function initialDateFrom(): string {
  const date = new Date()
  date.setDate(date.getDate() - 7)
  return dateInputValue(date)
}
```

Replace filters initialization:

```ts
const filters = ref<Required<Pick<TripFilters, 'status' | 'plate' | 'load_status' | 'date_from' | 'date_to' | 'direction'>>>({
  status: '',
  plate: '',
  load_status: '',
  date_from: initialDateFrom(),
  date_to: dateInputValue(new Date()),
  direction: '',
})
```

Add computed and exporting state:

```ts
const canViewReports = computed(() => authStore.can('reports.view'))
const exporting = ref<'csv' | 'pdf' | null>(null)
```

Add audit label helper:

```ts
function auditLabel(audit: TripEventLoadStatusAudit): string {
  const actor = audit.user.name ?? audit.user.email ?? 'Usuario'
  return `${actor}: ${loadLabel(audit.old_load_status)} para ${loadLabel(audit.new_load_status)}`
}
```

Add export method:

```ts
async function exportReport(format: 'csv' | 'pdf'): Promise<void> {
  exporting.value = format
  error.value = ''
  success.value = ''

  try {
    if (format === 'csv') {
      await reportsService.downloadCsv(filters.value)
    } else {
      await reportsService.downloadPdf(filters.value)
    }

    success.value = 'Relatorio solicitado.'
  } catch {
    error.value = 'Nao foi possivel baixar o relatorio.'
  } finally {
    exporting.value = null
  }
}
```

- [ ] **Step 4: Update TripsPage template**

Add export buttons in `page-header` after Atualizar:

```vue
<div class="header-actions">
  <BaseButton
    type="button"
    variant="secondary"
    @click="loadTrips"
  >
    Atualizar
  </BaseButton>
  <BaseButton
    v-if="canViewReports"
    data-test="export-csv"
    type="button"
    variant="secondary"
    :loading="exporting === 'csv'"
    @click="exportReport('csv')"
  >
    CSV
  </BaseButton>
  <BaseButton
    v-if="canViewReports"
    data-test="export-pdf"
    type="button"
    :loading="exporting === 'pdf'"
    @click="exportReport('pdf')"
  >
    PDF
  </BaseButton>
</div>
```

Add date and direction filters to `.filters-row`:

```vue
<BaseInput
  v-model="filters.date_from"
  label="De"
  type="date"
/>
<BaseInput
  v-model="filters.date_to"
  label="Ate"
  type="date"
/>
<BaseSelect
  v-model="filters.direction"
  label="Direcao"
  :options="directionOptions"
/>
```

Inside each `.trip-event`, after load action buttons, add:

```vue
<section
  v-if="event.load_status_audits?.length"
  class="audit-timeline"
>
  <h3>Historico de carga</h3>
  <ol>
    <li
      v-for="audit in event.load_status_audits"
      :key="audit.id"
    >
      <span>{{ auditLabel(audit) }}</span>
      <small>{{ formatDate(audit.changed_at) }}</small>
    </li>
  </ol>
</section>
```

- [ ] **Step 5: Update TripsPage styles**

Add:

```css
.header-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  justify-content: flex-end;
}

.audit-timeline {
  margin-top: 14px;
}

.audit-timeline h3 {
  font-size: 0.95rem;
  margin: 0 0 8px;
}

.audit-timeline ol {
  display: grid;
  gap: 6px;
  list-style: none;
  margin: 0;
  padding: 0;
}

.audit-timeline li {
  border-left: 3px solid var(--color-border);
  display: grid;
  gap: 2px;
  padding-left: 8px;
}

.audit-timeline small {
  color: var(--color-muted);
}
```

Change `.filters-row` columns:

```css
grid-template-columns: repeat(7, minmax(110px, 1fr));
```

- [ ] **Step 6: Run page tests and verify GREEN**

Run:

```bash
npm test -- --run src/pages/TripsPage.test.ts
```

Expected: PASS.

- [ ] **Step 7: Run frontend full suite and build**

Run:

```bash
npm test -- --run
npm run build
```

Expected: PASS.

- [ ] **Step 8: Commit frontend page task**

Run:

```bash
git add src/pages/TripsPage.vue src/pages/TripsPage.test.ts
git commit -m "feat: add trip report controls"
```

---

## Task 7: Documentation, Final Verification, And Integration Handoff

**Files:**

- Modify: `RIALMA-TrackVision-Backend/docs/api-parent-admin.md`
- Modify: `RIALMA-TrackVision-Backend/README.md`
- Modify: `RIALMA-TrackVision-Frontend/README.md`

**Interfaces:**

- Documents: audit behavior for load-status updates.
- Documents: CSV/PDF report endpoints, filters, permissions, limits, and frontend usage.
- Produces: final verification evidence for backend and frontend branches.

- [ ] **Step 1: Update backend API documentation**

Append to `RIALMA-TrackVision-Backend/docs/api-parent-admin.md`:

```markdown
## Trip Reports

Endpoints:

- `GET /reports/trips.csv`
- `GET /reports/trips.pdf`

Authorization:

- Requires `reports.view`.

Required filters:

- `date_from`
- `date_to`

Optional filters:

- `status`: `open`, `closed`, `needs_review`
- `plate`
- `vehicle_id`
- `location_id`
- `load_status`: `unknown`, `loaded`, `empty`, `needs_review`
- `direction`: `outbound`, `inbound`, `unknown`

Limits:

- CSV accepts up to 180 days and `TRACKVISION_REPORTS_CSV_MAX_ROWS` rows.
- PDF accepts up to 31 days and `TRACKVISION_REPORTS_PDF_MAX_TRIPS` trips.

CSV returns one row per trip event and does not include image bytes, storage paths, tokens, or signed URLs.

PDF returns `application/pdf` and embeds available private LPR/support images from storage as report evidence. Missing images are represented as absent evidence in the PDF.

## Trip Load Status Audit

`PATCH /trip-events/{tripEvent}/load-status` records an immutable audit entry when the new value differs from the current value.

Each audit entry stores:

- trip event;
- capture event;
- actor user;
- old load status;
- new load status;
- change timestamp.

Trip detail responses include `load_status_audits` for each event when audit relations are loaded.
```

- [ ] **Step 2: Update backend README**

Append:

```markdown
### Trip Reports And Audit

Trip reports are available from the parent admin API for users with `reports.view`.

Environment limits:

- `TRACKVISION_REPORTS_CSV_MAX_ROWS` defaults to `5000`.
- `TRACKVISION_REPORTS_PDF_MAX_TRIPS` defaults to `100`.

Manual load-status changes are audited inside the same database transaction that updates `trip_events.load_status` and `capture_events.load_status`.
```

- [ ] **Step 3: Update frontend README**

Append:

```markdown
### Trip Reports And Audit Timeline

The `/trips` screen shows CSV/PDF export buttons when the user has `reports.view`.

Report downloads use the current filters and send the Bearer token through the Authorization header. Tokens are never appended to report URLs.

Trip event details show the load-status audit timeline returned by the backend.
```

- [ ] **Step 4: Run backend final verification**

Run:

```bash
composer validate
php artisan test
php artisan route:list --path=api/v1/admin/reports
php artisan route:list --path=api/v1/admin/trip-events
php artisan route:list --path=api/v1/admin/trips
git diff --check
```

Expected:

- Composer valid.
- Full Laravel test suite passes.
- Report, trip-event, and trip routes are registered.
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
git commit -m "docs: add trip reports audit guide"
```

Frontend:

```bash
git add README.md
git commit -m "docs: add trip reports frontend guide"
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
- whether `dompdf/dompdf` was added successfully;
- any skipped backend test and reason;
- backend route list entries for reports;
- frontend build artifact result.

---

## Acceptance Checklist

- [ ] Toda alteracao manual real de carga gera auditoria.
- [ ] Alteracao sem mudanca de valor nao gera duplicidade.
- [ ] Historico mostra quem alterou, quando alterou, valor anterior e novo.
- [ ] Usuario com `reports.view` consegue exportar CSV e PDF.
- [ ] Usuario sem `reports.view` nao consegue exportar.
- [ ] CSV contem uma linha por evento de viagem.
- [ ] PDF contem dados da viagem e imagens LPR/apoio quando disponiveis.
- [ ] Imagens privadas continuam sem path interno exposto.
- [ ] Relatorios respeitam filtros, periodo e limites de volume.
- [ ] Frontend baixa relatorios com Authorization header.
- [ ] Backend segue Controllers magros, Form Requests, Actions/Services e Resources.
