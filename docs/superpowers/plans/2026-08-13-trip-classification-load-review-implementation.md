# Trip Classification And Load Review Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build trip grouping and manual load review so accepted parent captures become operational trips and operators can mark loaded/empty from support-camera images.

**Architecture:** The Laravel parent backend owns trip classification as a deterministic domain layer above `CaptureEvent` and `MediaAsset`. Backend controllers stay thin, Form Requests validate filters and load-status changes, Actions group captures into `Trip`/`TripEvent`, and Resources expose stable API payloads. The Vue frontend consumes the admin trip API through services, renders a dense operational review page, and loads private media through an authenticated blob service.

**Tech Stack:** Laravel 13, PHP 8.4, Laravel Passport, Spatie Laravel Permission, PostgreSQL/SQLite tests, private local storage, Vue 3, Vite, TypeScript, Pinia, Vue Router, Vitest, Vue Test Utils.

## Global Constraints

- Backend must follow `docs/backend-laravel-guidelines.md`.
- Frontend must follow `docs/frontend-vue-guidelines.md`.
- Backend code lives only in `RIALMA-TrackVision-Backend`.
- Frontend code lives only in `RIALMA-TrackVision-Frontend`.
- AgentOps documentation lives only in `RIALMA-TrackVision-AgentOps`.
- Controllers devem ficar magros.
- Validacao deve ficar em Form Requests.
- Regras de negocio devem ficar em Actions.
- Resources nao devem expor paths internos de storage.
- Imagens privadas devem exigir usuario autenticado e permissao `captures.view`.
- O frontend pode esconder botoes, mas o backend e autoridade final para `trips.manage`.
- Revisao de carga nao deve aceitar valores fora do enum `LoadStatus`.
- A API nao deve permitir atualizar carga de um evento que nao pertence a uma viagem.
- O frontend nao deve colocar token em query string.
- Imagens privadas devem ser carregadas pelo service com Authorization header e convertidas em Object URL local.
- Object URLs devem ser revogadas quando a imagem sair da tela.
- Relatorios formais, exportacao CSV/PDF, auditoria completa, visao computacional, reprocessamento em massa e dashboards em tempo real ficam fora desta fase.

---

## File Structure Map

Backend files:

```text
RIALMA-TrackVision-Backend/
|-- app/
|   |-- Actions/
|   |   `-- Trips/
|   |       |-- AttachCaptureToTripAction.php
|   |       |-- ClassifyTripDirectionAction.php
|   |       `-- UpdateLoadStatusAction.php
|   |-- Enums/
|   |   `-- TripStatus.php
|   |-- Http/
|   |   |-- Controllers/Api/V1/Admin/
|   |   |   |-- MediaAssetContentController.php
|   |   |   |-- TripController.php
|   |   |   `-- TripLoadReviewController.php
|   |   |-- Requests/Api/V1/Admin/
|   |   |   |-- IndexTripRequest.php
|   |   |   `-- UpdateTripEventLoadStatusRequest.php
|   |   `-- Resources/Api/V1/
|   |       |-- TripEventResource.php
|   |       `-- TripResource.php
|   |-- Models/
|   |   |-- Trip.php
|   |   `-- TripEvent.php
|   `-- Models/CaptureEvent.php
|-- database/
|   |-- factories/
|   |   |-- TripEventFactory.php
|   |   `-- TripFactory.php
|   `-- migrations/
|       |-- 2026_08_13_180001_create_trips_table.php
|       `-- 2026_08_13_180002_create_trip_events_table.php
|-- docs/api-parent-admin.md
|-- routes/api.php
`-- tests/Feature/
    |-- Trips/TripClassificationTest.php
    `-- Admin/TripAdminTest.php
```

Frontend files:

```text
RIALMA-TrackVision-Frontend/
|-- src/
|   |-- components/navigation/TheSidebar.vue
|   |-- components/navigation/TheSidebar.test.ts
|   |-- pages/
|   |   |-- TripsPage.vue
|   |   `-- TripsPage.test.ts
|   |-- router/
|   |   |-- index.ts
|   |   `-- routerGuard.test.ts
|   |-- services/
|   |   |-- mediaAssetsService.test.ts
|   |   |-- mediaAssetsService.ts
|   |   |-- tripsService.test.ts
|   |   `-- tripsService.ts
|   `-- types/admin.ts
`-- README.md
```

## Execution Setup

- Create an isolated backend worktree from `RIALMA-TrackVision-Backend/main` on branch `codex/trip-load-review-backend`.
- Create an isolated frontend worktree from `RIALMA-TrackVision-Frontend/main` on branch `codex/trip-load-review-frontend`.
- Copy `.env` into the backend worktree if the main checkout has one and tests require app keys/database settings.
- Baseline backend verification before Task 1:

```bash
composer validate
php artisan test
```

- Baseline frontend verification before Task 4:

```bash
npm test -- --run
npm run build
```

If either baseline fails, stop and report the exact failure before implementing this plan.

---

## Task 1: Backend Trip Schema And Models

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Enums/TripStatus.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/Trip.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/TripEvent.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/TripFactory.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/TripEventFactory.php`
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_13_180001_create_trips_table.php`
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_13_180002_create_trip_events_table.php`
- Modify: `RIALMA-TrackVision-Backend/app/Models/CaptureEvent.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Trips/TripClassificationTest.php`

**Interfaces:**

- Consumes: existing `App\Models\Vehicle`, `App\Models\Location`, `App\Models\CaptureEvent`.
- Consumes: existing enums `App\Enums\CameraPairDirection` and `App\Enums\LoadStatus`.
- Produces: `App\Enums\TripStatus` with values `open`, `closed`, `needs_review`.
- Produces: `Trip::vehicle(): BelongsTo`.
- Produces: `Trip::location(): BelongsTo`.
- Produces: `Trip::events(): HasMany`.
- Produces: `TripEvent::trip(): BelongsTo`.
- Produces: `TripEvent::captureEvent(): BelongsTo`.
- Produces: `CaptureEvent::tripEvent(): HasOne`.

- [ ] **Step 1: Write failing schema/model tests**

Create `tests/Feature/Trips/TripClassificationTest.php` with the first schema-focused tests:

```php
<?php

namespace Tests\Feature\Trips;

use App\Enums\CameraPairDirection;
use App\Enums\LoadStatus;
use App\Enums\TripStatus;
use App\Models\CameraPair;
use App\Models\CaptureEvent;
use App\Models\Trip;
use App\Models\TripEvent;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class TripClassificationTest extends TestCase
{
    use RefreshDatabase;

    public function test_trip_models_cast_status_dates_and_relationships(): void
    {
        $capture = CaptureEvent::factory()->create([
            'direction' => CameraPairDirection::Outbound,
            'load_status' => LoadStatus::Unknown,
            'event_time' => '2026-08-13T10:00:00Z',
        ]);
        $trip = Trip::factory()->create([
            'vehicle_id' => $capture->vehicle_id,
            'location_id' => $capture->cameraPair->location_id,
            'status' => TripStatus::Open,
            'opened_at' => '2026-08-13T10:00:00Z',
            'closed_at' => null,
            'review_required_reason' => null,
        ]);
        $event = TripEvent::factory()->create([
            'trip_id' => $trip->id,
            'capture_event_id' => $capture->id,
            'direction' => CameraPairDirection::Outbound,
            'load_status' => LoadStatus::Unknown,
            'occurred_at' => '2026-08-13T10:00:00Z',
        ]);

        $this->assertSame(TripStatus::Open, $trip->refresh()->status);
        $this->assertTrue($trip->vehicle->is($capture->vehicle));
        $this->assertTrue($trip->location->is($capture->cameraPair->location));
        $this->assertTrue($trip->events->first()->is($event));
        $this->assertTrue($event->captureEvent->is($capture));
        $this->assertTrue($capture->tripEvent->is($event));
        $this->assertSame(CameraPairDirection::Outbound, $event->direction);
        $this->assertSame(LoadStatus::Unknown, $event->load_status);
    }

    public function test_trip_event_capture_event_is_unique(): void
    {
        $capture = CaptureEvent::factory()->create();
        $trip = Trip::factory()->create([
            'vehicle_id' => $capture->vehicle_id,
            'location_id' => $capture->cameraPair->location_id,
        ]);

        TripEvent::factory()->create([
            'trip_id' => $trip->id,
            'capture_event_id' => $capture->id,
        ]);

        $this->expectException(\Illuminate\Database\QueryException::class);

        TripEvent::factory()->create([
            'trip_id' => $trip->id,
            'capture_event_id' => $capture->id,
        ]);
    }
}
```

- [ ] **Step 2: Run schema/model tests and verify RED**

Run:

```bash
php artisan test --filter=TripClassificationTest
```

Expected: FAIL because `TripStatus`, `Trip`, `TripEvent`, factories, tables, and `CaptureEvent::tripEvent()` do not exist.

- [ ] **Step 3: Create `TripStatus` enum**

Create `app/Enums/TripStatus.php`:

```php
<?php

namespace App\Enums;

enum TripStatus: string
{
    case Open = 'open';
    case Closed = 'closed';
    case NeedsReview = 'needs_review';
}
```

- [ ] **Step 4: Create trip migrations**

Create `database/migrations/2026_08_13_180001_create_trips_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('trips', function (Blueprint $table): void {
            $table->id();
            $table->uuid('uuid')->unique();
            $table->foreignId('vehicle_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
            $table->foreignId('location_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
            $table->string('status', 40)->index();
            $table->timestamp('opened_at')->index();
            $table->timestamp('closed_at')->nullable()->index();
            $table->string('review_required_reason', 80)->nullable();
            $table->timestamps();

            $table->index(['vehicle_id', 'location_id', 'status']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('trips');
    }
};
```

Create `database/migrations/2026_08_13_180002_create_trip_events_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('trip_events', function (Blueprint $table): void {
            $table->id();
            $table->uuid('uuid')->unique();
            $table->foreignId('trip_id')->constrained()->cascadeOnUpdate()->cascadeOnDelete();
            $table->foreignId('capture_event_id')->unique()->constrained()->cascadeOnUpdate()->restrictOnDelete();
            $table->string('direction', 30)->index();
            $table->string('load_status', 40)->index();
            $table->timestamp('occurred_at')->index();
            $table->timestamps();

            $table->index(['trip_id', 'occurred_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('trip_events');
    }
};
```

