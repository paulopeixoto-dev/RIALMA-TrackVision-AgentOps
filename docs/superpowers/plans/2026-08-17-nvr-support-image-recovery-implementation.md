# NVR Support Image Recovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable the edge backend to recover missing support-camera JPEG images from a local Intelbras NVR by capture time and attach them to existing LPR capture events.

**Architecture:** The LPR camera remains the only source of plate events. The edge backend stores NVR devices and camera-to-channel mappings, schedules recovery attempts when `support_image` is missing, and runs an idempotent command that fetches one still frame from the NVR and requeues the capture outbox message. The parent backend receives recovered media only through the existing edge sync; it never reaches into the local NVR.

**Tech Stack:** Laravel, Passport, Spatie Permission, Eloquent, MySQL, Laravel HTTP client, Vue 3, Vite, Pinia, Vuestic UI, Vitest, Vue Test Utils, Playwright.

## Global Constraints

- AgentOps root: `C:\projetos\rialma\RIALMA-TrackVision-AgentOps`.
- Backend repository: `RIALMA-TrackVision-Backend`.
- Frontend repository: `RIALMA-TrackVision-Frontend`.
- Keep Laravel Controllers thin; validation and authorization stay in Form Requests.
- Keep business rules in Actions or Services; Controllers only coordinate HTTP.
- Use Eloquent relationships, eager loading, Resources, and pagination for admin APIs.
- Application code reads environment values through `config()`, never through direct `env()` calls.
- Store NVR credentials with Laravel encrypted casts and never expose passwords in Resources, logs, bootstrap payloads, or frontend bundles.
- Use Vuestic Admin/Vuestic UI components first for screens, forms, alerts, modals, tables, and actions.
- The LPR event flow must keep working when the NVR is down, not configured, or outside retention.
- Recovery runs on the edge node inside the local network; parent nodes do not access cameras or NVRs.
- No scraping of the NVR web interface is allowed.
- Use TDD for behavior changes: write the failing test, run and confirm failure, implement, run and confirm pass.
- Commit after each task in the repository that was changed.

---

## File Structure

### Backend Files

- Create `RIALMA-TrackVision-Backend/database/migrations/2026_08_17_200001_create_recording_devices_table.php`: NVR/DVR inventory.
- Create `RIALMA-TrackVision-Backend/database/migrations/2026_08_17_200002_create_camera_recording_sources_table.php`: support camera to NVR channel mapping.
- Create `RIALMA-TrackVision-Backend/database/migrations/2026_08_17_200003_create_capture_media_recovery_attempts_table.php`: idempotent media recovery state.
- Create `RIALMA-TrackVision-Backend/app/Enums/RecordingDeviceVendor.php`: `intelbras`.
- Create `RIALMA-TrackVision-Backend/app/Enums/RecordingDeviceProtocol.php`: `http`, `https`.
- Create `RIALMA-TrackVision-Backend/app/Enums/RecordingDeviceAuthType.php`: `digest`, `basic`, `none`.
- Create `RIALMA-TrackVision-Backend/app/Enums/RecordingStream.php`: `main`, `sub`.
- Create `RIALMA-TrackVision-Backend/app/Enums/MediaRecoveryStatus.php`: `pending_configuration`, `pending`, `running`, `recovered`, `not_found`, `failed`.
- Create `RIALMA-TrackVision-Backend/app/Models/RecordingDevice.php`: NVR model with encrypted password and relationships.
- Create `RIALMA-TrackVision-Backend/app/Models/CameraRecordingSource.php`: channel mapping model.
- Create `RIALMA-TrackVision-Backend/app/Models/CaptureMediaRecoveryAttempt.php`: attempt model.
- Modify `RIALMA-TrackVision-Backend/app/Models/Camera.php`: add `recordingSource()` relationship.
- Modify `RIALMA-TrackVision-Backend/app/Models/CaptureEvent.php`: add `mediaRecoveryAttempts()` and `latestSupportImageRecoveryAttempt()` relationships.
- Create factories for `RecordingDevice`, `CameraRecordingSource`, and `CaptureMediaRecoveryAttempt`.
- Create `RIALMA-TrackVision-Backend/app/Actions/Admin/RecordingDevices/CreateRecordingDeviceAction.php`.
- Create `RIALMA-TrackVision-Backend/app/Actions/Admin/RecordingDevices/UpdateRecordingDeviceAction.php`.
- Create `RIALMA-TrackVision-Backend/app/Actions/Admin/CameraRecordingSources/UpsertCameraRecordingSourceAction.php`.
- Create `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/StoreRecordingDeviceRequest.php`.
- Create `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/UpdateRecordingDeviceRequest.php`.
- Create `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/StoreCameraRecordingSourceRequest.php`.
- Create `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/UpdateCameraRecordingSourceRequest.php`.
- Create `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/RecordingDeviceController.php`.
- Create `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/CameraRecordingSourceController.php`.
- Create `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/RecordingDeviceResource.php`.
- Create `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/CameraRecordingSourceResource.php`.
- Create `RIALMA-TrackVision-Backend/app/Data/Recordings/RecordingFrameData.php`.
- Create `RIALMA-TrackVision-Backend/app/Data/Recordings/RecordingFrameRequestData.php`.
- Create `RIALMA-TrackVision-Backend/app/Services/Recordings/RecordingPlaybackClient.php`.
- Create `RIALMA-TrackVision-Backend/app/Services/Recordings/IntelbrasNvrPlaybackClient.php`.
- Create `RIALMA-TrackVision-Backend/app/Services/Recordings/SupportImageRecoverySelector.php`.
- Create `RIALMA-TrackVision-Backend/app/Support/Recordings/BuildRecordingFrameUrl.php`.
- Create `RIALMA-TrackVision-Backend/app/Support/Recordings/SanitizeRecordingFailureMessage.php`.
- Create `RIALMA-TrackVision-Backend/app/Actions/Edge/ScheduleSupportImageRecoveryAction.php`.
- Create `RIALMA-TrackVision-Backend/app/Actions/Edge/RecoverSupportImageFromNvrAction.php`.
- Create `RIALMA-TrackVision-Backend/app/Actions/Edge/RecoverMissingSupportImagesAction.php`.
- Create `RIALMA-TrackVision-Backend/app/Actions/Edge/StoreRecoveredSupportImageAction.php`.
- Create `RIALMA-TrackVision-Backend/app/Console/Commands/EdgeRecoverSupportImagesCommand.php`.
- Create `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/CaptureSupportImageRecoveryController.php`.
- Modify `RIALMA-TrackVision-Backend/app/Actions/Edge/ProcessLprEventAction.php`: schedule recovery after missing support image.
- Modify `RIALMA-TrackVision-Backend/app/Actions/Edge/EnqueueCaptureForSyncAction.php`: allow requeueing an existing outbox message.
- Modify `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/TripEventResource.php`: expose recovery state.
- Modify `RIALMA-TrackVision-Backend/routes/api.php`: add admin NVR, mapping, and manual recovery routes.
- Modify `RIALMA-TrackVision-Backend/config/trackvision.php`: add NVR defaults and Intelbras frame endpoint template.
- Modify `RIALMA-TrackVision-Backend/.env.example`: document NVR config keys.

### Backend Tests

- Create `RIALMA-TrackVision-Backend/tests/Feature/Admin/RecordingDeviceAdminTest.php`.
- Create `RIALMA-TrackVision-Backend/tests/Feature/Admin/CameraRecordingSourceAdminTest.php`.
- Create `RIALMA-TrackVision-Backend/tests/Feature/Admin/CaptureSupportImageRecoveryTest.php`.
- Create `RIALMA-TrackVision-Backend/tests/Feature/Edge/SupportImageRecoverySchedulingTest.php`.
- Create `RIALMA-TrackVision-Backend/tests/Feature/Edge/RecoverSupportImagesCommandTest.php`.
- Create `RIALMA-TrackVision-Backend/tests/Unit/Recordings/IntelbrasNvrPlaybackClientTest.php`.
- Create `RIALMA-TrackVision-Backend/tests/Unit/Recordings/SupportImageRecoverySelectorTest.php`.
- Modify existing edge processing and trip resource tests when their assertions need the new recovery field.

### Frontend Files

