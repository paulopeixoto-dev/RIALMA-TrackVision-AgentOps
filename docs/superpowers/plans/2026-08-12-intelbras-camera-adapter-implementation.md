# Intelbras Camera Adapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implementar no backend Laravel o adaptador Intelbras para a VIP 5460 LPR IA com recebimento webhook-first, captura de snapshot da camera de apoio, persistencia local edge e fallback por assinatura de eventos.

**Architecture:** O endpoint edge recebe o webhook da camera, valida Basic Auth dedicado, delega parsing para services Intelbras e regra de negocio para Actions. O fluxo cria captura local, grava midias privadas, gera outbox idempotente e deixa o parent sync consumir os dados em fase posterior. A assinatura `snapManager.cgi?action=attachFileProc` usa os mesmos DTOs e Actions para evitar dois fluxos de dominio.

**Tech Stack:** Laravel 13, PHP 8.4, PostgreSQL, SQLite in-memory em testes, Laravel HTTP Client, Laravel Storage, PHPUnit, Laravel Passport ja existente, Spatie Permission ja existente.

## Global Constraints

- Codigo de backend deve ser alterado somente em `RIALMA-TrackVision-Backend`.
- Documentacao agentica deve ser alterada somente em `RIALMA-TrackVision-AgentOps`.
- Seguir `docs/backend-laravel-guidelines.md`: Controllers magros, Form Requests, Eloquent eficiente, Services ou Actions para regra de negocio.
- O endpoint webhook Intelbras existe apenas no perfil `edge`.
- O webhook usa Basic Auth dedicado por `TRACKVISION_INTELBRAS_WEBHOOK_USERNAME` e `TRACKVISION_INTELBRAS_WEBHOOK_PASSWORD`.
- As credenciais administrativas das cameras continuam criptografadas em `cameras.password_encrypted`.
- Chamadas HTTP para cameras devem ter timeout obrigatorio vindo de `config('trackvision.camera_default_timeout_seconds')`.
- O evento LPR padrao e `TrafficJunction`.
- A camera de apoio deve usar snapshot HTTP padrao `/cgi-bin/snapshot.cgi`.
- Imagens LPR e apoio devem ser gravadas como midias privadas separadas.
- Placas desconhecidas nao entram no fluxo parent de viagens.
- Toda escrita local de captura deve ser idempotente.
- Nunca registrar senhas, Authorization header ou bytes de imagem em logs.

---

## File Structure

Backend code files:

```text
RIALMA-TrackVision-Backend/
|-- app/
|   |-- Actions/Edge/
|   |   |-- EnqueueCaptureForSyncAction.php
|   |   |-- HandleIntelbrasLprWebhookAction.php
|   |   |-- ProcessLprEventAction.php
|   |   `-- StoreCaptureMediaAction.php
|   |-- Console/Commands/
|   |   `-- EdgeListenIntelbrasCommand.php
|   |-- Data/Cameras/
|   |   |-- SnapshotData.php
|   |   `-- TrafficCaptureData.php
|   |-- Enums/
|   |   |-- CaptureStatus.php
|   |   |-- EdgeOutboxStatus.php
|   |   |-- LoadStatus.php
|   |   `-- MediaAssetKind.php
|   |-- Http/
|   |   |-- Controllers/Api/V1/Edge/
|   |   |   `-- IntelbrasLprWebhookController.php
|   |   |-- Middleware/
|   |   |   `-- EnsureTrackVisionEdgeNode.php
|   |   `-- Requests/Api/V1/Edge/
|   |       `-- IntelbrasLprWebhookRequest.php
|   |-- Models/
|   |   |-- CaptureEvent.php
|   |   |-- EdgeOutboxMessage.php
|   |   `-- MediaAsset.php
|   |-- Services/Cameras/Intelbras/
|   |   |-- IntelbrasEventStreamClient.php
|   |   |-- IntelbrasHttpClient.php
|   |   |-- IntelbrasSnapshotClient.php
|   |   |-- IntelbrasWebhookParser.php
|   |   `-- TrafficEventParser.php
|   `-- Support/Cameras/
|       `-- BuildIntelbrasEventIdempotencyKey.php
|-- config/trackvision.php
|-- database/
|   |-- factories/
|   |   |-- CaptureEventFactory.php
|   |   |-- EdgeOutboxMessageFactory.php
|   |   `-- MediaAssetFactory.php
|   `-- migrations/
|       |-- 2026_08_12_160001_create_capture_events_table.php
|       |-- 2026_08_12_160002_create_media_assets_table.php
|       `-- 2026_08_12_160003_create_edge_outbox_messages_table.php
|-- routes/api.php
`-- tests/
    |-- Feature/Edge/
    |   `-- IntelbrasLprWebhookTest.php
    `-- Unit/Cameras/Intelbras/
        |-- IntelbrasEventStreamClientTest.php
        |-- IntelbrasSnapshotClientTest.php
        |-- IntelbrasWebhookParserTest.php
        |-- TrafficCaptureDataTest.php
        `-- TrafficEventParserTest.php
```

AgentOps documentation files:

```text
RIALMA-TrackVision-AgentOps/
`-- docs/operations/
    `-- intelbras-vip-5460-field-checklist.md
```

---

### Task 1: Configuration, DTOs, Enums, and Idempotency

**Files:**

- Modify: `RIALMA-TrackVision-Backend/config/trackvision.php`
- Modify: `RIALMA-TrackVision-Backend/.env.example`
- Create: `RIALMA-TrackVision-Backend/app/Data/Cameras/TrafficCaptureData.php`
- Create: `RIALMA-TrackVision-Backend/app/Data/Cameras/SnapshotData.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/CaptureStatus.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/EdgeOutboxStatus.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/LoadStatus.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/MediaAssetKind.php`
- Create: `RIALMA-TrackVision-Backend/app/Support/Cameras/BuildIntelbrasEventIdempotencyKey.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/Cameras/Intelbras/TrafficCaptureDataTest.php`

**Interfaces:**

- Consumes: `App\Support\Vehicles\NormalizePlate::__invoke(string $plate): string`
- Consumes: `App\Models\CameraPair`
- Produces: `App\Data\Cameras\TrafficCaptureData`
- Produces: `TrafficCaptureData::hasLprImage(): bool`
- Produces: `App\Data\Cameras\SnapshotData`
- Produces: `BuildIntelbrasEventIdempotencyKey::__invoke(CameraPair $cameraPair, TrafficCaptureData $event): string`
- Produces config keys under `config('trackvision.intelbras')` and `config('trackvision.edge')`

- [ ] **Step 1: Write failing DTO and idempotency tests**

Create `tests/Unit/Cameras/Intelbras/TrafficCaptureDataTest.php`:

```php
<?php

namespace Tests\Unit\Cameras\Intelbras;

use App\Data\Cameras\SnapshotData;
use App\Data\Cameras\TrafficCaptureData;
use App\Enums\CameraPairDirection;
use App\Models\CameraPair;
use App\Support\Cameras\BuildIntelbrasEventIdempotencyKey;
use Carbon\CarbonImmutable;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class TrafficCaptureDataTest extends TestCase
{
    use RefreshDatabase;

    public function test_traffic_capture_data_knows_when_lpr_image_is_present(): void
    {
        $event = new TrafficCaptureData(
            eventCode: 'TrafficJunction',
            action: 'Pulse',
            plateNumber: 'ABC-1D23',
            plateNormalized: 'ABC1D23',
            eventTime: CarbonImmutable::parse('2026-08-12 15:00:00'),
            lane: 1,
            groupId: 'group-10',
            indexInGroup: 1,
            rawPayload: ['TrafficCar' => ['PlateNumber' => 'ABC-1D23']],
            lprImageBytes: 'jpeg-bytes',
            lprImageContentType: 'image/jpeg',
        );

        $this->assertTrue($event->hasLprImage());
    }

    public function test_snapshot_data_calculates_hash_and_size(): void
    {
        $snapshot = SnapshotData::fromBytes(
            bytes: 'support-jpeg',
            contentType: 'image/jpeg',
            capturedAt: CarbonImmutable::parse('2026-08-12 15:00:01'),
            sourceCameraUuid: 'camera-uuid',
        );

        $this->assertSame(hash('sha256', 'support-jpeg'), $snapshot->sha256Hash);
        $this->assertSame(strlen('support-jpeg'), $snapshot->byteSize);
    }

    public function test_idempotency_key_is_stable_for_same_event(): void
    {
        config(['trackvision.edge_node_uuid' => 'edge-uuid-01']);

        $cameraPair = CameraPair::factory()->create([
            'uuid' => '11111111-1111-4111-8111-111111111111',
            'direction' => CameraPairDirection::Outbound,
        ]);

        $event = new TrafficCaptureData(
            eventCode: 'TrafficJunction',
            action: 'Pulse',
            plateNumber: 'ABC-1D23',
            plateNormalized: 'ABC1D23',
            eventTime: CarbonImmutable::parse('2026-08-12 15:00:00'),
            lane: 1,
            groupId: 'group-10',
            indexInGroup: 1,
            rawPayload: ['PTS' => '42949485818.0'],
            lprImageBytes: null,
            lprImageContentType: null,
        );

        $builder = new BuildIntelbrasEventIdempotencyKey;

        $this->assertSame($builder($cameraPair, $event), $builder($cameraPair, $event));
        $this->assertStringStartsWith('intelbras:edge-uuid-01:', $builder($cameraPair, $event));
    }
}
```

- [ ] **Step 2: Run the new test and verify it fails**

Run:

```bash
php artisan test --filter=TrafficCaptureDataTest
```

Expected: FAIL because `TrafficCaptureData`, `SnapshotData`, and `BuildIntelbrasEventIdempotencyKey` do not exist.

- [ ] **Step 3: Implement config keys**

Modify `config/trackvision.php` by adding:

```php
'edge' => [
    'store_unknown_plates' => (bool) env('TRACKVISION_EDGE_STORE_UNKNOWN_PLATES', false),
],