- [ ] **Step 5: Create models and relationships**

Create `app/Models/Trip.php`:

```php
<?php

namespace App\Models;

use App\Enums\TripStatus;
use App\Models\Concerns\HasPublicUuid;
use Database\Factories\TripFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

#[Fillable(['uuid', 'vehicle_id', 'location_id', 'status', 'opened_at', 'closed_at', 'review_required_reason'])]
class Trip extends Model
{
    /** @use HasFactory<TripFactory> */
    use HasFactory, HasPublicUuid;

    protected function casts(): array
    {
        return [
            'status' => TripStatus::class,
            'opened_at' => 'immutable_datetime',
            'closed_at' => 'immutable_datetime',
        ];
    }

    public function vehicle(): BelongsTo
    {
        return $this->belongsTo(Vehicle::class);
    }

    public function location(): BelongsTo
    {
        return $this->belongsTo(Location::class);
    }

    public function events(): HasMany
    {
        return $this->hasMany(TripEvent::class);
    }
}
```

Create `app/Models/TripEvent.php`:

```php
<?php

namespace App\Models;

use App\Enums\CameraPairDirection;
use App\Enums\LoadStatus;
use App\Models\Concerns\HasPublicUuid;
use Database\Factories\TripEventFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Fillable(['uuid', 'trip_id', 'capture_event_id', 'direction', 'load_status', 'occurred_at'])]
class TripEvent extends Model
{
    /** @use HasFactory<TripEventFactory> */
    use HasFactory, HasPublicUuid;

    protected function casts(): array
    {
        return [
            'direction' => CameraPairDirection::class,
            'load_status' => LoadStatus::class,
            'occurred_at' => 'immutable_datetime',
        ];
    }

    public function trip(): BelongsTo
    {
        return $this->belongsTo(Trip::class);
    }

    public function captureEvent(): BelongsTo
    {
        return $this->belongsTo(CaptureEvent::class);
    }
}
```

Modify `app/Models/CaptureEvent.php`:

```php
use Illuminate\Database\Eloquent\Relations\HasOne;
```

Add the relationship:

```php
public function tripEvent(): HasOne
{
    return $this->hasOne(TripEvent::class);
}
```

- [ ] **Step 6: Create factories**

Create `database/factories/TripFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Enums\TripStatus;
use App\Models\Location;
use App\Models\Trip;
use App\Models\Vehicle;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<Trip>
 */
class TripFactory extends Factory
{
    public function definition(): array
    {
        return [
            'vehicle_id' => Vehicle::factory(),
            'location_id' => Location::factory(),
            'status' => TripStatus::Open,
            'opened_at' => now(),
            'closed_at' => null,
            'review_required_reason' => null,
        ];
    }
}
```

Create `database/factories/TripEventFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Enums\CameraPairDirection;
use App\Enums\LoadStatus;
use App\Models\CaptureEvent;
use App\Models\Trip;
use App\Models\TripEvent;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<TripEvent>
 */
class TripEventFactory extends Factory
{
    public function definition(): array
    {
        $capture = CaptureEvent::factory()->create();

        return [
            'trip_id' => Trip::factory()->create([
                'vehicle_id' => $capture->vehicle_id,
                'location_id' => $capture->cameraPair->location_id,
            ]),
            'capture_event_id' => $capture->id,
            'direction' => CameraPairDirection::Outbound,
            'load_status' => LoadStatus::Unknown,
            'occurred_at' => $capture->event_time ?? now(),
        ];
    }
}
```

- [ ] **Step 7: Run schema/model tests and verify GREEN**

Run:

```bash
php artisan test --filter=TripClassificationTest
```

Expected: PASS for the two schema/model tests.

- [ ] **Step 8: Run backend focused smoke**

Run:

```bash
php artisan test tests/Feature/Admin/VehicleAdminTest.php tests/Feature/Edge/EdgeCaptureBatchIngestTest.php --stop-on-failure
```

Expected: PASS. This confirms the new relationships/migrations did not disturb existing admin and capture flows.

- [ ] **Step 9: Commit backend schema task**

Run:

```bash
git add app/Enums/TripStatus.php app/Models/Trip.php app/Models/TripEvent.php app/Models/CaptureEvent.php database/factories/TripFactory.php database/factories/TripEventFactory.php database/migrations/2026_08_13_180001_create_trips_table.php database/migrations/2026_08_13_180002_create_trip_events_table.php tests/Feature/Trips/TripClassificationTest.php
git commit -m "feat: add trip schema models"
```

---

## Task 2: Backend Trip Classification Actions

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Actions/Trips/ClassifyTripDirectionAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Trips/AttachCaptureToTripAction.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Edge/UpsertEdgeCaptureAction.php`
- Modify: `RIALMA-TrackVision-Backend/tests/Feature/Trips/TripClassificationTest.php`

**Interfaces:**

- Consumes: `TripStatus`, `Trip`, `TripEvent`, `CaptureEvent::tripEvent()`.
- Produces: `ClassifyTripDirectionAction::execute(CaptureEvent $capture): CameraPairDirection`.
- Produces: `AttachCaptureToTripAction::execute(CaptureEvent $capture): TripEvent`.
- Produces: automatic classification after successful parent capture ingest in `UpsertEdgeCaptureAction`.

- [ ] **Step 1: Add failing outbound classification test**

Append to `tests/Feature/Trips/TripClassificationTest.php`:

```php
public function test_outbound_capture_opens_trip(): void
{
    $capture = CaptureEvent::factory()->create([
        'direction' => CameraPairDirection::Outbound,
        'event_time' => '2026-08-13T10:00:00Z',
    ]);

    $event = app(\App\Actions\Trips\AttachCaptureToTripAction::class)->execute($capture);

    $this->assertSame(CameraPairDirection::Outbound, $event->direction);
    $this->assertSame(LoadStatus::Unknown, $event->load_status);
    $this->assertSame(TripStatus::Open, $event->trip->status);
    $this->assertSame($capture->vehicle_id, $event->trip->vehicle_id);
    $this->assertSame($capture->cameraPair->location_id, $event->trip->location_id);
    $this->assertSame('2026-08-13T10:00:00+00:00', $event->trip->opened_at->toIso8601String());
    $this->assertNull($event->trip->closed_at);
}
```

- [ ] **Step 2: Add failing inbound, unknown, ambiguous, and idempotency tests**

Append:

```php
public function test_inbound_capture_closes_open_trip_for_same_vehicle_and_location(): void
{
    $outbound = CaptureEvent::factory()->create([
        'direction' => CameraPairDirection::Outbound,
        'event_time' => '2026-08-13T10:00:00Z',
    ]);
    $openTrip = app(\App\Actions\Trips\AttachCaptureToTripAction::class)->execute($outbound)->trip;

    $inboundPair = CameraPair::factory()->create([
        'location_id' => $outbound->cameraPair->location_id,
        'direction' => CameraPairDirection::Inbound,
    ]);
    $inbound = CaptureEvent::factory()->create([
        'vehicle_id' => $outbound->vehicle_id,
        'camera_pair_id' => $inboundPair->id,
        'edge_node_id' => $inboundPair->edge_node_id,
        'lpr_camera_id' => $inboundPair->lpr_camera_id,
        'support_camera_id' => $inboundPair->support_camera_id,
        'direction' => CameraPairDirection::Inbound,
        'event_time' => '2026-08-13T12:00:00Z',
    ]);

    $event = app(\App\Actions\Trips\AttachCaptureToTripAction::class)->execute($inbound);

    $this->assertTrue($event->trip->is($openTrip));
    $this->assertSame(TripStatus::Closed, $event->trip->refresh()->status);
    $this->assertSame('2026-08-13T12:00:00+00:00', $event->trip->closed_at->toIso8601String());
}

public function test_inbound_without_open_outbound_creates_needs_review_trip(): void
{
    $capture = CaptureEvent::factory()->create([
        'direction' => CameraPairDirection::Inbound,
        'event_time' => '2026-08-13T12:00:00Z',
    ]);

    $event = app(\App\Actions\Trips\AttachCaptureToTripAction::class)->execute($capture);

    $this->assertSame(TripStatus::NeedsReview, $event->trip->status);
    $this->assertSame('inbound_without_outbound', $event->trip->review_required_reason);
    $this->assertSame('2026-08-13T12:00:00+00:00', $event->trip->opened_at->toIso8601String());
    $this->assertSame('2026-08-13T12:00:00+00:00', $event->trip->closed_at->toIso8601String());
}

public function test_unknown_direction_creates_needs_review_trip(): void
{
    $capture = CaptureEvent::factory()->create([
        'direction' => CameraPairDirection::Unknown,
        'event_time' => '2026-08-13T12:00:00Z',
    ]);

    $event = app(\App\Actions\Trips\AttachCaptureToTripAction::class)->execute($capture);

    $this->assertSame(TripStatus::NeedsReview, $event->trip->status);
    $this->assertSame('unknown_direction', $event->trip->review_required_reason);
    $this->assertSame(CameraPairDirection::Unknown, $event->direction);
}

public function test_new_outbound_marks_previous_open_trip_as_needs_review(): void
{
    $first = CaptureEvent::factory()->create([
        'direction' => CameraPairDirection::Outbound,
        'event_time' => '2026-08-13T10:00:00Z',
    ]);
    $firstTrip = app(\App\Actions\Trips\AttachCaptureToTripAction::class)->execute($first)->trip;

    $second = CaptureEvent::factory()->create([
        'vehicle_id' => $first->vehicle_id,
        'camera_pair_id' => $first->camera_pair_id,
        'edge_node_id' => $first->edge_node_id,
        'lpr_camera_id' => $first->lpr_camera_id,
        'support_camera_id' => $first->support_camera_id,
        'direction' => CameraPairDirection::Outbound,
        'event_time' => '2026-08-13T11:00:00Z',
    ]);

    $secondEvent = app(\App\Actions\Trips\AttachCaptureToTripAction::class)->execute($second);

    $this->assertSame(TripStatus::NeedsReview, $firstTrip->refresh()->status);
    $this->assertSame('new_outbound_before_inbound', $firstTrip->review_required_reason);
    $this->assertFalse($secondEvent->trip->is($firstTrip));
    $this->assertSame(TripStatus::Open, $secondEvent->trip->status);
}

