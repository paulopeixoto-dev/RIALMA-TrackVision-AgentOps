# Edge Parent Synchronization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implementar a sincronizacao v1 entre o backend edge local e o backend parent central, com bootstrap, heartbeat, batch multipart idempotente, imagens privadas e retry offline.

**Architecture:** O parent expoe endpoints versionados protegidos por Passport client credentials e pelo header `X-TrackVision-Edge-Node`. O edge usa um cliente HTTP dedicado para autenticar no parent, baixar bootstrap, enviar heartbeat e publicar lotes da outbox; regras de dominio ficam em Actions, validacao em Form Requests, payloads em DTOs/Services e Controllers permanecem finos.

**Tech Stack:** Laravel 13, PHP 8.4, Laravel Passport, PostgreSQL, SQLite in-memory em testes, Laravel HTTP Client, Laravel Storage, PHPUnit, private local disk.

## Global Constraints

- Codigo de backend deve ser alterado somente em `RIALMA-TrackVision-Backend`.
- Documentacao agentica deve ser alterada somente em `RIALMA-TrackVision-AgentOps`.
- Seguir `docs/backend-laravel-guidelines.md`: Controllers magros, Form Requests, Eloquent eficiente, Services ou Actions para regra de negocio.
- O parent deve usar Passport client credentials com scopes `edge:read`, `edge:write` e `captures:write`.
- Toda chamada edge-parent deve enviar `X-TrackVision-Edge-Node: {edge_node_uuid}`.
- O edge deve executar `php artisan edge:sync-parent`.
- O batch de capturas deve usar `multipart/form-data` com uma parte `metadata` JSON e arquivos JPEG separados.
- O parent deve deduplicar por `capture_events.idempotency_key`.
- Imagens devem continuar em storage privado.
- Camera passwords nunca podem aparecer no bootstrap, logs ou respostas de API.
- Tokens OAuth, Authorization headers, client secret e bytes de imagem nunca devem ser registrados em log.
- O edge so marca outbox como `synced` apos resposta parent com item em `accepted` ou `duplicates`.
- Itens rejeitados pelo parent ficam `failed` com `last_error`.
- Falhas HTTP ou rede mantem itens `pending` com `attempts`, `last_error` e `available_at` recalculado.
- Backoff v1: `min(60 minutos, 2 ^ attempts minutos)`.
- Veiculos recebidos no batch devem existir no cadastro global e estar ativos.
- `camera_pair_uuid`, camera LPR e camera de apoio devem pertencer ao edge node autenticado.
- O edge nao deve apagar capturas locais nesta fase.

---

## File Structure

Backend code files:

```text
RIALMA-TrackVision-Backend/
|-- app/
|   |-- Actions/Edge/
|   |   |-- BuildEdgeBootstrapPayloadAction.php
|   |   |-- IngestCaptureBatchAction.php
|   |   |-- PushOutboxBatchAction.php
|   |   |-- SendEdgeHeartbeatAction.php
|   |   |-- StoreEdgeHeartbeatAction.php
|   |   |-- StoreSyncedCaptureMediaAction.php
|   |   |-- SyncFromParentAction.php
|   |   `-- UpsertEdgeCaptureAction.php
|   |-- Console/Commands/
|   |   `-- EdgeSyncParentCommand.php
|   |-- Data/Edge/
|   |   |-- EdgeBatchItemResultData.php
|   |   |-- EdgeCaptureBatchData.php
|   |   |-- EdgeCaptureBatchItemData.php
|   |   |-- EdgeCaptureMediaData.php
|   |   `-- EdgeCaptureBatchResponseData.php
|   |-- Enums/
|   |   `-- EdgeSyncBatchStatus.php
|   |-- Http/
|   |   |-- Controllers/Api/V1/Edge/
|   |   |   |-- BootstrapController.php
|   |   |   |-- CaptureIngestController.php
|   |   |   `-- HeartbeatController.php
|   |   |-- Middleware/
|   |   |   `-- EnsureActiveParentEdgeNode.php
|   |   |-- Requests/Api/V1/Edge/
|   |   |   |-- StoreEdgeCaptureBatchRequest.php
|   |   |   `-- StoreEdgeHeartbeatRequest.php
|   |   `-- Resources/Api/V1/Edge/
|   |       |-- EdgeBootstrapResource.php
|   |       |-- EdgeCaptureBatchResultResource.php
|   |       `-- EdgeHeartbeatResource.php
|   |-- Models/
|   |   |-- EdgeSyncBatch.php
|   |   `-- EdgeSyncState.php
|   `-- Services/EdgeSync/
|       |-- EdgeCaptureBatchResponseParser.php
|       |-- EdgeCaptureBatchSerializer.php
|       |-- EdgeOutboxBackoff.php
|       `-- ParentApiClient.php
|-- config/trackvision.php
|-- database/
|   |-- factories/
|   |   |-- EdgeSyncBatchFactory.php
|   |   `-- EdgeSyncStateFactory.php
|   `-- migrations/
|       |-- 2026_08_12_170001_create_edge_sync_batches_table.php
|       `-- 2026_08_12_170002_create_edge_sync_state_table.php
|-- docs/
|   `-- api-edge-sync.md
|-- routes/api.php
`-- tests/
    |-- Feature/Edge/
    |   |-- EdgeCaptureBatchIngestTest.php
    |   |-- EdgeHeartbeatTest.php
    |   |-- EdgeSyncBootstrapTest.php
    |   `-- EdgeSyncParentCommandTest.php
    `-- Unit/EdgeSync/
        |-- EdgeCaptureBatchResponseParserTest.php
        |-- EdgeCaptureBatchSerializerTest.php
        `-- EdgeOutboxBackoffTest.php
```

AgentOps documentation files:

```text
RIALMA-TrackVision-AgentOps/
`-- docs/superpowers/plans/
    |-- 2026-08-12-trackvision-system-implementation.md
    `-- 2026-08-12-edge-parent-synchronization-implementation.md
```

---

### Task 1: Sync Tables, Models, Enums, and Configuration

**Files:**

- Modify: `RIALMA-TrackVision-Backend/config/trackvision.php`
- Modify: `RIALMA-TrackVision-Backend/.env.example`
- Create: `RIALMA-TrackVision-Backend/app/Enums/EdgeSyncBatchStatus.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/EdgeSyncBatch.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/EdgeSyncState.php`
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_12_170001_create_edge_sync_batches_table.php`
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_12_170002_create_edge_sync_state_table.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/EdgeSyncBatchFactory.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/EdgeSyncStateFactory.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Edge/EdgeSyncSchemaTest.php`

**Interfaces:**

- Consumes: existing `App\Models\EdgeNode`.
- Produces: `App\Enums\EdgeSyncBatchStatus::Received`, `::Partial`, `::Failed`.
- Produces: `App\Models\EdgeSyncBatch` with relationship `edgeNode(): BelongsTo`.
- Produces: `App\Models\EdgeSyncState`.
- Produces config keys `trackvision.parent_oauth_client_id`, `trackvision.parent_oauth_client_secret`, `trackvision.sync_timeout_seconds`, `trackvision.sync_connect_timeout_seconds`, `trackvision.sync_retry_max_minutes`.

- [ ] **Step 1: Write failing schema tests**

Create `tests/Feature/Edge/EdgeSyncSchemaTest.php`:

```php
<?php

namespace Tests\Feature\Edge;

use App\Enums\EdgeSyncBatchStatus;
use App\Models\EdgeNode;
use App\Models\EdgeSyncBatch;
use App\Models\EdgeSyncState;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Schema;
use Tests\TestCase;

class EdgeSyncSchemaTest extends TestCase
{
    use RefreshDatabase;

    public function test_edge_sync_batch_schema_exists(): void
    {
        $this->assertTrue(Schema::hasColumns('edge_sync_batches', [
            'uuid',
            'edge_node_id',
            'status',
            'items_count',
            'accepted_count',
            'duplicate_count',
            'rejected_count',
            'received_at',
            'summary',
        ]));
    }

    public function test_edge_sync_state_schema_exists(): void
    {
        $this->assertTrue(Schema::hasColumns('edge_sync_state', [
            'edge_node_uuid',
            'last_bootstrap_at',
            'last_push_at',
            'last_successful_push_at',
            'last_error',
            'bootstrap_payload_hash',
        ]));
    }

    public function test_models_cast_status_and_dates(): void
    {
        $edgeNode = EdgeNode::factory()->create();

        $batch = EdgeSyncBatch::factory()->create([
            'edge_node_id' => $edgeNode->id,
            'status' => EdgeSyncBatchStatus::Received,
            'summary' => ['accepted' => 1],
        ]);

        $state = EdgeSyncState::factory()->create([
            'edge_node_uuid' => $edgeNode->uuid,
            'bootstrap_payload_hash' => hash('sha256', 'payload'),
        ]);

        $this->assertTrue($batch->edgeNode->is($edgeNode));
        $this->assertSame(EdgeSyncBatchStatus::Received, $batch->status);
        $this->assertSame(['accepted' => 1], $batch->summary);
        $this->assertSame($edgeNode->uuid, $state->edge_node_uuid);
    }
}
```

- [ ] **Step 2: Run schema test and verify it fails**

Run:

```bash
php artisan test --filter=EdgeSyncSchemaTest
```

Expected: FAIL because `edge_sync_batches`, `edge_sync_state`, models, factories, and enum are absent.

- [ ] **Step 3: Add sync configuration**

Modify `config/trackvision.php`:

```php
'parent_oauth_client_id' => env('TRACKVISION_PARENT_OAUTH_CLIENT_ID'),
'parent_oauth_client_secret' => env('TRACKVISION_PARENT_OAUTH_CLIENT_SECRET'),
'sync_connect_timeout_seconds' => (int) env('TRACKVISION_SYNC_CONNECT_TIMEOUT_SECONDS', 5),
'sync_timeout_seconds' => (int) env('TRACKVISION_SYNC_TIMEOUT_SECONDS', 30),
'sync_retry_max_minutes' => (int) env('TRACKVISION_SYNC_RETRY_MAX_MINUTES', 60),
```

Append to `.env.example`:

```text
TRACKVISION_PARENT_OAUTH_CLIENT_ID=
TRACKVISION_PARENT_OAUTH_CLIENT_SECRET=
TRACKVISION_SYNC_CONNECT_TIMEOUT_SECONDS=5
TRACKVISION_SYNC_TIMEOUT_SECONDS=30
TRACKVISION_SYNC_RETRY_MAX_MINUTES=60
```

- [ ] **Step 4: Create `EdgeSyncBatchStatus` enum**

Create `app/Enums/EdgeSyncBatchStatus.php`:

```php
<?php

namespace App\Enums;

enum EdgeSyncBatchStatus: string
{
    case Received = 'received';
    case Partial = 'partial';
    case Failed = 'failed';
}
```

- [ ] **Step 5: Create sync migrations**

Create `database/migrations/2026_08_12_170001_create_edge_sync_batches_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('edge_sync_batches', function (Blueprint $table): void {
            $table->id();
            $table->uuid('uuid')->unique();
            $table->foreignId('edge_node_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
            $table->string('status', 30);
            $table->unsignedInteger('items_count')->default(0);
            $table->unsignedInteger('accepted_count')->default(0);
            $table->unsignedInteger('duplicate_count')->default(0);
            $table->unsignedInteger('rejected_count')->default(0);
            $table->timestamp('received_at')->index();
            $table->json('summary')->nullable();
            $table->timestamps();

            $table->index(['edge_node_id', 'received_at']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('edge_sync_batches');
    }
};
```