- Modify `RIALMA-TrackVision-Frontend/src/types/admin.ts`: add NVR, source mapping, and recovery-status types.
- Create `RIALMA-TrackVision-Frontend/src/services/recordingDevicesService.ts`.
- Create `RIALMA-TrackVision-Frontend/src/services/cameraRecordingSourcesService.ts`.
- Modify `RIALMA-TrackVision-Frontend/src/services/tripsService.ts`: add manual support image recovery call.
- Create `RIALMA-TrackVision-Frontend/src/components/forms/RecordingDeviceForm.vue`.
- Create `RIALMA-TrackVision-Frontend/src/components/forms/CameraRecordingSourceForm.vue`.
- Create `RIALMA-TrackVision-Frontend/src/pages/RecordingDevicesPage.vue`.
- Modify `RIALMA-TrackVision-Frontend/src/pages/TripsPage.vue`: show recovery state and action.
- Modify `RIALMA-TrackVision-Frontend/src/router/index.ts`: add `recording-devices` route with `cameras.manage`.
- Modify `RIALMA-TrackVision-Frontend/src/components/navigation/TheSidebar.vue`: add `Gravadores/NVRs` navigation item.
- Create `RIALMA-TrackVision-Frontend/src/pages/RecordingDevicesPage.test.ts`.
- Modify `RIALMA-TrackVision-Frontend/src/pages/TripsPage.test.ts`.
- Modify `RIALMA-TrackVision-Frontend/src/components/navigation/TheSidebar.test.ts`.

### Documentation Files

- Create `docs/operations/nvr-support-image-recovery.md`: field checklist and required routines.
- Modify `docs/operations/intelbras-vip-5460-field-checklist.md`: add NVR/NTP/channel validation.
- Modify `docs/project-context.md`: add operational summary for NVR recovery.

---

### Task 1: Backend Recording Schema And Domain Models

**Files:**
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_17_200001_create_recording_devices_table.php`
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_17_200002_create_camera_recording_sources_table.php`
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_17_200003_create_capture_media_recovery_attempts_table.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/RecordingDeviceVendor.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/RecordingDeviceProtocol.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/RecordingDeviceAuthType.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/RecordingStream.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/MediaRecoveryStatus.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/RecordingDevice.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/CameraRecordingSource.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/CaptureMediaRecoveryAttempt.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/RecordingDeviceFactory.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/CameraRecordingSourceFactory.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/CaptureMediaRecoveryAttemptFactory.php`
- Modify: `RIALMA-TrackVision-Backend/app/Models/Camera.php`
- Modify: `RIALMA-TrackVision-Backend/app/Models/CaptureEvent.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Edge/SupportImageRecoverySchedulingTest.php`

**Interfaces:**
- Consumes: existing `Camera`, `CaptureEvent`, `EdgeNode`, `Location`, `MediaAssetKind`.
- Produces: `Camera::recordingSource(): HasOne`, `CaptureEvent::mediaRecoveryAttempts(): HasMany`, `CaptureEvent::latestSupportImageRecoveryAttempt(): HasOne`, and three persisted domain models.

- [ ] **Step 1: Write the failing schema/model test**

```php
<?php

namespace Tests\Feature\Edge;

use App\Enums\CameraType;
use App\Enums\MediaAssetKind;
use App\Enums\MediaRecoveryStatus;
use App\Enums\RecordingDeviceAuthType;
use App\Enums\RecordingDeviceProtocol;
use App\Enums\RecordingDeviceVendor;
use App\Enums\RecordingStream;
use App\Models\Camera;
use App\Models\CameraRecordingSource;
use App\Models\CaptureEvent;
use App\Models\CaptureMediaRecoveryAttempt;
use App\Models\RecordingDevice;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class SupportImageRecoverySchedulingTest extends TestCase
{
    use RefreshDatabase;

    public function test_recording_models_cast_values_and_keep_nvr_password_private(): void
    {
        $supportCamera = Camera::factory()->create(['type' => CameraType::Support]);
        $recordingDevice = RecordingDevice::factory()->create([
            'location_id' => $supportCamera->location_id,
            'edge_node_id' => $supportCamera->edge_node_id,
            'vendor' => RecordingDeviceVendor::Intelbras,
            'protocol' => RecordingDeviceProtocol::Http,
            'auth_type' => RecordingDeviceAuthType::Digest,
            'password_encrypted' => 'secret-pass',
        ]);

        $source = CameraRecordingSource::factory()->create([
            'camera_id' => $supportCamera->id,
            'recording_device_id' => $recordingDevice->id,
            'channel' => 4,
            'stream' => RecordingStream::Main,
            'target_offset_seconds' => 2,
            'search_window_seconds' => 5,
        ]);

        $capture = CaptureEvent::factory()->create(['support_camera_id' => $supportCamera->id]);
        $attempt = CaptureMediaRecoveryAttempt::factory()->create([
            'capture_event_id' => $capture->id,
            'media_kind' => MediaAssetKind::SupportImage,
            'recording_device_id' => $recordingDevice->id,
            'camera_id' => $supportCamera->id,
            'status' => MediaRecoveryStatus::Pending,
        ]);

        $this->assertSame('secret-pass', $recordingDevice->refresh()->password_encrypted);
        $this->assertNotSame('secret-pass', $recordingDevice->getRawOriginal('password_encrypted'));
        $this->assertSame($source->id, $supportCamera->recordingSource->id);
        $this->assertSame($attempt->id, $capture->latestSupportImageRecoveryAttempt->id);
    }
}
```

- [ ] **Step 2: Run the schema/model test to verify it fails**

Run:

```powershell
php artisan test --filter=SupportImageRecoverySchedulingTest
```

Expected: FAIL because the new models, enums, factories, and relationships do not exist.

- [ ] **Step 3: Add migrations**

Use these table contracts:

```php
Schema::create('recording_devices', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->foreignId('edge_node_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
    $table->foreignId('location_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
    $table->string('name');
    $table->string('vendor', 30)->default('intelbras');
    $table->string('protocol', 10)->default('http');
    $table->string('host');
    $table->unsignedInteger('port')->default(80);
    $table->string('username')->nullable();
    $table->text('password_encrypted')->nullable();
    $table->string('auth_type', 20)->default('digest');
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();

    $table->index(['location_id', 'edge_node_id', 'vendor']);
});
```

```php
Schema::create('camera_recording_sources', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->foreignId('camera_id')->constrained('cameras')->cascadeOnUpdate()->cascadeOnDelete();
    $table->foreignId('recording_device_id')->constrained()->cascadeOnUpdate()->cascadeOnDelete();
    $table->unsignedInteger('channel');
    $table->string('stream', 20)->default('main');
    $table->integer('target_offset_seconds')->default(2);
    $table->unsignedInteger('search_window_seconds')->default(5);
    $table->boolean('is_active')->default(true);
    $table->timestamps();

    $table->unique('camera_id');
    $table->unique(['recording_device_id', 'channel']);
});
```

```php
Schema::create('capture_media_recovery_attempts', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->foreignId('capture_event_id')->constrained()->cascadeOnUpdate()->cascadeOnDelete();
    $table->string('media_kind', 40);
    $table->foreignId('recording_device_id')->nullable()->constrained()->cascadeOnUpdate()->nullOnDelete();
    $table->foreignId('camera_id')->nullable()->constrained('cameras')->cascadeOnUpdate()->nullOnDelete();
    $table->timestamp('target_time')->nullable()->index();
    $table->string('status', 40)->default('pending');
    $table->unsignedInteger('attempts')->default(0);
    $table->text('last_error')->nullable();
    $table->timestamp('next_attempt_at')->nullable()->index();
    $table->timestamps();

    $table->unique(['capture_event_id', 'media_kind']);
    $table->index(['status', 'next_attempt_at']);
});
```

- [ ] **Step 4: Add enum cases with string values**

```php
enum RecordingDeviceVendor: string
{
    case Intelbras = 'intelbras';
}

enum RecordingDeviceProtocol: string
{
    case Http = 'http';
    case Https = 'https';
}

enum RecordingDeviceAuthType: string
{
    case Digest = 'digest';
    case Basic = 'basic';
    case None = 'none';
}

enum RecordingStream: string
{
    case Main = 'main';
    case Sub = 'sub';
}