public function test_inbound_before_open_trip_creates_needs_review_without_closing_open_trip(): void
{
    $outbound = CaptureEvent::factory()->create([
        'direction' => CameraPairDirection::Outbound,
        'event_time' => '2026-08-13T10:00:00Z',
    ]);
    $openTrip = app(\App\Actions\Trips\AttachCaptureToTripAction::class)->execute($outbound)->trip;

    $inbound = CaptureEvent::factory()->create([
        'vehicle_id' => $outbound->vehicle_id,
        'camera_pair_id' => $outbound->camera_pair_id,
        'edge_node_id' => $outbound->edge_node_id,
        'lpr_camera_id' => $outbound->lpr_camera_id,
        'support_camera_id' => $outbound->support_camera_id,
        'direction' => CameraPairDirection::Inbound,
        'event_time' => '2026-08-13T09:55:00Z',
    ]);

    $event = app(\App\Actions\Trips\AttachCaptureToTripAction::class)->execute($inbound);

    $this->assertSame(TripStatus::Open, $openTrip->refresh()->status);
    $this->assertSame(TripStatus::NeedsReview, $event->trip->status);
    $this->assertSame('inbound_before_open_trip', $event->trip->review_required_reason);
}

public function test_reprocessing_same_capture_returns_existing_trip_event(): void
{
    $capture = CaptureEvent::factory()->create([
        'direction' => CameraPairDirection::Outbound,
    ]);
    $action = app(\App\Actions\Trips\AttachCaptureToTripAction::class);

    $first = $action->execute($capture);
    $second = $action->execute($capture->refresh());

    $this->assertTrue($first->is($second));
    $this->assertSame(1, TripEvent::query()->where('capture_event_id', $capture->id)->count());
}
```

- [ ] **Step 3: Run classification tests and verify RED**

Run:

```bash
php artisan test --filter=TripClassificationTest
```

Expected: FAIL because `AttachCaptureToTripAction` and `ClassifyTripDirectionAction` do not exist.

- [ ] **Step 4: Create `ClassifyTripDirectionAction`**

Create `app/Actions/Trips/ClassifyTripDirectionAction.php`:

```php
<?php

namespace App\Actions\Trips;

use App\Enums\CameraPairDirection;
use App\Models\CaptureEvent;

class ClassifyTripDirectionAction
{
    public function execute(CaptureEvent $capture): CameraPairDirection
    {
        if ($capture->direction instanceof CameraPairDirection) {
            return $capture->direction;
        }

        return CameraPairDirection::tryFrom((string) $capture->direction) ?? CameraPairDirection::Unknown;
    }
}
```

- [ ] **Step 5: Create `AttachCaptureToTripAction`**

Create `app/Actions/Trips/AttachCaptureToTripAction.php`:

```php
<?php

namespace App\Actions\Trips;

use App\Enums\CameraPairDirection;
use App\Enums\CaptureStatus;
use App\Enums\TripStatus;
use App\Models\CaptureEvent;
use App\Models\Trip;
use App\Models\TripEvent;
use Carbon\CarbonImmutable;
use Illuminate\Support\Facades\DB;
use RuntimeException;

class AttachCaptureToTripAction
{
    public function __construct(private readonly ClassifyTripDirectionAction $classifyDirection) {}

    public function execute(CaptureEvent $capture): TripEvent
    {
        return DB::transaction(function () use ($capture): TripEvent {
            $capture->loadMissing(['cameraPair', 'tripEvent']);

            if ($capture->tripEvent) {
                return $capture->tripEvent;
            }

            if ($capture->status !== CaptureStatus::Accepted || ! $capture->vehicle_id || ! $capture->camera_pair_id) {
                throw new RuntimeException('capture_not_classifiable');
            }

            $locationId = $capture->cameraPair?->location_id;
            if (! $locationId) {
                throw new RuntimeException('capture_location_missing');
            }

            $direction = $this->classifyDirection->execute($capture);

            return match ($direction) {
                CameraPairDirection::Outbound => $this->attachOutbound($capture, $locationId),
                CameraPairDirection::Inbound => $this->attachInbound($capture, $locationId),
                CameraPairDirection::Unknown => $this->attachNeedsReview($capture, $locationId, $direction, 'unknown_direction', null),
            };
        });
    }

    private function attachOutbound(CaptureEvent $capture, int $locationId): TripEvent
    {
        $openTrip = $this->openTrip($capture, $locationId);
        if ($openTrip) {
            $openTrip->update([
                'status' => TripStatus::NeedsReview,
                'review_required_reason' => 'new_outbound_before_inbound',
            ]);
        }

        $trip = Trip::query()->create([
            'vehicle_id' => $capture->vehicle_id,
            'location_id' => $locationId,
            'status' => TripStatus::Open,
            'opened_at' => $capture->event_time,
            'closed_at' => null,
            'review_required_reason' => null,
        ]);

        return $this->createEvent($trip, $capture, CameraPairDirection::Outbound);
    }

    private function attachInbound(CaptureEvent $capture, int $locationId): TripEvent
    {
        $openTrip = $this->openTrip($capture, $locationId);

        if (! $openTrip) {
            return $this->attachNeedsReview($capture, $locationId, CameraPairDirection::Inbound, 'inbound_without_outbound', $capture->event_time);
        }

        if ($capture->event_time && $openTrip->opened_at && $capture->event_time->lt($openTrip->opened_at)) {
            return $this->attachNeedsReview($capture, $locationId, CameraPairDirection::Inbound, 'inbound_before_open_trip', $capture->event_time);
        }

        $event = $this->createEvent($openTrip, $capture, CameraPairDirection::Inbound);
        $openTrip->update([
            'status' => TripStatus::Closed,
            'closed_at' => $capture->event_time,
            'review_required_reason' => null,
        ]);

        return $event;
    }

    private function attachNeedsReview(CaptureEvent $capture, int $locationId, CameraPairDirection $direction, string $reason, ?CarbonImmutable $closedAt): TripEvent
    {
        $trip = Trip::query()->create([
            'vehicle_id' => $capture->vehicle_id,
            'location_id' => $locationId,
            'status' => TripStatus::NeedsReview,
            'opened_at' => $capture->event_time,
            'closed_at' => $closedAt,
            'review_required_reason' => $reason,
        ]);

        return $this->createEvent($trip, $capture, $direction);
    }

    private function openTrip(CaptureEvent $capture, int $locationId): ?Trip
    {
        return Trip::query()
            ->where('vehicle_id', $capture->vehicle_id)
            ->where('location_id', $locationId)
            ->where('status', TripStatus::Open)
            ->latest('opened_at')
            ->lockForUpdate()
            ->first();
    }

    private function createEvent(Trip $trip, CaptureEvent $capture, CameraPairDirection $direction): TripEvent
    {
        return TripEvent::query()->create([
            'trip_id' => $trip->id,
            'capture_event_id' => $capture->id,
            'direction' => $direction,
            'load_status' => $capture->load_status,
            'occurred_at' => $capture->event_time,
        ]);
    }
}
```

- [ ] **Step 6: Integrate classification into parent capture ingest**

Modify `app/Actions/Edge/UpsertEdgeCaptureAction.php` constructor:

```php
use App\Actions\Trips\AttachCaptureToTripAction;
```

```php
public function __construct(
    private readonly StoreSyncedCaptureMediaAction $storeMedia,
    private readonly AttachCaptureToTripAction $attachCaptureToTrip,
) {}
```

After successful media storage for a newly created capture, before returning accepted:

```php
$this->attachCaptureToTrip->execute($capture->refresh());
```

When `existingResult()` returns a duplicate, call the action before returning duplicate only after `matchesReplay(...)` is true:

```php
$this->attachCaptureToTrip->execute($existingByIdempotencyKey);

return new EdgeBatchItemResultData($item->captureUuid, $item->idempotencyKey, 'duplicate');
```

Do not call trip classification for rejected, divergent, inactive vehicle, or camera-pair mismatch results.

- [ ] **Step 7: Add ingest integration test**

Append to `tests/Feature/Edge/EdgeCaptureBatchIngestTest.php`:

```php
public function test_accepted_parent_capture_is_attached_to_trip(): void
{
    [$edgeNode, $vehicle, $pair, $lpr, $support] = $this->fixtures();
    $this->actingAsCaptureWriter();

    $this->postBatch($edgeNode, $this->metadata($vehicle, $pair, $lpr, $support))
        ->assertOk()
        ->assertJsonPath('data.accepted.0.capture_uuid', 'aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa');

    $capture = CaptureEvent::query()->where('uuid', 'aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa')->firstOrFail();

    $this->assertDatabaseHas('trip_events', ['capture_event_id' => $capture->id]);
    $this->assertDatabaseHas('trips', [
        'vehicle_id' => $vehicle->id,
        'location_id' => $pair->location_id,
        'status' => 'open',
    ]);
}
```

- [ ] **Step 8: Run focused tests and verify GREEN**

Run:

```bash
php artisan test --filter=TripClassificationTest
php artisan test --filter=test_accepted_parent_capture_is_attached_to_trip
```

Expected: PASS.

- [ ] **Step 9: Run capture ingest regression suite**

Run:

```bash
php artisan test tests/Feature/Edge/EdgeCaptureBatchIngestTest.php
```

Expected: PASS. This guards idempotency and replay behavior after injecting trip classification.

- [ ] **Step 10: Commit backend classification task**

Run:

```bash
git add app/Actions/Trips/ClassifyTripDirectionAction.php app/Actions/Trips/AttachCaptureToTripAction.php app/Actions/Edge/UpsertEdgeCaptureAction.php tests/Feature/Trips/TripClassificationTest.php tests/Feature/Edge/EdgeCaptureBatchIngestTest.php
git commit -m "feat: classify captures into trips"
```

---

## Task 3: Backend Admin Trip API, Load Review, And Private Media

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Actions/Trips/UpdateLoadStatusAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/TripController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/TripLoadReviewController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/MediaAssetContentController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/IndexTripRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/UpdateTripEventLoadStatusRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/TripEventResource.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/TripResource.php`
- Modify: `RIALMA-TrackVision-Backend/routes/api.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Admin/TripAdminTest.php`