Create `database/migrations/2026_08_12_170002_create_edge_sync_state_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('edge_sync_state', function (Blueprint $table): void {
            $table->id();
            $table->uuid('edge_node_uuid')->unique();
            $table->timestamp('last_bootstrap_at')->nullable();
            $table->timestamp('last_push_at')->nullable();
            $table->timestamp('last_successful_push_at')->nullable();
            $table->text('last_error')->nullable();
            $table->string('bootstrap_payload_hash', 64)->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('edge_sync_state');
    }
};
```

- [ ] **Step 6: Create sync models**

Create `app/Models/EdgeSyncBatch.php`:

```php
<?php

namespace App\Models;

use App\Enums\EdgeSyncBatchStatus;
use App\Models\Concerns\HasPublicUuid;
use Database\Factories\EdgeSyncBatchFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Fillable(['uuid', 'edge_node_id', 'status', 'items_count', 'accepted_count', 'duplicate_count', 'rejected_count', 'received_at', 'summary'])]
class EdgeSyncBatch extends Model
{
    /** @use HasFactory<EdgeSyncBatchFactory> */
    use HasFactory, HasPublicUuid;

    protected function casts(): array
    {
        return [
            'status' => EdgeSyncBatchStatus::class,
            'items_count' => 'integer',
            'accepted_count' => 'integer',
            'duplicate_count' => 'integer',
            'rejected_count' => 'integer',
            'received_at' => 'immutable_datetime',
            'summary' => 'array',
        ];
    }

    public function edgeNode(): BelongsTo
    {
        return $this->belongsTo(EdgeNode::class);
    }
}
```

Create `app/Models/EdgeSyncState.php`:

```php
<?php

namespace App\Models;

use Database\Factories\EdgeSyncStateFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

#[Fillable(['edge_node_uuid', 'last_bootstrap_at', 'last_push_at', 'last_successful_push_at', 'last_error', 'bootstrap_payload_hash'])]
class EdgeSyncState extends Model
{
    /** @use HasFactory<EdgeSyncStateFactory> */
    use HasFactory;

    protected $table = 'edge_sync_state';

    protected function casts(): array
    {
        return [
            'last_bootstrap_at' => 'immutable_datetime',
            'last_push_at' => 'immutable_datetime',
            'last_successful_push_at' => 'immutable_datetime',
        ];
    }
}
```

- [ ] **Step 7: Create factories**

Create `database/factories/EdgeSyncBatchFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Enums\EdgeSyncBatchStatus;
use App\Models\EdgeNode;
use App\Models\EdgeSyncBatch;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<EdgeSyncBatch>
 */
class EdgeSyncBatchFactory extends Factory
{
    public function definition(): array
    {
        return [
            'edge_node_id' => EdgeNode::factory(),
            'status' => EdgeSyncBatchStatus::Received,
            'items_count' => 1,
            'accepted_count' => 1,
            'duplicate_count' => 0,
            'rejected_count' => 0,
            'received_at' => now(),
            'summary' => ['factory' => true],
        ];
    }
}
```

Create `database/factories/EdgeSyncStateFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\EdgeSyncState;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<EdgeSyncState>
 */
class EdgeSyncStateFactory extends Factory
{
    public function definition(): array
    {
        return [
            'edge_node_uuid' => fake()->uuid(),
            'last_bootstrap_at' => now(),
            'last_push_at' => now(),
            'last_successful_push_at' => now(),
            'last_error' => null,
            'bootstrap_payload_hash' => fake()->sha256(),
        ];
    }
}
```

- [ ] **Step 8: Run schema tests**

Run:

```bash
php artisan test --filter=EdgeSyncSchemaTest
```

Expected: PASS.

- [ ] **Step 9: Commit Task 1**

Run:

```bash
git add config/trackvision.php .env.example app/Enums/EdgeSyncBatchStatus.php app/Models/EdgeSyncBatch.php app/Models/EdgeSyncState.php database/migrations/2026_08_12_170001_create_edge_sync_batches_table.php database/migrations/2026_08_12_170002_create_edge_sync_state_table.php database/factories/EdgeSyncBatchFactory.php database/factories/EdgeSyncStateFactory.php tests/Feature/Edge/EdgeSyncSchemaTest.php
git commit -m "feat: add edge sync state models"
```

---

### Task 2: Parent Bootstrap and Heartbeat API

**Files:**

- Modify: `RIALMA-TrackVision-Backend/routes/api.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Middleware/EnsureActiveParentEdgeNode.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/BuildEdgeBootstrapPayloadAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/StoreEdgeHeartbeatAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Edge/BootstrapController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Edge/HeartbeatController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Edge/StoreEdgeHeartbeatRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/Edge/EdgeBootstrapResource.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/Edge/EdgeHeartbeatResource.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Edge/EdgeSyncBootstrapTest.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Edge/EdgeHeartbeatTest.php`

**Interfaces:**

- Consumes: Passport middleware `EnsureClientIsResourceOwner::using('edge:read')` and `EnsureClientIsResourceOwner::using('edge:write')`.
- Produces request attribute `trackvision.edge_node` containing `App\Models\EdgeNode`.
- Produces: `GET /api/v1/edge/bootstrap`.
- Produces: `POST /api/v1/edge/heartbeat`.
- Produces: `BuildEdgeBootstrapPayloadAction::execute(EdgeNode $edgeNode): array`.
- Produces: `StoreEdgeHeartbeatAction::execute(EdgeNode $edgeNode, array $data): EdgeNode`.

- [ ] **Step 1: Write failing bootstrap tests**

Create `tests/Feature/Edge/EdgeSyncBootstrapTest.php`:

```php
<?php

namespace Tests\Feature\Edge;

use App\Enums\CameraPairDirection;
use App\Enums\CameraType;
use App\Models\Camera;
use App\Models\CameraPair;
use App\Models\EdgeNode;
use App\Models\Vehicle;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Passport\Client;
use Laravel\Passport\Passport;
use Tests\TestCase;

class EdgeSyncBootstrapTest extends TestCase
{
    use RefreshDatabase;

    public function test_bootstrap_returns_active_operational_data_for_authenticated_edge(): void
    {
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['edge:read']);

        $edgeNode = EdgeNode::factory()->create(['uuid' => '11111111-1111-4111-8111-111111111111', 'is_active' => true]);
        $vehicle = Vehicle::factory()->create(['plate' => 'ABC-1D23', 'plate_normalized' => 'ABC1D23', 'is_active' => true]);
        Vehicle::factory()->create(['plate' => 'ZZZ-9999', 'plate_normalized' => 'ZZZ9999', 'is_active' => false]);
        $lpr = Camera::factory()->create(['edge_node_id' => $edgeNode->id, 'type' => CameraType::Lpr, 'password_encrypted' => 'secret']);
        $support = Camera::factory()->create(['edge_node_id' => $edgeNode->id, 'type' => CameraType::Support, 'password_encrypted' => 'secret']);
        $pair = CameraPair::factory()->create([
            'edge_node_id' => $edgeNode->id,
            'lpr_camera_id' => $lpr->id,
            'support_camera_id' => $support->id,
            'direction' => CameraPairDirection::Outbound,
            'is_active' => true,
        ]);

        $response = $this->withHeader('X-TrackVision-Edge-Node', $edgeNode->uuid)
            ->getJson('/api/v1/edge/bootstrap');

        $response->assertOk()
            ->assertJsonPath('data.edge_node.uuid', $edgeNode->uuid)
            ->assertJsonPath('data.vehicles.0.uuid', $vehicle->uuid)
            ->assertJsonPath('data.camera_pairs.0.uuid', $pair->uuid)
            ->assertJsonMissingPath('data.cameras.0.password_encrypted')
            ->assertJsonMissingPath('data.cameras.0.password');
    }

    public function test_bootstrap_requires_active_edge_node_header(): void
    {
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['edge:read']);

        $this->getJson('/api/v1/edge/bootstrap')->assertForbidden();
    }

    public function test_bootstrap_requires_edge_read_scope(): void
    {
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['edge:write']);
        $edgeNode = EdgeNode::factory()->create(['is_active' => true]);

        $this->withHeader('X-TrackVision-Edge-Node', $edgeNode->uuid)
            ->getJson('/api/v1/edge/bootstrap')
            ->assertForbidden();
    }
}
```

- [ ] **Step 2: Write failing heartbeat tests**

Create `tests/Feature/Edge/EdgeHeartbeatTest.php`:

```php
<?php

namespace Tests\Feature\Edge;

use App\Enums\EdgeNodeStatus;
use App\Models\EdgeNode;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Passport\Client;
use Laravel\Passport\Passport;
use Tests\TestCase;

class EdgeHeartbeatTest extends TestCase
{
    use RefreshDatabase;

    public function test_heartbeat_updates_edge_node_status_and_last_seen(): void
    {
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['edge:write']);
        $edgeNode = EdgeNode::factory()->create(['status' => EdgeNodeStatus::Offline, 'is_active' => true]);

        $response = $this->withHeader('X-TrackVision-Edge-Node', $edgeNode->uuid)
            ->postJson('/api/v1/edge/heartbeat', [
                'status' => 'online',
                'local_time' => '2026-08-12T15:00:00Z',
                'pending_outbox_count' => 12,
                'last_capture_at' => '2026-08-12T14:55:00Z',
                'disk_free_mb' => 51200,
            ]);

        $response->assertOk()
            ->assertJsonPath('data.edge_node.uuid', $edgeNode->uuid)
            ->assertJsonPath('data.edge_node.status', 'online');

        $this->assertDatabaseHas('edge_nodes', [
            'id' => $edgeNode->id,
            'status' => 'online',
        ]);
        $this->assertNotNull($edgeNode->refresh()->last_seen_at);
    }

    public function test_heartbeat_rejects_invalid_status(): void
    {
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['edge:write']);
        $edgeNode = EdgeNode::factory()->create(['is_active' => true]);

        $this->withHeader('X-TrackVision-Edge-Node', $edgeNode->uuid)
            ->postJson('/api/v1/edge/heartbeat', ['status' => 'broken'])
            ->assertUnprocessable();
    }
}
```

- [ ] **Step 3: Run bootstrap and heartbeat tests and verify they fail**

Run:

```bash
php artisan test --filter=EdgeSyncBootstrapTest
php artisan test --filter=EdgeHeartbeatTest
```

Expected: FAIL because endpoints still use the provisional bootstrap closure and heartbeat does not exist.

- [ ] **Step 4: Implement parent edge-node middleware**

Create `app/Http/Middleware/EnsureActiveParentEdgeNode.php`:

```php
<?php

namespace App\Http\Middleware;

use App\Enums\NodeRole;
use App\Models\EdgeNode;
use App\Support\NodeRoleResolver;
use Closure;
use Illuminate\Http\Request;

class EnsureActiveParentEdgeNode
{
    public function handle(Request $request, Closure $next): mixed
    {
        abort_unless(NodeRoleResolver::current() === NodeRole::Parent, 404);

        $uuid = (string) $request->header('X-TrackVision-Edge-Node');
        $edgeNode = EdgeNode::query()
            ->where('uuid', $uuid)
            ->where('is_active', true)
            ->first();

        abort_unless($edgeNode, 403);

        $request->attributes->set('trackvision.edge_node', $edgeNode);

        return $next($request);
    }
}
```

- [ ] **Step 5: Implement bootstrap payload action**

Create `app/Actions/Edge/BuildEdgeBootstrapPayloadAction.php`:

```php
<?php

namespace App\Actions\Edge;

use App\Models\EdgeNode;
use App\Models\Vehicle;

class BuildEdgeBootstrapPayloadAction
{
    public function execute(EdgeNode $edgeNode): array
    {
        $edgeNode->loadMissing([
            'cameras' => fn ($query) => $query->where('is_active', true)->orderBy('id'),
            'cameraPairs.lprCamera',
            'cameraPairs.supportCamera',
        ]);

        return [
            'edge_node' => [
                'uuid' => $edgeNode->uuid,
                'name' => $edgeNode->name,
                'status' => $edgeNode->status?->value,
            ],
            'vehicles' => Vehicle::query()
                ->where('is_active', true)
                ->orderBy('plate_normalized')
                ->get(['uuid', 'plate', 'plate_normalized', 'fleet_code', 'is_active', 'updated_at'])
                ->map(fn (Vehicle $vehicle): array => [
                    'uuid' => $vehicle->uuid,
                    'plate' => $vehicle->plate,
                    'plate_normalized' => $vehicle->plate_normalized,
                    'fleet_code' => $vehicle->fleet_code,
                    'is_active' => $vehicle->is_active,
                    'updated_at' => $vehicle->updated_at?->toISOString(),
                ])
                ->values()
                ->all(),
            'cameras' => $edgeNode->cameras->map(fn ($camera): array => [
                'uuid' => $camera->uuid,
                'type' => $camera->type?->value,
                'vendor' => $camera->vendor?->value,
                'host' => $camera->host,
                'port' => $camera->port,
                'channel' => $camera->channel,
                'is_active' => $camera->is_active,
            ])->values()->all(),
            'camera_pairs' => $edgeNode->cameraPairs
                ->where('is_active', true)
                ->map(fn ($pair): array => [
                    'uuid' => $pair->uuid,
                    'lpr_camera_uuid' => $pair->lprCamera?->uuid,
                    'support_camera_uuid' => $pair->supportCamera?->uuid,
                    'direction' => $pair->direction?->value,
                    'is_active' => $pair->is_active,
                ])
                ->values()
                ->all(),
            'sync' => [
                'server_time' => now()->toISOString(),
                'batch_size' => (int) config('trackvision.sync_batch_size', 100),
            ],
        ];
    }
}
```

- [ ] **Step 6: Implement heartbeat request and action**

Create `app/Http/Requests/Api/V1/Edge/StoreEdgeHeartbeatRequest.php`:

```php
<?php

namespace App\Http\Requests\Api\V1\Edge;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreEdgeHeartbeatRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->attributes->has('trackvision.edge_node');
    }

    public function rules(): array
    {
        return [
            'status' => ['required', Rule::in(['online', 'offline', 'degraded'])],
            'local_time' => ['nullable', 'date'],
            'pending_outbox_count' => ['nullable', 'integer', 'min:0'],
            'last_capture_at' => ['nullable', 'date'],
            'disk_free_mb' => ['nullable', 'integer', 'min:0'],
        ];
    }
}
```

Create `app/Actions/Edge/StoreEdgeHeartbeatAction.php`:

```php
<?php

namespace App\Actions\Edge;

use App\Enums\EdgeNodeStatus;
use App\Models\EdgeNode;

class StoreEdgeHeartbeatAction
{
    public function execute(EdgeNode $edgeNode, array $data): EdgeNode
    {
        $edgeNode->forceFill([
            'status' => EdgeNodeStatus::from($data['status']),
            'last_seen_at' => now(),
        ])->save();

        return $edgeNode->refresh();
    }
}
```

- [ ] **Step 7: Implement Resources and controllers**

Create `app/Http/Resources/Api/V1/Edge/EdgeBootstrapResource.php`:

```php
<?php

namespace App\Http\Resources\Api\V1\Edge;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class EdgeBootstrapResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return $this->resource;
    }
}
```

Create `app/Http/Resources/Api/V1/Edge/EdgeHeartbeatResource.php`:

```php
<?php

namespace App\Http\Resources\Api\V1\Edge;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class EdgeHeartbeatResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'edge_node' => [
                'uuid' => $this->uuid,
                'status' => $this->status?->value,
                'last_seen_at' => $this->last_seen_at?->toISOString(),
            ],
        ];
    }
}
```

Create `BootstrapController`:

```php
<?php

namespace App\Http\Controllers\Api\V1\Edge;

use App\Actions\Edge\BuildEdgeBootstrapPayloadAction;
use App\Http\Controllers\Controller;
use App\Http\Resources\Api\V1\Edge\EdgeBootstrapResource;
use App\Models\EdgeNode;
use Illuminate\Http\Request;

class BootstrapController extends Controller
{
    public function __invoke(Request $request, BuildEdgeBootstrapPayloadAction $action): EdgeBootstrapResource
    {
        /** @var EdgeNode $edgeNode */
        $edgeNode = $request->attributes->get('trackvision.edge_node');

        return EdgeBootstrapResource::make($action->execute($edgeNode));
    }
}
```

Create `HeartbeatController`:

```php
<?php

namespace App\Http\Controllers\Api\V1\Edge;

use App\Actions\Edge\StoreEdgeHeartbeatAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Api\V1\Edge\StoreEdgeHeartbeatRequest;
use App\Http\Resources\Api\V1\Edge\EdgeHeartbeatResource;
use App\Models\EdgeNode;

class HeartbeatController extends Controller
{
    public function __invoke(StoreEdgeHeartbeatRequest $request, StoreEdgeHeartbeatAction $action): EdgeHeartbeatResource
    {
        /** @var EdgeNode $edgeNode */
        $edgeNode = $request->attributes->get('trackvision.edge_node');

        return EdgeHeartbeatResource::make($action->execute($edgeNode, $request->validated()));
    }
}
```

- [ ] **Step 8: Register parent edge routes**

Modify `routes/api.php` imports:

```php
use App\Http\Controllers\Api\V1\Edge\BootstrapController;
use App\Http\Controllers\Api\V1\Edge\HeartbeatController;
use App\Http\Middleware\EnsureActiveParentEdgeNode;
```

Replace the provisional bootstrap closure with:

```php
Route::middleware(EnsureActiveParentEdgeNode::class)->group(function (): void {
    Route::middleware(EnsureClientIsResourceOwner::using('edge:read'))
        ->get('/edge/bootstrap', BootstrapController::class);

    Route::middleware(EnsureClientIsResourceOwner::using('edge:write'))
        ->post('/edge/heartbeat', HeartbeatController::class);
});
```

- [ ] **Step 9: Run bootstrap and heartbeat tests**

Run:

```bash
php artisan test --filter=EdgeSyncBootstrapTest
php artisan test --filter=EdgeHeartbeatTest
php artisan route:list --path=api/v1/edge
```

Expected: tests PASS and route list includes `GET api/v1/edge/bootstrap` plus `POST api/v1/edge/heartbeat`.

- [ ] **Step 10: Commit Task 2**

Run:

```bash
git add routes/api.php app/Http/Middleware/EnsureActiveParentEdgeNode.php app/Actions/Edge/BuildEdgeBootstrapPayloadAction.php app/Actions/Edge/StoreEdgeHeartbeatAction.php app/Http/Controllers/Api/V1/Edge/BootstrapController.php app/Http/Controllers/Api/V1/Edge/HeartbeatController.php app/Http/Requests/Api/V1/Edge/StoreEdgeHeartbeatRequest.php app/Http/Resources/Api/V1/Edge tests/Feature/Edge/EdgeSyncBootstrapTest.php tests/Feature/Edge/EdgeHeartbeatTest.php
git commit -m "feat: add parent edge bootstrap heartbeat"
```

---

### Task 3: Parent Capture Batch Ingest

**Files:**

- Modify: `RIALMA-TrackVision-Backend/routes/api.php`
- Create: `RIALMA-TrackVision-Backend/app/Data/Edge/EdgeCaptureBatchData.php`
- Create: `RIALMA-TrackVision-Backend/app/Data/Edge/EdgeCaptureBatchItemData.php`
- Create: `RIALMA-TrackVision-Backend/app/Data/Edge/EdgeCaptureMediaData.php`
- Create: `RIALMA-TrackVision-Backend/app/Data/Edge/EdgeBatchItemResultData.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/IngestCaptureBatchAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/UpsertEdgeCaptureAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/StoreSyncedCaptureMediaAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Edge/CaptureIngestController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Edge/StoreEdgeCaptureBatchRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/Edge/EdgeCaptureBatchResultResource.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Edge/EdgeCaptureBatchIngestTest.php`

**Interfaces:**

- Consumes: request attribute `trackvision.edge_node`.
- Consumes: `CaptureEvent`, `MediaAsset`, `Vehicle`, `Camera`, `CameraPair`, `EdgeSyncBatch`.
- Produces: `POST /api/v1/edge/captures/batch`.
- Produces: `StoreEdgeCaptureBatchRequest::batchData(): EdgeCaptureBatchData`.
- Produces: `IngestCaptureBatchAction::execute(EdgeNode $edgeNode, EdgeCaptureBatchData $batch, array $uploadedFiles): array`.
- Produces response arrays `accepted`, `duplicates`, `rejected`.

- [ ] **Step 1: Write failing parent batch ingest tests**

Create `tests/Feature/Edge/EdgeCaptureBatchIngestTest.php`:

```php
<?php

namespace Tests\Feature\Edge;

use App\Enums\CameraPairDirection;
use App\Enums\CameraType;
use App\Models\Camera;
use App\Models\CameraPair;
use App\Models\CaptureEvent;
use App\Models\EdgeNode;
use App\Models\Vehicle;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Laravel\Passport\Client;
use Laravel\Passport\Passport;
use Tests\TestCase;

class EdgeCaptureBatchIngestTest extends TestCase
{
    use RefreshDatabase;

    public function test_batch_creates_capture_and_private_media(): void
    {
        Storage::fake('local');
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['captures:write']);
        [$edgeNode, $vehicle, $pair, $lpr, $support] = $this->fixtures();

        $lprFile = UploadedFile::fake()->createWithContent('lpr.jpg', 'lpr-jpeg');
        $supportFile = UploadedFile::fake()->createWithContent('support.jpg', 'support-jpeg');

        $response = $this->withHeader('X-TrackVision-Edge-Node', $edgeNode->uuid)
            ->post('/api/v1/edge/captures/batch', [
                'metadata' => json_encode($this->metadata($vehicle, $pair, $lpr, $support, [
                    ['kind' => 'lpr_image', 'field' => 'files.capture-uuid.lpr_image', 'sha256_hash' => hash('sha256', 'lpr-jpeg'), 'content_type' => 'image/jpeg', 'byte_size' => strlen('lpr-jpeg')],
                    ['kind' => 'support_image', 'field' => 'files.capture-uuid.support_image', 'sha256_hash' => hash('sha256', 'support-jpeg'), 'content_type' => 'image/jpeg', 'byte_size' => strlen('support-jpeg')],
                ]), JSON_THROW_ON_ERROR),
                'files' => [
                    'capture-uuid' => [
                        'lpr_image' => $lprFile,
                        'support_image' => $supportFile,
                    ],
                ],
            ]);

        $response->assertOk()
            ->assertJsonPath('data.accepted.0.capture_uuid', 'capture-uuid')
            ->assertJsonPath('data.duplicates', [])
            ->assertJsonPath('data.rejected', []);

        $this->assertDatabaseHas('capture_events', ['uuid' => 'capture-uuid', 'idempotency_key' => 'edge-key-001']);
        $this->assertDatabaseHas('media_assets', ['kind' => 'lpr_image', 'sha256_hash' => hash('sha256', 'lpr-jpeg')]);
        $this->assertDatabaseHas('media_assets', ['kind' => 'support_image', 'sha256_hash' => hash('sha256', 'support-jpeg')]);
        Storage::disk('local')->assertExists('captures/capture-uuid/lpr_image/'.hash('sha256', 'lpr-jpeg').'.jpg');
    }

    public function test_duplicate_idempotency_key_does_not_create_second_capture(): void
    {
        Storage::fake('local');
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['captures:write']);
        [$edgeNode, $vehicle, $pair, $lpr, $support] = $this->fixtures();
        CaptureEvent::factory()->create(['uuid' => 'existing-capture', 'idempotency_key' => 'edge-key-001']);

        $response = $this->withHeader('X-TrackVision-Edge-Node', $edgeNode->uuid)
            ->post('/api/v1/edge/captures/batch', [
                'metadata' => json_encode($this->metadata($vehicle, $pair, $lpr, $support, []), JSON_THROW_ON_ERROR),
            ]);

        $response->assertOk()
            ->assertJsonPath('data.accepted', [])
            ->assertJsonPath('data.duplicates.0.idempotency_key', 'edge-key-001');

        $this->assertSame(1, CaptureEvent::where('idempotency_key', 'edge-key-001')->count());
    }

    public function test_idempotency_conflict_is_rejected_without_replacing_existing_capture(): void
    {
        Storage::fake('local');
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['captures:write']);
        [$edgeNode, $vehicle, $pair, $lpr, $support] = $this->fixtures();
        CaptureEvent::factory()->create(['uuid' => 'existing-capture', 'idempotency_key' => 'edge-key-001']);
        $metadata = $this->metadata($vehicle, $pair, $lpr, $support, []);
        $metadata['items'][0]['capture_uuid'] = 'different-capture';

        $response = $this->withHeader('X-TrackVision-Edge-Node', $edgeNode->uuid)
            ->post('/api/v1/edge/captures/batch', [
                'metadata' => json_encode($metadata, JSON_THROW_ON_ERROR),
            ]);

        $response->assertOk()
            ->assertJsonPath('data.rejected.0.reason', 'idempotency_conflict');

        $this->assertDatabaseHas('capture_events', ['uuid' => 'existing-capture', 'idempotency_key' => 'edge-key-001']);
        $this->assertDatabaseMissing('capture_events', ['uuid' => 'different-capture']);
    }

    public function test_batch_rejects_item_when_declared_file_is_missing(): void
    {
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['captures:write']);
        [$edgeNode, $vehicle, $pair, $lpr, $support] = $this->fixtures();

        $response = $this->withHeader('X-TrackVision-Edge-Node', $edgeNode->uuid)
            ->post('/api/v1/edge/captures/batch', [
                'metadata' => json_encode($this->metadata($vehicle, $pair, $lpr, $support, [
                    ['kind' => 'lpr_image', 'field' => 'files.capture-uuid.lpr_image', 'sha256_hash' => hash('sha256', 'missing'), 'content_type' => 'image/jpeg', 'byte_size' => 7],
                ]), JSON_THROW_ON_ERROR),
            ]);

        $response->assertOk()
            ->assertJsonPath('data.rejected.0.capture_uuid', 'capture-uuid')
            ->assertJsonPath('data.rejected.0.reason', 'media_file_missing');

        $this->assertDatabaseMissing('capture_events', ['uuid' => 'capture-uuid']);
    }

    public function test_batch_rejects_camera_pair_from_another_edge_node(): void
    {
        $client = Client::factory()->asClientCredentials()->create();
        Passport::actingAsClient($client, ['captures:write']);
        [$edgeNode, $vehicle, $pair, $lpr, $support] = $this->fixtures();
        $otherEdge = EdgeNode::factory()->create(['is_active' => true]);

        $response = $this->withHeader('X-TrackVision-Edge-Node', $otherEdge->uuid)
            ->post('/api/v1/edge/captures/batch', [
                'metadata' => json_encode($this->metadata($vehicle, $pair, $lpr, $support, []), JSON_THROW_ON_ERROR),
            ]);

        $response->assertOk()
            ->assertJsonPath('data.rejected.0.reason', 'camera_pair_not_available_for_edge');
    }

    private function fixtures(): array
    {
        $edgeNode = EdgeNode::factory()->create(['is_active' => true]);
        $vehicle = Vehicle::factory()->create(['plate' => 'ABC-1D23', 'plate_normalized' => 'ABC1D23', 'is_active' => true]);
        $lpr = Camera::factory()->create(['edge_node_id' => $edgeNode->id, 'type' => CameraType::Lpr]);
        $support = Camera::factory()->create(['edge_node_id' => $edgeNode->id, 'type' => CameraType::Support]);
        $pair = CameraPair::factory()->create([
            'edge_node_id' => $edgeNode->id,
            'lpr_camera_id' => $lpr->id,
            'support_camera_id' => $support->id,
            'direction' => CameraPairDirection::Outbound,
            'is_active' => true,
        ]);

        return [$edgeNode, $vehicle, $pair, $lpr, $support];
    }

    private function metadata(Vehicle $vehicle, CameraPair $pair, Camera $lpr, Camera $support, array $media): array
    {
        return [
            'batch_uuid' => 'batch-uuid-001',
            'sent_at' => '2026-08-12T15:00:00Z',
            'items' => [[
                'capture_uuid' => 'capture-uuid',
                'idempotency_key' => 'edge-key-001',
                'vehicle_uuid' => $vehicle->uuid,
                'camera_pair_uuid' => $pair->uuid,
                'lpr_camera_uuid' => $lpr->uuid,
                'support_camera_uuid' => $support->uuid,
                'plate' => 'ABC-1D23',
                'plate_normalized' => 'ABC1D23',
                'event_code' => 'TrafficJunction',
                'action' => 'Pulse',
                'event_time' => '2026-08-12T14:55:00Z',
                'lane' => 1,
                'direction' => 'outbound',
                'status' => 'accepted',
                'load_status' => 'unknown',
                'raw_payload' => ['Code' => 'TrafficJunction'],
                'media' => $media,
            ]],
        ];
    }
}
```

- [ ] **Step 2: Run parent ingest test and verify it fails**

Run:

```bash
php artisan test --filter=EdgeCaptureBatchIngestTest
```

Expected: FAIL because the batch ingest route, request, data objects, and actions are absent.

- [ ] **Step 3: Create batch data objects**

Create `app/Data/Edge/EdgeCaptureMediaData.php`:

```php
<?php

namespace App\Data\Edge;

readonly class EdgeCaptureMediaData
{
    public function __construct(
        public string $kind,
        public string $field,
        public string $sha256Hash,
        public string $contentType,
        public int $byteSize,
    ) {}

    public static function fromArray(array $data): self
    {
        return new self(
            kind: (string) $data['kind'],
            field: (string) $data['field'],
            sha256Hash: (string) $data['sha256_hash'],
            contentType: (string) $data['content_type'],
            byteSize: (int) $data['byte_size'],
        );
    }
}
```

Create `app/Data/Edge/EdgeCaptureBatchItemData.php`:

```php
<?php

namespace App\Data\Edge;

use Carbon\CarbonImmutable;

readonly class EdgeCaptureBatchItemData
{
    public function __construct(
        public string $captureUuid,
        public string $idempotencyKey,
        public string $vehicleUuid,
        public string $cameraPairUuid,
        public string $lprCameraUuid,
        public string $supportCameraUuid,
        public string $plate,
        public string $plateNormalized,
        public string $eventCode,
        public string $action,
        public CarbonImmutable $eventTime,
        public ?int $lane,
        public string $direction,
        public string $status,
        public string $loadStatus,
        public array $rawPayload,
        public array $media,
    ) {}

    public static function fromArray(array $data): self
    {
        return new self(
            captureUuid: (string) $data['capture_uuid'],
            idempotencyKey: (string) $data['idempotency_key'],
            vehicleUuid: (string) $data['vehicle_uuid'],
            cameraPairUuid: (string) $data['camera_pair_uuid'],
            lprCameraUuid: (string) $data['lpr_camera_uuid'],
            supportCameraUuid: (string) $data['support_camera_uuid'],
            plate: (string) $data['plate'],
            plateNormalized: (string) $data['plate_normalized'],
            eventCode: (string) $data['event_code'],
            action: (string) $data['action'],
            eventTime: CarbonImmutable::parse($data['event_time']),
            lane: isset($data['lane']) ? (int) $data['lane'] : null,
            direction: (string) $data['direction'],
            status: (string) $data['status'],
            loadStatus: (string) $data['load_status'],
            rawPayload: (array) $data['raw_payload'],
            media: array_map(fn (array $media): EdgeCaptureMediaData => EdgeCaptureMediaData::fromArray($media), array_key_exists('media', $data) ? $data['media'] : []),
        );
    }
}
```

Create `app/Data/Edge/EdgeCaptureBatchData.php`:

```php
<?php

namespace App\Data\Edge;

use Carbon\CarbonImmutable;

readonly class EdgeCaptureBatchData
{
    public function __construct(
        public string $batchUuid,
        public CarbonImmutable $sentAt,
        public array $items,
    ) {}

    public static function fromArray(array $data): self
    {
        return new self(
            batchUuid: (string) $data['batch_uuid'],
            sentAt: CarbonImmutable::parse($data['sent_at']),
            items: array_map(fn (array $item): EdgeCaptureBatchItemData => EdgeCaptureBatchItemData::fromArray($item), $data['items']),
        );
    }
}
```

Create `app/Data/Edge/EdgeBatchItemResultData.php`:

```php
<?php

namespace App\Data\Edge;

readonly class EdgeBatchItemResultData
{
    public function __construct(
        public string $captureUuid,
        public string $idempotencyKey,
        public string $status,
        public ?string $reason = null,
    ) {}

    public function toArray(): array
    {
        return array_filter([
            'capture_uuid' => $this->captureUuid,
            'idempotency_key' => $this->idempotencyKey,
            'status' => $this->status,
            'reason' => $this->reason,
        ], fn ($value): bool => $value !== null);
    }
}
```

- [ ] **Step 4: Implement batch request**

Create `app/Http/Requests/Api/V1/Edge/StoreEdgeCaptureBatchRequest.php`:

```php
<?php

namespace App\Http\Requests\Api\V1\Edge;

use App\Data\Edge\EdgeCaptureBatchData;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Validator;
use JsonException;

class StoreEdgeCaptureBatchRequest extends FormRequest
{
    private ?array $decodedMetadata = null;

    public function authorize(): bool
    {
        return $this->attributes->has('trackvision.edge_node');
    }

    public function rules(): array
    {
        return [
            'metadata' => ['required', 'string'],
            'files' => ['sometimes', 'array'],
        ];
    }

    public function after(): array
    {
        return [
            function (Validator $validator): void {
                try {
                    $this->decodedMetadata = json_decode((string) $this->input('metadata'), true, 512, JSON_THROW_ON_ERROR);
                } catch (JsonException) {
                    $validator->errors()->add('metadata', 'The metadata field must contain valid JSON.');
                    return;
                }

                foreach (['batch_uuid', 'sent_at', 'items'] as $field) {
                    if (! array_key_exists($field, $this->decodedMetadata)) {
                        $validator->errors()->add('metadata', "The metadata.$field field is required.");
                    }
                }

                if ($validator->errors()->any()) {
                    return;
                }

                if (! array_key_exists('items', $this->decodedMetadata) || ! is_array($this->decodedMetadata['items'])) {
                    $validator->errors()->add('metadata', 'The metadata.items field must be an array.');
                    return;
                }

                foreach ($this->decodedMetadata['items'] as $index => $item) {
                    foreach (['capture_uuid', 'idempotency_key', 'vehicle_uuid', 'camera_pair_uuid', 'lpr_camera_uuid', 'support_camera_uuid', 'plate', 'plate_normalized', 'event_code', 'action', 'event_time', 'direction', 'status', 'load_status', 'raw_payload'] as $field) {
                        if (! array_key_exists($field, $item)) {
                            $validator->errors()->add('metadata', "The metadata.items.$index.$field field is required.");
                        }
                    }

                    foreach ((array_key_exists('media', $item) ? $item['media'] : []) as $mediaIndex => $media) {
                        foreach (['kind', 'field', 'sha256_hash', 'content_type', 'byte_size'] as $field) {
                            if (! array_key_exists($field, $media)) {
                                $validator->errors()->add('metadata', "The metadata.items.$index.media.$mediaIndex.$field field is required.");
                            }
                        }
                    }
                }
            },
        ];
    }

    public function batchData(): EdgeCaptureBatchData
    {
        return EdgeCaptureBatchData::fromArray($this->decodedMetadata ?: []);
    }
}
```

