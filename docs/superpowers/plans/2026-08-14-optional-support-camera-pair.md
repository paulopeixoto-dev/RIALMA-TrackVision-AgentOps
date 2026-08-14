# Optional Support Camera Pair Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow TrackVision capture points to operate with only an Intelbras LPR camera, while still supporting a secondary support camera when available.

**Architecture:** The camera pair remains the operational unit that gives Intelbras events a stable `camera_pair_uuid`. `support_camera_id` becomes nullable in admin, capture, and edge sync flows. LPR captures continue to create trips and outbox messages; support image capture runs only when a support camera exists.

**Tech Stack:** Laravel 12, Passport, Eloquent, Spatie permissions, MySQL, Vue 3, Vuestic UI, Vitest.

## Global Constraints

- Keep Laravel Controllers thin, validation in Form Requests, business rules in Actions or Services.
- Keep Vuestic Admin/Vuestic UI components as the frontend default.
- Do not store camera or webhook secrets in committed files.
- Use TDD for behavior changes.

---

### Task 1: Backend Optional Support Camera

**Files:**
- Modify: `RIALMA-TrackVision-Backend/tests/Feature/Admin/CameraPairAdminTest.php`
- Modify: `RIALMA-TrackVision-Backend/tests/Feature/Edge/ProcessLprEventTest.php`
- Modify: `RIALMA-TrackVision-Backend/database/migrations/*_make_support_camera_optional_on_capture_flows.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/StoreCameraPairRequest.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Admin/UpdateCameraPairRequest.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Admin/CameraPairs/CreateCameraPairAction.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Admin/CameraPairs/UpdateCameraPairAction.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Admin/CameraPairs/ValidateCameraPairCamerasAction.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Edge/ProcessLprEventAction.php`

**Interfaces:**
- Consumes: existing `CameraPair` model and admin API.
- Produces: admin API accepts omitted or null `support_camera_id`; LPR event processing stores captures with `support_camera_id = null`.

- [ ] Write failing admin and LPR processing tests.
- [ ] Run focused tests and confirm they fail for required support camera constraints.
- [ ] Add migration and minimal backend updates.
- [ ] Run focused tests and confirm they pass.

### Task 2: Edge Sync Optional Support Camera

**Files:**
- Modify: `RIALMA-TrackVision-Backend/app/Services/EdgeSync/EdgeCaptureBatchSerializer.php`
- Modify: `RIALMA-TrackVision-Backend/app/Data/Edge/EdgeCaptureBatchItemData.php`
- Modify: `RIALMA-TrackVision-Backend/app/Http/Requests/Api/V1/Edge/StoreEdgeCaptureBatchRequest.php`
- Modify: `RIALMA-TrackVision-Backend/app/Actions/Edge/UpsertEdgeCaptureAction.php`
- Modify: focused edge sync tests.

**Interfaces:**
- Consumes: nullable `support_camera_id` captures.
- Produces: serialized and ingested edge batches with `support_camera_uuid = null`.

- [ ] Write failing sync tests for no support camera.
- [ ] Run focused tests and confirm they fail for required support camera UUID.
- [ ] Make serializer/request/data/action null-safe.
- [ ] Run focused tests and confirm they pass.

### Task 3: Frontend Optional Support Camera

**Files:**
- Modify: `RIALMA-TrackVision-Frontend/src/types/admin.ts`
- Modify: `RIALMA-TrackVision-Frontend/src/components/forms/CameraPairForm.vue`
- Modify: `RIALMA-TrackVision-Frontend/src/pages/CameraPairsPage.vue`
- Modify: `RIALMA-TrackVision-Frontend/src/pages/CameraPairsPage.test.ts`

**Interfaces:**
- Consumes: API accepts `support_camera_id: null`.
- Produces: Vuestic form can submit a pair without support camera.

- [ ] Write/adjust test for rendering a pair without support camera.
- [ ] Update types/form payload to use `number | null`.
- [ ] Run frontend tests and build.

### Task 4: Local Configuration

**Files:**
- No committed files.

**Interfaces:**
- Consumes: camera `df18ae14-5e96-4acc-9fa2-d1faaa1974c3`.
- Produces: local `camera_pair_uuid` ready for Intelbras integration.

- [ ] Run migrations against local MySQL.
- [ ] Create or update local pair for `Intelbras VIP-5460-LPR-IA` with no support camera.
- [ ] Validate admin API, snapshot, and endpoint path.