'intelbras' => [
    'webhook_username' => env('TRACKVISION_INTELBRAS_WEBHOOK_USERNAME'),
    'webhook_password' => env('TRACKVISION_INTELBRAS_WEBHOOK_PASSWORD'),
    'event_code' => env('TRACKVISION_INTELBRAS_LPR_EVENT_CODE', 'TrafficJunction'),
    'snapshot_type' => (int) env('TRACKVISION_INTELBRAS_SNAPSHOT_TYPE', 0),
    'event_stream_heartbeat_seconds' => (int) env('TRACKVISION_INTELBRAS_EVENT_STREAM_HEARTBEAT_SECONDS', 5),
    'event_dedupe_window_seconds' => (int) env('TRACKVISION_INTELBRAS_EVENT_DEDUPE_WINDOW_SECONDS', 30),
],
```

Append to `.env.example`:

```text
TRACKVISION_EDGE_STORE_UNKNOWN_PLATES=false
TRACKVISION_INTELBRAS_WEBHOOK_USERNAME=
TRACKVISION_INTELBRAS_WEBHOOK_PASSWORD=
TRACKVISION_INTELBRAS_LPR_EVENT_CODE=TrafficJunction
TRACKVISION_INTELBRAS_SNAPSHOT_TYPE=0
TRACKVISION_INTELBRAS_EVENT_STREAM_HEARTBEAT_SECONDS=5
TRACKVISION_INTELBRAS_EVENT_DEDUPE_WINDOW_SECONDS=30
```

- [ ] **Step 4: Implement enums**

Create enum files with these cases:

```php
<?php

namespace App\Enums;

enum CaptureStatus: string
{
    case Accepted = 'accepted';
    case IgnoredUnknownPlate = 'ignored_unknown_plate';
    case FailedSupportCapture = 'failed_support_capture';
}
```

```php
<?php

namespace App\Enums;

enum LoadStatus: string
{
    case Unknown = 'unknown';
    case Loaded = 'loaded';
    case Empty = 'empty';
    case NeedsReview = 'needs_review';
}
```

```php
<?php

namespace App\Enums;

enum MediaAssetKind: string
{
    case LprImage = 'lpr_image';
    case SupportImage = 'support_image';
}
```

```php
<?php

namespace App\Enums;

enum EdgeOutboxStatus: string
{
    case Pending = 'pending';
    case Synced = 'synced';
    case Failed = 'failed';
}
```

- [ ] **Step 5: Implement DTOs**

Create `TrafficCaptureData`:

```php
<?php

namespace App\Data\Cameras;

use Carbon\CarbonImmutable;

readonly class TrafficCaptureData
{
    public function __construct(
        public string $eventCode,
        public string $action,
        public ?string $plateNumber,
        public ?string $plateNormalized,
        public CarbonImmutable $eventTime,
        public ?int $lane,
        public ?string $groupId,
        public ?int $indexInGroup,
        public array $rawPayload,
        public ?string $lprImageBytes,
        public ?string $lprImageContentType,
    ) {}

    public function hasLprImage(): bool
    {
        return $this->lprImageBytes !== null && $this->lprImageBytes !== '';
    }
}
```

Create `SnapshotData`:

```php
<?php

namespace App\Data\Cameras;

use Carbon\CarbonImmutable;

readonly class SnapshotData
{
    public function __construct(
        public string $bytes,
        public string $contentType,
        public string $sha256Hash,
        public int $byteSize,
        public CarbonImmutable $capturedAt,
        public string $sourceCameraUuid,
    ) {}

    public static function fromBytes(
        string $bytes,
        string $contentType,
        CarbonImmutable $capturedAt,
        string $sourceCameraUuid,
    ): self {
        return new self($bytes, $contentType, hash('sha256', $bytes), strlen($bytes), $capturedAt, $sourceCameraUuid);
    }
}
```

- [ ] **Step 6: Implement idempotency key builder**

Create `BuildIntelbrasEventIdempotencyKey`:

```php
<?php

namespace App\Support\Cameras;

use App\Data\Cameras\TrafficCaptureData;
use App\Models\CameraPair;

class BuildIntelbrasEventIdempotencyKey
{
    public function __invoke(CameraPair $cameraPair, TrafficCaptureData $event): string
    {
        $edgeUuid = config('trackvision.edge_node_uuid') ?: $cameraPair->edgeNode()->value('uuid');
        $payloadHash = hash('sha256', json_encode($event->rawPayload, JSON_THROW_ON_ERROR));

        return implode(':', [
            'intelbras',
            $edgeUuid,
            $cameraPair->uuid,
            $event->plateNormalized ?: 'no-plate',
            $event->eventTime->toISOString(),
            $event->groupId ?: 'no-group',
            $event->indexInGroup !== null ? $event->indexInGroup : 'no-index',
            $payloadHash,
        ]);
    }
}
```

- [ ] **Step 7: Run DTO tests and full focused unit test**

Run:

```bash
php artisan test --filter=TrafficCaptureDataTest
php artisan test --filter=NormalizePlateTest
```

Expected: PASS.

- [ ] **Step 8: Commit Task 1**

Run:

```bash
git add config/trackvision.php .env.example app/Data/Cameras app/Enums app/Support/Cameras tests/Unit/Cameras/Intelbras/TrafficCaptureDataTest.php
git commit -m "feat: add intelbras capture data contracts"
```

---

### Task 2: Intelbras Event and Webhook Parsing

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Services/Cameras/Intelbras/TrafficEventParser.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/Cameras/Intelbras/IntelbrasWebhookParser.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/Cameras/Intelbras/TrafficEventParserTest.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/Cameras/Intelbras/IntelbrasWebhookParserTest.php`

**Interfaces:**

- Consumes: `TrafficCaptureData`
- Consumes: `NormalizePlate::__invoke(string $plate): string`
- Produces: `TrafficEventParser::parse(array|string $payload, ?string $lprImageBytes = null, ?string $lprImageContentType = null): TrafficCaptureData`
- Produces: `IntelbrasWebhookParser::parse(Illuminate\Http\Request $request): TrafficCaptureData`

- [ ] **Step 1: Write failing parser tests for JSON and key-value payloads**

Create `TrafficEventParserTest.php`:

```php
<?php

namespace Tests\Unit\Cameras\Intelbras;

use App\Services\Cameras\Intelbras\TrafficEventParser;
use App\Support\Vehicles\NormalizePlate;
use Tests\TestCase;

class TrafficEventParserTest extends TestCase
{
    public function test_parses_event_http_upload_json_payload(): void
    {
        $payload = [
            'Code' => 'TrafficJunction',
            'Action' => 'Pulse',
            'Index' => 0,
            'Data' => [
                'UTC' => 1786543200,
                'Lane' => 2,
                'TrafficCar' => ['PlateNumber' => 'abc-1d23'],
            ],
        ];

        $event = (new TrafficEventParser(new NormalizePlate))->parse($payload);

        $this->assertSame('TrafficJunction', $event->eventCode);
        $this->assertSame('Pulse', $event->action);
        $this->assertSame('ABC1D23', $event->plateNormalized);
        $this->assertSame(2, $event->lane);
    }

    public function test_parses_attach_file_proc_key_value_payload(): void
    {
        $payload = implode("\n", [
            'Events[0].EventBaseInfo.Code=TrafficJunction',
            'Events[0].EventBaseInfo.Action=Pulse',
            'Events[0].Lane=1',
            'Events[0].CountInGroup=3',
            'Events[0].IndexInGroup=1',
            'Events[0].PTS=42949485818.0',
            'Events[0].TrafficCar.PlateNumber=ZZZ12345',
        ]);

        $event = (new TrafficEventParser(new NormalizePlate))->parse($payload);

        $this->assertSame('TrafficJunction', $event->eventCode);
        $this->assertSame('ZZZ12345', $event->plateNormalized);
        $this->assertSame('3', $event->groupId);
        $this->assertSame(1, $event->indexInGroup);
    }
}
```

