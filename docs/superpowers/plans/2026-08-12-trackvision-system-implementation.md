# TrackVision System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build RIALMA TrackVision as a parent + local edge vehicle monitoring system that manages users, permissions, vehicles, Intelbras camera pairs, offline captures, parent synchronization, trip classification, load review, and image-backed reports.

**Architecture:** Use a Laravel backend deployed in two profiles: `parent` for the central server and `edge` for the local site server. The edge profile listens to Intelbras LPR events, checks if the plate belongs to an active registered vehicle, captures the support-camera image, stores the capture locally, and synchronizes idempotently to the parent when internet is available. The Vue frontend talks to the parent API for admin, operational dashboards, trip review, and reports.

**Tech Stack:** Laravel 13, PHP 8.4, Laravel Passport, Spatie Laravel Permission, PostgreSQL, Laravel queues, private media storage, Vue 3, Vite, TypeScript, Pinia, Vue Router, Vitest, Vue Test Utils, Playwright.

## Global Constraints

- Backend must follow `docs/backend-laravel-guidelines.md`.
- Frontend must follow `docs/frontend-vue-guidelines.md`.
- Backend code lives only in `RIALMA-TrackVision-Backend`.
- Frontend code lives only in `RIALMA-TrackVision-Frontend`.
- AgentOps documentation lives only in `RIALMA-TrackVision-AgentOps`.
- Use Laravel Passport for API token authentication and edge machine-to-machine client credentials.
- Use Spatie Laravel Permission for roles and permissions.
- Controllers stay thin; business logic goes into Services or Actions.
- Form Requests validate API input.
- API responses use Resources.
- Vue components stay small; API calls go through services/clients.
- Intelbras device credentials, parent sync secrets, JWT/OAuth secrets, database passwords, and camera passwords must never be committed.
- The first load-status workflow is manual review from support-camera images: `unknown`, `loaded`, `empty`, or `needs_review`.
- The first direction workflow uses the configured camera-pair direction: `outbound`, `inbound`, or `unknown`.
- Unknown plates may be stored as local edge audit events, but parent trip/report flows only use registered active vehicles.

---

## Scope Check

This is a multi-subsystem product. Implement it in phases. Each phase must finish with working, testable software, a focused commit in the correct repository, and updated documentation where behavior or contracts change.

Recommended phase order:

1. Backend foundation and security.
2. Parent admin domain.
3. Frontend admin shell.
4. Camera and location modeling.
5. Intelbras integration adapter, using webhook-first delivery for VIP 5460 LPR IA and event-stream fallback when required by firmware or field setup.
6. Edge local capture engine.
7. Edge-to-parent synchronization.
8. Trip classification and load review.
9. Reports and audit trail.
10. Deployment, operations, and field validation.

Before implementing any large phase, create a detailed phase plan from this master plan.

## System Topology

```text
Intelbras LPR Camera ----\
                         > Edge Backend ---- local DB/media ---- sync queue ---- Parent Backend ---- Vue Admin
Intelbras Support Camera /
```

Parent backend responsibilities:

- authoritative users, roles, permissions, vehicles, locations, cameras, edge nodes, captures, trips, reports, and audit logs;
- admin API consumed by Vue;
- edge sync API consumed by local edge backends;
- trip grouping and report generation.

Edge backend responsibilities:

- local copy of active vehicles and camera-pair configuration;
- Intelbras LPR listener;
- support-camera snapshot capture;
- local storage for captures and media;
- durable outbox for offline sync;
- idempotent sync with the parent server.

Vue frontend responsibilities:

- admin login and permission-aware navigation;
- user, role, permission, vehicle, location, camera, and camera-pair management;
- capture/trip review;
- load-status marking from support-camera image;
- reports with LPR image, support image, direction, load status, and trip data.

## Domain Model

Core parent tables:

- `users`
- `roles`, `permissions`, and Spatie pivot tables
- `vehicles`
- `locations`
- `edge_nodes`
- `cameras`
- `camera_pairs`
- `capture_events`
- `media_assets`
- `trips`
- `trip_events`
- `audit_logs`
- `edge_sync_batches`

Core edge-only tables:

- `edge_outbox_messages`
- `edge_sync_state`
- local copies of active `vehicles`, `cameras`, and `camera_pairs`

Important enums:

- `NodeRole`: `parent`, `edge`
- `CameraType`: `lpr`, `support`
- `CameraVendor`: `intelbras`
- `CameraPairDirection`: `outbound`, `inbound`, `unknown`
- `CaptureStatus`: `accepted`, `ignored_unknown_plate`, `failed_support_capture`, `synced`
- `LoadStatus`: `unknown`, `loaded`, `empty`, `needs_review`
- `TripStatus`: `open`, `closed`, `needs_review`

## File Structure Map

Backend:

```text
RIALMA-TrackVision-Backend/
|-- app/
|   |-- Actions/
|   |   |-- Admin/
|   |   |-- Cameras/
|   |   |-- Edge/
|   |   |-- Reports/
|   |   |-- Trips/
|   |   `-- Vehicles/
|   |-- Data/
|   |-- Enums/
|   |-- Http/
|   |   |-- Controllers/Api/V1/
|   |   |-- Requests/Api/V1/
|   |   `-- Resources/Api/V1/
|   |-- Models/
|   |-- Policies/
|   |-- Services/
|   |   |-- Cameras/Intelbras/
|   |   |-- EdgeSync/
|   |   |-- Media/
|   |   `-- Trips/
|   `-- Support/
|-- config/trackvision.php
|-- database/
|-- routes/api.php
|-- tests/Feature/
`-- tests/Unit/
```

Frontend:

```text
RIALMA-TrackVision-Frontend/
|-- src/
|   |-- components/
|   |   |-- base/
|   |   |-- cameras/
|   |   |-- trips/
|   |   |-- users/
|   |   `-- vehicles/
|   |-- composables/
|   |-- layouts/
|   |-- pages/
|   |-- router/
|   |-- services/
|   |-- stores/
|   |-- types/
|   `-- utils/
|-- tests/
`-- playwright/
```

---

## Task 1: Backend Foundation and Runtime Profiles

**Files:**

- Create: `RIALMA-TrackVision-Backend/config/trackvision.php`
- Create: `RIALMA-TrackVision-Backend/app/Enums/NodeRole.php`
- Create: `RIALMA-TrackVision-Backend/app/Support/NodeRoleResolver.php`
- Create: `RIALMA-TrackVision-Backend/tests/Unit/Support/NodeRoleResolverTest.php`
- Modify: `RIALMA-TrackVision-Backend/.env.example`
- Modify: `RIALMA-TrackVision-Backend/README.md`

**Interfaces:**

- Produces: `NodeRole::Parent`
- Produces: `NodeRole::Edge`
- Produces: `NodeRoleResolver::current(): NodeRole`
- Consumes: `TRACKVISION_NODE_ROLE=parent|edge`

**Steps:**

- [ ] Scaffold Laravel 13 in `RIALMA-TrackVision-Backend`.
- [ ] Configure PostgreSQL for runtime and SQLite in-memory for tests.
- [ ] Add `TRACKVISION_NODE_ROLE=parent` to `.env.example`.
- [ ] Add `config/trackvision.php` with `node_role`, `parent_api_url`, `edge_node_uuid`, `camera_default_timeout_seconds`, and `sync_batch_size`.
- [ ] Write `NodeRoleResolverTest` for `parent`, `edge`, and invalid values.
- [ ] Implement `NodeRole` enum and `NodeRoleResolver`.
- [ ] Run `php artisan test --filter=NodeRoleResolverTest`.
- [ ] Commit as `chore: scaffold laravel backend foundation`.

**Acceptance Criteria:**

- Backend app boots.
- Test suite passes.
- Runtime profile is explicit.
- `.env.example` contains no secrets.

---

## Task 2: Authentication With Laravel Passport

**Files:**

- Modify: `RIALMA-TrackVision-Backend/composer.json`
- Modify: `RIALMA-TrackVision-Backend/app/Models/User.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Auth/LoginController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Auth/LogoutController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Auth/LoginRequest.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Resources/Api/V1/UserResource.php`
- Modify: `RIALMA-TrackVision-Backend/routes/api.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Auth/LoginTest.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Auth/EdgeClientCredentialsTest.php`

**Interfaces:**

- Produces: `POST /api/v1/auth/login`
- Produces: `POST /api/v1/auth/logout`
- Produces: `GET /api/v1/me`
- Produces: scoped OAuth client credentials for edge nodes.

**Steps:**