**Interfaces:**

- Consumes: `Trip`, `TripEvent`, `MediaAsset`, `LoadStatus`, `TripStatus`.
- Produces: `GET /api/v1/admin/trips`.
- Produces: `GET /api/v1/admin/trips/{trip}`.
- Produces: `PATCH /api/v1/admin/trip-events/{tripEvent}/load-status`.
- Produces: `GET /api/v1/admin/media-assets/{mediaAsset}/content`.
- Produces: `UpdateLoadStatusAction::execute(TripEvent $tripEvent, LoadStatus $loadStatus): TripEvent`.

- [ ] **Step 1: Write failing admin API tests**

Create `tests/Feature/Admin/TripAdminTest.php`:

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
use App\Models\User;
use Database\Seeders\PermissionSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Storage;
use Laravel\Passport\Passport;
use Tests\TestCase;

class TripAdminTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_with_capture_permission_can_list_trips(): void
    {
        $this->actingWithRole('auditor');
        [$trip] = $this->tripFixture();

        $this->getJson('/api/v1/admin/trips')
            ->assertOk()
            ->assertJsonPath('data.0.id', $trip->id)
            ->assertJsonPath('data.0.status', 'open')
            ->assertJsonPath('data.0.events_count', 1)
            ->assertJsonPath('data.0.current_load_status', 'unknown');
    }

    public function test_trip_list_filters_by_plate_status_and_load_status(): void
    {
        $this->actingWithRole('super_admin');
        [$trip] = $this->tripFixture(['plate_normalized' => 'ABC1D23']);
        $otherCapture = CaptureEvent::factory()->create(['plate_normalized' => 'XYZ9Z99']);
        $otherTrip = Trip::factory()->create([
            'vehicle_id' => $otherCapture->vehicle_id,
            'location_id' => $otherCapture->cameraPair->location_id,
            'status' => TripStatus::Closed,
        ]);
        TripEvent::factory()->create([
            'trip_id' => $otherTrip->id,
            'capture_event_id' => $otherCapture->id,
            'load_status' => LoadStatus::Loaded,
        ]);

        $this->getJson('/api/v1/admin/trips?plate=ABC-1D23&status=open&load_status=unknown')
            ->assertOk()
            ->assertJsonCount(1, 'data')
            ->assertJsonPath('data.0.id', $trip->id);
    }

    public function test_user_with_capture_permission_can_show_trip_detail_with_media_references(): void
    {
        $this->actingWithRole('auditor');
        [$trip, $event, $capture, $media] = $this->tripFixture();

        $this->getJson("/api/v1/admin/trips/{$trip->id}")
            ->assertOk()
            ->assertJsonPath('data.id', $trip->id)
            ->assertJsonPath('data.events.0.id', $event->id)
            ->assertJsonPath('data.events.0.capture.id', $capture->id)
            ->assertJsonPath('data.events.0.media.lpr_image.id', $media->id)
            ->assertJsonPath('data.events.0.media.lpr_image.content_endpoint', "/api/v1/admin/media-assets/{$media->id}/content");
    }

    public function test_load_status_update_requires_trips_manage(): void
    {
        $this->actingWithRole('auditor');
        [, $event] = $this->tripFixture();

        $this->patchJson("/api/v1/admin/trip-events/{$event->id}/load-status", [
            'load_status' => 'loaded',
        ])->assertForbidden();
    }

    public function test_user_with_trips_manage_can_update_load_status_and_capture_event(): void
    {
        $this->actingWithRole('operator');
        [, $event, $capture] = $this->tripFixture();

        $this->patchJson("/api/v1/admin/trip-events/{$event->id}/load-status", [
            'load_status' => 'loaded',
        ])
            ->assertOk()
            ->assertJsonPath('data.load_status', 'loaded');

        $this->assertSame(LoadStatus::Loaded, $event->refresh()->load_status);
        $this->assertSame(LoadStatus::Loaded, $capture->refresh()->load_status);
    }

    public function test_invalid_load_status_is_rejected(): void
    {
        $this->actingWithRole('operator');
        [, $event] = $this->tripFixture();

        $this->patchJson("/api/v1/admin/trip-events/{$event->id}/load-status", [
            'load_status' => 'invented',
        ])
            ->assertUnprocessable()
            ->assertJsonValidationErrors('load_status');
    }

    public function test_private_media_content_requires_capture_permission(): void
    {
        $this->actingWithRole('edge_service');
        [, , , $media] = $this->tripFixture();

        $this->get("/api/v1/admin/media-assets/{$media->id}/content")
            ->assertForbidden();
    }

    public function test_private_media_content_returns_jpeg(): void
    {
        Storage::fake('local');
        $this->actingWithRole('auditor');
        [, , , $media] = $this->tripFixture();
        Storage::disk('local')->put($media->path, 'jpeg-bytes');

        $response = $this->get("/api/v1/admin/media-assets/{$media->id}/content");

        $response->assertOk();
        $response->assertHeader('content-type', 'image/jpeg');
        $this->assertSame('jpeg-bytes', $response->streamedContent());
    }

    public function test_private_media_content_returns_not_found_when_file_is_missing(): void
    {
        Storage::fake('local');
        $this->actingWithRole('auditor');
        [, , , $media] = $this->tripFixture();

        $this->get("/api/v1/admin/media-assets/{$media->id}/content")
            ->assertNotFound();
    }

    private function actingWithRole(string $role): User
    {
        $this->seed(PermissionSeeder::class);
        $user = User::factory()->create();
        $user->assignRole($role);
        Passport::actingAs($user, ['admin:read']);

        return $user;
    }

    private function tripFixture(array $captureOverrides = []): array
    {
        $capture = CaptureEvent::factory()->create(array_merge([
            'direction' => CameraPairDirection::Outbound,
            'load_status' => LoadStatus::Unknown,
        ], $captureOverrides));
        $trip = Trip::factory()->create([
            'vehicle_id' => $capture->vehicle_id,
            'location_id' => $capture->cameraPair->location_id,
            'status' => TripStatus::Open,
            'opened_at' => $capture->event_time,
        ]);
        $event = TripEvent::factory()->create([
            'trip_id' => $trip->id,
            'capture_event_id' => $capture->id,
            'direction' => $capture->direction,
            'load_status' => $capture->load_status,
            'occurred_at' => $capture->event_time,
        ]);
        $media = MediaAsset::factory()->create([
            'capture_event_id' => $capture->id,
            'camera_id' => $capture->lpr_camera_id,
            'kind' => MediaAssetKind::LprImage,
            'disk' => 'local',
            'path' => 'captures/'.$capture->uuid.'/lpr_image/test.jpg',
            'content_type' => 'image/jpeg',
            'byte_size' => 10,
        ]);

        return [$trip, $event, $capture, $media];
    }
}
```

- [ ] **Step 2: Run admin API tests and verify RED**

Run:

```bash
php artisan test --filter=TripAdminTest
```

Expected: FAIL because controllers, requests, resources, routes, and action do not exist.

- [ ] **Step 3: Create `IndexTripRequest`**

Create `app/Http/Requests/Api/V1/Admin/IndexTripRequest.php`:

```php
<?php

namespace App\Http\Requests\Api\V1\Admin;

use App\Enums\LoadStatus;
use App\Enums\TripStatus;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class IndexTripRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('captures.view') === true;
    }

    public function rules(): array
    {
        return [
            'status' => ['sometimes', Rule::enum(TripStatus::class)],
            'vehicle_id' => ['sometimes', 'integer', 'exists:vehicles,id'],
            'location_id' => ['sometimes', 'integer', 'exists:locations,id'],
            'plate' => ['sometimes', 'string', 'max:20'],
            'load_status' => ['sometimes', Rule::enum(LoadStatus::class)],
            'date_from' => ['sometimes', 'date'],
            'date_to' => ['sometimes', 'date'],
        ];
    }
}
```

- [ ] **Step 4: Create `UpdateTripEventLoadStatusRequest`**

Create `app/Http/Requests/Api/V1/Admin/UpdateTripEventLoadStatusRequest.php`:

```php
<?php

namespace App\Http\Requests\Api\V1\Admin;

use App\Enums\LoadStatus;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdateTripEventLoadStatusRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()?->can('trips.manage') === true;
    }

    public function rules(): array
    {
        return [
            'load_status' => ['required', Rule::enum(LoadStatus::class)],
        ];
    }
}
```

- [ ] **Step 5: Create `UpdateLoadStatusAction`**

Create `app/Actions/Trips/UpdateLoadStatusAction.php`:

```php
<?php

namespace App\Actions\Trips;

use App\Enums\LoadStatus;
use App\Models\TripEvent;
use Illuminate\Support\Facades\DB;

class UpdateLoadStatusAction
{
    public function execute(TripEvent $tripEvent, LoadStatus $loadStatus): TripEvent
    {
        return DB::transaction(function () use ($tripEvent, $loadStatus): TripEvent {
            $tripEvent->update(['load_status' => $loadStatus]);
            $tripEvent->captureEvent()->update(['load_status' => $loadStatus]);

            return $tripEvent->refresh()->load(['captureEvent.mediaAssets', 'captureEvent.cameraPair']);
        });
    }
}
```

- [ ] **Step 6: Create Resources**

Create `app/Http/Resources/Api/V1/TripEventResource.php`:

```php
<?php

namespace App\Http\Resources\Api\V1;