- [ ] **Step 5: Implement parent media storage action**

Create `app/Actions/Edge/StoreSyncedCaptureMediaAction.php`:

```php
<?php

namespace App\Actions\Edge;

use App\Data\Edge\EdgeCaptureMediaData;
use App\Enums\MediaAssetKind;
use App\Models\Camera;
use App\Models\CaptureEvent;
use App\Models\MediaAsset;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use RuntimeException;

class StoreSyncedCaptureMediaAction
{
    public function execute(CaptureEvent $capture, Camera $camera, EdgeCaptureMediaData $media, UploadedFile $file): MediaAsset
    {
        if ($media->contentType !== 'image/jpeg') {
            throw new RuntimeException('media_content_type_invalid');
        }

        $bytes = $file->get();

        if (hash('sha256', $bytes) !== $media->sha256Hash) {
            throw new RuntimeException('media_hash_mismatch');
        }

        if (strlen($bytes) !== $media->byteSize) {
            throw new RuntimeException('media_size_mismatch');
        }

        $path = sprintf('captures/%s/%s/%s.jpg', $capture->uuid, $media->kind, $media->sha256Hash);
        Storage::disk('local')->put($path, $bytes);

        return $capture->mediaAssets()->firstOrCreate(
            ['kind' => MediaAssetKind::from($media->kind), 'camera_id' => $camera->id],
            [
                'disk' => 'local',
                'path' => $path,
                'content_type' => $media->contentType,
                'sha256_hash' => $media->sha256Hash,
                'byte_size' => $media->byteSize,
                'captured_at' => $capture->event_time,
            ],
        );
    }
}
```

- [ ] **Step 6: Implement capture upsert action**

Create `app/Actions/Edge/UpsertEdgeCaptureAction.php`:

```php
<?php

namespace App\Actions\Edge;

use App\Data\Edge\EdgeBatchItemResultData;
use App\Data\Edge\EdgeCaptureBatchItemData;
use App\Enums\CameraPairDirection;
use App\Enums\CaptureStatus;
use App\Enums\LoadStatus;
use App\Models\Camera;
use App\Models\CameraPair;
use App\Models\CaptureEvent;
use App\Models\EdgeNode;
use App\Models\Vehicle;
use Illuminate\Http\UploadedFile;
use Throwable;

class UpsertEdgeCaptureAction
{
    public function __construct(private readonly StoreSyncedCaptureMediaAction $storeMedia) {}

    public function execute(EdgeNode $edgeNode, EdgeCaptureBatchItemData $item, array $uploadedFiles): EdgeBatchItemResultData
    {
        $existingCapture = CaptureEvent::query()->where('idempotency_key', $item->idempotencyKey)->first();

        if ($existingCapture && $existingCapture->uuid !== $item->captureUuid) {
            return new EdgeBatchItemResultData($item->captureUuid, $item->idempotencyKey, 'rejected', 'idempotency_conflict');
        }

        if ($existingCapture) {
            return new EdgeBatchItemResultData($item->captureUuid, $item->idempotencyKey, 'duplicate');
        }

        $vehicle = Vehicle::query()->where('uuid', $item->vehicleUuid)->where('is_active', true)->first();
        $pair = CameraPair::query()->where('uuid', $item->cameraPairUuid)->where('edge_node_id', $edgeNode->id)->where('is_active', true)->first();
        $lprCamera = Camera::query()->where('uuid', $item->lprCameraUuid)->where('edge_node_id', $edgeNode->id)->first();
        $supportCamera = Camera::query()->where('uuid', $item->supportCameraUuid)->where('edge_node_id', $edgeNode->id)->first();

        if (! $vehicle) {
            return new EdgeBatchItemResultData($item->captureUuid, $item->idempotencyKey, 'rejected', 'vehicle_not_active');
        }

        if (! $pair || ! $lprCamera || ! $supportCamera) {
            return new EdgeBatchItemResultData($item->captureUuid, $item->idempotencyKey, 'rejected', 'camera_pair_not_available_for_edge');
        }

        try {
            $capture = CaptureEvent::query()->create([
                'uuid' => $item->captureUuid,
                'camera_pair_id' => $pair->id,
                'edge_node_id' => $edgeNode->id,
                'vehicle_id' => $vehicle->id,
                'lpr_camera_id' => $lprCamera->id,
                'support_camera_id' => $supportCamera->id,
                'plate' => $item->plate,
                'plate_normalized' => $item->plateNormalized,
                'event_code' => $item->eventCode,
                'action' => $item->action,
                'event_time' => $item->eventTime,
                'lane' => $item->lane,
                'direction' => CameraPairDirection::from($item->direction),
                'status' => CaptureStatus::from($item->status),
                'load_status' => LoadStatus::from($item->loadStatus),
                'idempotency_key' => $item->idempotencyKey,
                'raw_payload' => $item->rawPayload,
            ]);

            foreach ($item->media as $media) {
                $file = data_get($uploadedFiles, $media->field);
                if (! $file instanceof UploadedFile) {
                    throw new \RuntimeException('media_file_missing');
                }

                $camera = $media->kind === 'lpr_image' ? $lprCamera : $supportCamera;
                $this->storeMedia->execute($capture, $camera, $media, $file);
            }
        } catch (Throwable $exception) {
            CaptureEvent::query()->where('uuid', $item->captureUuid)->delete();

            return new EdgeBatchItemResultData($item->captureUuid, $item->idempotencyKey, 'rejected', $exception->getMessage());
        }

        return new EdgeBatchItemResultData($item->captureUuid, $item->idempotencyKey, 'accepted');
    }
}
```

- [ ] **Step 7: Implement batch ingest action**

Create `app/Actions/Edge/IngestCaptureBatchAction.php`:

```php
<?php

namespace App\Actions\Edge;

use App\Data\Edge\EdgeCaptureBatchData;
use App\Enums\EdgeSyncBatchStatus;
use App\Models\EdgeNode;
use App\Models\EdgeSyncBatch;
use Illuminate\Support\Facades\DB;

class IngestCaptureBatchAction
{
    public function __construct(private readonly UpsertEdgeCaptureAction $upsertCapture) {}

    public function execute(EdgeNode $edgeNode, EdgeCaptureBatchData $batch, array $uploadedFiles): array
    {
        $accepted = [];
        $duplicates = [];
        $rejected = [];

        DB::transaction(function () use ($edgeNode, $batch, $uploadedFiles, &$accepted, &$duplicates, &$rejected): void {
            foreach ($batch->items as $item) {
                $result = $this->upsertCapture->execute($edgeNode, $item, $uploadedFiles)->toArray();

                match ($result['status']) {
                    'accepted' => $accepted[] = $result,
                    'duplicate' => $duplicates[] = $result,
                    default => $rejected[] = $result,
                };
            }

            EdgeSyncBatch::query()->updateOrCreate(
                ['uuid' => $batch->batchUuid],
                [
                    'edge_node_id' => $edgeNode->id,
                    'status' => $this->statusFor(count($accepted), count($duplicates), count($rejected)),
                    'items_count' => count($batch->items),
                    'accepted_count' => count($accepted),
                    'duplicate_count' => count($duplicates),
                    'rejected_count' => count($rejected),
                    'received_at' => now(),
                    'summary' => compact('accepted', 'duplicates', 'rejected'),
                ],
            );
        });

        return [
            'batch_uuid' => $batch->batchUuid,
            'accepted' => $accepted,
            'duplicates' => $duplicates,
            'rejected' => $rejected,
        ];
    }

    private function statusFor(int $acceptedCount, int $duplicateCount, int $rejectedCount): EdgeSyncBatchStatus
    {
        if ($rejectedCount > 0 && ($acceptedCount + $duplicateCount) === 0) {
            return EdgeSyncBatchStatus::Failed;
        }

        if ($rejectedCount > 0) {
            return EdgeSyncBatchStatus::Partial;
        }

        return EdgeSyncBatchStatus::Received;
    }
}
```

- [ ] **Step 8: Implement Resource, controller, and route**

Create `app/Http/Resources/Api/V1/Edge/EdgeCaptureBatchResultResource.php`:

```php
<?php

namespace App\Http\Resources\Api\V1\Edge;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class EdgeCaptureBatchResultResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return $this->resource;
    }
}
```

Create `app/Http/Controllers/Api/V1/Edge/CaptureIngestController.php`:

```php
<?php

namespace App\Http\Controllers\Api\V1\Edge;

use App\Actions\Edge\IngestCaptureBatchAction;
use App\Http\Controllers\Controller;
use App\Http\Requests\Api\V1\Edge\StoreEdgeCaptureBatchRequest;
use App\Http\Resources\Api\V1\Edge\EdgeCaptureBatchResultResource;
use App\Models\EdgeNode;

class CaptureIngestController extends Controller
{
    public function __invoke(StoreEdgeCaptureBatchRequest $request, IngestCaptureBatchAction $action): EdgeCaptureBatchResultResource
    {
        /** @var EdgeNode $edgeNode */
        $edgeNode = $request->attributes->get('trackvision.edge_node');

        return EdgeCaptureBatchResultResource::make(
            $action->execute($edgeNode, $request->batchData(), $request->allFiles()),
        );
    }
}
```

Modify `routes/api.php` inside the `EnsureActiveParentEdgeNode` group:

```php
Route::middleware(EnsureClientIsResourceOwner::using('captures:write'))
    ->post('/edge/captures/batch', CaptureIngestController::class);
```

- [ ] **Step 9: Run parent ingest tests**

Run:

```bash
php artisan test --filter=EdgeCaptureBatchIngestTest
php artisan route:list --path=api/v1/edge/captures
```

Expected: tests PASS and route list includes `POST api/v1/edge/captures/batch`.

- [ ] **Step 10: Commit Task 3**

Run:

```bash
git add routes/api.php app/Data/Edge app/Actions/Edge/IngestCaptureBatchAction.php app/Actions/Edge/UpsertEdgeCaptureAction.php app/Actions/Edge/StoreSyncedCaptureMediaAction.php app/Http/Controllers/Api/V1/Edge/CaptureIngestController.php app/Http/Requests/Api/V1/Edge/StoreEdgeCaptureBatchRequest.php app/Http/Resources/Api/V1/Edge/EdgeCaptureBatchResultResource.php tests/Feature/Edge/EdgeCaptureBatchIngestTest.php
git commit -m "feat: add parent capture batch ingest"
```

---

### Task 4: Edge Batch Serializer and Parent API Client

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Data/Edge/EdgeCaptureBatchResponseData.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/EdgeSync/EdgeCaptureBatchResponseParser.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/EdgeSync/EdgeCaptureBatchSerializer.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/EdgeSync/ParentApiClient.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/EdgeSync/EdgeCaptureBatchResponseParserTest.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/EdgeSync/EdgeCaptureBatchSerializerTest.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/EdgeSync/ParentApiClientTest.php`

**Interfaces:**

- Consumes: `EdgeOutboxMessage`, `CaptureEvent`, `MediaAsset`, `Storage`.
- Produces: `EdgeCaptureBatchSerializer::serialize(iterable $messages): array`.
- Produces: serialized shape `['metadata' => array, 'files' => array]`.
- Produces: `EdgeCaptureBatchResponseParser::parse(array $payload): EdgeCaptureBatchResponseData`.
- Produces: `ParentApiClient::bootstrap(): array`.
- Produces: `ParentApiClient::heartbeat(array $payload): array`.
- Produces: `ParentApiClient::sendCaptureBatch(array $metadata, array $files): EdgeCaptureBatchResponseData`.

- [ ] **Step 1: Write failing response parser test**

Create `tests/Unit/EdgeSync/EdgeCaptureBatchResponseParserTest.php`:

```php
<?php