- [ ] Install Laravel Passport.
- [ ] Run Passport migrations.
- [ ] Configure Passport scopes: `admin:read`, `admin:write`, `edge:read`, `edge:write`, `captures:write`, `reports:read`.
- [ ] Add Passport API token behavior to `User`.
- [ ] Write `LoginTest` for valid login, invalid login, logout, and authenticated current-user endpoint.
- [ ] Write `EdgeClientCredentialsTest` proving an edge client can call edge bootstrap and cannot call admin user endpoints.
- [ ] Implement Form Request validation and thin auth controllers.
- [ ] Run `php artisan test --filter=Auth`.
- [ ] Commit as `feat: add passport authentication`.

**Acceptance Criteria:**

- Admin users authenticate through API.
- Edge nodes authenticate with client credentials.
- Tokens are scoped.
- Password grant is not required for machine clients.

---

## Task 3: Roles and Permissions With Spatie

**Files:**

- Modify: `RIALMA-TrackVision-Backend/composer.json`
- Modify: `RIALMA-TrackVision-Backend/app/Models/User.php`
- Create: `RIALMA-TrackVision-Backend/app/Support/Permissions/PermissionCatalog.php`
- Create: `RIALMA-TrackVision-Backend/database/seeders/PermissionSeeder.php`
- Create: `RIALMA-TrackVision-Backend/database/seeders/AdminUserSeeder.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/UserController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/RoleController.php`
- Create: `RIALMA-TrackVision-Backend/app/Http/Controllers/Api/V1/Admin/PermissionController.php`
- Create: `RIALMA-TrackVision-Backend/tests/Feature/Admin/RbacTest.php`

**Interfaces:**

- Produces roles: `super_admin`, `admin`, `operator`, `auditor`, `edge_service`.
- Produces permissions: `users.manage`, `permissions.manage`, `vehicles.manage`, `cameras.manage`, `captures.view`, `trips.manage`, `reports.view`, `edge.sync`.
- Produces admin CRUD endpoints protected by permission middleware.

**Steps:**

- [ ] Install `spatie/laravel-permission`.
- [ ] Publish and run package migrations.
- [ ] Add `HasRoles` to `User`.
- [ ] Create `PermissionCatalog` with role-permission mapping.
- [ ] Seed roles and permissions idempotently.
- [ ] Seed first admin user from safe environment values.
- [ ] Write tests proving `operator` cannot manage permissions, `admin` can manage vehicles, and `super_admin` has all permissions.
- [ ] Implement User, Role, and Permission controllers with Form Requests and Resources.
- [ ] Run `php artisan test --filter=RbacTest`.
- [ ] Commit as `feat: add role based access control`.

**Acceptance Criteria:**

- Permissions are database-backed.
- UI can list roles and permissions.
- Admin routes are protected by Spatie permissions.
- Seeders are repeatable.

---

## Task 4: Parent Admin Domain

**Files:**

- Create migrations for `vehicles`, `locations`, `edge_nodes`, `cameras`, and `camera_pairs`.
- Create models: `Vehicle`, `Location`, `EdgeNode`, `Camera`, `CameraPair`.
- Create enums: `CameraType`, `CameraVendor`, `CameraPairDirection`, `CameraConnectionStatus`.
- Create controllers under `app/Http/Controllers/Api/V1/Admin`.
- Create requests under `app/Http/Requests/Api/V1/Admin`.
- Create resources under `app/Http/Resources/Api/V1`.
- Create tests under `tests/Feature/Admin`.

**Interfaces:**

- Produces: `GET|POST /api/v1/admin/vehicles`
- Produces: `GET|PATCH|DELETE /api/v1/admin/vehicles/{vehicle}`
- Produces: `GET|POST /api/v1/admin/locations`
- Produces: `GET|POST /api/v1/admin/edge-nodes`
- Produces: `GET|POST /api/v1/admin/cameras`
- Produces: `GET|POST /api/v1/admin/camera-pairs`

**Steps:**

- [ ] Create migrations with foreign keys, UUID public identifiers, and unique indexes.
- [ ] Store normalized license plates in `vehicles.plate_normalized`.
- [ ] Store camera credentials encrypted and never expose them through Resources.
- [ ] Require each camera pair to contain exactly one `lpr` camera and one `support` camera.
- [ ] Add `camera_pairs.direction` with `outbound`, `inbound`, or `unknown`.
- [ ] Write tests for creating a vehicle, rejecting duplicate plate, creating a valid camera pair, rejecting two LPR cameras in one pair, and forbidding unauthorized users.
- [ ] Implement thin CRUD controllers with Form Requests and Resources.
- [ ] Run `php artisan test --filter=Admin`.
- [ ] Commit as `feat: add parent admin domain`.