use App\Enums\MediaAssetKind;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class TripEventResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        $this->resource->loadMissing(['captureEvent.mediaAssets', 'captureEvent.cameraPair']);
        $capture = $this->captureEvent;
        $media = $capture->mediaAssets
            ->mapWithKeys(fn ($asset): array => [
                $asset->kind->value => [
                    'id' => $asset->id,
                    'uuid' => $asset->uuid,
                    'kind' => $asset->kind->value,
                    'content_type' => $asset->content_type,
                    'byte_size' => $asset->byte_size,
                    'content_endpoint' => "/api/v1/admin/media-assets/{$asset->id}/content",
                ],
            ]);

        return [
            'id' => $this->id,
            'uuid' => $this->uuid,
            'direction' => $this->direction?->value,
            'load_status' => $this->load_status?->value,
            'occurred_at' => $this->occurred_at?->toJSON(),
            'capture' => [
                'id' => $capture->id,
                'uuid' => $capture->uuid,
                'plate' => $capture->plate,
                'plate_normalized' => $capture->plate_normalized,
                'event_time' => $capture->event_time?->toJSON(),
                'camera_pair' => [
                    'id' => $capture->cameraPair?->id,
                    'uuid' => $capture->cameraPair?->uuid,
                    'name' => $capture->cameraPair?->name,
                ],
            ],
            'media' => [
                MediaAssetKind::LprImage->value => $media->get(MediaAssetKind::LprImage->value),
                MediaAssetKind::SupportImage->value => $media->get(MediaAssetKind::SupportImage->value),
            ],
        ];
    }
}
```

Create `app/Http/Resources/Api/V1/TripResource.php`:

```php
<?php

namespace App\Http\Resources\Api\V1;

use App\Enums\LoadStatus;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class TripResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        $this->resource->loadMissing(['vehicle', 'location']);

        return [
            'id' => $this->id,
            'uuid' => $this->uuid,
            'status' => $this->status?->value,
            'opened_at' => $this->opened_at?->toJSON(),
            'closed_at' => $this->closed_at?->toJSON(),
            'review_required_reason' => $this->review_required_reason,
            'vehicle' => [
                'id' => $this->vehicle?->id,
                'uuid' => $this->vehicle?->uuid,
                'plate' => $this->vehicle?->plate,
                'plate_normalized' => $this->vehicle?->plate_normalized,
                'fleet_code' => $this->vehicle?->fleet_code,
            ],
            'location' => [
                'id' => $this->location?->id,
                'uuid' => $this->location?->uuid,
                'name' => $this->location?->name,
            ],
            'events_count' => $this->whenCounted('events'),
            'current_load_status' => $this->currentLoadStatus(),
            'events' => TripEventResource::collection($this->whenLoaded('events')),
        ];
    }

    private function currentLoadStatus(): string
    {
        if (! $this->relationLoaded('events')) {
            return LoadStatus::Unknown->value;
        }

        return $this->events
            ->sortByDesc('occurred_at')
            ->first(fn ($event): bool => $event->load_status !== LoadStatus::Unknown)
            ?->load_status
            ?->value ?? LoadStatus::Unknown->value;
    }
}
```

- [ ] **Step 7: Create controllers**

Create `app/Http/Controllers/Api/V1/Admin/TripController.php`:

```php
<?php

namespace App\Http\Controllers\Api\V1\Admin;

use App\Http\Controllers\Controller;
use App\Http\Requests\Api\V1\Admin\IndexTripRequest;
use App\Http\Resources\Api\V1\TripResource;
use App\Models\Trip;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class TripController extends Controller
{
    public function index(IndexTripRequest $request, \App\Support\Vehicles\NormalizePlate $normalizePlate): AnonymousResourceCollection
    {
        $filters = $request->validated();

        $query = Trip::query()
            ->with([
                'vehicle',
                'location',
                'events' => fn ($query) => $query->select('id', 'trip_id', 'load_status', 'occurred_at')->orderBy('occurred_at'),
            ])
            ->withCount('events')
            ->when($filters['status'] ?? null, fn ($query, $status) => $query->where('status', $status))
            ->when($filters['vehicle_id'] ?? null, fn ($query, $vehicleId) => $query->where('vehicle_id', $vehicleId))
            ->when($filters['location_id'] ?? null, fn ($query, $locationId) => $query->where('location_id', $locationId))
            ->when($filters['plate'] ?? null, function ($query, $plate): void {
                $query->whereHas('vehicle', fn ($vehicleQuery) => $vehicleQuery->where('plate_normalized', 'like', '%'.$normalizePlate($plate).'%'));
            })
            ->when($filters['load_status'] ?? null, fn ($query, $loadStatus) => $query->whereHas('events', fn ($eventQuery) => $eventQuery->where('load_status', $loadStatus)))
            ->when($filters['date_from'] ?? null, fn ($query, $dateFrom) => $query->where('opened_at', '>=', $dateFrom))
            ->when($filters['date_to'] ?? null, fn ($query, $dateTo) => $query->where('opened_at', '<=', $dateTo))
            ->latest('opened_at');

        return TripResource::collection($query->paginate());
    }

    public function show(Trip $trip): TripResource
    {
        return TripResource::make(
            $trip->load([
                'vehicle',
                'location',
                'events' => fn ($query) => $query->orderBy('occurred_at'),
                'events.captureEvent.cameraPair',
                'events.captureEvent.mediaAssets',
            ]),
        );
    }
}
```

Create `app/Http/Controllers/Api/V1/Admin/TripLoadReviewController.php`:

```php
<?php

namespace App\Http\Controllers\Api\V1\Admin;

use App\Actions\Trips\UpdateLoadStatusAction;
use App\Enums\LoadStatus;
use App\Http\Controllers\Controller;
use App\Http\Requests\Api\V1\Admin\UpdateTripEventLoadStatusRequest;
use App\Http\Resources\Api\V1\TripEventResource;
use App\Models\TripEvent;

class TripLoadReviewController extends Controller
{
    public function __invoke(UpdateTripEventLoadStatusRequest $request, TripEvent $tripEvent, UpdateLoadStatusAction $action): TripEventResource
    {
        return TripEventResource::make(
            $action->execute($tripEvent, LoadStatus::from($request->validated('load_status'))),
        );
    }
}
```

Create `app/Http/Controllers/Api/V1/Admin/MediaAssetContentController.php`:

```php
<?php

namespace App\Http\Controllers\Api\V1\Admin;

use App\Http\Controllers\Controller;
use App\Models\MediaAsset;
use Illuminate\Support\Facades\Storage;
use Symfony\Component\HttpFoundation\StreamedResponse;

class MediaAssetContentController extends Controller
{
    public function __invoke(MediaAsset $mediaAsset): StreamedResponse
    {
        if (! Storage::disk($mediaAsset->disk)->exists($mediaAsset->path)) {
            abort(404);
        }

        return Storage::disk($mediaAsset->disk)->response(
            $mediaAsset->path,
            basename($mediaAsset->path),
            ['Content-Type' => $mediaAsset->content_type],
        );
    }
}
```

- [ ] **Step 8: Register routes**

Modify `routes/api.php` imports:

```php
use App\Http\Controllers\Api\V1\Admin\MediaAssetContentController;
use App\Http\Controllers\Api\V1\Admin\TripController;
use App\Http\Controllers\Api\V1\Admin\TripLoadReviewController;
```

Inside the existing `/admin` group, after `camera-pairs` routes:

```php
Route::get('/trips', [TripController::class, 'index'])
    ->middleware('permission:captures.view,api');

Route::get('/trips/{trip}', [TripController::class, 'show'])
    ->middleware('permission:captures.view,api');

Route::patch('/trip-events/{tripEvent}/load-status', TripLoadReviewController::class)
    ->middleware('permission:trips.manage,api');

Route::get('/media-assets/{mediaAsset}/content', MediaAssetContentController::class)
    ->middleware('permission:captures.view,api');
```

- [ ] **Step 9: Run admin API tests and verify GREEN**

Run:

```bash
php artisan test --filter=TripAdminTest
```

Expected: PASS.

- [ ] **Step 10: Run classification and route checks**

Run:

```bash
php artisan test --filter=TripClassificationTest
php artisan route:list --path=api/v1/admin/trips
php artisan route:list --path=api/v1/admin/trip-events
php artisan route:list --path=api/v1/admin/media-assets
```

Expected: tests PASS and all four admin trip/media routes are visible across the three route-list commands.

- [ ] **Step 11: Run backend full suite**

Run:

```bash
composer validate
php artisan test
```

Expected: PASS.

- [ ] **Step 12: Commit backend admin API task**

Run:

```bash
git add app/Actions/Trips/UpdateLoadStatusAction.php app/Http/Controllers/Api/V1/Admin/TripController.php app/Http/Controllers/Api/V1/Admin/TripLoadReviewController.php app/Http/Controllers/Api/V1/Admin/MediaAssetContentController.php app/Http/Requests/Api/V1/Admin/IndexTripRequest.php app/Http/Requests/Api/V1/Admin/UpdateTripEventLoadStatusRequest.php app/Http/Resources/Api/V1/TripEventResource.php app/Http/Resources/Api/V1/TripResource.php routes/api.php tests/Feature/Admin/TripAdminTest.php
git commit -m "feat: add trip review admin api"
```

---

## Task 4: Frontend Trip Types, Services, Routing, And Navigation

**Files:**

- Modify: `RIALMA-TrackVision-Frontend/src/types/admin.ts`
- Create: `RIALMA-TrackVision-Frontend/src/services/tripsService.ts`
- Create: `RIALMA-TrackVision-Frontend/src/services/tripsService.test.ts`
- Create: `RIALMA-TrackVision-Frontend/src/services/mediaAssetsService.ts`
- Create: `RIALMA-TrackVision-Frontend/src/services/mediaAssetsService.test.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/router/index.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/router/routerGuard.test.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/components/navigation/TheSidebar.vue`
- Modify: `RIALMA-TrackVision-Frontend/src/components/navigation/TheSidebar.test.ts`
- Create: `RIALMA-TrackVision-Frontend/src/pages/TripsPage.vue`

**Interfaces:**

- Consumes: backend `GET /admin/trips`, `GET /admin/trips/{trip}`, `PATCH /admin/trip-events/{id}/load-status`, `GET /admin/media-assets/{id}/content`.
- Produces: `tripsService.list(filters?: TripFilters): Promise<LaravelPaginated<Trip>>`.
- Produces: `tripsService.show(trip: Trip): Promise<Trip>`.
- Produces: `tripsService.updateLoadStatus(tripEvent: TripEvent, loadStatus: LoadStatus): Promise<TripEvent>`.
- Produces: `mediaAssetsService.fetchObjectUrl(endpoint: string): Promise<string>`.
- Produces: route name `trips` at `/trips`, meta permission `captures.view`.

- [ ] **Step 1: Create initial TripsPage shell**

Create `src/pages/TripsPage.vue` so router imports can compile in this task:

```vue
<script setup lang="ts">
</script>