enum MediaRecoveryStatus: string
{
    case PendingConfiguration = 'pending_configuration';
    case Pending = 'pending';
    case Running = 'running';
    case Recovered = 'recovered';
    case NotFound = 'not_found';
    case Failed = 'failed';
}
```

- [ ] **Step 5: Add models, casts, fillable fields, relationships, and factories**

`RecordingDevice` must cast `password_encrypted` as `encrypted`, `vendor`, `protocol`, `auth_type`, and `is_active`.

`CameraRecordingSource` must cast `stream`, `target_offset_seconds`, `search_window_seconds`, and `is_active`.

`CaptureMediaRecoveryAttempt` must cast `media_kind`, `status`, `attempts`, `target_time`, and `next_attempt_at`.

- [ ] **Step 6: Run the schema/model test to verify it passes**

Run:

```powershell
php artisan test --filter=SupportImageRecoverySchedulingTest
```

Expected: PASS.

- [ ] **Step 7: Commit backend schema**

```powershell
git add database/migrations app/Enums app/Models database/factories tests/Feature/Edge/SupportImageRecoverySchedulingTest.php
git commit -m "feat: add nvr recording schema"
```

---

### Task 2: Backend Admin API For NVR Devices And Camera Mappings

**Files:**
- Create: `RIALMA-TrackVision-Backend/app/Actions/Admin/RecordingDevices/CreateRecordingDeviceAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Admin/RecordingDevices/UpdateRecordingDeviceAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Admin/CameraRecordingSources/UpsertCameraRecordingSourceAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/StoreRecordingDeviceRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/UpdateRecordingDeviceRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/StoreCameraRecordingSourceRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/UpdateCameraRecordingSourceRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/RecordingDeviceController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/CameraRecordingSourceController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/RecordingDeviceResource.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/CameraRecordingSourceResource.php`
- Modify: `RIALMA-TrackVision-Backend/routes/api.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Admin/RecordingDeviceAdminTest.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Admin/CameraRecordingSourceAdminTest.php`

**Interfaces:**
- Consumes: schema and models from Task 1.
- Produces: `/api/v1/admin/recording-devices` CRUD and `/api/v1/admin/camera-recording-sources` CRUD protected by `permission:cameras.manage,api`.

- [ ] **Step 1: Write failing NVR admin API tests**

```php
public function test_user_with_camera_permission_can_create_recording_device_without_exposing_password(): void
{
    $user = User::factory()->create();
    $user->givePermissionTo('cameras.manage');
    $location = Location::factory()->create();
    $edgeNode = EdgeNode::factory()->create(['location_id' => $location->id]);

    $response = $this->actingAs($user, 'api')->postJson('/api/v1/admin/recording-devices', [
        'location_id' => $location->id,
        'edge_node_id' => $edgeNode->id,
        'name' => 'NVR Portaria 01',
        'vendor' => 'intelbras',
        'protocol' => 'http',
        'host' => '10.0.8.150',
        'port' => 80,
        'username' => 'trackvision_ro',
        'password' => 'readonly-secret',
        'auth_type' => 'digest',
        'is_active' => true,
    ]);

    $response->assertCreated()
        ->assertJsonPath('data.name', 'NVR Portaria 01')
        ->assertJsonPath('data.has_password', true)
        ->assertJsonMissing(['password' => 'readonly-secret'])
        ->assertJsonMissing(['password_encrypted' => 'readonly-secret']);

    $device = RecordingDevice::query()->firstOrFail();
    $this->assertSame('readonly-secret', $device->password_encrypted);
    $this->assertNotSame('readonly-secret', $device->getRawOriginal('password_encrypted'));
}
```

```php
public function test_recording_source_requires_support_camera_on_same_edge_and_location(): void
{
    $user = User::factory()->create();
    $user->givePermissionTo('cameras.manage');
    $supportCamera = Camera::factory()->create(['type' => CameraType::Support]);
    $lprCamera = Camera::factory()->create([
        'location_id' => $supportCamera->location_id,
        'edge_node_id' => $supportCamera->edge_node_id,
        'type' => CameraType::Lpr,
    ]);
    $recordingDevice = RecordingDevice::factory()->create([
        'location_id' => $supportCamera->location_id,
        'edge_node_id' => $supportCamera->edge_node_id,
    ]);

    $this->actingAs($user, 'api')->postJson('/api/v1/admin/camera-recording-sources', [
        'camera_id' => $lprCamera->id,
        'recording_device_id' => $recordingDevice->id,
        'channel' => 3,
        'stream' => 'main',
        'target_offset_seconds' => 2,
        'search_window_seconds' => 5,
        'is_active' => true,
    ])->assertUnprocessable();

    $this->actingAs($user, 'api')->postJson('/api/v1/admin/camera-recording-sources', [
        'camera_id' => $supportCamera->id,
        'recording_device_id' => $recordingDevice->id,
        'channel' => 3,
        'stream' => 'main',
        'target_offset_seconds' => 2,
        'search_window_seconds' => 5,
        'is_active' => true,
    ])->assertCreated()
      ->assertJsonPath('data.channel', 3)
      ->assertJsonPath('data.camera.id', $supportCamera->id)
      ->assertJsonPath('data.recording_device.id', $recordingDevice->id);
}
```

- [ ] **Step 2: Run admin API tests to verify they fail**

Run:

```powershell
php artisan test --filter=RecordingDeviceAdminTest
php artisan test --filter=CameraRecordingSourceAdminTest
```

Expected: FAIL because routes, requests, actions, controllers, and resources do not exist.

- [ ] **Step 3: Implement Form Requests**

`StoreRecordingDeviceRequest::rules()`:

```php
return [
    'location_id' => ['required', 'integer', Rule::exists('locations', 'id')],
    'edge_node_id' => ['required', 'integer', Rule::exists('edge_nodes', 'id')],
    'name' => ['required', 'string', 'max:255'],
    'vendor' => ['required', Rule::enum(RecordingDeviceVendor::class)],
    'protocol' => ['required', Rule::enum(RecordingDeviceProtocol::class)],
    'host' => ['required', 'string', 'max:255'],
    'port' => ['required', 'integer', 'between:1,65535'],
    'username' => ['nullable', 'string', 'max:255'],
    'password' => ['nullable', 'string', 'max:255'],
    'auth_type' => ['required', Rule::enum(RecordingDeviceAuthType::class)],
    'is_active' => ['required', 'boolean'],
];
```

`UpdateRecordingDeviceRequest` uses `sometimes` for fields and keeps `password` nullable; an empty password does not overwrite the stored credential.

`StoreCameraRecordingSourceRequest::rules()`:

```php
return [
    'camera_id' => ['required', 'integer', Rule::exists('cameras', 'id')],
    'recording_device_id' => ['required', 'integer', Rule::exists('recording_devices', 'id')],
    'channel' => ['required', 'integer', 'between:0,255'],
    'stream' => ['required', Rule::enum(RecordingStream::class)],
    'target_offset_seconds' => ['required', 'integer', 'between:-60,60'],
    'search_window_seconds' => ['required', 'integer', 'between:1,120'],
    'is_active' => ['required', 'boolean'],
];
```

- [ ] **Step 4: Implement Actions**

`CreateRecordingDeviceAction::execute(array $data): RecordingDevice` stores `password` into `password_encrypted` and unsets `password`.

`UpdateRecordingDeviceAction::execute(RecordingDevice $recordingDevice, array $data): RecordingDevice` updates password only when `array_key_exists('password', $data)` and `filled($data['password'])`.

`UpsertCameraRecordingSourceAction::execute(array $data): CameraRecordingSource` must load `Camera` and `RecordingDevice`, reject non-support cameras with `ValidationException::withMessages(['camera_id' => 'A camera vinculada ao NVR deve ser do tipo apoio.'])`, reject location or edge mismatch, and update or create the single source row for `camera_id`.

- [ ] **Step 5: Implement thin Controllers and Resources**

`RecordingDeviceResource` returns:

```php
[
    'id' => $this->id,
    'uuid' => $this->uuid,
    'name' => $this->name,
    'vendor' => $this->vendor?->value,
    'protocol' => $this->protocol?->value,
    'host' => $this->host,
    'port' => $this->port,
    'username' => $this->username,
    'auth_type' => $this->auth_type?->value,
    'has_password' => filled($this->password_encrypted),
    'is_active' => $this->is_active,
    'location' => new LocationResource($this->whenLoaded('location')),
    'edge_node' => new EdgeNodeResource($this->whenLoaded('edgeNode')),
]
```

`CameraRecordingSourceResource` returns `camera`, `recording_device`, `channel`, `stream`, `target_offset_seconds`, `search_window_seconds`, and `is_active`.

- [ ] **Step 6: Register routes**

In `routes/api.php` admin group:

```php
Route::apiResource('recording-devices', RecordingDeviceController::class)
    ->middleware('permission:cameras.manage,api');