- [ ] **Step 2: Write failing webhook parser tests**

Create `IntelbrasWebhookParserTest.php`:

```php
<?php

namespace Tests\Unit\Cameras\Intelbras;

use App\Services\Cameras\Intelbras\IntelbrasWebhookParser;
use App\Services\Cameras\Intelbras\TrafficEventParser;
use App\Support\Vehicles\NormalizePlate;
use Illuminate\Http\Request;
use Illuminate\Http\UploadedFile;
use Tests\TestCase;

class IntelbrasWebhookParserTest extends TestCase
{
    public function test_parses_json_webhook_request(): void
    {
        $request = Request::create(
            uri: '/api/v1/edge/intelbras/camera-pairs/pair/lpr-events',
            method: 'POST',
            server: ['CONTENT_TYPE' => 'application/json'],
            content: json_encode([
                'Code' => 'TrafficJunction',
                'Action' => 'Pulse',
                'Data' => ['UTC' => 1786543200, 'TrafficCar' => ['PlateNumber' => 'ABC-1D23']],
            ], JSON_THROW_ON_ERROR),
        );

        $event = $this->parser()->parse($request);

        $this->assertSame('ABC1D23', $event->plateNormalized);
        $this->assertFalse($event->hasLprImage());
    }

    public function test_parses_multipart_webhook_request_with_jpeg(): void
    {
        $image = UploadedFile::fake()->image('plate.jpg', 320, 120);
        $request = Request::create(
            uri: '/api/v1/edge/intelbras/camera-pairs/pair/lpr-events',
            method: 'POST',
            parameters: [
                'Code' => 'TrafficJunction',
                'Action' => 'Pulse',
                'Data' => ['TrafficCar' => ['PlateNumber' => 'BRA2E19']],
            ],
            files: ['image' => $image],
        );

        $event = $this->parser()->parse($request);

        $this->assertSame('BRA2E19', $event->plateNormalized);
        $this->assertTrue($event->hasLprImage());
        $this->assertSame('image/jpeg', $event->lprImageContentType);
    }

    private function parser(): IntelbrasWebhookParser
    {
        return new IntelbrasWebhookParser(new TrafficEventParser(new NormalizePlate));
    }
}
```

- [ ] **Step 3: Run parser tests and verify they fail**

Run:

```bash
php artisan test --filter=TrafficEventParserTest
php artisan test --filter=IntelbrasWebhookParserTest
```

Expected: FAIL because the parser classes do not exist.

- [ ] **Step 4: Implement `TrafficEventParser`**

Create `TrafficEventParser` with array normalization and key-value parsing:

```php
<?php

namespace App\Services\Cameras\Intelbras;

use App\Data\Cameras\TrafficCaptureData;
use App\Support\Vehicles\NormalizePlate;
use Carbon\CarbonImmutable;
use InvalidArgumentException;

class TrafficEventParser
{
    public function __construct(private readonly NormalizePlate $normalizePlate) {}

    public function parse(array|string $payload, ?string $lprImageBytes = null, ?string $lprImageContentType = null): TrafficCaptureData
    {
        $data = is_string($payload) ? $this->parseKeyValuePayload($payload) : $payload;
        $event = $this->flattenEvent($data);
        $plate = array_key_exists('plate_number', $event) ? $event['plate_number'] : null;

        if (! array_key_exists('event_code', $event) || $event['event_code'] === null) {
            throw new InvalidArgumentException('Intelbras event code is missing.');
        }

        return new TrafficCaptureData(
            eventCode: (string) $event['event_code'],
            action: (string) (array_key_exists('action', $event) ? $event['action'] : 'Pulse'),
            plateNumber: $plate,
            plateNormalized: $plate ? ($this->normalizePlate)($plate) : null,
            eventTime: $this->eventTime($event),
            lane: isset($event['lane']) ? (int) $event['lane'] : null,
            groupId: isset($event['group_id']) ? (string) $event['group_id'] : null,
            indexInGroup: isset($event['index_in_group']) ? (int) $event['index_in_group'] : null,
            rawPayload: is_string($payload) ? ['raw' => $payload] : $payload,
            lprImageBytes: $lprImageBytes,
            lprImageContentType: $lprImageContentType,
        );
    }
}
```

Implement private methods in the same class:

```php
private function flattenEvent(array $payload): array
{
    $data = data_get($payload, 'Data', $payload);
    $event = data_get($payload, 'Events.0', $data);

    return [
        'event_code' => data_get($payload, 'Code') ?: data_get($event, 'EventBaseInfo.Code') ?: data_get($event, 'Code'),
        'action' => data_get($payload, 'Action') ?: data_get($event, 'EventBaseInfo.Action') ?: data_get($event, 'Action'),
        'plate_number' => data_get($data, 'TrafficCar.PlateNumber') ?: data_get($event, 'TrafficCar.PlateNumber'),
        'lane' => data_get($data, 'Lane') ?: data_get($event, 'Lane'),
        'group_id' => data_get($event, 'CountInGroup') ?: data_get($data, 'GroupId'),
        'index_in_group' => data_get($event, 'IndexInGroup') ?: data_get($data, 'IndexInGroup'),
        'utc' => data_get($data, 'UTC') ?: data_get($event, 'UTC'),
        'pts' => data_get($event, 'PTS') ?: data_get($data, 'PTS'),
    ];
}

private function parseKeyValuePayload(string $payload): array
{
    $result = [];

    foreach (preg_split('/\r\n|\r|\n/', trim($payload)) as $line) {
        if ($line === '' || ! str_contains($line, '=')) {
            continue;
        }

        [$key, $value] = explode('=', $line, 2);
        $this->assignDottedKey($result, str_replace(['Events[0].', 'EventBaseInfo.'], ['Events.0.', 'EventBaseInfo.'], $key), $value);
    }

    return $result;
}
```

The `assignDottedKey()` private method must split on `.` and create nested arrays. Keep it private and covered through the public parser tests.

- [ ] **Step 5: Implement `IntelbrasWebhookParser`**

Create `IntelbrasWebhookParser`:

```php
<?php

namespace App\Services\Cameras\Intelbras;

use Illuminate\Http\Request;
use Illuminate\Http\UploadedFile;

class IntelbrasWebhookParser
{
    public function __construct(private readonly TrafficEventParser $trafficEventParser) {}

    public function parse(Request $request): \App\Data\Cameras\TrafficCaptureData
    {
        [$imageBytes, $imageContentType] = $this->extractFirstJpeg($request);

        if (str_contains((string) $request->headers->get('content-type'), 'application/json')) {
            return $this->trafficEventParser->parse($request->json()->all(), $imageBytes, $imageContentType);
        }

        if ($request->request->count() > 0) {
            return $this->trafficEventParser->parse($request->request->all(), $imageBytes, $imageContentType);
        }

        return $this->trafficEventParser->parse($request->getContent(), $imageBytes, $imageContentType);
    }

    private function extractFirstJpeg(Request $request): array
    {
        foreach ($request->allFiles() as $file) {
            if ($file instanceof UploadedFile && str_starts_with((string) $file->getMimeType(), 'image/')) {
                return [$file->get(), $file->getMimeType()];
            }
        }

        return [null, null];
    }
}
```

- [ ] **Step 6: Run parser tests**

Run:

```bash
php artisan test --filter=TrafficEventParserTest
php artisan test --filter=IntelbrasWebhookParserTest
```

Expected: PASS.

- [ ] **Step 7: Commit Task 2**

Run:

```bash
git add app/Services/Cameras/Intelbras tests/Unit/Cameras/Intelbras
git commit -m "feat: parse intelbras lpr event payloads"
```

---

### Task 3: Intelbras HTTP and Snapshot Client

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Services/Cameras/Intelbras/IntelbrasHttpClient.php`
- Create: `RIALMA-TrackVision-Backend/app/Services/Cameras/Intelbras/IntelbrasSnapshotClient.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/Cameras/Intelbras/IntelbrasSnapshotClientTest.php`

**Interfaces:**

- Consumes: `App\Models\Camera`
- Consumes: `SnapshotData::fromBytes(string $bytes, string $contentType, CarbonImmutable $capturedAt, string $sourceCameraUuid): SnapshotData`
- Produces: `IntelbrasHttpClient::get(Camera $camera, string $path, array $query = []): Illuminate\Http\Client\Response`
- Produces: `IntelbrasHttpClient::buildUrl(Camera $camera, string $path): string`
- Produces: `IntelbrasSnapshotClient::capture(Camera $camera): SnapshotData`

- [ ] **Step 1: Write failing snapshot client tests**

Create `IntelbrasSnapshotClientTest.php`:

```php
<?php

namespace Tests\Unit\Cameras\Intelbras;