**Acceptance Criteria:**

- Parent can manage users, permissions, vehicles, locations, edge nodes, cameras, and camera pairs.
- Vehicle plate normalization is deterministic.
- Camera credentials never appear in API responses.

---

## Task 5: Frontend Admin Shell

**Files:**

- Create Vue 3 + Vite app in `RIALMA-TrackVision-Frontend`.
- Create: `src/services/apiClient.ts`
- Create: `src/services/authService.ts`
- Create: `src/stores/authStore.ts`
- Create: `src/router/index.ts`
- Create pages: `LoginPage.vue`, `DashboardPage.vue`, `UsersPage.vue`, `RolesPage.vue`, `VehiclesPage.vue`, `LocationsPage.vue`, `CamerasPage.vue`, `CameraPairsPage.vue`.
- Create base components: `BaseButton.vue`, `BaseInput.vue`, `BaseTable.vue`, `BaseModal.vue`, `BaseAlert.vue`.
- Create tests under `tests/unit` and `playwright`.

**Interfaces:**

- Consumes parent auth and admin endpoints.
- Produces authenticated admin panel shell.

**Steps:**

- [ ] Scaffold Vue 3 + Vite + TypeScript.
- [ ] Configure Pinia, Vue Router, Vitest, Vue Test Utils, and Playwright.
- [ ] Implement `apiClient` with bearer token handling, error normalization, and `VITE_API_BASE_URL`.
- [ ] Implement `authStore` with login, logout, current user, roles, and permissions.
- [ ] Create route guards based on authentication and permission meta fields.
- [ ] Create admin layout with permission-aware navigation.
- [ ] Create CRUD pages for users, roles, vehicles, locations, cameras, and camera pairs.
- [ ] Add loading, error, empty, and success states in each page.
- [ ] Write tests for login, route guard, and vehicle list rendering.
- [ ] Run `npm run test` and `npm run build`.
- [ ] Commit as `feat: add vue admin shell`.

**Acceptance Criteria:**

- Admin can log in.
- Navigation respects permissions.
- Admin pages consume services, not direct fetches inside components.
- Vue style guide rules are followed.

---

## Task 6: Intelbras Camera Adapter

**Design Source:** `docs/superpowers/specs/2026-08-12-intelbras-webhook-camera-adapter-design.md`

**Files:**

- Create: `app/Services/Cameras/Intelbras/IntelbrasHttpClient.php`
- Create: `app/Services/Cameras/Intelbras/IntelbrasSnapshotClient.php`
- Create: `app/Services/Cameras/Intelbras/IntelbrasEventStreamClient.php`
- Create: `app/Services/Cameras/Intelbras/IntelbrasWebhookParser.php`
- Create: `app/Services/Cameras/Intelbras/TrafficEventParser.php`
- Create: `app/Data/Cameras/TrafficCaptureData.php`
- Create: `app/Data/Cameras/SnapshotData.php`
- Create: `app/Http/Controllers/Api/V1/Edge/IntelbrasLprWebhookController.php`
- Create: `app/Http/Requests/Api/V1/Edge/IntelbrasLprWebhookRequest.php`
- Create: `app/Actions/Edge/HandleIntelbrasLprWebhookAction.php`
- Create: `tests/Unit/Cameras/Intelbras/TrafficEventParserTest.php`
- Create: `tests/Unit/Cameras/Intelbras/IntelbrasWebhookParserTest.php`
- Create: `tests/Unit/Cameras/Intelbras/IntelbrasSnapshotClientTest.php`
- Create: `tests/Feature/Edge/IntelbrasLprWebhookTest.php`

**Interfaces:**

- Produces: `IntelbrasSnapshotClient::capture(Camera $camera): SnapshotData`
- Produces: `POST /api/v1/edge/intelbras/camera-pairs/{cameraPair:uuid}/lpr-events`
- Produces: `TrafficEventParser::parse(string $payload): TrafficCaptureData`
- Produces event data: `event_code`, `action`, `plate_number`, `plate_normalized`, `event_time`, `lane`, `group_id`, `index_in_group`, `raw_payload`, `lpr_image_bytes`.

**Steps:**