<template>
  <section class="page-section">
    <header class="page-header">
      <div>
        <p class="page-eyebrow">
          Operacao
        </p>
        <h1>Viagens</h1>
      </div>
    </header>
  </section>
</template>
```

- [ ] **Step 2: Add failing service and navigation tests**

Create `src/services/tripsService.test.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { tripsService } from './tripsService'

const fetchMock = vi.fn()
vi.stubGlobal('fetch', fetchMock)

describe('tripsService', () => {
  beforeEach(() => {
    localStorage.setItem('trackvision.token', 'token-123')
    fetchMock.mockReset()
  })

  it('lists trips with filters', async () => {
    fetchMock.mockResolvedValueOnce(new Response(JSON.stringify({ data: [] }), { status: 200 }))

    await tripsService.list({ status: 'open', plate: 'ABC-1D23', load_status: 'unknown' })

    const [url, init] = fetchMock.mock.calls[0]
    expect(url).toContain('/admin/trips?')
    expect(url).toContain('status=open')
    expect(url).toContain('plate=ABC-1D23')
    expect(url).toContain('load_status=unknown')
    expect((init.headers as Headers).get('Authorization')).toBe('Bearer token-123')
  })

  it('updates trip event load status', async () => {
    fetchMock.mockResolvedValueOnce(new Response(JSON.stringify({
      data: { id: 10, uuid: 'event-uuid', direction: 'outbound', load_status: 'loaded', occurred_at: '2026-08-13T10:00:00Z', capture: {}, media: {} },
    }), { status: 200 }))

    const event = await tripsService.updateLoadStatus({ id: 10 } as never, 'loaded')

    expect(event.load_status).toBe('loaded')
    expect(fetchMock.mock.calls[0][0]).toContain('/admin/trip-events/10/load-status')
    expect(fetchMock.mock.calls[0][1].method).toBe('PATCH')
    expect(fetchMock.mock.calls[0][1].body).toBe(JSON.stringify({ load_status: 'loaded' }))
  })
})
```

Create `src/services/mediaAssetsService.test.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { mediaAssetsService } from './mediaAssetsService'

const fetchMock = vi.fn()
vi.stubGlobal('fetch', fetchMock)

describe('mediaAssetsService', () => {
  beforeEach(() => {
    localStorage.setItem('trackvision.token', 'token-123')
    fetchMock.mockReset()
    vi.stubGlobal('URL', {
      createObjectURL: vi.fn().mockReturnValue('blob:trackvision-image'),
      revokeObjectURL: vi.fn(),
    })
  })

  it('fetches private media as an authenticated blob object url', async () => {
    fetchMock.mockResolvedValueOnce(new Response(new Blob(['jpeg-bytes'], { type: 'image/jpeg' }), { status: 200 }))

    const url = await mediaAssetsService.fetchObjectUrl('/api/v1/admin/media-assets/5/content')

    expect(url).toBe('blob:trackvision-image')
    expect(fetchMock.mock.calls[0][0]).toContain('/api/v1/admin/media-assets/5/content')
    expect((fetchMock.mock.calls[0][1].headers as Headers).get('Authorization')).toBe('Bearer token-123')
  })

  it('revokes object urls', () => {
    mediaAssetsService.revokeObjectUrl('blob:trackvision-image')

    expect(URL.revokeObjectURL).toHaveBeenCalledWith('blob:trackvision-image')
  })
})
```

Modify `src/router/routerGuard.test.ts` by adding:

```ts
it('allows authenticated users with captures view to access trips', async () => {
  const store = useAuthStore()
  store.token = 'token-123'
  store.user = { id: 1, name: 'Paulo', email: 'paulo@example.com' }
  store.permissions = ['captures.view']

  const router = createAppRouter()

  await router.push('/trips')
  await router.isReady()

  expect(router.currentRoute.value.name).toBe('trips')
})
```

Modify `src/components/navigation/TheSidebar.test.ts` so routes include `/trips`, then assert:

```ts
authStore.permissions = ['captures.view']

expect(wrapper.text()).toContain('Viagens')
expect(wrapper.text()).not.toContain('Veiculos')
```

- [ ] **Step 3: Run frontend focused tests and verify RED**

Run:

```bash
npm test -- --run src/services/tripsService.test.ts src/services/mediaAssetsService.test.ts src/router/routerGuard.test.ts src/components/navigation/TheSidebar.test.ts
```

Expected: FAIL because services, types, route, and sidebar item are missing.

- [ ] **Step 4: Add trip types**

Modify `src/types/admin.ts`:

```ts
export type TripStatus = 'open' | 'closed' | 'needs_review'
export type TripEventDirection = 'outbound' | 'inbound' | 'unknown'
export type LoadStatus = 'unknown' | 'loaded' | 'empty' | 'needs_review'

export interface TripMediaAsset {
  id: number
  uuid: string
  kind: 'lpr_image' | 'support_image'
  content_type: string
  byte_size: number
  content_endpoint: string
}

export interface TripEvent {
  id: number
  uuid: string
  direction: TripEventDirection
  load_status: LoadStatus
  occurred_at: string | null
  capture: {
    id: number
    uuid: string
    plate: string | null
    plate_normalized: string | null
    event_time: string | null
    camera_pair?: {
      id: number | null
      uuid: string | null
      name: string | null
    }
  }
  media: {
    lpr_image?: TripMediaAsset | null
    support_image?: TripMediaAsset | null
  }
}

export interface Trip {
  id: number
  uuid: string
  status: TripStatus
  opened_at: string | null
  closed_at: string | null
  review_required_reason: string | null
  current_load_status: LoadStatus
  events_count?: number
  vehicle?: Pick<Vehicle, 'id' | 'uuid' | 'plate' | 'plate_normalized' | 'fleet_code'>
  location?: Pick<Location, 'id' | 'uuid' | 'name'>
  events?: TripEvent[]
}

export interface TripFilters {
  status?: TripStatus | ''
  plate?: string
  load_status?: LoadStatus | ''
}
```

- [ ] **Step 5: Create `tripsService`**

Create `src/services/tripsService.ts`:

```ts
import { getAppConfig } from '@/app/config'
import type { LaravelPaginated, LaravelResource } from '@/types/api'
import type { LoadStatus, Trip, TripEvent, TripFilters } from '@/types/admin'
import { createApiClient } from './apiClient'

const client = createApiClient({
  apiBaseUrl: getAppConfig().apiBaseUrl,
  getToken: () => localStorage.getItem('trackvision.token'),
})

function queryFrom(filters: TripFilters = {}): string {
  const params = new URLSearchParams()

  if (filters.status) params.set('status', filters.status)
  if (filters.plate?.trim()) params.set('plate', filters.plate.trim())
  if (filters.load_status) params.set('load_status', filters.load_status)

  const query = params.toString()
  return query ? `?${query}` : ''
}

export const tripsService = {
  list(filters: TripFilters = {}): Promise<LaravelPaginated<Trip>> {
    return client.get<LaravelPaginated<Trip>>(`/admin/trips${queryFrom(filters)}`)
  },

  async show(trip: Trip): Promise<Trip> {
    const response = await client.get<LaravelResource<Trip>>(`/admin/trips/${trip.id}`)
    return response.data
  },

  async updateLoadStatus(tripEvent: TripEvent, loadStatus: LoadStatus): Promise<TripEvent> {
    const response = await client.patch<LaravelResource<TripEvent>>(
      `/admin/trip-events/${tripEvent.id}/load-status`,
      { load_status: loadStatus },
    )
    return response.data
  },
}
```

- [ ] **Step 6: Create `mediaAssetsService`**

Create `src/services/mediaAssetsService.ts`:

```ts
import { getAppConfig } from '@/app/config'
import { ApiError } from './apiClient'

function joinUrl(baseUrl: string, path: string): string {
  return `${baseUrl.replace(/\/$/, '')}/${path.replace(/^\//, '')}`
}

function normalizeApiEndpoint(endpoint: string): string {
  return endpoint.replace(/^\/api\/v1(?=\/)/, '')
}

export const mediaAssetsService = {
  async fetchObjectUrl(endpoint: string): Promise<string> {
    const headers = new Headers()
    headers.set('Accept', 'image/jpeg')

    const token = localStorage.getItem('trackvision.token')
    if (token) {
      headers.set('Authorization', `Bearer ${token}`)
    }

    const response = await fetch(joinUrl(getAppConfig().apiBaseUrl, normalizeApiEndpoint(endpoint)), { headers })

    if (!response.ok) {
      throw new ApiError(response.status, 'Nao foi possivel carregar a imagem privada.')
    }

    return URL.createObjectURL(await response.blob())
  },

  revokeObjectUrl(url: string): void {
    URL.revokeObjectURL(url)
  },
}
```

- [ ] **Step 7: Register route and sidebar item**

Modify `src/router/index.ts`:

```ts
import TripsPage from '@/pages/TripsPage.vue'
```

Add child route under admin layout:

```ts
{
  path: 'trips',
  name: 'trips',
  component: TripsPage,
  meta: { permission: 'captures.view' },
},
```

Modify `src/components/navigation/TheSidebar.vue` imports:

```ts
import { Route } from 'lucide-vue-next'
```

Add navigation item:

```ts
{ label: 'Viagens', route: 'trips', permission: 'captures.view', icon: Route },
```

- [ ] **Step 8: Run focused frontend tests and verify GREEN**

Run:

```bash
npm test -- --run src/services/tripsService.test.ts src/services/mediaAssetsService.test.ts src/router/routerGuard.test.ts src/components/navigation/TheSidebar.test.ts
```

Expected: PASS.

- [ ] **Step 9: Run frontend build**

Run:

```bash
npm run build
```

Expected: PASS.

- [ ] **Step 10: Commit frontend routing/services task**

Run:

```bash
git add src/types/admin.ts src/services/tripsService.ts src/services/tripsService.test.ts src/services/mediaAssetsService.ts src/services/mediaAssetsService.test.ts src/router/index.ts src/router/routerGuard.test.ts src/components/navigation/TheSidebar.vue src/components/navigation/TheSidebar.test.ts src/pages/TripsPage.vue
git commit -m "feat: add trip review routes and services"
```

---

## Task 5: Frontend Trip Review Page

**Files:**

- Modify: `RIALMA-TrackVision-Frontend/src/pages/TripsPage.vue`
- Create: `RIALMA-TrackVision-Frontend/src/pages/TripsPage.test.ts`

**Interfaces:**

- Consumes: `tripsService.list`, `tripsService.show`, `tripsService.updateLoadStatus`.
- Consumes: `mediaAssetsService.fetchObjectUrl`, `mediaAssetsService.revokeObjectUrl`.
- Consumes: `useAuthStore().can('trips.manage')`.
- Produces: operational page that lists trips, shows selected trip details, displays private LPR/support images, and updates event load status.

- [ ] **Step 1: Add TripsPage behavior tests**

Create `src/pages/TripsPage.test.ts`:

```ts
import { mount } from '@vue/test-utils'
import { createPinia, setActivePinia } from 'pinia'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { useAuthStore } from '@/stores/authStore'
import TripsPage from './TripsPage.vue'