namespace Tests\Unit\EdgeSync;

use App\Services\EdgeSync\EdgeCaptureBatchResponseParser;
use Tests\TestCase;

class EdgeCaptureBatchResponseParserTest extends TestCase
{
    public function test_parses_accepted_duplicates_and_rejected_items(): void
    {
        $response = (new EdgeCaptureBatchResponseParser)->parse([
            'data' => [
                'batch_uuid' => 'batch-uuid',
                'accepted' => [['capture_uuid' => 'cap-1', 'idempotency_key' => 'key-1', 'status' => 'accepted']],
                'duplicates' => [['capture_uuid' => 'cap-2', 'idempotency_key' => 'key-2', 'status' => 'duplicate']],
                'rejected' => [['capture_uuid' => 'cap-3', 'idempotency_key' => 'key-3', 'status' => 'rejected', 'reason' => 'vehicle_not_active']],
            ],
        ]);

        $this->assertSame('batch-uuid', $response->batchUuid);
        $this->assertSame(['key-1'], $response->acceptedKeys());
        $this->assertSame(['key-2'], $response->duplicateKeys());
        $this->assertSame('vehicle_not_active', $response->rejected[0]['reason']);
    }
}
```

- [ ] **Step 2: Write failing serializer test**

Create `tests/Unit/EdgeSync/EdgeCaptureBatchSerializerTest.php`:

```php
<?php

namespace Tests\Unit\EdgeSync;

use App\Enums\MediaAssetKind;
use App\Models\CaptureEvent;
use App\Models\EdgeOutboxMessage;
use App\Models\MediaAsset;
use App\Services\EdgeSync\EdgeCaptureBatchSerializer;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

class EdgeCaptureBatchSerializerTest extends TestCase
{
    use RefreshDatabase;

    public function test_serializes_outbox_capture_with_metadata_and_private_files(): void
    {
        Storage::fake('local');
        $capture = CaptureEvent::factory()->create(['uuid' => 'capture-uuid', 'idempotency_key' => 'edge-key-001']);
        Storage::disk('local')->put('captures/capture-uuid/lpr_image/hash.jpg', 'lpr-jpeg');
        MediaAsset::factory()->create([
            'capture_event_id' => $capture->id,
            'camera_id' => $capture->lpr_camera_id,
            'kind' => MediaAssetKind::LprImage,
            'path' => 'captures/capture-uuid/lpr_image/hash.jpg',
            'sha256_hash' => hash('sha256', 'lpr-jpeg'),
            'byte_size' => strlen('lpr-jpeg'),
        ]);
        $message = EdgeOutboxMessage::factory()->create([
            'aggregate_uuid' => $capture->uuid,
            'idempotency_key' => $capture->idempotency_key,
        ]);

        $serialized = (new EdgeCaptureBatchSerializer)->serialize(collect([$message]));

        $this->assertSame('edge-key-001', $serialized['metadata']['items'][0]['idempotency_key']);
        $this->assertSame('files.capture-uuid.lpr_image', $serialized['metadata']['items'][0]['media'][0]['field']);
        $this->assertArrayHasKey('capture-uuid', $serialized['files']);
        $this->assertArrayHasKey('lpr_image', $serialized['files']['capture-uuid']);
    }
}
```

- [ ] **Step 3: Write failing parent API client test**

Create `tests/Unit/EdgeSync/ParentApiClientTest.php`:

```php
<?php

namespace Tests\Unit\EdgeSync;

use App\Services\EdgeSync\EdgeCaptureBatchResponseParser;
use App\Services\EdgeSync\ParentApiClient;
use Illuminate\Support\Facades\Http;
use Tests\TestCase;

class ParentApiClientTest extends TestCase
{
    public function test_client_authenticates_and_calls_bootstrap_with_edge_header(): void
    {
        config([
            'trackvision.parent_api_url' => 'https://parent.test',
            'trackvision.edge_node_uuid' => 'edge-uuid',
            'trackvision.parent_oauth_client_id' => 'client-id',
            'trackvision.parent_oauth_client_secret' => 'client-secret',
        ]);

        Http::fake([
            'parent.test/oauth/token' => Http::response(['access_token' => 'token-123'], 200),
            'parent.test/api/v1/edge/bootstrap' => Http::response(['data' => ['vehicles' => []]], 200),
        ]);

        $payload = $this->client()->bootstrap();

        $this->assertSame([], $payload['data']['vehicles']);
        Http::assertSent(fn ($request): bool => $request->url() === 'https://parent.test/api/v1/edge/bootstrap'
            && $request->hasHeader('X-TrackVision-Edge-Node', 'edge-uuid')
            && $request->hasHeader('Authorization', 'Bearer token-123'));
    }

    public function test_client_sends_capture_batch_as_multipart(): void
    {
        config([
            'trackvision.parent_api_url' => 'https://parent.test',
            'trackvision.edge_node_uuid' => 'edge-uuid',
            'trackvision.parent_oauth_client_id' => 'client-id',
            'trackvision.parent_oauth_client_secret' => 'client-secret',
        ]);

        Http::fake([
            'parent.test/oauth/token' => Http::response(['access_token' => 'token-123'], 200),
            'parent.test/api/v1/edge/captures/batch' => Http::response([
                'data' => ['batch_uuid' => 'batch-uuid', 'accepted' => [], 'duplicates' => [], 'rejected' => []],
            ], 200),
        ]);

        $response = $this->client()->sendCaptureBatch(['batch_uuid' => 'batch-uuid', 'items' => []], []);

        $this->assertSame('batch-uuid', $response->batchUuid);
        Http::assertSent(fn ($request): bool => $request->url() === 'https://parent.test/api/v1/edge/captures/batch'
            && $request->hasHeader('X-TrackVision-Edge-Node', 'edge-uuid'));
    }

    private function client(): ParentApiClient
    {
        return new ParentApiClient(new EdgeCaptureBatchResponseParser);
    }
}
```

- [ ] **Step 4: Run serializer/client tests and verify they fail**

Run:

```bash
php artisan test --filter=EdgeCaptureBatchResponseParserTest
php artisan test --filter=EdgeCaptureBatchSerializerTest
php artisan test --filter=ParentApiClientTest
```

Expected: FAIL because edge sync services and response DTO are absent.

- [ ] **Step 5: Implement response DTO and parser**

Create `app/Data/Edge/EdgeCaptureBatchResponseData.php`:

```php
<?php

namespace App\Data\Edge;

readonly class EdgeCaptureBatchResponseData
{
    public function __construct(
        public string $batchUuid,
        public array $accepted,
        public array $duplicates,
        public array $rejected,
    ) {}

    public function acceptedKeys(): array
    {
        return array_values(array_map(fn (array $item): string => $item['idempotency_key'], $this->accepted));
    }

    public function duplicateKeys(): array
    {
        return array_values(array_map(fn (array $item): string => $item['idempotency_key'], $this->duplicates));
    }
}
```

Create `app/Services/EdgeSync/EdgeCaptureBatchResponseParser.php`:

```php
<?php

namespace App\Services\EdgeSync;

use App\Data\Edge\EdgeCaptureBatchResponseData;

class EdgeCaptureBatchResponseParser
{
    public function parse(array $payload): EdgeCaptureBatchResponseData
    {
        $data = $payload['data'];

        return new EdgeCaptureBatchResponseData(
            batchUuid: (string) $data['batch_uuid'],
            accepted: array_key_exists('accepted', $data) ? $data['accepted'] : [],
            duplicates: array_key_exists('duplicates', $data) ? $data['duplicates'] : [],
            rejected: array_key_exists('rejected', $data) ? $data['rejected'] : [],
        );
    }
}
```

- [ ] **Step 6: Implement batch serializer**

Create `app/Services/EdgeSync/EdgeCaptureBatchSerializer.php`:

```php
<?php

namespace App\Services\EdgeSync;

use App\Models\CaptureEvent;
use App\Models\EdgeOutboxMessage;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class EdgeCaptureBatchSerializer
{
    public function serialize(iterable $messages): array
    {
        $batchUuid = (string) Str::uuid();
        $files = [];
        $items = Collection::make($messages)
            ->map(function (EdgeOutboxMessage $message) use (&$files): array {
                $capture = CaptureEvent::query()
                    ->with(['vehicle', 'cameraPair', 'lprCamera', 'supportCamera', 'mediaAssets'])
                    ->where('uuid', $message->aggregate_uuid)
                    ->firstOrFail();

                $media = $capture->mediaAssets->map(function ($asset) use (&$files, $capture): array {
                    $files[$capture->uuid][$asset->kind->value] = Storage::disk($asset->disk)->path($asset->path);

                    return [
                        'kind' => $asset->kind->value,
                        'field' => 'files.'.$capture->uuid.'.'.$asset->kind->value,
                        'sha256_hash' => $asset->sha256_hash,
                        'content_type' => $asset->content_type,
                        'byte_size' => $asset->byte_size,
                    ];
                })->values()->all();

                return [
                    'capture_uuid' => $capture->uuid,
                    'idempotency_key' => $message->idempotency_key,
                    'vehicle_uuid' => $capture->vehicle?->uuid,
                    'camera_pair_uuid' => $capture->cameraPair->uuid,
                    'lpr_camera_uuid' => $capture->lprCamera->uuid,
                    'support_camera_uuid' => $capture->supportCamera->uuid,
                    'plate' => $capture->plate,
                    'plate_normalized' => $capture->plate_normalized,
                    'event_code' => $capture->event_code,
                    'action' => $capture->action,
                    'event_time' => $capture->event_time?->toISOString(),
                    'lane' => $capture->lane,
                    'direction' => $capture->direction?->value,
                    'status' => $capture->status?->value,
                    'load_status' => $capture->load_status?->value,
                    'raw_payload' => $capture->raw_payload ?: [],
                    'media' => $media,
                ];
            })
            ->values()
            ->all();

        return [
            'metadata' => [
                'batch_uuid' => $batchUuid,
                'sent_at' => now()->toISOString(),
                'items' => $items,
            ],
            'files' => $files,
        ];
    }
}
```

- [ ] **Step 7: Implement parent API client**

Create `app/Services/EdgeSync/ParentApiClient.php`:

```php
<?php

namespace App\Services\EdgeSync;

use App\Data\Edge\EdgeCaptureBatchResponseData;
use Illuminate\Http\Client\PendingRequest;
use Illuminate\Support\Facades\Http;
use RuntimeException;

class ParentApiClient
{
    public function __construct(private readonly EdgeCaptureBatchResponseParser $responseParser) {}

    public function bootstrap(): array
    {
        return $this->authorized()->get($this->url('/api/v1/edge/bootstrap'))->throw()->json();
    }

    public function heartbeat(array $payload): array
    {
        return $this->authorized()->post($this->url('/api/v1/edge/heartbeat'), $payload)->throw()->json();
    }

    public function sendCaptureBatch(array $metadata, array $files): EdgeCaptureBatchResponseData
    {
        $request = $this->authorized()->asMultipart();

        foreach ($files as $captureUuid => $filesByKind) {
            foreach ($filesByKind as $kind => $path) {
                $request->attach("files[$captureUuid][$kind]", fopen($path, 'r'), basename($path), ['Content-Type' => 'image/jpeg']);
            }
        }

        return $this->responseParser->parse(
            $request->post($this->url('/api/v1/edge/captures/batch'), [
                'metadata' => json_encode($metadata, JSON_THROW_ON_ERROR),
            ])->throw()->json(),
        );
    }

    private function authorized(): PendingRequest
    {
        return Http::timeout((int) config('trackvision.sync_timeout_seconds', 30))
            ->connectTimeout((int) config('trackvision.sync_connect_timeout_seconds', 5))
            ->withToken($this->accessToken())
            ->withHeaders(['X-TrackVision-Edge-Node' => (string) config('trackvision.edge_node_uuid')]);
    }