Route::apiResource('camera-recording-sources', CameraRecordingSourceController::class)
    ->middleware('permission:cameras.manage,api');
```

- [ ] **Step 7: Run admin API tests to verify they pass**

Run:

```powershell
php artisan test --filter=RecordingDeviceAdminTest
php artisan test --filter=CameraRecordingSourceAdminTest
```

Expected: PASS.

- [ ] **Step 8: Commit backend admin API**

```powershell
git add app/Actions/Admin app/Http/Controllers/Api/V1/Admin app/Http/Requests/Api/V1/Admin app/Http/Resources/Api/V1 routes/api.php tests/Feature/Admin
git commit -m "feat: add nvr admin api"
```

---

### Task 3: Backend NVR Playback Client And Safe Configuration

**Files:**
- Create: `RIALMA-TrackVision-Backend/app/Data/Recordings/RecordingFrameData.php`
- Create: `RIALMA-TrackVision-Backend/app/Data/Recordings/RecordingFrameRequestData.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/Recordings/RecordingPlaybackClient.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/Recordings/IntelbrasNvrPlaybackClient.php`
- Create: `RIALMA-TrackVision-Backend/app/Support/Recordings/BuildRecordingFrameUrl.php`
- Create: `RIALMA-TrackVision-Backend/app/Support/Recordings/SanitizeRecordingFailureMessage.php`
- Modify: `RIALMA-TrackVision-Backend/app/Providers/AppServiceProvider.php`
- Modify: `RIALMA-TrackVision-Backend/config/trackvision.php`
- Modify: `RIALMA-TrackVision-Backend/.env.example`
- Test: `RIALMA-TrackVision-Backend/tests/Unit/Recordings/IntelbrasNvrPlaybackClientTest.php`

**Interfaces:**
- Consumes: `RecordingDevice`, `CameraRecordingSource`, and NVR config.
- Produces: `RecordingPlaybackClient::captureFrame(RecordingFrameRequestData $request): ?RecordingFrameData`.

- [ ] **Step 1: Write failing client tests**

```php
public function test_intelbras_client_downloads_jpeg_using_configured_frame_template(): void
{
    config()->set('trackvision.nvr.intelbras_frame_endpoint_template', '/cgi-bin/recording-frame.cgi?channel={channel}&time={timestamp_local}&stream={stream}');

    Http::fake([
        '10.0.8.150:80/cgi-bin/recording-frame.cgi*' => Http::response("\xff\xd8\xff\xe0fakejpeg", 200, [
            'Content-Type' => 'image/jpeg',
        ]),
    ]);

    $device = RecordingDevice::factory()->make([
        'protocol' => RecordingDeviceProtocol::Http,
        'host' => '10.0.8.150',
        'port' => 80,
        'username' => 'trackvision_ro',
        'password_encrypted' => 'secret',
        'auth_type' => RecordingDeviceAuthType::Digest,
    ]);
    $source = CameraRecordingSource::factory()->make([
        'channel' => 4,
        'stream' => RecordingStream::Main,
        'search_window_seconds' => 5,
    ]);

    $frame = app(IntelbrasNvrPlaybackClient::class)->captureFrame(new RecordingFrameRequestData(
        recordingDevice: $device,
        recordingSource: $source,
        targetTime: CarbonImmutable::parse('2026-08-17 10:30:15', 'America/Sao_Paulo'),
    ));

    $this->assertNotNull($frame);
    $this->assertSame('image/jpeg', $frame->contentType);
    $this->assertSame(hash('sha256', "\xff\xd8\xff\xe0fakejpeg"), $frame->sha256Hash);
    Http::assertSent(fn ($request) => str_contains($request->url(), 'channel=4')
        && str_contains(urldecode($request->url()), '2026-08-17 10:30:15')
    );
}

public function test_intelbras_client_refuses_to_attach_when_template_is_not_configured(): void
{
    config()->set('trackvision.nvr.intelbras_frame_endpoint_template', null);

    $frame = app(IntelbrasNvrPlaybackClient::class)->captureFrame(new RecordingFrameRequestData(
        recordingDevice: RecordingDevice::factory()->make(),
        recordingSource: CameraRecordingSource::factory()->make(),
        targetTime: CarbonImmutable::parse('2026-08-17 10:30:15', 'America/Sao_Paulo'),
    ));

    $this->assertNull($frame);
}
```

- [ ] **Step 2: Run client tests to verify they fail**

Run:

```powershell
php artisan test --filter=IntelbrasNvrPlaybackClientTest
```

Expected: FAIL because DTOs, client, URL builder, and container binding do not exist.

- [ ] **Step 3: Add NVR config keys**

In `config/trackvision.php`:

```php
'nvr' => [
    'default_search_window_seconds' => (int) env('TRACKVISION_NVR_DEFAULT_SEARCH_WINDOW_SECONDS', 5),
    'default_target_offset_seconds' => (int) env('TRACKVISION_NVR_DEFAULT_TARGET_OFFSET_SECONDS', 2),
    'recovery_retry_minutes' => (int) env('TRACKVISION_NVR_RECOVERY_RETRY_MINUTES', 10),
    'max_attempts' => (int) env('TRACKVISION_NVR_RECOVERY_MAX_ATTEMPTS', 5),
    'timeout_seconds' => (int) env('TRACKVISION_NVR_TIMEOUT_SECONDS', 15),
    'intelbras_frame_endpoint_template' => env('TRACKVISION_NVR_INTELBRAS_FRAME_ENDPOINT_TEMPLATE'),
],
```

In `.env.example`:

```text
TRACKVISION_NVR_DEFAULT_SEARCH_WINDOW_SECONDS=5
TRACKVISION_NVR_DEFAULT_TARGET_OFFSET_SECONDS=2
TRACKVISION_NVR_RECOVERY_RETRY_MINUTES=10
TRACKVISION_NVR_RECOVERY_MAX_ATTEMPTS=5
TRACKVISION_NVR_TIMEOUT_SECONDS=15
TRACKVISION_NVR_INTELBRAS_FRAME_ENDPOINT_TEMPLATE=
```

The endpoint template supports placeholders `{channel}`, `{stream}`, `{timestamp_iso}`, `{timestamp_local}`, `{unix_timestamp}`, and `{search_window_seconds}`. The value stays empty until the NVR firmware endpoint is validated in the field.

- [ ] **Step 4: Implement DTOs and URL builder**

```php
final readonly class RecordingFrameRequestData
{
    public function __construct(
        public RecordingDevice $recordingDevice,
        public CameraRecordingSource $recordingSource,
        public CarbonImmutable $targetTime,
    ) {}
}

