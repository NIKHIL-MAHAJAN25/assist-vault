# API contract

**This is the critical shared page.** Once a route's shape is locked here, `companion-app`,
`family-dashboard`, and `watch-app` should each only need this page plus their own repo — not each
other's internals, not the backend's source. Changing a route here is a breaking change; update this
page and flag affected repos in the same session.

Base URL (dev): `http://localhost:3000`

## Auth

`POST /api/auth/login`
- Body: `{ email, password }`
- Returns: `{ token, user: { id, name, role, patient_id, patient_name } }`
- `patient_name` is the related patient's name, added so the dashboard can say "Priya is at home"
  rather than "the patient is at home". Additive — every field that was here before is unchanged.
- Public.

`POST /api/devices/register`
- Body: `{ patient_id, device_type: "esp32cam" | "watch", identifier }`
- Returns: `{ device_id, api_key }` — **`api_key` is shown once, only the hash is stored.**
- Public (one-time registration; treat the returned key as a secret from then on).

## Ingestion (device API key required — `Authorization: Bearer <device_api_key>`)

`POST /api/frames`
- Body: `{ device_id, image: <base64 or multipart>, captured_at }`
- Pipeline: Gemini 2.5 Flash vision call → structured JSON → Cloudinary upload → `log_entries` write
  (`entry_type: "frame"`).
- Returns: `{ log_entry_id, requires_attention }`

`POST /api/geofence-check`
- Body: `{ device_id, lat, lng, timestamp }`
- Checks against the patient's `geofence_zones`; writes a `log_entries` row (`entry_type: "geofence"`).
- Returns: `{ log_entry_id, inside_safe_zone: bool }`

`POST /api/heartrate-events`
- Body: `{ device_id, scenario: "normal" | "confusion" | "emergency", heart_rate, motion_pattern, timestamp }`
- Scripted watch-app data, not live sensor polling. Writes `log_entries` (`entry_type: "heartrate"`).
- Returns: `{ log_entry_id }`

## Log entries (JWT required — `Authorization: Bearer <jwt>`)

`GET /api/log-entries?patient_id=&type=&from=&to=&acknowledged=` — timeline feed.
`acknowledged=true|false` is optional; omitted means no filtering (the pre-existing behaviour).
`GET /api/log-entries/:id`
`PATCH /api/log-entries/:id` — body `{ is_saved?: bool, acknowledged?: bool }`. Either field may
be sent alone or together. `is_saved` toggles retention exemption, unchanged. `acknowledged: true`
stamps `acknowledged_at` with the current time; `false` clears it back to null.

Every log entry in every response now also carries **`acknowledgedAt`** (ISO string or null).
Note the camelCase: these responses serialize the Prisma object directly, so this field matches
its siblings (`isSaved`, `requiresAttention`, `imagePublicId`, `createdAt`) rather than the
snake_case used in request bodies. The underlying column is `log_entries.acknowledged_at`.

### Patient status (JWT, `family_caregiver` role only)

`GET /api/patients/:id/status` — a single derived "right now" summary. Pure read; no new tables
beyond the `acknowledged_at` column above. Returns every key on every call, null or 0 when the
data is missing, so clients never have to guess at an absent field:

```json
{
  "patient": { "id": "<uuid>", "name": "<string>" },
  "last_update": "<ISO8601>|null",
  "is_stale": true,
  "minutes_since_update": 12,
  "current_activity": "<string>|null",
  "location": {
    "inside_safe_zone": true, "zone_id": "<uuid>|null",
    "zone_name": "<string>|null", "lat": 0.0, "lng": 0.0
  },
  "needs_attention_count": 2,
  "today": { "entry_count": 18, "quiz_correct": 1, "quiz_total": 3 }
}
```

- `is_stale` is true when the newest entry is more than **45 minutes** old, or when there are no
  entries at all. It is computed server-side deliberately: the dashboard's landing screen uses it
  to refuse to present old readings as current, and that judgement should not be duplicated in
  each client. **Clients must not recompute or override it.**
- `zone_name` is resolved by haversine distance from the newest geofence point against the
  patient's zones; on overlap the smallest radius wins. Null when nothing matches.
- `needs_attention_count` counts entries with `requires_attention = true` AND
  `acknowledged_at IS NULL`.
- 403 when the JWT's `patient_id` does not match `:id`; 404 when the patient does not exist.

## Geofence zones (JWT, `family_caregiver` role only)

`POST /api/geofence-zones` — body `{ patient_id, name, center_lat, center_lng, radius_meters }`
`GET /api/geofence-zones?patient_id=`
`PATCH /api/geofence-zones/:id`
`DELETE /api/geofence-zones/:id`

## Enrolled faces (JWT, `family_caregiver` role only)

`POST /api/enrolled-faces` — body `{ patient_id, name, reference_embedding }` (Phase 0: caller supplies
the pre-computed descriptor array directly; Phase 2 will add server-side computation from a
`reference_image` via face-api.js — see decisions-log.md)
`GET /api/enrolled-faces?patient_id=`
`DELETE /api/enrolled-faces/:id`

## Quiz

`GET /api/quiz/today?device_id=` — generate-if-absent-or-fetch today's questions, scoped to the
authenticated device's patient (device API key — called by the companion app; there's no patient login)
`POST /api/quiz/attempts` — body `{ device_id, quiz_question_id, answered, correct }` (device API key —
rejects with 404 if the question doesn't belong to this device's patient)
`GET /api/quiz/history?patient_id=` (JWT, dashboard use)

## Recall & assist (device API key — called on the patient's behalf by the companion app)

`POST /api/recall/query` — body `{ device_id, query_text }` → object/day recall, grounded retrieval,
returns `{ answer_text }` for TTS
`POST /api/assist/clarify` — body `{ device_id, raw_transcript }` → `{ clarified_response }`

## Contract changes — 2026-09-13 (Session 5)

All three of the above were added in one session and are **strictly additive**: no existing route
lost or renamed a field, and no request shape changed. Consumers affected, per the working
agreement in [../CLAUDE.md](../CLAUDE.md):

- `family-dashboard` — **updated in the same session.** Uses `/patients/:id/status` for its new
  Today landing screen, `acknowledged` for a clearable alerts list, and `patient_name` for copy.
- `companion-app` — **not affected.** Device-authenticated only; touches none of these routes.
- `watch-app` — **not affected**, and not yet built.

## Status

Phase 0 (today): routes, request/response shapes, and auth model defined here and implemented against a
single Gemini call (no agentic decision layer, no short-term memory context yet). `agent_decision` on
`log_entries` will start being populated in Phase 1 once the two-step decision layer
([ai-prompting-patterns.md](ai-prompting-patterns.md) §3) lands.