use App\Enums\CameraType;
use App\Models\Camera;
use App\Services\Cameras\Intelbras\IntelbrasHttpClient;
use App\Services\Cameras\Intelbras\IntelbrasSnapshotClient;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Http;
use Tests\TestCase;

class IntelbrasSnapshotClientTest extends TestCase
{
    use RefreshDatabase;

    public function test_captures_jpeg_snapshot_from_camera_channel(): void
    {
        config(['trackvision.camera_default_timeout_seconds' => 7]);

        $camera = Camera::factory()->create([
            'uuid' => '22222222-2222-4222-8222-222222222222',
            'type' => CameraType::Support,
            'host' => '192.168.10.20',
            'port' => 8080,
            'channel' => 1,
            'username' => 'admin',
            'password_encrypted' => 'camera-secret',
        ]);

        Http::fake([
            '192.168.10.20:8080/cgi-bin/snapshot.cgi*' => Http::response('jpeg-binary', 200, ['Content-Type' => 'image/jpeg']),
        ]);

        $snapshot = (new IntelbrasSnapshotClient(new IntelbrasHttpClient))->capture($camera);

        $this->assertSame('image/jpeg', $snapshot->contentType);
        $this->assertSame('jpeg-binary', $snapshot->bytes);
        $this->assertSame(hash('sha256', 'jpeg-binary'), $snapshot->sha256Hash);
        $this->assertSame('22222222-2222-4222-8222-222222222222', $snapshot->sourceCameraUuid);

        Http::assertSent(fn ($request) => str_contains($request->url(), '/cgi-bin/snapshot.cgi')
            && str_contains($request->url(), 'channel=1')
            && str_contains($request->url(), 'type=0'));
    }
}
```

- [ ] **Step 2: Run snapshot test and verify it fails**

Run:

```bash
php artisan test --filter=IntelbrasSnapshotClientTest
```

Expected: FAIL because HTTP and snapshot client classes do not exist.

- [ ] **Step 3: Implement `IntelbrasHttpClient`**

Create the client:

```php
<?php

namespace App\Services\Cameras\Intelbras;

use App\Models\Camera;
use Illuminate\Http\Client\Response;
use Illuminate\Support\Facades\Http;

class IntelbrasHttpClient
{
    public function get(Camera $camera, string $path, array $query = []): Response
    {
        return Http::timeout((int) config('trackvision.camera_default_timeout_seconds', 10))
            ->withDigestAuth((string) $camera->username, (string) $camera->password_encrypted)
            ->get($this->buildUrl($camera, $path), $query)
            ->throw();
    }

    public function buildUrl(Camera $camera, string $path): string
    {
        $host = trim((string) $camera->host);
        $port = (int) ($camera->port ?: 80);
        $normalizedPath = '/'.ltrim($path, '/');

        return "http://{$host}:{$port}{$normalizedPath}";
    }
}
```

- [ ] **Step 4: Implement `IntelbrasSnapshotClient`**

Create the snapshot client:

```php
<?php

namespace App\Services\Cameras\Intelbras;

use App\Data\Cameras\SnapshotData;
use App\Models\Camera;
use Carbon\CarbonImmutable;

class IntelbrasSnapshotClient
{
    public function __construct(private readonly IntelbrasHttpClient $httpClient) {}

    public function capture(Camera $camera): SnapshotData
    {
        $response = $this->httpClient->get($camera, '/cgi-bin/snapshot.cgi', [
            'channel' => $camera->channel ?: 1,
            'type' => (int) config('trackvision.intelbras.snapshot_type', 0),
        ]);

        return SnapshotData::fromBytes(
            bytes: $response->body(),
            contentType: $response->header('Content-Type') ?: 'image/jpeg',
            capturedAt: CarbonImmutable::now(),
            sourceCameraUuid: $camera->uuid,
        );
    }
}
```

- [ ] **Step 5: Run snapshot and existing camera tests**

Run:

```bash
php artisan test --filter=IntelbrasSnapshotClientTest
php artisan test --filter=CameraAdminTest
```

Expected: PASS.

- [ ] **Step 6: Commit Task 3**

Run:

```bash
git add app/Services/Cameras/Intelbras/IntelbrasHttpClient.php app/Services/Cameras/Intelbras/IntelbrasSnapshotClient.php tests/Unit/Cameras/Intelbras/IntelbrasSnapshotClientTest.php
git commit -m "feat: add intelbras snapshot client"
```

---

### Task 4: Local Capture Persistence and Outbox

**Files:**

- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_12_160001_create_capture_events_table.php`
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_12_160002_create_media_assets_table.php`
- Create: `RIALMA-TrackVision-Backend/database/migrations/2026_08_12_160003_create_edge_outbox_messages_table.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/CaptureEvent.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/MediaAsset.php`
- Create: `RIALMA-TrackVision-Backend/app/Models/EdgeOutboxMessage.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/CaptureEventFactory.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/MediaAssetFactory.php`
- Create: `RIALMA-TrackVision-Backend/database/factories/EdgeOutboxMessageFactory.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/StoreCaptureMediaAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/EnqueueCaptureForSyncAction.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/ProcessLprEventAction.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Edge/ProcessLprEventTest.php`

**Interfaces:**

- Consumes: `TrafficCaptureData`
- Consumes: `SnapshotData`
- Consumes: `IntelbrasSnapshotClient::capture(Camera $camera): SnapshotData`
- Consumes: `BuildIntelbrasEventIdempotencyKey::__invoke(CameraPair $cameraPair, TrafficCaptureData $event): string`
- Produces: `ProcessLprEventAction::execute(CameraPair $cameraPair, TrafficCaptureData $event): ?CaptureEvent`
- Produces: `StoreCaptureMediaAction::execute(CaptureEvent $captureEvent, Camera $camera, MediaAssetKind $kind, SnapshotData $snapshot): MediaAsset`
- Produces: `EnqueueCaptureForSyncAction::execute(CaptureEvent $captureEvent): EdgeOutboxMessage`

- [ ] **Step 1: Write failing persistence feature tests**

Create `ProcessLprEventTest.php`:

```php
<?php

namespace Tests\Feature\Edge;

use App\Actions\Edge\ProcessLprEventAction;
use App\Data\Cameras\SnapshotData;
use App\Data\Cameras\TrafficCaptureData;
use App\Enums\CameraPairDirection;
use App\Enums\CameraType;
use App\Models\Camera;
use App\Models\CameraPair;
use App\Models\Vehicle;
use App\Services\Cameras\Intelbras\IntelbrasSnapshotClient;
use Carbon\CarbonImmutable;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Storage;
use Mockery;
use Tests\TestCase;

class ProcessLprEventTest extends TestCase
{
    use RefreshDatabase;

    public function test_registered_vehicle_creates_capture_media_and_outbox(): void
    {
        Storage::fake('local');
        $pair = $this->cameraPair();
        Vehicle::factory()->create(['plate' => 'ABC-1D23', 'plate_normalized' => 'ABC1D23', 'is_active' => true]);

        $this->mockSnapshots([
            $pair->supportCamera->uuid => 'support-jpeg',
        ]);

        $capture = app(ProcessLprEventAction::class)->execute($pair, $this->event(lprImageBytes: 'lpr-jpeg'));

        $this->assertNotNull($capture);
        $this->assertDatabaseHas('capture_events', ['plate_normalized' => 'ABC1D23', 'status' => 'accepted']);
        $this->assertDatabaseHas('media_assets', ['kind' => 'lpr_image']);
        $this->assertDatabaseHas('media_assets', ['kind' => 'support_image']);
        $this->assertDatabaseHas('edge_outbox_messages', ['type' => 'capture.created', 'status' => 'pending']);
    }

    public function test_unknown_plate_is_ignored_when_storage_is_disabled(): void
    {
        config(['trackvision.edge.store_unknown_plates' => false]);
        Storage::fake('local');
        $pair = $this->cameraPair();

        $capture = app(ProcessLprEventAction::class)->execute($pair, $this->event(plate: 'ZZZ9999', normalized: 'ZZZ9999'));

        $this->assertNull($capture);
        $this->assertDatabaseCount('capture_events', 0);
        $this->assertDatabaseCount('edge_outbox_messages', 0);
    }

    public function test_missing_lpr_image_triggers_lpr_snapshot_fallback(): void
    {
        Storage::fake('local');
        $pair = $this->cameraPair();
        Vehicle::factory()->create(['plate' => 'ABC-1D23', 'plate_normalized' => 'ABC1D23', 'is_active' => true]);

        $this->mockSnapshots([
            $pair->lprCamera->uuid => 'lpr-fallback-jpeg',
            $pair->supportCamera->uuid => 'support-jpeg',
        ]);

        app(ProcessLprEventAction::class)->execute($pair, $this->event(lprImageBytes: null));

        $this->assertDatabaseHas('media_assets', ['kind' => 'lpr_image', 'sha256_hash' => hash('sha256', 'lpr-fallback-jpeg')]);
        $this->assertDatabaseHas('media_assets', ['kind' => 'support_image', 'sha256_hash' => hash('sha256', 'support-jpeg')]);
    }