final readonly class RecordingFrameData
{
    public function __construct(
        public string $bytes,
        public string $contentType,
        public string $sha256Hash,
        public CarbonImmutable $capturedAt,
        public string $sourceRecordingDeviceUuid,
        public int $sourceChannel,
    ) {}
}
```

```php
public function execute(RecordingFrameRequestData $request, string $template): string
{
    $replacements = [
        '{channel}' => (string) $request->recordingSource->channel,
        '{stream}' => $request->recordingSource->stream->value,
        '{timestamp_iso}' => rawurlencode($request->targetTime->toIso8601String()),
        '{timestamp_local}' => rawurlencode($request->targetTime->format('Y-m-d H:i:s')),
        '{unix_timestamp}' => (string) $request->targetTime->getTimestamp(),
        '{search_window_seconds}' => (string) $request->recordingSource->search_window_seconds,
    ];

    $path = strtr($template, $replacements);
    $path = '/'.ltrim($path, '/');

    return "{$request->recordingDevice->protocol->value}://{$request->recordingDevice->host}:{$request->recordingDevice->port}{$path}";
}
```

- [ ] **Step 5: Implement `IntelbrasNvrPlaybackClient`**

Behavior:

- return `null` when `trackvision.nvr.intelbras_frame_endpoint_template` is blank;
- use `Http::timeout(config('trackvision.nvr.timeout_seconds'))`;
- apply auth from `RecordingDeviceAuthType`: digest, basic, or none;
- accept only JPEG bytes when `Content-Type` contains `image/jpeg`, `image/jpg`, or bytes begin with `\xff\xd8`;
- return `null` for HTTP 404 and 204;
- throw the original exception for timeout, 401, 403, and 5xx so the recovery action can mark `failed` with a sanitized message.

- [ ] **Step 6: Bind the interface**

In `AppServiceProvider::register()`:

```php
$this->app->bind(RecordingPlaybackClient::class, IntelbrasNvrPlaybackClient::class);
```

- [ ] **Step 7: Run client tests to verify they pass**

Run:

```powershell
php artisan test --filter=IntelbrasNvrPlaybackClientTest
```

Expected: PASS.

- [ ] **Step 8: Commit backend NVR client**

```powershell
git add app/Data/Recordings app/Services/Recordings app/Support/Recordings app/Providers/AppServiceProvider.php config/trackvision.php .env.example tests/Unit/Recordings/IntelbrasNvrPlaybackClientTest.php
git commit -m "feat: add nvr playback client"
```

---

### Task 4: Backend Recovery Scheduling, Command, And Outbox Requeue

**Files:**
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/ScheduleSupportImageRecoveryAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/RecoverSupportImageFromNvrAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/RecoverMissingSupportImagesAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/StoreRecoveredSupportImageAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Console/Commands/EdgeRecoverSupportImagesCommand.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/Recordings/SupportImageRecoverySelector.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Edge/ProcessLprEventAction.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Edge/EnqueueCaptureForSyncAction.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Edge/SupportImageRecoverySchedulingTest.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Edge/RecoverSupportImagesCommandTest.php`
- Test: `RIALMA-TrackVision-Backend/tests/Unit/Recordings/SupportImageRecoverySelectorTest.php`

**Interfaces:**
- Consumes: `RecordingPlaybackClient::captureFrame(RecordingFrameRequestData $request): ?RecordingFrameData`.
- Produces: `ScheduleSupportImageRecoveryAction::execute(CaptureEvent $captureEvent): ?CaptureMediaRecoveryAttempt`, `RecoverSupportImageFromNvrAction::execute(CaptureMediaRecoveryAttempt $attempt): CaptureMediaRecoveryAttempt`, and command `edge:recover-support-images {cameraPairUuid?} {--from=} {--to=} {--missing-only} {--limit=50}`.

- [ ] **Step 1: Write failing scheduling tests**

```php
public function test_process_lpr_event_schedules_recovery_when_support_snapshot_fails(): void
{
    config()->set('trackvision.edge.auto_register_unknown_vehicles', true);

    $pair = CameraPair::factory()->withSupportCamera()->create();
    RecordingDevice::factory()
        ->for($pair->supportCamera->location, 'location')
        ->for($pair->supportCamera->edgeNode, 'edgeNode')
        ->has(CameraRecordingSource::factory()->state([
            'camera_id' => $pair->support_camera_id,
            'channel' => 4,
            'target_offset_seconds' => 2,
            'search_window_seconds' => 5,
        ]), 'recordingSources')
        ->create();

    $this->mock(IntelbrasSnapshotClient::class, function ($mock): void {
        $mock->shouldReceive('capture')->andThrow(new RuntimeException('support camera offline'));
    });

    $event = TrafficCaptureData::fromIntelbrasPayload([
        'PlateNumber' => 'ABC1D23',
        'UTC' => '2026-08-17 10:30:15',
        'Action' => 'Pulse',
        'Code' => 'TrafficJunction',
    ], null);

    app(ProcessLprEventAction::class)->execute($pair, $event);

    $this->assertDatabaseHas('capture_media_recovery_attempts', [
        'media_kind' => 'support_image',
        'status' => 'pending',
    ]);
}

public function test_process_lpr_event_records_pending_configuration_when_support_camera_has_no_nvr_source(): void
{
    config()->set('trackvision.edge.auto_register_unknown_vehicles', true);
    $pair = CameraPair::factory()->withSupportCamera()->create();

    $this->mock(IntelbrasSnapshotClient::class, function ($mock): void {
        $mock->shouldReceive('capture')->andThrow(new RuntimeException('support camera offline'));
    });

    app(ProcessLprEventAction::class)->execute($pair, TrafficCaptureData::fakePlate('ABC1D23'));

    $this->assertDatabaseHas('capture_media_recovery_attempts', [
        'media_kind' => 'support_image',
        'status' => 'pending_configuration',
        'recording_device_id' => null,
    ]);
}
```

- [ ] **Step 2: Write failing recovery command tests**

```php
public function test_recovery_command_attaches_support_image_and_requeues_capture_outbox(): void
{
    $capture = CaptureEvent::factory()->accepted()->create();
    $source = CameraRecordingSource::factory()->create(['camera_id' => $capture->support_camera_id]);
    $attempt = CaptureMediaRecoveryAttempt::factory()->create([
        'capture_event_id' => $capture->id,
        'media_kind' => MediaAssetKind::SupportImage,
        'recording_device_id' => $source->recording_device_id,
        'camera_id' => $source->camera_id,
        'status' => MediaRecoveryStatus::Pending,
        'target_time' => $capture->event_time->addSeconds($source->target_offset_seconds),
    ]);
    $outbox = EdgeOutboxMessage::factory()->create([
        'idempotency_key' => $capture->idempotency_key,
        'status' => EdgeOutboxStatus::Synced,
        'synced_at' => now(),
    ]);

    $this->mock(RecordingPlaybackClient::class, function ($mock) use ($source): void {
        $mock->shouldReceive('captureFrame')->once()->andReturn(new RecordingFrameData(
            bytes: "\xff\xd8\xff\xe0fakejpeg",
            contentType: 'image/jpeg',
            sha256Hash: hash('sha256', "\xff\xd8\xff\xe0fakejpeg"),
            capturedAt: CarbonImmutable::parse('2026-08-17 10:30:17'),
            sourceRecordingDeviceUuid: $source->recordingDevice->uuid,
            sourceChannel: $source->channel,
        ));
    });

    $this->artisan('edge:recover-support-images', ['--limit' => 10])->assertExitCode(0);

    $this->assertDatabaseHas('media_assets', [
        'capture_event_id' => $capture->id,
        'kind' => 'support_image',
        'sha256_hash' => hash('sha256', "\xff\xd8\xff\xe0fakejpeg"),
    ]);
    $this->assertDatabaseHas('capture_media_recovery_attempts', [
        'id' => $attempt->id,
        'status' => 'recovered',
    ]);
    $this->assertDatabaseHas('edge_outbox_messages', [
        'id' => $outbox->id,
        'status' => 'pending',
        'synced_at' => null,
    ]);
}

public function test_recovery_command_is_idempotent_when_support_image_already_exists(): void
{
    $capture = CaptureEvent::factory()->accepted()->create();
    MediaAsset::factory()->supportImage()->create(['capture_event_id' => $capture->id]);
    CaptureMediaRecoveryAttempt::factory()->create([
        'capture_event_id' => $capture->id,
        'media_kind' => MediaAssetKind::SupportImage,
        'status' => MediaRecoveryStatus::Pending,
    ]);

    $this->mock(RecordingPlaybackClient::class, function ($mock): void {
        $mock->shouldNotReceive('captureFrame');
    });

    $this->artisan('edge:recover-support-images')->assertExitCode(0);

    $this->assertSame(1, MediaAsset::where('capture_event_id', $capture->id)->where('kind', MediaAssetKind::SupportImage)->count());
}
```

- [ ] **Step 3: Run recovery tests to verify they fail**

Run:

```powershell
php artisan test --filter=SupportImageRecoverySchedulingTest
php artisan test --filter=RecoverSupportImagesCommandTest
php artisan test --filter=SupportImageRecoverySelectorTest
```

Expected: FAIL because scheduling action, recovery action, selector, command, and outbox requeue behavior do not exist.

- [ ] **Step 4: Implement `ScheduleSupportImageRecoveryAction`**

Behavior:

- return `null` when `support_camera_id` is null;
- return `null` when a `support_image` media asset already exists;
- load `supportCamera.recordingSource.recordingDevice`;
- create or update one attempt per `capture_event_id + support_image`;
- when mapping exists, set `status = pending`, `recording_device_id`, `camera_id`, `target_time = event_time + target_offset_seconds`, `next_attempt_at = now()`;
- when mapping is missing, set `status = pending_configuration`, `last_error = 'support_camera_has_no_recording_source'`, and leave `next_attempt_at = null`.