const trip = {
  id: 1,
  uuid: 'trip-uuid',
  status: 'open',
  opened_at: '2026-08-13T10:00:00Z',
  closed_at: null,
  review_required_reason: null,
  current_load_status: 'unknown',
  events_count: 1,
  vehicle: { id: 1, uuid: 'vehicle-uuid', plate: 'ABC-1D23', plate_normalized: 'ABC1D23', fleet_code: 'TRUCK-01' },
  location: { id: 1, uuid: 'location-uuid', name: 'Portaria' },
}

const detailedTrip = {
  ...trip,
  events: [{
    id: 7,
    uuid: 'event-uuid',
    direction: 'outbound',
    load_status: 'unknown',
    occurred_at: '2026-08-13T10:00:00Z',
    capture: {
      id: 3,
      uuid: 'capture-uuid',
      plate: 'ABC-1D23',
      plate_normalized: 'ABC1D23',
      event_time: '2026-08-13T10:00:00Z',
      camera_pair: { id: 1, uuid: 'pair-uuid', name: 'Entrada 01' },
    },
    media: {
      lpr_image: { id: 11, uuid: 'lpr-media', kind: 'lpr_image', content_type: 'image/jpeg', byte_size: 10, content_endpoint: '/api/v1/admin/media-assets/11/content' },
      support_image: { id: 12, uuid: 'support-media', kind: 'support_image', content_type: 'image/jpeg', byte_size: 10, content_endpoint: '/api/v1/admin/media-assets/12/content' },
    },
  }],
}

vi.mock('@/services/tripsService', () => ({
  tripsService: {
    list: vi.fn().mockResolvedValue({ data: [trip] }),
    show: vi.fn().mockResolvedValue(detailedTrip),
    updateLoadStatus: vi.fn().mockResolvedValue({ ...detailedTrip.events[0], load_status: 'loaded' }),
  },
}))

vi.mock('@/services/mediaAssetsService', () => ({
  mediaAssetsService: {
    fetchObjectUrl: vi.fn().mockResolvedValue('blob:image-url'),
    revokeObjectUrl: vi.fn(),
  },
}))

describe('TripsPage', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders trips returned by the API', async () => {
    const wrapper = mount(TripsPage)
    await new Promise((resolve) => setTimeout(resolve, 0))

    expect(wrapper.text()).toContain('ABC-1D23')
    expect(wrapper.text()).toContain('Portaria')
    expect(wrapper.text()).toContain('Aberta')
  })

  it('loads selected trip detail with LPR and support images', async () => {
    const wrapper = mount(TripsPage)
    await new Promise((resolve) => setTimeout(resolve, 0))

    await wrapper.get('[data-test="select-trip"]').trigger('click')
    await new Promise((resolve) => setTimeout(resolve, 0))

    expect(wrapper.text()).toContain('Entrada 01')
    expect(wrapper.findAll('img')).toHaveLength(2)
  })

  it('shows load actions only when user can manage trips', async () => {
    const authStore = useAuthStore()
    authStore.permissions = ['captures.view']
    const wrapper = mount(TripsPage)
    await new Promise((resolve) => setTimeout(resolve, 0))
    await wrapper.get('[data-test="select-trip"]').trigger('click')
    await new Promise((resolve) => setTimeout(resolve, 0))

    expect(wrapper.text()).not.toContain('Carregado')

    authStore.permissions = ['captures.view', 'trips.manage']
    const allowedWrapper = mount(TripsPage)
    await new Promise((resolve) => setTimeout(resolve, 0))
    await allowedWrapper.get('[data-test="select-trip"]').trigger('click')
    await new Promise((resolve) => setTimeout(resolve, 0))

    expect(allowedWrapper.text()).toContain('Carregado')
    expect(allowedWrapper.text()).toContain('Vazio')
  })
})
```

- [ ] **Step 2: Run page test and verify RED**

Run:

```bash
npm test -- --run src/pages/TripsPage.test.ts
```

Expected: FAIL because `TripsPage.vue` only contains the initial shell.

- [ ] **Step 3: Implement TripsPage script**

Replace `src/pages/TripsPage.vue` script:

```vue
<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import BaseAlert from '@/components/base/BaseAlert.vue'
import BaseButton from '@/components/base/BaseButton.vue'
import BaseInput from '@/components/base/BaseInput.vue'
import BaseSelect from '@/components/base/BaseSelect.vue'
import BaseTable from '@/components/base/BaseTable.vue'
import { mediaAssetsService } from '@/services/mediaAssetsService'
import { tripsService } from '@/services/tripsService'
import { useAuthStore } from '@/stores/authStore'
import type { LoadStatus, Trip, TripEvent, TripFilters } from '@/types/admin'

const authStore = useAuthStore()
const columns = [
  { key: 'plate', label: 'Placa' },
  { key: 'location', label: 'Local' },
  { key: 'status', label: 'Status' },
  { key: 'opened_at', label: 'Abertura' },
  { key: 'closed_at', label: 'Fechamento' },
  { key: 'load_status', label: 'Carga' },
  { key: 'actions', label: 'Acoes' },
]
const statusOptions = [
  { label: 'Todos', value: '' },
  { label: 'Aberta', value: 'open' },
  { label: 'Fechada', value: 'closed' },
  { label: 'Revisao', value: 'needs_review' },
]
const loadOptions = [
  { label: 'Todas', value: '' },
  { label: 'Nao revisada', value: 'unknown' },
  { label: 'Carregado', value: 'loaded' },
  { label: 'Vazio', value: 'empty' },
  { label: 'Precisa revisao', value: 'needs_review' },
]
const trips = ref<Trip[]>([])
const selectedTrip = ref<Trip | null>(null)
const loading = ref(true)
const detailLoading = ref(false)
const savingEventId = ref<number | null>(null)
const error = ref('')
const success = ref('')
const filters = ref<TripFilters>({ status: '', plate: '', load_status: '' })
const imageUrls = ref<Record<number, string>>({})
const canManageTrips = computed(() => authStore.can('trips.manage'))

function tripFrom(row: unknown): Trip {
  return row as Trip
}

function statusLabel(status: string): string {
  return ({ open: 'Aberta', closed: 'Fechada', needs_review: 'Revisao' } as Record<string, string>)[status] ?? status
}

function loadLabel(loadStatus: string): string {
  return ({ unknown: 'Nao revisada', loaded: 'Carregado', empty: 'Vazio', needs_review: 'Precisa revisao' } as Record<string, string>)[loadStatus] ?? loadStatus
}

function directionLabel(direction: string): string {
  return ({ outbound: 'Ida', inbound: 'Volta', unknown: 'Indefinida' } as Record<string, string>)[direction] ?? direction
}

function formatDate(value: string | null | undefined): string {
  return value ? new Intl.DateTimeFormat('pt-BR', { dateStyle: 'short', timeStyle: 'short' }).format(new Date(value)) : '-'
}

function revokeImages(): void {
  Object.values(imageUrls.value).forEach((url) => mediaAssetsService.revokeObjectUrl(url))
  imageUrls.value = {}
}

async function loadTrips(): Promise<void> {
  loading.value = true
  error.value = ''

  try {
    const response = await tripsService.list(filters.value)
    trips.value = response.data
  } catch {
    error.value = 'Nao foi possivel carregar viagens.'
  } finally {
    loading.value = false
  }
}

async function selectTrip(trip: Trip): Promise<void> {
  detailLoading.value = true
  error.value = ''
  revokeImages()

  try {
    selectedTrip.value = await tripsService.show(trip)
    await loadImages(selectedTrip.value.events ?? [])
  } catch {
    error.value = 'Nao foi possivel carregar detalhes da viagem.'
  } finally {
    detailLoading.value = false
  }
}

async function loadImages(events: TripEvent[]): Promise<void> {
  const pairs = events.flatMap((event) => [event.media.lpr_image, event.media.support_image].filter(Boolean))

  for (const media of pairs) {
    if (media) {
      imageUrls.value[media.id] = await mediaAssetsService.fetchObjectUrl(media.content_endpoint)
    }
  }
}

function mediaUrl(event: TripEvent, kind: 'lpr_image' | 'support_image'): string | undefined {
  const media = event.media[kind]
  return media ? imageUrls.value[media.id] : undefined
}

async function updateLoadStatus(event: TripEvent, loadStatus: LoadStatus): Promise<void> {
  savingEventId.value = event.id
  success.value = ''
  error.value = ''

  try {
    const updated = await tripsService.updateLoadStatus(event, loadStatus)
    event.load_status = updated.load_status
    success.value = 'Carga atualizada.'
    await loadTrips()
  } catch {
    error.value = 'Nao foi possivel atualizar a carga.'
  } finally {
    savingEventId.value = null
  }
}