    public function test_duplicate_event_returns_existing_capture_without_creating_second_outbox_message(): void
    {
        Storage::fake('local');
        $pair = $this->cameraPair();
        Vehicle::factory()->create(['plate' => 'ABC-1D23', 'plate_normalized' => 'ABC1D23', 'is_active' => true]);
        $this->mockSnapshots([$pair->supportCamera->uuid => 'support-jpeg']);

        $action = app(ProcessLprEventAction::class);
        $first = $action->execute($pair, $this->event(lprImageBytes: 'lpr-jpeg'));
        $second = $action->execute($pair, $this->event(lprImageBytes: 'lpr-jpeg'));

        $this->assertSame($first->id, $second->id);
        $this->assertDatabaseCount('capture_events', 1);
        $this->assertDatabaseCount('edge_outbox_messages', 1);
    }
}
```

Add private helpers in the same test file:

```php
private function cameraPair(): CameraPair
{
    $pair = CameraPair::factory()->create(['direction' => CameraPairDirection::Outbound]);
    $pair->loadMissing(['lprCamera', 'supportCamera', 'edgeNode']);

    return $pair;
}

private function event(?string $lprImageBytes = 'lpr-jpeg', string $plate = 'ABC-1D23', string $normalized = 'ABC1D23'): TrafficCaptureData
{
    return new TrafficCaptureData(
        eventCode: 'TrafficJunction',
        action: 'Pulse',
        plateNumber: $plate,
        plateNormalized: $normalized,
        eventTime: CarbonImmutable::parse('2026-08-12 15:00:00'),
        lane: 1,
        groupId: '3',
        indexInGroup: 1,
        rawPayload: ['PTS' => '42949485818.0'],
        lprImageBytes: $lprImageBytes,
        lprImageContentType: $lprImageBytes ? 'image/jpeg' : null,
    );
}

private function mockSnapshots(array $bytesByCameraUuid): void
{
    $mock = Mockery::mock(IntelbrasSnapshotClient::class);
    $mock->shouldReceive('capture')->andReturnUsing(function (Camera $camera) use ($bytesByCameraUuid): SnapshotData {
        return SnapshotData::fromBytes(
            bytes: $bytesByCameraUuid[$camera->uuid],
            contentType: 'image/jpeg',
            capturedAt: CarbonImmutable::parse('2026-08-12 15:00:01'),
            sourceCameraUuid: $camera->uuid,
        );
    });

    $this->app->instance(IntelbrasSnapshotClient::class, $mock);
}
```

- [ ] **Step 2: Run persistence tests and verify they fail**

Run:

```bash
php artisan test --filter=ProcessLprEventTest
```

Expected: FAIL because capture models, tables, and Actions do not exist.

- [ ] **Step 3: Implement capture migrations**

Create `capture_events` migration:

```php
Schema::create('capture_events', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->foreignId('camera_pair_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
    $table->foreignId('edge_node_id')->constrained()->cascadeOnUpdate()->restrictOnDelete();
    $table->foreignId('vehicle_id')->nullable()->constrained()->cascadeOnUpdate()->nullOnDelete();
    $table->foreignId('lpr_camera_id')->constrained('cameras')->cascadeOnUpdate()->restrictOnDelete();
    $table->foreignId('support_camera_id')->constrained('cameras')->cascadeOnUpdate()->restrictOnDelete();
    $table->string('plate', 20)->nullable();
    $table->string('plate_normalized', 20)->nullable()->index();
    $table->string('event_code', 64);
    $table->string('action', 32);
    $table->timestamp('event_time')->nullable()->index();
    $table->unsignedInteger('lane')->nullable();
    $table->string('direction', 30)->default('unknown');
    $table->string('status', 40);
    $table->string('load_status', 40)->default('unknown');
    $table->string('idempotency_key')->unique();
    $table->json('raw_payload')->nullable();
    $table->timestamps();
});
```

Create `media_assets` migration:

```php
Schema::create('media_assets', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->foreignId('capture_event_id')->constrained()->cascadeOnUpdate()->cascadeOnDelete();
    $table->foreignId('camera_id')->constrained('cameras')->cascadeOnUpdate()->restrictOnDelete();
    $table->string('kind', 40);
    $table->string('disk', 40)->default('local');
    $table->string('path');
    $table->string('content_type', 120);
    $table->string('sha256_hash', 64)->index();
    $table->unsignedBigInteger('byte_size');
    $table->timestamp('captured_at')->nullable();
    $table->timestamps();

    $table->unique(['capture_event_id', 'kind', 'camera_id']);
});
```

Create `edge_outbox_messages` migration:

```php
Schema::create('edge_outbox_messages', function (Blueprint $table): void {
    $table->id();
    $table->uuid('uuid')->unique();
    $table->string('type', 80);
    $table->uuid('aggregate_uuid');
    $table->string('idempotency_key')->unique();
    $table->json('payload');
    $table->string('status', 30)->default('pending');
    $table->unsignedInteger('attempts')->default(0);
    $table->timestamp('available_at')->nullable();
    $table->timestamp('synced_at')->nullable();
    $table->text('last_error')->nullable();
    $table->timestamps();
});
```

- [ ] **Step 4: Implement models and factories**

Each model must use `HasFactory` and `HasPublicUuid`, define fillable attributes, casts for enums/datetime/json, and relationships:

```php
public function captureEvent(): BelongsTo
{
    return $this->belongsTo(CaptureEvent::class);
}
```

`CaptureEvent` relationships:

```php
public function cameraPair(): BelongsTo;
public function edgeNode(): BelongsTo;
public function vehicle(): BelongsTo;
public function lprCamera(): BelongsTo;
public function supportCamera(): BelongsTo;
public function mediaAssets(): HasMany;
```

Factories must create valid rows using existing `VehicleFactory`, `CameraPairFactory`, and `CameraFactory`.

- [ ] **Step 5: Implement `StoreCaptureMediaAction`**

Create action:

```php
public function execute(CaptureEvent $captureEvent, Camera $camera, MediaAssetKind $kind, SnapshotData $snapshot): MediaAsset
{
    $path = sprintf(
        'captures/%s/%s/%s.jpg',
        $captureEvent->uuid,
        $kind->value,
        $snapshot->sha256Hash,
    );

    Storage::disk('local')->put($path, $snapshot->bytes);

    return $captureEvent->mediaAssets()->create([
        'camera_id' => $camera->id,
        'kind' => $kind,
        'disk' => 'local',
        'path' => $path,
        'content_type' => $snapshot->contentType,
        'sha256_hash' => $snapshot->sha256Hash,
        'byte_size' => $snapshot->byteSize,
        'captured_at' => $snapshot->capturedAt,
    ]);
}
```

- [ ] **Step 6: Implement `EnqueueCaptureForSyncAction`**

Create action:

```php
public function execute(CaptureEvent $captureEvent): EdgeOutboxMessage
{
    return EdgeOutboxMessage::firstOrCreate(
        ['idempotency_key' => $captureEvent->idempotency_key],
        [
            'type' => 'capture.created',
            'aggregate_uuid' => $captureEvent->uuid,
            'payload' => [
                'capture_event_uuid' => $captureEvent->uuid,
                'plate_normalized' => $captureEvent->plate_normalized,
                'event_time' => optional($captureEvent->event_time)->toISOString(),
            ],
            'status' => EdgeOutboxStatus::Pending,
            'available_at' => now(),
        ],
    );
}
```

- [ ] **Step 7: Implement `ProcessLprEventAction`**

Use a database transaction and this flow:

```php
public function execute(CameraPair $cameraPair, TrafficCaptureData $event): ?CaptureEvent
{
    $cameraPair->loadMissing(['lprCamera', 'supportCamera', 'edgeNode']);
    $idempotencyKey = ($this->buildIdempotencyKey)($cameraPair, $event);

    if ($existing = CaptureEvent::where('idempotency_key', $idempotencyKey)->first()) {
        return $existing;
    }

    $vehicle = $event->plateNormalized
        ? Vehicle::query()->where('plate_normalized', $event->plateNormalized)->where('is_active', true)->first()
        : null;

    if (! $vehicle && ! config('trackvision.edge.store_unknown_plates', false)) {
        return null;
    }

    return DB::transaction(function () use ($cameraPair, $event, $idempotencyKey, $vehicle): CaptureEvent {
        $status = $vehicle ? CaptureStatus::Accepted : CaptureStatus::IgnoredUnknownPlate;
        $supportSnapshot = null;

        if ($vehicle) {
            try {
                $supportSnapshot = $this->snapshotClient->capture($cameraPair->supportCamera);
            } catch (Throwable $exception) {
                report($exception);
                $status = CaptureStatus::FailedSupportCapture;
            }
        }

        $capture = CaptureEvent::create([
            'camera_pair_id' => $cameraPair->id,
            'edge_node_id' => $cameraPair->edge_node_id,
            'vehicle_id' => $vehicle?->id,
            'lpr_camera_id' => $cameraPair->lpr_camera_id,
            'support_camera_id' => $cameraPair->support_camera_id,
            'plate' => $event->plateNumber,
            'plate_normalized' => $event->plateNormalized,
            'event_code' => $event->eventCode,
            'action' => $event->action,
            'event_time' => $event->eventTime,
            'lane' => $event->lane,
            'direction' => $cameraPair->direction,
            'status' => $status,
            'load_status' => LoadStatus::Unknown,
            'idempotency_key' => $idempotencyKey,
            'raw_payload' => $event->rawPayload,
        ]);

        $this->storeImages($capture, $cameraPair, $event, $supportSnapshot);
        $this->enqueueCaptureForSyncAction->execute($capture);

        return $capture;
    });
}
```

The private `storeImages()` method must:

- save LPR bytes from the event when `TrafficCaptureData::hasLprImage()` is true;
- call `IntelbrasSnapshotClient::capture($cameraPair->lprCamera)` when event LPR image is missing;
- save support image only when `$supportSnapshot` is not null.

- [ ] **Step 8: Run persistence tests**

Run:

```bash
php artisan test --filter=ProcessLprEventTest
```

Expected: PASS.

- [ ] **Step 9: Commit Task 4**

Run:

```bash
git add database/migrations database/factories app/Models app/Actions/Edge tests/Feature/Edge/ProcessLprEventTest.php
git commit -m "feat: persist edge lpr captures locally"
```

---

### Task 5: Intelbras Webhook Endpoint

**Files:**

- Modify: `RIALMA-TrackVision-Backend/routes/api.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Middleware/EnsureTrackVisionEdgeNode.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Edge/IntelbrasLprWebhookRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Edge/IntelbrasLprWebhookController.php`
- Create: `RIALMA-TrackVision-Backend/app/Actions/Edge/HandleIntelbrasLprWebhookAction.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Edge/IntelbrasLprWebhookTest.php`

**Interfaces:**

- Consumes: `IntelbrasWebhookParser::parse(Request $request): TrafficCaptureData`
- Consumes: `ProcessLprEventAction::execute(CameraPair $cameraPair, TrafficCaptureData $event): ?CaptureEvent`
- Produces: `POST /api/v1/edge/intelbras/camera-pairs/{cameraPair:uuid}/lpr-events`
- Produces: `EnsureTrackVisionEdgeNode::handle(Request $request, Closure $next): mixed`
- Produces: `HandleIntelbrasLprWebhookAction::execute(CameraPair $cameraPair, Request $request): ?CaptureEvent`

- [ ] **Step 1: Write failing webhook feature tests**

Create `IntelbrasLprWebhookTest.php`:

```php
<?php