- [ ] Implement an HTTP client with timeout and Digest Auth support.
- [ ] Implement Basic Auth verification for Intelbras webhook uploads using edge-only environment values.
- [ ] Implement local webhook endpoint for VIP 5460 LPR IA `TrafficJunction` events.
- [ ] Parse `PictureHttpUpload` multipart payloads when the camera sends event plus JPEG.
- [ ] Parse `EventHttpUpload` JSON payloads when the camera sends event without image.
- [ ] Implement support-camera snapshot capture through Intelbras HTTP API snapshot endpoint.
- [ ] Capture an LPR snapshot when the webhook payload does not include an LPR image.
- [ ] Keep LPR event subscription with `snapManager.cgi?action=attachFileProc` as operational fallback.
- [ ] Parse traffic/LPR event payload into `TrafficCaptureData`.
- [ ] Treat LPR image and support snapshot as separate media assets.
- [ ] Load camera timeout and retry policy from `config/trackvision.php`.
- [ ] Write parser tests using saved sample payload fixtures.
- [ ] Write webhook tests for registered vehicle, unknown plate, missing LPR image fallback, invalid credentials, and duplicate event dedupe.
- [ ] Write snapshot tests with fake HTTP responses returning JPEG bytes.
- [ ] Run `php artisan test --filter=Intelbras`.
- [ ] Commit as `feat: add intelbras camera adapter`.

**Acceptance Criteria:**

- Adapter does not depend on Controllers.
- Adapter can parse plate number and traffic metadata.
- Snapshot result includes content type, bytes, hash, and captured timestamp.
- HTTP timeout is mandatory.
- Webhook-first path supports VIP 5460 LPR IA `PictureHttpUpload` and `EventHttpUpload`.
- Event-stream subscription remains available as fallback when webhook setup is unavailable.

---

## Task 7: Edge Local Capture Engine

**Files:**

- Create migrations: `capture_events`, `media_assets`, `edge_outbox_messages`, `edge_sync_state`.
- Create models: `CaptureEvent`, `MediaAsset`, `EdgeOutboxMessage`, `EdgeSyncState`.
- Create actions:
  - `Actions/Edge/ProcessLprEventAction.php`
  - `Actions/Edge/CaptureSupportImageAction.php`
  - `Actions/Edge/StoreCaptureMediaAction.php`
  - `Actions/Edge/EnqueueCaptureForSyncAction.php`
- Create command: `Console/Commands/EdgeListenIntelbrasCommand.php`
- Create tests: `tests/Feature/Edge/ProcessLprEventTest.php`

**Interfaces:**

- Consumes: `TrafficCaptureData`
- Produces: local `CaptureEvent`
- Produces: local `MediaAsset` records for LPR and support images
- Produces: outbox message `capture.created`

**Steps:**

- [ ] Create local capture and outbox tables.
- [ ] Implement `ProcessLprEventAction`.
- [ ] If plate is not registered and active, store minimal ignored event only when `trackvision.edge.store_unknown_plates=true`.
- [ ] If plate is registered and active, capture support-camera snapshot.
- [ ] Store LPR image and support image as private media files.
- [ ] Set `capture_events.direction` from `camera_pairs.direction`.
- [ ] Set `capture_events.load_status=unknown`.
- [ ] Create `edge_outbox_messages` with a stable idempotency key.
- [ ] Implement `edge:listen-intelbras --camera-pair={uuid}`.
- [ ] Write Feature test with registered vehicle and fake camera clients.
- [ ] Write Feature test with unknown plate and store-unknown disabled.
- [ ] Run `php artisan test --filter=ProcessLprEventTest`.
- [ ] Commit as `feat: add edge capture engine`.

**Acceptance Criteria:**

- Edge works without internet.
- Registered plate triggers support snapshot capture.
- Unknown plates do not pollute parent trip data.
- Every synced event has a stable idempotency key.

---

## Task 8: Edge-to-Parent Synchronization

**Files:**

- Create parent controllers:
  - `Api/V1/Edge/BootstrapController.php`
  - `Api/V1/Edge/CaptureIngestController.php`
  - `Api/V1/Edge/HeartbeatController.php`
- Create edge actions:
  - `Actions/Edge/SyncFromParentAction.php`
  - `Actions/Edge/PushOutboxBatchAction.php`
- Create parent actions:
  - `Actions/Edge/IngestCaptureBatchAction.php`
  - `Actions/Edge/UpsertEdgeCaptureAction.php`