onMounted(loadTrips)
onBeforeUnmount(revokeImages)
</script>
```

- [ ] **Step 4: Implement TripsPage template**

Use this template after the script:

```vue
<template>
  <section class="page-section">
    <header class="page-header">
      <div>
        <p class="page-eyebrow">
          Operacao
        </p>
        <h1>Viagens</h1>
      </div>
      <BaseButton
        type="button"
        variant="secondary"
        @click="loadTrips"
      >
        Atualizar
      </BaseButton>
    </header>

    <div class="filters-row">
      <BaseSelect
        v-model="filters.status"
        label="Status"
        :options="statusOptions"
      />
      <BaseInput
        v-model="filters.plate"
        label="Placa"
        placeholder="ABC-1D23"
      />
      <BaseSelect
        v-model="filters.load_status"
        label="Carga"
        :options="loadOptions"
      />
      <BaseButton
        type="button"
        @click="loadTrips"
      >
        Filtrar
      </BaseButton>
    </div>

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

    <div class="trips-layout">
      <BaseTable
        :columns="columns"
        empty-text="Nenhuma viagem encontrada."
        :loading="loading"
        :rows="trips"
      >
        <template #row="{ row }">
          <td>{{ tripFrom(row).vehicle?.plate ?? '-' }}</td>
          <td>{{ tripFrom(row).location?.name ?? '-' }}</td>
          <td>{{ statusLabel(tripFrom(row).status) }}</td>
          <td>{{ formatDate(tripFrom(row).opened_at) }}</td>
          <td>{{ formatDate(tripFrom(row).closed_at) }}</td>
          <td>{{ loadLabel(tripFrom(row).current_load_status) }}</td>
          <td>
            <BaseButton
              data-test="select-trip"
              type="button"
              variant="secondary"
              @click="selectTrip(tripFrom(row))"
            >
              Revisar
            </BaseButton>
          </td>
        </template>
      </BaseTable>

      <aside class="trip-detail">
        <p
          v-if="detailLoading"
          class="muted"
        >
          Carregando detalhes...
        </p>
        <p
          v-else-if="!selectedTrip"
          class="muted"
        >
          Selecione uma viagem para revisar imagens e carga.
        </p>
        <template v-else>
          <h2>{{ selectedTrip.vehicle?.plate ?? 'Sem placa' }}</h2>
          <p class="muted">
            {{ selectedTrip.location?.name ?? '-' }} · {{ statusLabel(selectedTrip.status) }}
          </p>
          <p
            v-if="selectedTrip.review_required_reason"
            class="review-reason"
          >
            {{ selectedTrip.review_required_reason }}
          </p>

          <article
            v-for="event in selectedTrip.events ?? []"
            :key="event.id"
            class="trip-event"
          >
            <header>
              <strong>{{ directionLabel(event.direction) }}</strong>
              <span>{{ formatDate(event.occurred_at) }}</span>
            </header>
            <p>{{ event.capture.camera_pair?.name ?? '-' }} · {{ loadLabel(event.load_status) }}</p>

            <div class="media-grid">
              <figure>
                <figcaption>LPR</figcaption>
                <img
                  v-if="mediaUrl(event, 'lpr_image')"
                  alt="Imagem LPR da viagem"
                  :src="mediaUrl(event, 'lpr_image')"
                >
                <span v-else>Sem imagem LPR</span>
              </figure>
              <figure>
                <figcaption>Apoio</figcaption>
                <img
                  v-if="mediaUrl(event, 'support_image')"
                  alt="Imagem de apoio da viagem"
                  :src="mediaUrl(event, 'support_image')"
                >
                <span v-else>Sem imagem de apoio</span>
              </figure>
            </div>

            <div
              v-if="canManageTrips"
              class="row-actions"
            >
              <BaseButton
                type="button"
                :disabled="savingEventId === event.id"
                @click="updateLoadStatus(event, 'loaded')"
              >
                Carregado
              </BaseButton>
              <BaseButton
                type="button"
                variant="secondary"
                :disabled="savingEventId === event.id"
                @click="updateLoadStatus(event, 'empty')"
              >
                Vazio
              </BaseButton>
              <BaseButton
                type="button"
                variant="secondary"
                :disabled="savingEventId === event.id"
                @click="updateLoadStatus(event, 'needs_review')"
              >
                Precisa revisao
              </BaseButton>
            </div>
          </article>
        </template>
      </aside>
    </div>
  </section>
</template>
```

- [ ] **Step 5: Add scoped page styles**

Add:

```vue
<style scoped>
.filters-row {
  align-items: end;
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(4, minmax(140px, 1fr));
  margin-bottom: 16px;
}

.trips-layout {
  align-items: start;
  display: grid;
  gap: 20px;
  grid-template-columns: minmax(0, 1.3fr) minmax(320px, 0.7fr);
}

.trip-detail {
  border: 1px solid var(--color-border);
  border-radius: 8px;
  padding: 16px;
}

.muted {
  color: var(--color-muted);
}

.review-reason {
  color: var(--color-danger);
  font-weight: 600;
}

.trip-event {
  border-top: 1px solid var(--color-border);
  padding: 16px 0;
}

.trip-event header {
  display: flex;
  justify-content: space-between;
  gap: 12px;
}

.media-grid {
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.media-grid figure {
  margin: 0;
}

.media-grid img {
  aspect-ratio: 4 / 3;
  border-radius: 6px;
  display: block;
  object-fit: cover;
  width: 100%;
}

@media (max-width: 980px) {
  .filters-row,
  .trips-layout {
    grid-template-columns: 1fr;
  }
}
</style>
```

The variables `--color-border`, `--color-muted`, and `--color-danger` already exist in `src/styles/main.css`; use those variables exactly.

- [ ] **Step 6: Run page test and verify GREEN**

Run:

```bash
npm test -- --run src/pages/TripsPage.test.ts
```

Expected: PASS.

- [ ] **Step 7: Run frontend full tests and build**

Run:

```bash
npm test -- --run
npm run build
```

Expected: PASS.

- [ ] **Step 8: Commit frontend trip review page**

Run:

```bash
git add src/pages/TripsPage.vue src/pages/TripsPage.test.ts
git commit -m "feat: add trip load review page"
```

---

## Task 6: Documentation, Verification, And Final Polish

**Files:**

- Modify: `RIALMA-TrackVision-Backend/docs/api-parent-admin.md`
- Modify: `RIALMA-TrackVision-Backend/README.md`
- Modify: `RIALMA-TrackVision-Frontend/README.md`
- Modify tests only if final verification exposes a real gap in this phase's behavior.

**Interfaces:**

- Consumes: all backend trip endpoints and frontend `/trips` route from Tasks 1-5.
- Produces: documented admin API contracts for trips, trip events, load review, and private media.
- Produces: frontend README note for the Trips page and required permissions.

- [ ] **Step 1: Update backend API documentation**

Append to `RIALMA-TrackVision-Backend/docs/api-parent-admin.md`:

```markdown
## Trips

Endpoints:

- `GET /trips`
- `GET /trips/{trip}`
- `PATCH /trip-events/{tripEvent}/load-status`
- `GET /media-assets/{mediaAsset}/content`

Authorization:

- `captures.view` can list trips, show trip details, and fetch private media content.
- `trips.manage` can update a trip event `load_status`.

Trip filters:

- `status`: `open`, `closed`, `needs_review`
- `vehicle_id`
- `location_id`
- `plate`
- `load_status`: `unknown`, `loaded`, `empty`, `needs_review`
- `date_from`
- `date_to`

Trip detail includes ordered events, capture plate data, camera pair data, and media references. Media references expose `content_endpoint`; they never expose local storage paths.

Load review payload:

```json
{
  "load_status": "loaded"
}
```

Accepted load values are `unknown`, `loaded`, `empty`, and `needs_review`. Updating a trip event also updates the linked `capture_events.load_status`.

Private media content must be fetched with the same Bearer token used for the admin API. The frontend must not put tokens in query strings.
```

- [ ] **Step 2: Update backend README**

Add a short operational note:

```markdown
### Trip Classification And Load Review

Accepted parent captures are classified into trips by vehicle, location, and camera-pair direction. Outbound opens a trip, inbound closes the latest open trip for the same vehicle/location, and ambiguous cases become `needs_review`.

Operators need `captures.view` to see trips/media and `trips.manage` to mark load status from support-camera images.
```

- [ ] **Step 3: Update frontend README**

Add:

```markdown
### Trips Page

The `/trips` route is the operational load-review screen. It requires `captures.view` to open and shows the `loaded`, `empty`, and `needs_review` actions only when the effective permissions include `trips.manage`.

Private media is fetched through the API with the Bearer token and rendered as temporary Object URLs. Tokens are never appended to media URLs.
```

- [ ] **Step 4: Run backend final verification**

Run in backend worktree:

```bash
composer validate
php artisan test
php artisan route:list --path=api/v1/admin/trips
php artisan route:list --path=api/v1/admin/trip-events
php artisan route:list --path=api/v1/admin/media-assets
git diff --check
```

Expected:

- Composer valid.
- Full test suite passes.
- Trip, trip-event, and media routes are registered.
- `git diff --check` has no output.

- [ ] **Step 5: Run frontend final verification**

Run in frontend worktree:

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
git commit -m "docs: add trip review api guide"
```

Frontend:

```bash
git add README.md
git commit -m "docs: add trip review frontend guide"
```

- [ ] **Step 7: Final review handoff**

Prepare final review package for both repos:

```bash
git log --oneline main..HEAD
git diff --stat main..HEAD
```

Reviewer should verify:

- Backend classification is idempotent.
- Ambiguous trips remain visible as `needs_review`.
- Load status updates both `trip_events` and `capture_events`.
- Private media endpoints do not expose storage paths and require `captures.view`.
- Frontend does not put tokens in URLs.
- Object URLs are revoked.
- Controllers remain thin, Form Requests own validation, Actions own business logic.

---

## Acceptance Checklist

- [ ] Capturas aceitas no parent sao agrupadas em viagens.
- [ ] Viagens outbound/inbound sao classificadas de forma deterministica.
- [ ] Casos ambiguos ficam visiveis como `needs_review`.
- [ ] Operador consegue revisar carga pela imagem de apoio.
- [ ] Imagem LPR e imagem de apoio continuam privadas.
- [ ] Backend protege visualizacao e alteracao por permissao.
- [ ] Frontend oferece tela operacional usavel com estados de loading, erro, vazio e sucesso.
- [ ] Testes cobrem classificacao, permissao, idempotencia, media privada e revisao de carga.
- [ ] Backend e frontend foram verificados em suas respectivas worktrees antes de merge/push.