- [ ] **Step 5: Call scheduling from `ProcessLprEventAction`**

Inject `ScheduleSupportImageRecoveryAction` in the constructor. After `storeImages()` and before returning the transaction result:

```php
if ($capture->support_camera_id !== null
    && ! $capture->mediaAssets()->where('kind', MediaAssetKind::SupportImage)->exists()) {
    $this->scheduleSupportImageRecoveryAction->execute($capture->refresh());
}
```

- [ ] **Step 6: Implement outbox requeue option**

Change signature:

```php
public function execute(CaptureEvent $captureEvent, bool $forcePending = false): EdgeOutboxMessage
```

If a row exists and `$forcePending` is true:

```php
$message->forceFill([
    'status' => EdgeOutboxStatus::Pending,
    'attempts' => 0,
    'available_at' => now(),
    'synced_at' => null,
    'last_error' => null,
])->save();
```

Keep the current `firstOrCreate` behavior unchanged when `$forcePending` is false.

- [ ] **Step 7: Implement `StoreRecoveredSupportImageAction`**

Behavior:

- exit when the capture already has `support_image`;
- create one `MediaAssetKind::SupportImage` media asset through `StoreCaptureMediaAction`;
- call `EnqueueCaptureForSyncAction::execute($captureEvent, forcePending: true)`;
- return the new or existing `MediaAsset`.

- [ ] **Step 8: Implement recovery selector and command**

`SupportImageRecoverySelector::query(?string $cameraPairUuid, ?CarbonImmutable $from, ?CarbonImmutable $to, bool $missingOnly): Builder` must:

- include statuses `pending` and `failed` when `next_attempt_at <= now()` or null;
- exclude `recovered` and `not_found`;
- filter by camera pair UUID when provided;
- filter by `capture_events.event_time` for `--from` and `--to`;
- when `missingOnly` is true, exclude captures that already have `support_image`.

`EdgeRecoverSupportImagesCommand` must parse:

```php
protected $signature = 'edge:recover-support-images {cameraPairUuid?} {--from=} {--to=} {--missing-only} {--limit=50}';
```

- [ ] **Step 9: Implement `RecoverSupportImageFromNvrAction`**

Behavior:

- mark attempt `running` before calling the client;
- skip and mark `recovered` when `support_image` already exists;
- call `RecordingPlaybackClient`;
- when frame is `null`, mark `not_found`, increment `attempts`, and set safe `last_error = 'frame_not_found_in_recording_window'`;
- when frame exists, store media, mark `recovered`, clear `last_error`, and clear `next_attempt_at`;
- on exception, increment `attempts`, set `failed`, set `next_attempt_at = now()->addMinutes(config('trackvision.nvr.recovery_retry_minutes'))` while attempts are below `max_attempts`, sanitize `last_error`, and never include credentials.

- [ ] **Step 10: Run recovery tests to verify they pass**

Run:

```powershell
php artisan test --filter=SupportImageRecoverySchedulingTest
php artisan test --filter=RecoverSupportImagesCommandTest
php artisan test --filter=SupportImageRecoverySelectorTest
```

Expected: PASS.

- [ ] **Step 11: Commit backend recovery flow**

```powershell
git add app/Actions/Edge app/Console/Commands app/Services/Recordings app/Actions/Edge/ProcessLprEventAction.php tests/Feature/Edge tests/Unit/Recordings
git commit -m "feat: recover support images from nvr"
```

---

### Task 5: Backend Trip API Recovery State And Manual Retry Endpoint

**Files:**
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/CaptureSupportImageRecoveryController.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/TripEventResource.php`
- Modify: `RIALMA-TrackVision-Backend/routes/api.php`
- Test: `RIALMA-TrackVision-Backend/tests/Feature/Admin/CaptureSupportImageRecoveryTest.php`
- Modify: trip API tests if they assert exact JSON shape.

**Interfaces:**
- Consumes: `ScheduleSupportImageRecoveryAction::execute(CaptureEvent $captureEvent)` and `RecoverSupportImageFromNvrAction::execute(CaptureMediaRecoveryAttempt $attempt)`.
- Produces: `POST /api/v1/admin/captures/{captureEvent}/support-image-recovery` protected by `permission:trips.manage,api`.

- [ ] **Step 1: Write failing trip resource and manual retry tests**

```php
public function test_trip_event_resource_exposes_latest_support_image_recovery_state(): void
{
    $user = User::factory()->create();
    $user->givePermissionTo('captures.view');
    $trip = Trip::factory()->hasEvents(1)->create();
    $event = $trip->events()->firstOrFail();
    CaptureMediaRecoveryAttempt::factory()->create([
        'capture_event_id' => $event->capture_event_id,
        'media_kind' => MediaAssetKind::SupportImage,
        'status' => MediaRecoveryStatus::Pending,
        'attempts' => 2,
        'last_error' => 'frame_not_found_in_recording_window',
    ]);

    $response = $this->actingAs($user, 'api')->getJson("/api/v1/admin/trips/{$trip->id}");

    $response->assertOk()
        ->assertJsonPath('data.events.0.support_image_recovery.status', 'pending')
        ->assertJsonPath('data.events.0.support_image_recovery.attempts', 2)
        ->assertJsonPath('data.events.0.support_image_recovery.last_error', 'frame_not_found_in_recording_window');
}

public function test_user_with_trip_permission_can_request_support_image_recovery(): void
{
    $user = User::factory()->create();
    $user->givePermissionTo('trips.manage');
    $capture = CaptureEvent::factory()->accepted()->create();

    $response = $this->actingAs($user, 'api')->postJson("/api/v1/admin/captures/{$capture->id}/support-image-recovery");

    $response->assertAccepted()
        ->assertJsonPath('data.capture_event_id', $capture->id)
        ->assertJsonPath('data.media_kind', 'support_image');
}
```

- [ ] **Step 2: Run manual recovery tests to verify they fail**

Run:

```powershell
php artisan test --filter=CaptureSupportImageRecoveryTest
```

Expected: FAIL because the Resource field and endpoint do not exist.

- [ ] **Step 3: Update `TripEventResource`**

Load latest attempt:

```php
$this->resource->loadMissing([
    'captureEvent.mediaAssets',
    'captureEvent.cameraPair',
    'captureEvent.latestSupportImageRecoveryAttempt',
]);
```

Add:

```php
'support_image_recovery' => $capture->latestSupportImageRecoveryAttempt ? [
    'id' => $capture->latestSupportImageRecoveryAttempt->id,
    'uuid' => $capture->latestSupportImageRecoveryAttempt->uuid,
    'status' => $capture->latestSupportImageRecoveryAttempt->status?->value,
    'attempts' => $capture->latestSupportImageRecoveryAttempt->attempts,
    'last_error' => $capture->latestSupportImageRecoveryAttempt->last_error,
    'next_attempt_at' => $capture->latestSupportImageRecoveryAttempt->next_attempt_at?->toJSON(),
    'updated_at' => $capture->latestSupportImageRecoveryAttempt->updated_at?->toJSON(),
] : null,
```

- [ ] **Step 4: Implement manual retry Controller**

The invokable Controller receives `CaptureEvent $captureEvent`, calls `ScheduleSupportImageRecoveryAction`, and returns `202 Accepted` with the attempt resource or JSON data. It does not call the NVR synchronously from the HTTP request.

- [ ] **Step 5: Register manual recovery route**

```php
Route::post('/captures/{captureEvent}/support-image-recovery', CaptureSupportImageRecoveryController::class)
    ->middleware('permission:trips.manage,api');
