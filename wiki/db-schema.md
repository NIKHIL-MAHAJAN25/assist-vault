# Database schema

**Design principle for extending this schema**: fields and tables may be added or adjusted as real needs
emerge during the build — this is expected and fine. All changes must go through **Prisma migrations**
(`npx prisma migrate dev`), never hand-edited directly against the live database. Any new field must
follow proper normalization (1NF: atomic values, no comma-separated lists; 2NF/3NF: no field duplicating
data that belongs in another table, no field depending on something other than its table's primary key).
Do not denormalize for convenience.

The authoritative schema lives in `backend-server/prisma/schema.prisma` — this page is the human-readable
mirror of it. If they ever disagree, the Prisma schema wins and this page needs updating.

## `patients`
- `id` (PK), `name`, `dob`
- No dedicated `home_address` field — safe zones are handled entirely via `geofence_zones` (see
  [decisions-log.md](decisions-log.md), session 1: a home address is just a geofence zone with a
  conventional name like "Home"; a separate field would duplicate what `geofence_zones` already stores
  and risk drifting out of sync with it).

## `users`
- `id` (PK), `name`, `email` (unique), `password_hash`, `role` (`patient` | `family_caregiver`),
  `patient_id` (FK → `patients.id`)

## `devices`
- `id` (PK), `patient_id` (FK → `patients.id`), `device_type` (`esp32cam` | `watch`), `identifier`,
  `api_key_hash` (device authentication, separate from user JWT — see below)

## `log_entries` — the spine
- `id` (PK), `patient_id` (FK), `device_id` (FK, nullable — quiz-generated entries have no device),
  `entry_type` (`frame` | `geofence` | `heartrate` | `quiz`), `timestamp`, `payload` (`JSONB` — the
  Gemini/agent structured output), `requires_attention` (bool), `agent_decision` (`log_only` |
  `alert_family` | `check_in` — populated starting Phase 1), `image_public_id` (nullable, Cloudinary
  reference, `frame` entries only), `is_saved` (bool, default `false`), `created_at`

No separate "alerts" table — an alert is `log_entries` filtered by `requires_attention = true`.

## `log_entry_faces` (junction table)
1NF fix: one row per recognized person per log entry, not a comma-separated list in `log_entries`.
- `log_entry_id` (FK → `log_entries.id`), `enrolled_face_id` (FK → `enrolled_faces.id`)

## `enrolled_faces`
- `id` (PK), `patient_id` (FK), `name` (e.g. "Son"), `reference_embedding` (`JSONB` array of floats — the
  face-api.js descriptor vector stored directly, rather than a reference image recomputed at match time;
  see [decisions-log.md](decisions-log.md) session 1 for why)

## `geofence_zones`
- `id` (PK), `patient_id` (FK), `name`, `center_lat`, `center_lng`, `radius_meters`, `created_by` (FK →
  `users.id`)

## `quiz_questions`
- `id` (PK), `patient_id` (FK), `date`, `question`, `correct_answer`, `source_log_entry_id` (FK →
  `log_entries.id` — traceability to what it was grounded in)

## `quiz_attempts`
- `id` (PK), `quiz_question_id` (FK), `answered`, `correct` (bool), `timestamp`

## Auth model — two distinct systems, do not conflate

- **Users** (dashboard logins — family caregivers, and patient view if applicable): JWT-based, RBAC
  checked against `role` on every protected route.
- **Devices** (ESP32-CAM via companion app, companion app itself, watch app): authenticate via a
  **device API key** issued at `POST /api/devices/register`, hashed and stored in `devices.api_key_hash`.
  Devices are not "logging in" as a person — don't reuse the JWT scheme for them.


## `invites` — added Session 13
- `id` (PK), `code` (unique), `patient_id` (FK → `patients.id`, cascade), `kind` (`caregiver` | `device`),
  `created_by` (FK → `users.id`), `created_at`, `expires_at`, `used_at` (nullable), `used_by` (nullable —
  the new user id or device id; not an FK because it can point at either table).
- `log_entries.payload.decision` (JSON inside payload, no column) holds the decision layer's record;
  `log_entries.agent_decision` is now actually written (`log_only` | `alert_family` | `check_in`).


## `ai_jobs` — added Session 14
- `id`, `kind` (`frame` | `summary`), `status` (`queued` | `running` | `done` | `failed` | `dropped`), `patient_id`,
  `payload` (frame: device_id, captured_at, prompt, image base64 — the image is removed once finished;
  summary: prompt, cache_key), `result`, `error`, `attempts`, `avoid_provider`, `provider`, `worker_id`,
  `created_at`, `started_at`, `finished_at`. Finished rows are deleted after a day. Not an FK to patients on purpose
  (queue rows are disposable).


## `memories` — added Session 18
- `id`, `patient_id`, `image_url` (Cloudinary), `caption` (≤200, the family's words, read aloud), `created_by`
  (user id), `creator_name`, `created_at`. Index `(patient_id, created_at)`. Migration `20261018182153_memories`.
- Demo history uses existing tables: `log_entries` from device `demo:history` (`payload.simulated = true`),
  `quiz_questions` anchored to one of those entries via `source_log_entry_id`, and a `family_events` fall check
  with `ref_id` → a history entry, so all of it can be found and removed again.