- Create command: `Console/Commands/EdgeSyncParentCommand.php`
- Create tests: `tests/Feature/Edge/EdgeSyncTest.php`

**Interfaces:**

- Produces: `GET /api/v1/edge/bootstrap`
- Produces: `POST /api/v1/edge/heartbeat`
- Produces: `POST /api/v1/edge/captures/batch`
- Consumes: OAuth client credentials token with `edge:read`, `edge:write`, and `captures:write`.

**Steps:**

- [ ] Implement parent bootstrap endpoint returning active vehicles, camera pairs, and sync cursor for the edge node.
- [ ] Implement heartbeat endpoint storing edge online/offline health.
- [ ] Implement capture batch ingest with idempotency key dedupe.
- [ ] Implement edge sync command that pulls bootstrap data and pushes outbox batches.
- [ ] Mark outbox messages as `synced` only after parent acknowledges them.
- [ ] Retry failed messages with backoff and keep error details.
- [ ] Write tests proving duplicate batch submit does not duplicate captures.
- [ ] Write tests proving edge can queue captures while parent is unavailable.
- [ ] Run `php artisan test --filter=EdgeSyncTest`.
- [ ] Commit as `feat: add edge parent synchronization`.

**Acceptance Criteria:**

- Sync is idempotent.
- Edge can recover after internet outage.
- Parent can identify last heartbeat per edge node.
- Edge receives current registered vehicles and camera config.

---

## Task 9: Trip Classification and Load Review

**Files:**

- Create migrations: `trips`, `trip_events`.
- Create models: `Trip`, `TripEvent`.
- Create actions:
  - `Actions/Trips/AttachCaptureToTripAction.php`
  - `Actions/Trips/ClassifyTripDirectionAction.php`
  - `Actions/Trips/UpdateLoadStatusAction.php`
- Create controllers:
  - `Api/V1/Admin/TripController.php`
  - `Api/V1/Admin/TripLoadReviewController.php`
- Create resources:
  - `TripResource`
  - `TripEventResource`
- Create tests: `tests/Feature/Trips/TripClassificationTest.php`

**Interfaces:**

- Produces: `GET /api/v1/admin/trips`
- Produces: `GET /api/v1/admin/trips/{trip}`
- Produces: `PATCH /api/v1/admin/trip-events/{tripEvent}/load-status`
- Consumes: ingested `CaptureEvent`

**Steps:**

- [ ] Create trip and trip-event tables.
- [ ] Attach each accepted capture to an open trip for the vehicle.
- [ ] Use camera-pair direction to classify event as outbound, inbound, or unknown.
- [ ] Close trip when matching inbound event is found after outbound event.
- [ ] Keep trip as `needs_review` when direction is unknown or event ordering is ambiguous.
- [ ] Expose support image, LPR image, plate, vehicle, timestamp, direction, and load status in trip detail.
- [ ] Implement load-status update with permission `trips.manage`.
- [ ] Write tests for outbound capture, inbound close, unknown direction, and load-status update.
- [ ] Run `php artisan test --filter=TripClassificationTest`.
- [ ] Commit as `feat: add trip classification and load review`.

**Acceptance Criteria:**

- Trips group vehicle movement events.
- Operator can mark loaded/empty from the support-camera image.
- Unknown cases are visible for review rather than silently guessed.

---

## Task 10: Reports and Audit Trail

**Files:**

- Create: `Api/V1/Admin/ReportController.php`
- Create: `Actions/Reports/BuildTripReportQueryAction.php`
- Create: `Actions/Reports/ExportTripReportCsvAction.php`
- Create: `Models/AuditLog.php`
- Create: `Services/Audit/AuditLogger.php`
- Create frontend pages:
  - `src/pages/reports/TripReportsPage.vue`
  - `src/pages/trips/TripReviewPage.vue`
- Create tests:
  - `tests/Feature/Reports/TripReportTest.php`
  - `tests/Feature/Audit/AuditLogTest.php`

**Interfaces:**

- Produces: `GET /api/v1/admin/reports/trips`
- Produces: `GET /api/v1/admin/reports/trips/export.csv`
- Produces report rows with vehicle, plate, outbound time, inbound time, load status, LPR image URL, support image URL, location, and camera pair.

**Steps:**

