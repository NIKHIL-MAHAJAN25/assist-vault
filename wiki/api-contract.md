# API contract

**This is the critical shared page.** Once a route's shape is locked here, `companion-app`,
`family-dashboard`, and `watch-app` should each only need this page plus their own repo — not each
other's internals, not the backend's source. Changing a route here is a breaking change; update this
page and flag affected repos in the same session.

Base URL (dev): `http://localhost:3000`

## Auth

`POST /api/auth/login`
- Body: `{ email, password }`
- Returns: `{ token, user: { id, name, role, patient_id } }`
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

`GET /api/log-entries?patient_id=&type=&from=&to=` — timeline feed
`GET /api/log-entries/:id`
`PATCH /api/log-entries/:id` — body `{ is_saved: bool }`, toggles retention exemption

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

`GET /api/quiz/today?patient_id=` — generate-if-absent-or-fetch today's questions (JWT)
`POST /api/quiz/attempts` — body `{ quiz_question_id, answered, correct }` (JWT)
`GET /api/quiz/history?patient_id=` (JWT)

## Recall & assist (device API key — called on the patient's behalf by the companion app)

`POST /api/recall/query` — body `{ device_id, query_text }` → object/day recall, grounded retrieval,
returns `{ answer_text }` for TTS
`POST /api/assist/clarify` — body `{ device_id, raw_transcript }` → `{ clarified_response }`

## Status

Phase 0 (today): routes, request/response shapes, and auth model defined here and implemented against a
single Gemini call (no agentic decision layer, no short-term memory context yet). `agent_decision` on
`log_entries` will start being populated in Phase 1 once the two-step decision layer
([ai-prompting-patterns.md](ai-prompting-patterns.md) §3) lands.