```

- [ ] **Step 6: Run manual recovery tests to verify they pass**

Run:

```powershell
php artisan test --filter=CaptureSupportImageRecoveryTest
```

Expected: PASS.

- [ ] **Step 7: Commit trip recovery API**

```powershell
git add app/Http/Controllers/Api/V1/Admin/CaptureSupportImageRecoveryController.php app/Http/Resources/Api/V1/TripEventResource.php routes/api.php tests/Feature/Admin/CaptureSupportImageRecoveryTest.php
git commit -m "feat: expose support image recovery state"
```

---

### Task 6: Frontend NVR Management Screen With Vuestic Components

**Files:**
- Modify: `RIALMA-TrackVision-Frontend/src/types/admin.ts`
- Create: `RIALMA-TrackVision-Frontend/src/services/recordingDevicesService.ts`
- Create: `RIALMA-TrackVision-Frontend/src/services/cameraRecordingSourcesService.ts`
- Create: `RIALMA-TrackVision-Frontend/src/components/forms/RecordingDeviceForm.vue`
- Create: `RIALMA-TrackVision-Frontend/src/components/forms/CameraRecordingSourceForm.vue`
- Create: `RIALMA-TrackVision-Frontend/src/pages/RecordingDevicesPage.vue`
- Create: `RIALMA-TrackVision-Frontend/src/pages/RecordingDevicesPage.test.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/router/index.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/components/navigation/TheSidebar.vue`
- Modify: `RIALMA-TrackVision-Frontend/src/components/navigation/TheSidebar.test.ts`

**Interfaces:**
- Consumes: backend admin APIs from Task 2.
- Produces: route `recording-devices`, page title `Gravadores/NVRs`, Vuestic forms, and services for CRUD and mapping.

- [ ] **Step 1: Write failing frontend page tests**

```ts
it('renders nvr devices and opens the Vuestic form', async () => {
  vi.spyOn(recordingDevicesService, 'list').mockResolvedValue({
    data: [{
      id: 1,
      uuid: 'nvr-uuid',
      name: 'NVR Portaria 01',
      vendor: 'intelbras',
      protocol: 'http',
      host: '10.0.8.150',
      port: 80,
      username: 'trackvision_ro',
      auth_type: 'digest',
      has_password: true,
      is_active: true,
      location: { id: 1, uuid: 'loc-uuid', name: 'Portaria' },
      edge_node: { id: 1, uuid: 'edge-uuid', name: 'Edge Local' },
    }],
  })

  const wrapper = mount(RecordingDevicesPage, {
    global: { plugins: [createTestingPinia(), createVuesticTestPlugin()] },
  })
  await flushPromises()

  expect(wrapper.text()).toContain('Gravadores/NVRs')
  expect(wrapper.text()).toContain('NVR Portaria 01')
  await wrapper.get('[data-test="create-recording-device"]').trigger('click')
  expect(wrapper.get('[data-test="recording-device-form"]').exists()).toBe(true)
})

it('omits empty password when editing a recording device', async () => {
  const submit = vi.fn()
  const wrapper = mount(RecordingDeviceForm, {
    props: {
      modelValue: {
        location_id: 1,
        edge_node_id: 1,
        name: 'NVR Portaria 01',
        vendor: 'intelbras',
        protocol: 'http',
        host: '10.0.8.150',
        port: 80,
        username: 'trackvision_ro',
        auth_type: 'digest',
        password: '',
        is_active: true,
      },
      isEditing: true,
    },
    global: { plugins: [createVuesticTestPlugin()] },
  })

  wrapper.vm.$emit('submit', wrapper.vm.payloadFromForm())
  expect(submit).not.toHaveBeenCalled()
  expect(wrapper.vm.payloadFromForm()).not.toHaveProperty('password')
})
```

- [ ] **Step 2: Run frontend NVR page tests to verify they fail**

Run:

```powershell
npm run test -- src/pages/RecordingDevicesPage.test.ts --reporter=dot
```

Expected: FAIL because page, services, route, and forms do not exist.

- [ ] **Step 3: Add TypeScript contracts**

```ts
export type RecordingDeviceVendor = 'intelbras'
export type RecordingDeviceProtocol = 'http' | 'https'
export type RecordingDeviceAuthType = 'digest' | 'basic' | 'none'
export type RecordingStream = 'main' | 'sub'
export type MediaRecoveryStatus = 'pending_configuration' | 'pending' | 'running' | 'recovered' | 'not_found' | 'failed'

export interface RecordingDevice {
  id: number
  uuid: string
  name: string
  vendor: RecordingDeviceVendor
  protocol: RecordingDeviceProtocol
  host: string
  port: number
  username: string | null
  auth_type: RecordingDeviceAuthType
  has_password: boolean
  is_active: boolean
  location?: Pick<Location, 'id' | 'uuid' | 'name'>
  edge_node?: Pick<EdgeNode, 'id' | 'uuid' | 'name'>
}

export interface RecordingDeviceInput {
  location_id: number
  edge_node_id: number
  name: string
  vendor: RecordingDeviceVendor
  protocol: RecordingDeviceProtocol
  host: string
  port: number
  username: string | null
  password?: string
  auth_type: RecordingDeviceAuthType
  is_active: boolean
}

export interface CameraRecordingSource {
  id: number
  uuid: string
  camera?: Camera
  recording_device?: RecordingDevice
  channel: number
  stream: RecordingStream
  target_offset_seconds: number
  search_window_seconds: number
  is_active: boolean
}
```

- [ ] **Step 4: Add services**

`recordingDevicesService` methods: `list()`, `create(input)`, `update(device, input)`, `remove(device)`.

`cameraRecordingSourcesService` methods: `list()`, `create(input)`, `update(source, input)`, `remove(source)`.

Both use `createApiClient` and Laravel resource/pagination types used by existing services.

- [ ] **Step 5: Build Vuestic forms**

`RecordingDeviceForm.vue` uses `VaForm`, `VaInput`, `VaSelect`, `VaSwitch`, `VaButton`, and emits `submit` and `cancel`. The edit payload omits empty password:

```ts
function payloadFromForm(): RecordingDeviceInput {
  const payload = { ...form.value }
  if (props.isEditing && !payload.password?.trim()) {
    delete payload.password
  }
  return payload
}
```

`CameraRecordingSourceForm.vue` uses `VaSelect` for support cameras and NVR devices, `VaInput type="number"` for channel, offset, and window, and `VaSwitch` for active state.

- [ ] **Step 6: Build `RecordingDevicesPage.vue`**

The page must include:

- `VaCard` around filter/actions;
- `VaDataTable` for NVR devices;
- `VaModal` with `RecordingDeviceForm`;
- `VaModal` with `CameraRecordingSourceForm`;
- `VaAlert` for load/save errors;
- loading, empty, success, create, edit, deactivate, and mapping states.

- [ ] **Step 7: Add route and sidebar item**

In router:

```ts
{
  path: 'recording-devices',
  name: 'recording-devices',
  component: RecordingDevicesPage,
  meta: { permission: 'cameras.manage' },
}
```

In sidebar navigation:

```ts
{ label: 'Gravadores/NVRs', route: 'recording-devices', permission: 'cameras.manage', icon: Server }
```

- [ ] **Step 8: Run frontend NVR page tests to verify they pass**

Run:

```powershell
npm run test -- src/pages/RecordingDevicesPage.test.ts src/components/navigation/TheSidebar.test.ts --reporter=dot
```

Expected: PASS.

- [ ] **Step 9: Commit frontend NVR management**

```powershell
git add src/types/admin.ts src/services src/components/forms src/pages/RecordingDevicesPage.vue src/pages/RecordingDevicesPage.test.ts src/router/index.ts src/components/navigation/TheSidebar.vue src/components/navigation/TheSidebar.test.ts
git commit -m "feat: add nvr management screen"
```

---

### Task 7: Frontend Trip Recovery Status And Manual Recovery Action

**Files:**
- Modify: `RIALMA-TrackVision-Frontend/src/types/admin.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/services/tripsService.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/pages/TripsPage.vue`
- Modify: `RIALMA-TrackVision-Frontend/src/pages/TripsPage.test.ts`

**Interfaces:**
- Consumes: backend trip resource field `support_image_recovery`.
- Produces: operator-visible recovery state for missing support image and `tripsService.requestSupportImageRecovery(event: TripEvent): Promise<SupportImageRecoveryAttempt>`.

- [ ] **Step 1: Write failing trip recovery UI tests**

```ts
it('shows pending support image recovery when support image is missing', async () => {
  mockTripsShow({
    events: [{
      id: 10,
      uuid: 'event-uuid',
      direction: 'outbound',
      load_status: 'unknown',
      occurred_at: '2026-08-17T10:30:15Z',
      capture: { id: 99, uuid: 'capture-uuid', plate: 'ABC1D23', plate_normalized: 'ABC1D23', event_time: '2026-08-17T10:30:15Z' },
      media: { lpr_image: null, support_image: null },
      support_image_recovery: { status: 'pending', attempts: 1, last_error: null, next_attempt_at: null, updated_at: null },
    }],
  })

  const wrapper = await mountTripsPageAndSelectFirstTrip()

  expect(wrapper.text()).toContain('Recuperacao pendente')
  expect(wrapper.get('[data-test="request-support-recovery"]').exists()).toBe(true)
})