- [ ] Implement report query filters for date range, vehicle, plate, location, direction, and load status.
- [ ] Include signed private media URLs for LPR and support images.
- [ ] Add CSV export for operational use.
- [ ] Add audit logging for login, user changes, permission changes, vehicle changes, camera changes, edge sync failures, and load-status changes.
- [ ] Create Vue report page with filters, table, thumbnails, image preview modal, empty state, and export action.
- [ ] Create Vue trip review page showing side-by-side LPR and support image.
- [ ] Write report API tests for filters and permissions.
- [ ] Write audit tests for load-status change.
- [ ] Run backend tests and frontend tests.
- [ ] Commit backend as `feat: add trip reports and audit trail`.
- [ ] Commit frontend as `feat: add trip reports UI`.

**Acceptance Criteria:**

- Report always shows LPR image and support image when both exist.
- Load status is visible and filterable.
- Sensitive media is private and exposed through authorized URLs only.
- Auditable actions leave trace.

---

## Task 11: Deployment and Field Operations

**Files:**

- Create backend docs:
  - `RIALMA-TrackVision-Backend/docs/deployment-parent.md`
  - `RIALMA-TrackVision-Backend/docs/deployment-edge.md`
  - `RIALMA-TrackVision-Backend/docs/intelbras-field-checklist.md`
- Create AgentOps doc:
  - `RIALMA-TrackVision-AgentOps/docs/operations/field-validation-checklist.md`

**Interfaces:**

- Produces operational checklists for parent server and edge site.

**Steps:**

- [ ] Document required environment variables for parent profile.
- [ ] Document required environment variables for edge profile.
- [ ] Document camera-pair setup: LPR camera IP, support camera IP, credentials, direction, channel, and test snapshot.
- [ ] Document offline validation: disconnect internet, trigger LPR event, verify local capture, reconnect, verify parent sync.
- [ ] Document backup strategy for parent DB and media.
- [ ] Document local edge disk monitoring and media retention.
- [ ] Document command schedule for `edge:listen-intelbras` and `edge:sync-parent`.
- [ ] Commit as `docs: add deployment and field validation guides`.

**Acceptance Criteria:**

- A field technician can configure a camera pair.
- A developer can validate offline behavior.
- Operators know how to confirm sync health and report completeness.

---

## Risks and Decisions

- Intelbras camera API details must be validated against the exact camera firmware before implementation.
- VIP 5460 LPR IA integration is designed as webhook-first. If `PictureHttpUpload` is unavailable, use `EventHttpUpload` with LPR snapshot fallback. If active upload cannot be configured in field, use event-stream subscription with `snapManager.cgi?action=attachFileProc`.
- Automatic loaded/empty classification is out of v1 unless a computer-vision model is explicitly selected and validated.
- Edge local storage must be sized for image volume and offline duration.
- Parent sync must be idempotent from the first implementation.
- RBAC must be in place before exposing admin CRUD.

## Verification Matrix

Backend:

- `composer validate`
- `php artisan test`
- `php artisan route:list`
- endpoint Feature tests for auth, RBAC, vehicles, cameras, edge sync, trips, and reports

Frontend:

- `npm run lint`
- `npm run test`
- `npm run build`
- Playwright login and report-flow smoke tests

Field:

- LPR event from registered vehicle creates local capture.
- Support camera image is captured for the same event.
- Edge remains functional without internet.
- Parent receives queued capture after internet returns.
- Report shows LPR image, support image, direction, vehicle, and load status.

## Technical References

- Laravel Authentication: https://laravel.com/docs/13.x/authentication
- Laravel Passport: https://laravel.com/docs/13.x/passport
- Spatie Laravel Permission: https://spatie.be/docs/laravel-permission/v8/introduction
- Spatie Laravel Permission repository: https://github.com/spatie/laravel-permission
- Intelbras HTTP API SDK: https://botminio.apps.intelbras.com.br/sdk-api/HTTP%20API%20V3.35_Intelbras.pdf
- Vue Style Guide: https://vuejs.org/style-guide/
- Vue Router: https://router.vuejs.org/
- Pinia: https://pinia.vuejs.org/

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-08-12-trackvision-system-implementation.md`.

Two execution options:

1. **Subagent-Driven (recommended)** - dispatch a fresh subagent per phase, review between phases, fast iteration.
2. **Inline Execution** - execute phases in this session using `superpowers:executing-plans`, with checkpoints for review.

Before starting implementation, create a detailed phase-level plan for Task 1 through Task 3 because they establish the foundation used by all later phases.