namespace Tests\Feature\Edge;

use App\Enums\CameraPairDirection;
use App\Models\CameraPair;
use App\Models\Vehicle;
use App\Services\Cameras\Intelbras\IntelbrasSnapshotClient;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Storage;
use Mockery;
use Tests\TestCase;

class IntelbrasLprWebhookTest extends TestCase
{
    use RefreshDatabase;

    public function test_edge_webhook_accepts_valid_intelbras_event(): void
    {
        $this->edgeConfig();
        Storage::fake('local');
        $pair = CameraPair::factory()->create(['direction' => CameraPairDirection::Inbound]);
        Vehicle::factory()->create(['plate' => 'ABC-1D23', 'plate_normalized' => 'ABC1D23', 'is_active' => true]);
        $this->mockSnapshotClient();

        $this->withBasicAuth('intelbras-upload', 'secret-upload')
            ->postJson("/api/v1/edge/intelbras/camera-pairs/{$pair->uuid}/lpr-events", $this->payload('ABC-1D23'))
            ->assertOk()
            ->assertSee('OK');

        $this->assertDatabaseHas('capture_events', ['plate_normalized' => 'ABC1D23']);
    }

    public function test_webhook_rejects_invalid_basic_auth(): void
    {
        $this->edgeConfig();
        $pair = CameraPair::factory()->create();

        $this->withBasicAuth('intelbras-upload', 'wrong')
            ->postJson("/api/v1/edge/intelbras/camera-pairs/{$pair->uuid}/lpr-events", $this->payload('ABC-1D23'))
            ->assertForbidden();
    }

    public function test_webhook_returns_not_found_when_node_role_is_parent(): void
    {
        config(['trackvision.node_role' => 'parent']);
        $pair = CameraPair::factory()->create();

        $this->withBasicAuth('intelbras-upload', 'secret-upload')
            ->postJson("/api/v1/edge/intelbras/camera-pairs/{$pair->uuid}/lpr-events", $this->payload('ABC-1D23'))
            ->assertNotFound();
    }
}
```

Add helpers in the same test file:

```php
private function edgeConfig(): void
{
    config([
        'trackvision.node_role' => 'edge',
        'trackvision.intelbras.webhook_username' => 'intelbras-upload',
        'trackvision.intelbras.webhook_password' => 'secret-upload',
    ]);
}

private function payload(string $plate): array
{
    return [
        'Code' => 'TrafficJunction',
        'Action' => 'Pulse',
        'Data' => [
            'UTC' => 1786543200,
            'Lane' => 1,
            'TrafficCar' => ['PlateNumber' => $plate],
        ],
    ];
}

private function mockSnapshotClient(): void
{
    $mock = Mockery::mock(IntelbrasSnapshotClient::class);
    $mock->shouldReceive('capture')->andReturnUsing(fn ($camera) => \App\Data\Cameras\SnapshotData::fromBytes(
        bytes: 'snapshot-'.$camera->uuid,
        contentType: 'image/jpeg',
        capturedAt: \Carbon\CarbonImmutable::parse('2026-08-12 15:00:01'),
        sourceCameraUuid: $camera->uuid,
    ));

    $this->app->instance(IntelbrasSnapshotClient::class, $mock);
}
```

- [ ] **Step 2: Run webhook tests and verify they fail**

Run:

```bash
php artisan test --filter=IntelbrasLprWebhookTest
```

Expected: FAIL because route, middleware, request, controller, and action do not exist.

- [ ] **Step 3: Implement edge-only middleware**

Create `EnsureTrackVisionEdgeNode`:

```php
<?php

namespace App\Http\Middleware;

use App\Enums\NodeRole;
use App\Support\NodeRoleResolver;
use Closure;
use Illuminate\Http\Request;

class EnsureTrackVisionEdgeNode
{
    public function handle(Request $request, Closure $next): mixed
    {
        abort_unless(NodeRoleResolver::current() === NodeRole::Edge, 404);

        return $next($request);
    }
}
```

- [ ] **Step 4: Implement Form Request auth**

Create `IntelbrasLprWebhookRequest`:

```php
<?php

namespace App\Http\Requests\Api\V1\Edge;

use Illuminate\Foundation\Http\FormRequest;

class IntelbrasLprWebhookRequest extends FormRequest
{
    public function authorize(): bool
    {
        $username = (string) config('trackvision.intelbras.webhook_username');
        $password = (string) config('trackvision.intelbras.webhook_password');

        return $username !== ''
            && $password !== ''
            && hash_equals($username, (string) $this->getUser())
            && hash_equals($password, (string) $this->getPassword())
            && (bool) $this->route('cameraPair')?->is_active;
    }

    public function rules(): array
    {
        return [];
    }
}
```

- [ ] **Step 5: Implement webhook action and controller**

Create `HandleIntelbrasLprWebhookAction`:

```php
public function __construct(
    private readonly IntelbrasWebhookParser $parser,
    private readonly ProcessLprEventAction $processLprEventAction,
) {}

public function execute(CameraPair $cameraPair, Request $request): ?CaptureEvent
{
    $event = $this->parser->parse($request);

    if ($event->eventCode !== config('trackvision.intelbras.event_code', 'TrafficJunction')) {
        return null;
    }

    return $this->processLprEventAction->execute($cameraPair, $event);
}
```

Create `IntelbrasLprWebhookController`:

```php
public function __invoke(
    IntelbrasLprWebhookRequest $request,
    CameraPair $cameraPair,
    HandleIntelbrasLprWebhookAction $action,
): Response {
    $action->execute($cameraPair, $request);

    return response('OK');
}
```

- [ ] **Step 6: Register the route**

Modify `routes/api.php`:

```php
use App\Http\Controllers\Api\V1\Edge\IntelbrasLprWebhookController;
use App\Http\Middleware\EnsureTrackVisionEdgeNode;
```

Inside `Route::prefix('v1')->group(...)`, add before authenticated route groups:

```php
Route::post('/edge/intelbras/camera-pairs/{cameraPair:uuid}/lpr-events', IntelbrasLprWebhookController::class)
    ->middleware(EnsureTrackVisionEdgeNode::class);