    private function accessToken(): string
    {
        $response = Http::asForm()->post($this->url('/oauth/token'), [
            'grant_type' => 'client_credentials',
            'client_id' => config('trackvision.parent_oauth_client_id'),
            'client_secret' => config('trackvision.parent_oauth_client_secret'),
        ])->throw()->json();

        $token = array_key_exists('access_token', $response) ? $response['access_token'] : null;

        if (! is_string($token) || $token === '') {
            throw new RuntimeException('parent_access_token_missing');
        }

        return $token;
    }

    private function url(string $path): string
    {
        return rtrim((string) config('trackvision.parent_api_url'), '/').$path;
    }
}
```

- [ ] **Step 8: Run serializer/client tests**

Run:

```bash
php artisan test --filter=EdgeCaptureBatchResponseParserTest
php artisan test --filter=EdgeCaptureBatchSerializerTest
php artisan test --filter=ParentApiClientTest
```

Expected: PASS.

- [ ] **Step 9: Commit Task 4**

Run:

```bash
git add app/Data/Edge/EdgeCaptureBatchResponseData.php app/Services/EdgeSync tests/Unit/EdgeSync
git commit -m "feat: add edge parent sync client"
```

---

### Task 5: Edge Sync Command, State, and Outbox Transitions

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/SyncFromParentAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/SendEdgeHeartbeatAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/PushOutboxBatchAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/EdgeSync/EdgeOutboxBackoff.php`
- Create: `RIALMA-TrackVision-Backend/app/Console/Commands/EdgeSyncParentCommand.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/EdgeSync/EdgeOutboxBackoffTest.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Edge/EdgeSyncParentCommandTest.php`

**Interfaces:**

- Consumes: `ParentApiClient`, `EdgeCaptureBatchSerializer`, `EdgeCaptureBatchResponseData`, `EdgeOutboxMessage`, `EdgeSyncState`.
- Produces: `EdgeOutboxBackoff::availableAt(int $attempts): CarbonImmutable`.
- Produces: `SyncFromParentAction::execute(): void`.
- Produces: `SendEdgeHeartbeatAction::execute(): void`.
- Produces: `PushOutboxBatchAction::execute(): int`.
- Produces command `edge:sync-parent`.

- [ ] **Step 1: Write failing backoff test**

Create `tests/Unit/EdgeSync/EdgeOutboxBackoffTest.php`:

```php
<?php

namespace Tests\Unit\EdgeSync;

use App\Services\EdgeSync\EdgeOutboxBackoff;
use Illuminate\Support\Carbon;
use Tests\TestCase;

class EdgeOutboxBackoffTest extends TestCase
{
    public function test_backoff_doubles_minutes_until_sixty_minute_cap(): void
    {
        Carbon::setTestNow('2026-08-12 15:00:00');
        $backoff = new EdgeOutboxBackoff;

        $this->assertSame('2026-08-12 15:02:00', $backoff->availableAt(1)->format('Y-m-d H:i:s'));
        $this->assertSame('2026-08-12 16:00:00', $backoff->availableAt(10)->format('Y-m-d H:i:s'));
    }
}
```

- [ ] **Step 2: Write failing edge command tests**

Create `tests/Feature/Edge/EdgeSyncParentCommandTest.php`:

```php
<?php

namespace Tests\Feature\Edge;

use App\Enums\EdgeOutboxStatus;
use App\Models\CaptureEvent;
use App\Models\EdgeOutboxMessage;
use App\Models\EdgeSyncState;
use App\Services\EdgeSync\ParentApiClient;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Mockery;
use Tests\TestCase;

class EdgeSyncParentCommandTest extends TestCase
{
    use RefreshDatabase;

    public function test_command_rejects_parent_profile(): void
    {
        config(['trackvision.node_role' => 'parent']);

        $this->artisan('edge:sync-parent')
            ->expectsOutput('This command can only run on an edge node.')
            ->assertExitCode(1);
    }

    public function test_command_bootstraps_heartbeats_and_marks_accepted_outbox_as_synced(): void
    {
        config(['trackvision.node_role' => 'edge', 'trackvision.edge_node_uuid' => 'edge-uuid']);
        $capture = CaptureEvent::factory()->create(['idempotency_key' => 'edge-key-001']);
        EdgeOutboxMessage::factory()->create([
            'aggregate_uuid' => $capture->uuid,
            'idempotency_key' => 'edge-key-001',
            'status' => EdgeOutboxStatus::Pending,
            'available_at' => now(),
        ]);

        $client = Mockery::mock(ParentApiClient::class);
        $client->shouldReceive('bootstrap')->once()->andReturn(['data' => ['vehicles' => []]]);
        $client->shouldReceive('heartbeat')->once()->andReturn(['data' => ['edge_node' => ['uuid' => 'edge-uuid']]]);
        $client->shouldReceive('sendCaptureBatch')->once()->andReturn(new \App\Data\Edge\EdgeCaptureBatchResponseData(
            batchUuid: 'batch-uuid',
            accepted: [['capture_uuid' => $capture->uuid, 'idempotency_key' => 'edge-key-001', 'status' => 'accepted']],
            duplicates: [],
            rejected: [],
        ));
        $this->app->instance(ParentApiClient::class, $client);

        $this->artisan('edge:sync-parent')
            ->expectsOutput('Edge sync completed.')
            ->assertExitCode(0);

        $this->assertDatabaseHas('edge_outbox_messages', [
            'idempotency_key' => 'edge-key-001',
            'status' => 'synced',
        ]);
        $this->assertNotNull(EdgeSyncState::where('edge_node_uuid', 'edge-uuid')->first()?->last_successful_push_at);
    }

    public function test_parent_failure_keeps_outbox_pending_and_schedules_retry(): void
    {
        config(['trackvision.node_role' => 'edge', 'trackvision.edge_node_uuid' => 'edge-uuid']);
        $capture = CaptureEvent::factory()->create(['idempotency_key' => 'edge-key-001']);
        EdgeOutboxMessage::factory()->create([
            'aggregate_uuid' => $capture->uuid,
            'idempotency_key' => 'edge-key-001',
            'status' => EdgeOutboxStatus::Pending,
            'attempts' => 0,
            'available_at' => now(),
        ]);

        $client = Mockery::mock(ParentApiClient::class);
        $client->shouldReceive('bootstrap')->once()->andThrow(new \RuntimeException('network_down'));
        $this->app->instance(ParentApiClient::class, $client);

        $this->artisan('edge:sync-parent')
            ->expectsOutput('Edge sync failed: network_down')
            ->assertExitCode(1);

        $message = EdgeOutboxMessage::where('idempotency_key', 'edge-key-001')->firstOrFail();
        $this->assertSame(EdgeOutboxStatus::Pending, $message->status);
        $this->assertSame(1, $message->attempts);
        $this->assertSame('network_down', $message->last_error);
        $this->assertNotNull($message->available_at);
    }

    public function test_rejected_items_are_marked_failed(): void
    {
        config(['trackvision.node_role' => 'edge', 'trackvision.edge_node_uuid' => 'edge-uuid']);
        $capture = CaptureEvent::factory()->create(['idempotency_key' => 'edge-key-001']);
        EdgeOutboxMessage::factory()->create([
            'aggregate_uuid' => $capture->uuid,
            'idempotency_key' => 'edge-key-001',
            'status' => EdgeOutboxStatus::Pending,
            'available_at' => now(),
        ]);

        $client = Mockery::mock(ParentApiClient::class);
        $client->shouldReceive('bootstrap')->once()->andReturn(['data' => ['vehicles' => []]]);
        $client->shouldReceive('heartbeat')->once()->andReturn(['data' => ['edge_node' => ['uuid' => 'edge-uuid']]]);
        $client->shouldReceive('sendCaptureBatch')->once()->andReturn(new \App\Data\Edge\EdgeCaptureBatchResponseData(
            batchUuid: 'batch-uuid',
            accepted: [],
            duplicates: [],
            rejected: [['capture_uuid' => $capture->uuid, 'idempotency_key' => 'edge-key-001', 'status' => 'rejected', 'reason' => 'vehicle_not_active']],
        ));
        $this->app->instance(ParentApiClient::class, $client);

        $this->artisan('edge:sync-parent')->assertExitCode(0);

        $this->assertDatabaseHas('edge_outbox_messages', [
            'idempotency_key' => 'edge-key-001',
            'status' => 'failed',
            'last_error' => 'vehicle_not_active',
        ]);
    }
}
```

- [ ] **Step 3: Run command tests and verify they fail**

Run:

```bash
php artisan test --filter=EdgeOutboxBackoffTest
php artisan test --filter=EdgeSyncParentCommandTest
```

Expected: FAIL because edge command actions and backoff service are absent.

- [ ] **Step 4: Implement backoff service**

Create `app/Services/EdgeSync/EdgeOutboxBackoff.php`:

```php
<?php

namespace App\Services\EdgeSync;

use Carbon\CarbonImmutable;

class EdgeOutboxBackoff
{
    public function availableAt(int $attempts): CarbonImmutable
    {
        $minutes = min((int) config('trackvision.sync_retry_max_minutes', 60), 2 ** max(1, $attempts));

        return CarbonImmutable::now()->addMinutes($minutes);
    }
}
```

- [ ] **Step 5: Implement bootstrap sync action**

Create `app/Actions/Edge/SyncFromParentAction.php`:

```php
<?php

namespace App\Actions\Edge;

use App\Models\EdgeSyncState;
use App\Services\EdgeSync\ParentApiClient;

class SyncFromParentAction
{
    public function __construct(private readonly ParentApiClient $client) {}

    public function execute(): void
    {
        $payload = $this->client->bootstrap();
        $encoded = json_encode($payload, JSON_THROW_ON_ERROR);

        EdgeSyncState::query()->updateOrCreate(
            ['edge_node_uuid' => (string) config('trackvision.edge_node_uuid')],
            [
                'last_bootstrap_at' => now(),
                'bootstrap_payload_hash' => hash('sha256', $encoded),
                'last_error' => null,
            ],
        );
    }
}
```

- [ ] **Step 6: Implement heartbeat action**

Create `app/Actions/Edge/SendEdgeHeartbeatAction.php`:

```php
<?php

namespace App\Actions\Edge;

use App\Enums\EdgeOutboxStatus;
use App\Models\CaptureEvent;
use App\Models\EdgeOutboxMessage;
use App\Services\EdgeSync\ParentApiClient;

class SendEdgeHeartbeatAction
{
    public function __construct(private readonly ParentApiClient $client) {}

    public function execute(): void
    {
        $this->client->heartbeat([
            'status' => 'online',
            'local_time' => now()->toISOString(),
            'pending_outbox_count' => EdgeOutboxMessage::query()->where('status', EdgeOutboxStatus::Pending)->count(),
            'last_capture_at' => CaptureEvent::query()->max('event_time'),
            'disk_free_mb' => null,
        ]);
    }
}
```

- [ ] **Step 7: Implement outbox push action**

Create `app/Actions/Edge/PushOutboxBatchAction.php`:

```php
<?php

namespace App\Actions\Edge;

use App\Enums\EdgeOutboxStatus;
use App\Models\EdgeOutboxMessage;
use App\Models\EdgeSyncState;
use App\Services\EdgeSync\EdgeCaptureBatchSerializer;
use App\Services\EdgeSync\EdgeOutboxBackoff;
use App\Services\EdgeSync\ParentApiClient;

class PushOutboxBatchAction
{
    public function __construct(
        private readonly EdgeCaptureBatchSerializer $serializer,
        private readonly ParentApiClient $client,
        private readonly EdgeOutboxBackoff $backoff,
    ) {}

    public function execute(): int
    {
        $messages = EdgeOutboxMessage::query()
            ->where('status', EdgeOutboxStatus::Pending)
            ->where(fn ($query) => $query->whereNull('available_at')->orWhere('available_at', '<=', now()))
            ->orderBy('id')
            ->limit((int) config('trackvision.sync_batch_size', 100))
            ->get();

        if ($messages->isEmpty()) {
            return 0;
        }

        $serialized = $this->serializer->serialize($messages);
        $response = $this->client->sendCaptureBatch($serialized['metadata'], $serialized['files']);
        $acceptedKeys = array_merge($response->acceptedKeys(), $response->duplicateKeys());

        EdgeOutboxMessage::query()
            ->whereIn('idempotency_key', $acceptedKeys)
            ->update([
                'status' => EdgeOutboxStatus::Synced,
                'synced_at' => now(),
                'last_error' => null,
            ]);

        foreach ($response->rejected as $item) {
            EdgeOutboxMessage::query()
                ->where('idempotency_key', $item['idempotency_key'])
                ->update([
                    'status' => EdgeOutboxStatus::Failed,
                    'last_error' => array_key_exists('reason', $item) ? $item['reason'] : 'rejected_by_parent',
                ]);
        }

        EdgeSyncState::query()->updateOrCreate(
            ['edge_node_uuid' => (string) config('trackvision.edge_node_uuid')],
            ['last_push_at' => now(), 'last_successful_push_at' => now(), 'last_error' => null],
        );

        return count($acceptedKeys);
    }

    public function scheduleFailure(string $error): void
    {
        EdgeOutboxMessage::query()
            ->where('status', EdgeOutboxStatus::Pending)
            ->where(fn ($query) => $query->whereNull('available_at')->orWhere('available_at', '<=', now()))
            ->orderBy('id')
            ->limit((int) config('trackvision.sync_batch_size', 100))
            ->get()
            ->each(function (EdgeOutboxMessage $message) use ($error): void {
                $attempts = $message->attempts + 1;
                $message->forceFill([
                    'attempts' => $attempts,
                    'available_at' => $this->backoff->availableAt($attempts),
                    'last_error' => $error,
                ])->save();
            });

        EdgeSyncState::query()->updateOrCreate(
            ['edge_node_uuid' => (string) config('trackvision.edge_node_uuid')],
            ['last_push_at' => now(), 'last_error' => $error],
        );
    }
}
```

- [ ] **Step 8: Implement sync command**

Create `app/Console/Commands/EdgeSyncParentCommand.php`:

```php
<?php

namespace App\Console\Commands;

use App\Actions\Edge\PushOutboxBatchAction;
use App\Actions\Edge\SendEdgeHeartbeatAction;
use App\Actions\Edge\SyncFromParentAction;
use App\Enums\NodeRole;
use App\Support\NodeRoleResolver;
use Illuminate\Console\Command;
use Throwable;

class EdgeSyncParentCommand extends Command
{
    protected $signature = 'edge:sync-parent';
    protected $description = 'Synchronize local edge state and capture outbox with the parent server.';

    public function handle(
        SyncFromParentAction $syncFromParent,
        SendEdgeHeartbeatAction $sendHeartbeat,
        PushOutboxBatchAction $pushOutbox,
    ): int {
        if (NodeRoleResolver::current() !== NodeRole::Edge) {
            $this->error('This command can only run on an edge node.');
            return self::FAILURE;
        }

        try {
            $syncFromParent->execute();
            $sendHeartbeat->execute();
            $pushOutbox->execute();
        } catch (Throwable $exception) {
            $pushOutbox->scheduleFailure($exception->getMessage());
            $this->error('Edge sync failed: '.$exception->getMessage());
            return self::FAILURE;
        }

        $this->info('Edge sync completed.');

        return self::SUCCESS;
    }
}
```

- [ ] **Step 9: Run command tests**

Run:

```bash
php artisan test --filter=EdgeOutboxBackoffTest
php artisan test --filter=EdgeSyncParentCommandTest
```

Expected: PASS.

- [ ] **Step 10: Commit Task 5**

Run:

```bash
git add app/Actions/Edge/SyncFromParentAction.php app/Actions/Edge/SendEdgeHeartbeatAction.php app/Actions/Edge/PushOutboxBatchAction.php app/Services/EdgeSync/EdgeOutboxBackoff.php app/Console/Commands/EdgeSyncParentCommand.php tests/Unit/EdgeSync/EdgeOutboxBackoffTest.php tests/Feature/Edge/EdgeSyncParentCommandTest.php
git commit -m "feat: add edge parent sync command"
```

---

### Task 6: API Documentation, Full Verification, and Operational Handoff

**Files:**

- Modify: `RIALMA-TrackVision-Backend/README.md`
- Create: `RIALMA-TrackVision-Backend/docs/api-edge-sync.md`
- Modify: `RIALMA-TrackVision-AgentOps/docs/superpowers/plans/2026-08-12-trackvision-system-implementation.md`

**Interfaces:**

- Consumes: implemented `GET /api/v1/edge/bootstrap`.
- Consumes: implemented `POST /api/v1/edge/heartbeat`.
- Consumes: implemented `POST /api/v1/edge/captures/batch`.
- Consumes: implemented command `edge:sync-parent`.
- Produces: backend docs for env vars, scheduling, endpoint contracts and verification commands.

- [ ] **Step 1: Add backend sync docs**

Create `RIALMA-TrackVision-Backend/docs/api-edge-sync.md`:

```markdown
# Edge Sync API

## Authentication

Use Laravel Passport client credentials.

Required header:

```text
X-TrackVision-Edge-Node: {edge_node_uuid}
```

Required scopes:

- `edge:read` for `GET /api/v1/edge/bootstrap`.
- `edge:write` for `POST /api/v1/edge/heartbeat`.
- `captures:write` for `POST /api/v1/edge/captures/batch`.

## Bootstrap

```text
GET /api/v1/edge/bootstrap
```

Returns active vehicles, active cameras for the edge node, active camera pairs for the edge node and sync settings. Camera passwords are never returned.

## Heartbeat

```text
POST /api/v1/edge/heartbeat
```

Payload:

```json
{
  "status": "online",
  "local_time": "2026-08-12T15:00:00Z",
  "pending_outbox_count": 12,
  "last_capture_at": "2026-08-12T14:55:00Z",
  "disk_free_mb": 51200
}
```

## Capture Batch

```text
POST /api/v1/edge/captures/batch
```

Use `multipart/form-data`.

Parts:

- `metadata`: JSON batch metadata.
- `files[{capture_uuid}][lpr_image]`: JPEG.
- `files[{capture_uuid}][support_image]`: JPEG.

The parent returns `accepted`, `duplicates`, and `rejected`. The edge marks outbox as synced only for `accepted` and `duplicates`.
```

- [ ] **Step 2: Update backend README with sync environment and commands**

Add to `RIALMA-TrackVision-Backend/README.md`:

```markdown
## Edge Parent Sync

Required edge environment variables:

```env
TRACKVISION_NODE_ROLE=edge
TRACKVISION_PARENT_API_URL=
TRACKVISION_EDGE_NODE_UUID=
TRACKVISION_PARENT_OAUTH_CLIENT_ID=
TRACKVISION_PARENT_OAUTH_CLIENT_SECRET=
TRACKVISION_SYNC_BATCH_SIZE=100
TRACKVISION_SYNC_CONNECT_TIMEOUT_SECONDS=5
TRACKVISION_SYNC_TIMEOUT_SECONDS=30
TRACKVISION_SYNC_RETRY_MAX_MINUTES=60
```

Command:

```bash
php artisan edge:sync-parent
```

Focused verification:

```bash
php artisan test --filter=EdgeSync
php artisan test --filter=EdgeCaptureBatchIngestTest
php artisan test --filter=EdgeSyncParentCommandTest
php artisan route:list --path=api/v1/edge
```
```

- [ ] **Step 3: Update AgentOps master plan**

In `RIALMA-TrackVision-AgentOps/docs/superpowers/plans/2026-08-12-trackvision-system-implementation.md`, keep Task 8 pointing to both the design and this implementation plan:

```markdown
**Design Source:** `docs/superpowers/specs/2026-08-12-edge-parent-synchronization-design.md`

**Implementation Plan:** `docs/superpowers/plans/2026-08-12-edge-parent-synchronization-implementation.md`
```

- [ ] **Step 4: Run focused backend verification**

Run in `RIALMA-TrackVision-Backend`:

```bash
php artisan test --filter=EdgeSyncSchemaTest
php artisan test --filter=EdgeSyncBootstrapTest
php artisan test --filter=EdgeHeartbeatTest
php artisan test --filter=EdgeCaptureBatchIngestTest
php artisan test --filter=EdgeCaptureBatchResponseParserTest
php artisan test --filter=EdgeCaptureBatchSerializerTest
php artisan test --filter=ParentApiClientTest
php artisan test --filter=EdgeOutboxBackoffTest
php artisan test --filter=EdgeSyncParentCommandTest
```

Expected: all focused tests PASS.

- [ ] **Step 5: Run full backend verification**

Run in `RIALMA-TrackVision-Backend`:

```bash
composer validate
php artisan test
php artisan route:list --path=api/v1/edge
```

Expected: `composer validate` returns success, all backend tests pass, and route list includes bootstrap, heartbeat, capture batch, and Intelbras webhook routes.

- [ ] **Step 6: Check repositories before commits**

Run:

```bash
git -C C:/projetos/rialma/RIALMA-TrackVision-AgentOps/RIALMA-TrackVision-Backend status --short --branch
git -C C:/projetos/rialma/RIALMA-TrackVision-AgentOps status --short --branch
```

Expected: backend shows only implementation files for this sync phase; AgentOps shows only documentation updates for this sync phase.

- [ ] **Step 7: Commit backend implementation docs**

Run in `RIALMA-TrackVision-Backend`:

```bash
git add README.md docs/api-edge-sync.md
git commit -m "docs: add edge sync api guide"
```

- [ ] **Step 8: Commit AgentOps documentation**

Run in `RIALMA-TrackVision-AgentOps`:

```bash
git add docs/superpowers/plans/2026-08-12-trackvision-system-implementation.md docs/superpowers/plans/2026-08-12-edge-parent-synchronization-implementation.md
git commit -m "docs: add edge parent sync implementation plan"
```

---

## Verification Matrix

Backend focused checks:

```bash
php artisan test --filter=EdgeSyncSchemaTest
php artisan test --filter=EdgeSyncBootstrapTest
php artisan test --filter=EdgeHeartbeatTest
php artisan test --filter=EdgeCaptureBatchIngestTest
php artisan test --filter=EdgeCaptureBatchResponseParserTest
php artisan test --filter=EdgeCaptureBatchSerializerTest
php artisan test --filter=ParentApiClientTest
php artisan test --filter=EdgeOutboxBackoffTest
php artisan test --filter=EdgeSyncParentCommandTest
```

Backend full checks:

```bash
composer validate
php artisan test
php artisan route:list --path=api/v1/edge
```

Manual edge schedule after deployment:

```bash
php artisan edge:sync-parent
```

Expected command result when parent is reachable:

```text
Edge sync completed.
```

Expected command result when parent is unreachable:

```text
Edge sync failed: network_down
```

Operational acceptance:

- Edge calls bootstrap with `edge:read`.
- Edge sends heartbeat with `edge:write`.
- Edge sends capture batch with `captures:write`.
- Parent rejects missing or inactive edge node header.
- Parent stores LPR and support images privately.
- Re-sending the same capture returns duplicate without creating another `capture_events` row.
- Internet failure keeps outbox pending and schedules retry.
- Accepted and duplicate items become synced.
- Rejected items become failed with `last_error`.

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-08-12-edge-parent-synchronization-implementation.md`.

Two execution options:

1. **Subagent-Driven (recommended)** - dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** - execute tasks in this session using `superpowers:executing-plans`, batch execution with checkpoints.

Which approach?