it('requests manual recovery through the trips service', async () => {
  const request = vi.spyOn(tripsService, 'requestSupportImageRecovery').mockResolvedValue({
    status: 'pending',
    attempts: 0,
    last_error: null,
    next_attempt_at: null,
    updated_at: null,
  })

  const wrapper = await mountTripsPageAndSelectFirstTrip()
  await wrapper.get('[data-test="request-support-recovery"]').trigger('click')

  expect(request).toHaveBeenCalledWith(expect.objectContaining({ id: 10 }))
  expect(wrapper.text()).toContain('Recuperacao solicitada.')
})
```

- [ ] **Step 2: Run trip page tests to verify they fail**

Run:

```powershell
npm run test -- src/pages/TripsPage.test.ts --reporter=dot
```

Expected: FAIL because types, service method, and UI action do not exist.

- [ ] **Step 3: Add frontend recovery types**

```ts
export interface SupportImageRecoveryAttempt {
  id?: number
  uuid?: string
  status: MediaRecoveryStatus
  attempts: number
  last_error: string | null
  next_attempt_at: string | null
  updated_at: string | null
}
```

Add `support_image_recovery?: SupportImageRecoveryAttempt | null` to `TripEvent`.

- [ ] **Step 4: Add service method**

```ts
async requestSupportImageRecovery(tripEvent: TripEvent): Promise<SupportImageRecoveryAttempt> {
  const response = await client.post<LaravelResource<SupportImageRecoveryAttempt>>(
    `/admin/captures/${tripEvent.capture.id}/support-image-recovery`,
    {},
  )
  return response.data
}
```

- [ ] **Step 5: Update `TripsPage.vue` with Vuestic UI**

In each event block, under the support image figure:

```vue
<VaAlert
  v-if="!event.media.support_image && event.support_image_recovery"
  class="recovery-alert"
  :color="recoveryColor(event.support_image_recovery.status)"
>
  {{ recoveryLabel(event.support_image_recovery.status) }}
  <span v-if="event.support_image_recovery.last_error">
    {{ event.support_image_recovery.last_error }}
  </span>
</VaAlert>

<VaButton
  v-if="canManageTrips && !event.media.support_image"
  class="base-button"
  color="secondary"
  data-test="request-support-recovery"
  type="button"
  :loading="recoveringEventId === event.id"
  @click="requestSupportRecovery(event)"
>
  Recuperar apoio
</VaButton>
```

Add helper labels:

```ts
function recoveryLabel(status: MediaRecoveryStatus): string {
  return ({
    pending_configuration: 'NVR nao configurado',
    pending: 'Recuperacao pendente',
    running: 'Recuperando imagem',
    recovered: 'Imagem recuperada',
    not_found: 'Imagem nao encontrada no NVR',
    failed: 'Falha na recuperacao',
  } as Record<MediaRecoveryStatus, string>)[status]
}
```

- [ ] **Step 6: Run trip page tests to verify they pass**

Run:

```powershell
npm run test -- src/pages/TripsPage.test.ts --reporter=dot
```

Expected: PASS.

- [ ] **Step 7: Commit frontend trip recovery UI**

```powershell
git add src/types/admin.ts src/services/tripsService.ts src/pages/TripsPage.vue src/pages/TripsPage.test.ts
git commit -m "feat: show support image recovery status"
```

---

### Task 8: Operations Documentation, Local Verification, And Push

**Files:**
- Create: `docs/operations/nvr-support-image-recovery.md`
- Modify: `docs/operations/intelbras-vip-5460-field-checklist.md`
- Modify: `docs/project-context.md`

**Interfaces:**
- Consumes: finished backend and frontend implementation.
- Produces: operational runbook explaining required services, scheduled command, NVR field data, and validation.

- [ ] **Step 1: Add NVR operations runbook**

Create `docs/operations/nvr-support-image-recovery.md` with:

```markdown
# NVR Support Image Recovery

## O Que Deve Ficar Rodando

- Backend Laravel edge.
- Listener LPR Intelbras ou webhook configurado para o endpoint do edge.
- MySQL local.
- Comando recorrente `php artisan edge:recover-support-images --missing-only --limit=50`.
- Sincronizacao edge-to-parent quando houver internet.

## Dados De Campo Obrigatorios

- IP, porta, protocolo e usuario somente leitura do NVR.
- Canal do NVR para cada camera de apoio.
- Confirmacao de gravacao ativa no canal.
- Retencao minima esperada.
- NTP alinhado entre LPR, camera de apoio, NVR, edge e parent.
- Template HTTP validado para recuperar JPEG historico no firmware instalado.

## Teste De Campo

1. Criar uma passagem real pela LPR.
2. Confirmar `CaptureEvent` criado no edge.
3. Confirmar ausencia ou presenca da `support_image`.
4. Rodar `php artisan edge:recover-support-images --missing-only --limit=10`.
5. Abrir a viagem no frontend e confirmar a imagem de apoio.
6. Exportar PDF e confirmar que a imagem de apoio aparece no relatorio.
```

- [ ] **Step 2: Update VIP 5460 checklist and project context**

Add the NVR/NTP/channel section to the field checklist and a short summary in `docs/project-context.md` under operational architecture.

- [ ] **Step 3: Run full backend verification**

Run:

```powershell
cd C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Backend
php artisan test
php artisan migrate --force
```

Expected: all tests pass and migrations are applied to the local MySQL configured for the project.

- [ ] **Step 4: Run full frontend verification**

Run:

```powershell
cd C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Frontend
npm run lint
npm run test -- --reporter=dot
npm run build
```

Expected: lint, tests, and build exit `0`.

- [ ] **Step 5: Verify runtime endpoints**

Run:

```powershell
Invoke-WebRequest -Uri 'http://127.0.0.1:5173/recording-devices' -UseBasicParsing
Invoke-WebRequest -Uri 'http://127.0.0.1:8000/api/v1/auth/login' -Method OPTIONS -UseBasicParsing
```

Expected: frontend returns HTTP 200 for the Vite app and backend responds without a connection error.

- [ ] **Step 6: Run Playwright headed smoke**

Use the browser skill or local Playwright with headed mode:

```powershell
npx playwright test --headed
```

Expected:

- login renders with Vuestic template;
- sidebar contains `Gravadores/NVRs`;
- NVR page opens;
- Trips page shows recovery state for a seeded or test capture without `support_image`;
- PDF export still downloads after recovered media exists.

- [ ] **Step 7: Commit docs**

```powershell
cd C:\projetos\rialma\RIALMA-TrackVision-AgentOps
git add docs/operations/nvr-support-image-recovery.md docs/operations/intelbras-vip-5460-field-checklist.md docs/project-context.md docs/superpowers/plans/2026-08-17-nvr-support-image-recovery-implementation.md
git commit -m "docs: plan nvr support image recovery implementation"
```

- [ ] **Step 8: Push all changed repositories**

```powershell
cd C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Backend
git push origin main

cd C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Frontend
git push origin main

cd C:\projetos\rialma\RIALMA-TrackVision-AgentOps
git push origin main
```

Expected: backend, frontend, and AgentOps `main` branches are updated on GitHub.

---

## Self-Review

- Spec coverage: data model, NVR admin, camera-to-channel mapping, encrypted credentials, recovery scheduling, command, idempotent media attach, outbox requeue, trip API state, manual retry, Vuestic frontend, operations runbook, NTP, and field validation are covered in Tasks 1-8.
- Endpoint uncertainty coverage: Task 3 prevents unsafe image attachment while the Intelbras historical-frame endpoint template is empty, and the operations runbook requires field validation before production recovery.
- Security coverage: Resources omit passwords, logs store sanitized errors, frontend does not receive secrets, and parent nodes never access the NVR.
- Type consistency: `MediaRecoveryStatus`, `RecordingStream`, `RecordingPlaybackClient::captureFrame()`, `ScheduleSupportImageRecoveryAction::execute()`, and `tripsService.requestSupportImageRecovery()` keep the same names and payload shapes across backend and frontend tasks.
- Marker scan: the plan contains no unresolved work markers, no vague validation instructions, and no repeated task shortcuts.