```

- [ ] **Step 7: Run webhook and route tests**

Run:

```bash
php artisan test --filter=IntelbrasLprWebhookTest
php artisan route:list --path=api/v1/edge/intelbras
```

Expected: tests PASS and route list includes `POST api/v1/edge/intelbras/camera-pairs/{cameraPair}/lpr-events`.

- [ ] **Step 8: Commit Task 5**

Run:

```bash
git add routes/api.php app/Http/Middleware/EnsureTrackVisionEdgeNode.php app/Http/Requests/Api/V1/Edge app/Http/Controllers/Api/V1/Edge app/Actions/Edge/HandleIntelbrasLprWebhookAction.php tests/Feature/Edge/IntelbrasLprWebhookTest.php
git commit -m "feat: add intelbras lpr webhook endpoint"
```

---

### Task 6: Event Stream Fallback Command

**Files:**

- Create: `RIALMA-TrackVision-Backend/app/Services/Cameras/Intelbras/IntelbrasEventStreamClient.php`
- Create: `RIALMA-TrackVision-Backend/app/Console/Commands/EdgeListenIntelbrasCommand.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/Cameras/Intelbras/IntelbrasEventStreamClientTest.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Edge/EdgeListenIntelbrasCommandTest.php`

**Interfaces:**

- Consumes: `IntelbrasHttpClient::buildUrl(Camera $camera, string $path): string`
- Consumes: `TrafficEventParser::parse(array|string $payload, ?string $lprImageBytes = null, ?string $lprImageContentType = null): TrafficCaptureData`
- Consumes: `ProcessLprEventAction::execute(CameraPair $cameraPair, TrafficCaptureData $event): ?CaptureEvent`
- Produces: `IntelbrasHttpClient::stream(Camera $camera, string $path, array $query, callable $onChunk): void`
- Produces: `IntelbrasEventStreamClient::attachUrl(Camera $camera): string`
- Produces: `IntelbrasEventStreamClient::eventsFromMultipartBody(string $body): array`
- Produces: `IntelbrasEventStreamClient::listen(Camera $camera, callable $onEvent): void`
- Produces command `edge:listen-intelbras {cameraPairUuid}`

- [ ] **Step 1: Write failing event stream unit test**

Create `IntelbrasEventStreamClientTest.php`:

```php
<?php

namespace Tests\Unit\Cameras\Intelbras;

use App\Enums\CameraType;
use App\Models\Camera;
use App\Services\Cameras\Intelbras\IntelbrasEventStreamClient;
use App\Services\Cameras\Intelbras\IntelbrasHttpClient;
use App\Services\Cameras\Intelbras\TrafficEventParser;
use App\Support\Vehicles\NormalizePlate;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class IntelbrasEventStreamClientTest extends TestCase
{
    use RefreshDatabase;

    public function test_builds_attach_file_proc_url_for_traffic_junction(): void
    {
        $camera = Camera::factory()->create(['type' => CameraType::Lpr, 'host' => '192.168.10.10', 'port' => 80, 'channel' => 1]);
        $client = $this->client();

        $url = $client->attachUrl($camera);

        $this->assertStringContainsString('/cgi-bin/snapManager.cgi?action=attachFileProc', $url);
        $this->assertStringContainsString('Events=%5BTrafficJunction%5D', $url);
        $this->assertStringContainsString('heartbeat=5', $url);
    }

    public function test_extracts_traffic_event_from_multipart_body(): void
    {
        $body = "--boundary\r\nContent-Type: text/plain\r\n\r\nEvents[0].EventBaseInfo.Code=TrafficJunction\r\nEvents[0].EventBaseInfo.Action=Pulse\r\nEvents[0].TrafficCar.PlateNumber=ABC1D23\r\n--boundary--";

        $events = $this->client()->eventsFromMultipartBody($body);

        $this->assertCount(1, $events);
        $this->assertSame('ABC1D23', $events[0]->plateNormalized);
    }

    public function test_listen_streams_chunks_into_event_callback(): void
    {
        $camera = Camera::factory()->create(['type' => CameraType::Lpr]);
        $http = \Mockery::mock(IntelbrasHttpClient::class);
        $http->shouldReceive('stream')->once()->andReturnUsing(function ($camera, $path, $query, $onChunk): void {
            $onChunk("--boundary\r\nContent-Type: text/plain\r\n\r\nEvents[0].EventBaseInfo.Code=TrafficJunction\r\nEvents[0].EventBaseInfo.Action=Pulse\r\nEvents[0].TrafficCar.PlateNumber=ABC1D23\r\n");
            $onChunk("--boundary--");
        });

        $client = new IntelbrasEventStreamClient($http, new TrafficEventParser(new NormalizePlate));
        $received = [];

        $client->listen($camera, function ($event) use (&$received): void {
            $received[] = $event;
        });

        $this->assertSame('ABC1D23', $received[0]->plateNormalized);
    }

    private function client(): IntelbrasEventStreamClient
    {
        return new IntelbrasEventStreamClient(new IntelbrasHttpClient, new TrafficEventParser(new NormalizePlate));
    }
}
```

- [ ] **Step 2: Write failing command test**

Create `EdgeListenIntelbrasCommandTest.php`:

```php
<?php

namespace Tests\Feature\Edge;

use App\Models\CameraPair;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class EdgeListenIntelbrasCommandTest extends TestCase
{
    use RefreshDatabase;

    public function test_command_rejects_parent_profile(): void
    {
        config(['trackvision.node_role' => 'parent']);
        $pair = CameraPair::factory()->create();

        $this->artisan("edge:listen-intelbras {$pair->uuid}")
            ->expectsOutput('This command can only run on an edge node.')
            ->assertExitCode(1);
    }

    public function test_command_rejects_inactive_camera_pair(): void
    {
        config(['trackvision.node_role' => 'edge']);
        $pair = CameraPair::factory()->create(['is_active' => false]);

        $this->artisan("edge:listen-intelbras {$pair->uuid}")
            ->expectsOutput('Camera pair is inactive or unavailable.')
            ->assertExitCode(1);
    }

    public function test_command_passes_active_pair_lpr_camera_to_stream_client(): void
    {
        config(['trackvision.node_role' => 'edge']);
        $pair = CameraPair::factory()->create(['is_active' => true]);

        $stream = \Mockery::mock(\App\Services\Cameras\Intelbras\IntelbrasEventStreamClient::class);
        $stream->shouldReceive('listen')->once()->withArgs(fn ($camera, $callback) => $camera->is($pair->lprCamera) && is_callable($callback));
        $this->app->instance(\App\Services\Cameras\Intelbras\IntelbrasEventStreamClient::class, $stream);

        $this->artisan("edge:listen-intelbras {$pair->uuid}")
            ->expectsOutput('Listening to Intelbras TrafficJunction events.')
            ->assertExitCode(0);
    }
}
```

- [ ] **Step 3: Run event stream tests and verify they fail**

Run:

```bash
php artisan test --filter=IntelbrasEventStreamClientTest
php artisan test --filter=EdgeListenIntelbrasCommandTest
```

Expected: FAIL because the client and command do not exist.

- [ ] **Step 4: Extend `IntelbrasHttpClient` with streaming**

Add this method to `IntelbrasHttpClient`:

```php
public function stream(Camera $camera, string $path, array $query, callable $onChunk): void
{
    $client = new \GuzzleHttp\Client([
        'timeout' => 0,
        'read_timeout' => ((int) config('trackvision.intelbras.event_stream_heartbeat_seconds', 5)) + 10,
    ]);

    $response = $client->request('GET', $this->buildUrl($camera, $path), [
        'auth' => [(string) $camera->username, (string) $camera->password_encrypted, 'digest'],
        'query' => $query,
        'stream' => true,
    ]);

    $body = $response->getBody();

    while (! $body->eof()) {
        $chunk = $body->read(8192);

        if ($chunk !== '') {
            $onChunk($chunk);
        }
    }
}
```

- [ ] **Step 5: Implement event stream fallback client**

Create `IntelbrasEventStreamClient`:

```php
public function __construct(
    private readonly IntelbrasHttpClient $httpClient,
    private readonly TrafficEventParser $trafficEventParser,
) {}

public function attachUrl(Camera $camera): string
{
    return $this->httpClient->buildUrl($camera, '/cgi-bin/snapManager.cgi').'?'.http_build_query([
        'action' => 'attachFileProc',
        'channel' => $camera->channel ?: 1,
        'heartbeat' => (int) config('trackvision.intelbras.event_stream_heartbeat_seconds', 5),
        'Flags[0]' => 'Event',
        'Events' => '['.config('trackvision.intelbras.event_code', 'TrafficJunction').']',
    ]);
}

public function listen(Camera $camera, callable $onEvent): void
{
    $buffer = '';

    $this->httpClient->stream($camera, '/cgi-bin/snapManager.cgi', $this->attachQuery($camera), function (string $chunk) use (&$buffer, $onEvent): void {
        $buffer .= $chunk;

        foreach ($this->eventsFromMultipartBody($buffer) as $event) {
            $onEvent($event);
        }

        if (str_contains($buffer, '--boundary--')) {
            $buffer = '';
        }
    });
}

public function eventsFromMultipartBody(string $body): array
{
    $events = [];

    foreach (preg_split('/--[^\r\n]+/', $body) as $part) {
        if (str_contains($part, 'TrafficJunction')) {
            $events[] = $this->trafficEventParser->parse(trim(preg_replace('/^Content-Type:.*?\r\n\r\n/s', '', $part)));
        }
    }

    return $events;
}

private function attachQuery(Camera $camera): array
{
    return [
        'action' => 'attachFileProc',
        'channel' => $camera->channel ?: 1,
        'heartbeat' => (int) config('trackvision.intelbras.event_stream_heartbeat_seconds', 5),
        'Flags[0]' => 'Event',
        'Events' => '['.config('trackvision.intelbras.event_code', 'TrafficJunction').']',
    ];
}
```

- [ ] **Step 6: Implement fallback command**

Create `EdgeListenIntelbrasCommand`:

```php
protected $signature = 'edge:listen-intelbras {cameraPairUuid}';
protected $description = 'Listen to Intelbras LPR events through attachFileProc fallback.';

public function handle(IntelbrasEventStreamClient $streamClient, ProcessLprEventAction $processLprEventAction): int
{
    if (NodeRoleResolver::current() !== NodeRole::Edge) {
        $this->error('This command can only run on an edge node.');
        return self::FAILURE;
    }

    $pair = CameraPair::query()
        ->with(['lprCamera', 'supportCamera'])
        ->where('uuid', $this->argument('cameraPairUuid'))
        ->where('is_active', true)
        ->first();

    if (! $pair) {
        $this->error('Camera pair is inactive or unavailable.');
        return self::FAILURE;
    }

    $this->info('Listening to Intelbras TrafficJunction events.');
    $streamClient->listen($pair->lprCamera, function (TrafficCaptureData $event) use ($pair, $processLprEventAction): void {
        $processLprEventAction->execute($pair, $event);
    });

    return self::SUCCESS;
}
```

- [ ] **Step 7: Run event stream tests**

Run:

```bash
php artisan test --filter=IntelbrasEventStreamClientTest
php artisan test --filter=EdgeListenIntelbrasCommandTest
```

Expected: PASS.

- [ ] **Step 8: Commit Task 6**

Run:

```bash
git add app/Services/Cameras/Intelbras/IntelbrasHttpClient.php app/Services/Cameras/Intelbras/IntelbrasEventStreamClient.php app/Console/Commands/EdgeListenIntelbrasCommand.php tests/Unit/Cameras/Intelbras/IntelbrasEventStreamClientTest.php tests/Feature/Edge/EdgeListenIntelbrasCommandTest.php
git commit -m "feat: add intelbras event stream fallback"
```

---

### Task 7: Field Checklist, Full Verification, and Documentation Commit

**Files:**

- Create: `RIALMA-TrackVision-AgentOps/docs/operations/intelbras-vip-5460-field-checklist.md`
- Modify: `RIALMA-TrackVision-AgentOps/docs/superpowers/plans/2026-08-12-trackvision-system-implementation.md`
- Modify: `RIALMA-TrackVision-Backend/README.md`

**Interfaces:**

- Consumes: implemented endpoint `POST /api/v1/edge/intelbras/camera-pairs/{cameraPair:uuid}/lpr-events`
- Consumes: implemented command `edge:listen-intelbras {cameraPairUuid}`
- Produces: field checklist for VIP 5460 LPR IA setup
- Produces: backend README section with environment variables and verification commands

- [ ] **Step 1: Write field checklist in AgentOps**

Create `docs/operations/intelbras-vip-5460-field-checklist.md` with:

```markdown
# Intelbras VIP 5460 LPR IA Field Checklist

## Dados Necessarios

- IP local do backend edge.
- Porta HTTP ou HTTPS do backend edge.
- UUID do camera pair cadastrado no TrackVision.
- Usuario Basic Auth dedicado para upload Intelbras.
- Senha Basic Auth dedicada para upload Intelbras.
- IP, porta, canal, usuario e senha da VIP 5460 LPR IA.
- IP, porta, canal, usuario e senha da camera de apoio.

## Configuracao Recomendada

- Habilitar `PictureHttpUpload` quando disponivel.
- Evento: `TrafficJunction`.
- Destino: `/api/v1/edge/intelbras/camera-pairs/{camera_pair_uuid}/lpr-events`.
- Autenticacao: Basic Auth dedicado.

## Validacao

- Passar um veiculo cadastrado pela LPR.
- Confirmar `capture_events.status=accepted`.
- Confirmar uma midia `lpr_image`.
- Confirmar uma midia `support_image`.
- Desligar internet do local e repetir passagem.
- Confirmar que o evento fica em `edge_outbox_messages.status=pending`.
```

- [ ] **Step 2: Update backend README with environment variables**

Add a section:

~~~markdown
## Intelbras Edge Integration

Required edge environment variables:

```text
TRACKVISION_NODE_ROLE=edge
TRACKVISION_EDGE_NODE_UUID=
TRACKVISION_EDGE_STORE_UNKNOWN_PLATES=false
TRACKVISION_INTELBRAS_WEBHOOK_USERNAME=
TRACKVISION_INTELBRAS_WEBHOOK_PASSWORD=
TRACKVISION_INTELBRAS_LPR_EVENT_CODE=TrafficJunction
TRACKVISION_INTELBRAS_SNAPSHOT_TYPE=0
TRACKVISION_INTELBRAS_EVENT_STREAM_HEARTBEAT_SECONDS=5
TRACKVISION_INTELBRAS_EVENT_DEDUPE_WINDOW_SECONDS=30
```

Verification commands:

```bash
php artisan test --filter=Intelbras
php artisan test --filter=ProcessLprEventTest
php artisan test --filter=IntelbrasLprWebhookTest
php artisan route:list --path=api/v1/edge/intelbras
```
~~~

- [ ] **Step 3: Update master plan status language**

In `docs/superpowers/plans/2026-08-12-trackvision-system-implementation.md`, keep Task 6 pointing to the design and this implementation plan:

```markdown
**Implementation Plan:** `docs/superpowers/plans/2026-08-12-intelbras-camera-adapter-implementation.md`
```

- [ ] **Step 4: Run full backend verification**

Run in `RIALMA-TrackVision-Backend`:

```bash
composer validate
php artisan test
php artisan route:list --path=api/v1/edge/intelbras
```

Expected: `composer validate` returns success, all backend tests pass, and route list shows the Intelbras webhook route.

- [ ] **Step 5: Check both repositories**

Run:

```bash
git -C C:/projetos/rialma/RIALMA-TrackVision-AgentOps status --short --branch
git -C C:/projetos/rialma/RIALMA-TrackVision-AgentOps/RIALMA-TrackVision-Backend status --short --branch
```

Expected: only files from this Intelbras phase are modified or staged.

- [ ] **Step 6: Commit backend implementation**

Run in `RIALMA-TrackVision-Backend`:

```bash
git add .
git commit -m "feat: add intelbras camera adapter"
```

- [ ] **Step 7: Commit AgentOps documentation**

Run in `RIALMA-TrackVision-AgentOps`:

```bash
git add docs/operations/intelbras-vip-5460-field-checklist.md docs/superpowers/plans/2026-08-12-trackvision-system-implementation.md
git commit -m "docs: add intelbras adapter implementation plan"
```

---

## Verification Matrix

Backend focused checks:

```bash
php artisan test --filter=TrafficCaptureDataTest
php artisan test --filter=TrafficEventParserTest
php artisan test --filter=IntelbrasWebhookParserTest
php artisan test --filter=IntelbrasSnapshotClientTest
php artisan test --filter=ProcessLprEventTest
php artisan test --filter=IntelbrasLprWebhookTest
php artisan test --filter=IntelbrasEventStreamClientTest
php artisan test --filter=EdgeListenIntelbrasCommandTest
```

Backend full checks:

```bash
composer validate
php artisan test
php artisan route:list --path=api/v1/edge/intelbras
```

Manual edge check after deployment:

```bash
curl -u "$TRACKVISION_INTELBRAS_WEBHOOK_USERNAME:$TRACKVISION_INTELBRAS_WEBHOOK_PASSWORD" \
  -H "Content-Type: application/json" \
  -d '{"Code":"TrafficJunction","Action":"Pulse","Data":{"UTC":1786543200,"TrafficCar":{"PlateNumber":"ABC-1D23"}}}' \
  http://EDGE_HOST/api/v1/edge/intelbras/camera-pairs/CAMERA_PAIR_UUID/lpr-events
```

Expected curl response:

```text
OK
```

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-08-12-intelbras-camera-adapter-implementation.md`.

Two execution options:

1. **Subagent-Driven (recommended)** - dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** - execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
