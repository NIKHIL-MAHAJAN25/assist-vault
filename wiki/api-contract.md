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
- **JWT, `family_caregiver`, own patient only** (since Session 13 — was public). The patient's own
  phone uses `POST /api/devices/pair` with a code instead. **Breaking** for anything that registered
  without a token; the companion app's onboarding was switched to pairing in the same session.

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


## Device context (device API key) — added 2026-10-02

`GET /api/devices/me?device_id=` — what the phone needs to know about the person it serves.
Returns `{ patient: { id, name }, places: [{ id, name, center_lat, center_lng, radius_meters }] }`.
Used by `companion-app` for the Home greeting and the "Get home" compass (target = the place named
"Home", else the first place). Additive; no existing route changed. Consumer: `companion-app` only.

---

## Emergency (NFC) tag — added Session 11

A sticker holds `<PUBLIC_BASE_URL>/t/<code>`. The backend serves that page itself (`public/tag.html`),
so one exposed port is enough. `code` is 7 chars from `[a-z2-9]` minus look-alikes.

**Caregiver (JWT, family_caregiver, own patient only)**
- `GET /api/patients/:id/tag` → `{ code, url, note, home_label, show_home, has_home_zone, contacts: [{name, relation, phone}] }`
  (creates the code on first call)
- `PUT /api/patients/:id/tag` body any of `{ note, home_label, show_home, contacts }` (contacts ≤ 5, each needs name + phone; replaces the list) → same shape
- `POST /api/patients/:id/tag/rotate` → new code; the old sticker stops working

**Public (no auth — whoever taps the sticker)**
- `GET /api/public/tag/:code` → `{ first_name, note, contacts, home: {label, lat, lng} | null }`.
  Only the first name, never surname or DOB. Also logs a `sighting` entry `{kind:'scan'}`, `requires_attention: true`,
  at most once per 10 minutes per tag.
- `POST /api/public/tag/:code/sighting` body `{ lat?, lng?, accuracy?, message? }` → 201. Logs `sighting`
  `{kind:'location', lat, lng, accuracy}`, `requires_attention: true`. Throttled per IP (20 s).
- Unknown/rotated code → 404 `{ error: 'This tag is no longer active.' }`

**Schema:** `patients` gains `tag_code` (unique), `tag_note`, `home_label`, `tag_show_home`; new table
`emergency_contacts`; `LogEntryType` gains `sighting`. Consumers: dashboard + app `story` logic treat a
sighting as a place moment and title its incident "Someone found them and tapped the emergency tag".


## Accounts, invites & phone pairing — added Session 13 (2026-10-17)

Codes are 6 chars from `A-Z2-9` minus look-alikes (no 0/O/1/I/L), single-use. Input is normalised
(`"gtc-n46"` → `"GTCN46"`). One live code per kind per patient: making a new one deletes the unused old one.

**Public**
- `POST /api/auth/signup` body `{ name, email, password (≥8), patient_name, patient_dob?, home_label? }`
  → 201 `{ token, user }` (same shape as login). Creates the patient and the first `family_caregiver`.
  409 if the email exists. Emails are stored lower-cased; login lower-cases too.
- `GET /api/invites/:code?kind=caregiver|device` → `{ kind, patient_first_name, invited_by, expires_at }`.
  404 unknown / wrong kind, 410 used or expired.
- `POST /api/auth/join` body `{ code, name, email, password }` → 201 `{ token, user }`; consumes a
  `caregiver` code atomically (two racing joins: one gets 410).
- `POST /api/devices/pair` body `{ code, identifier, device_type? = "esp32cam" }` →
  201 `{ device_id, api_key, patient_id, patient_name }`; consumes a `device` code.

**Caregiver (JWT, family_caregiver, own patient only — 403 otherwise)**
- `GET /api/patients/:id/caregivers` → `[{ id, name, email, joined_at, is_you }]`
- `GET /api/patients/:id/invites` → unused, unexpired `[{ code, kind, created_at, expires_at, created_by }]`
- `POST /api/patients/:id/invites` body `{ kind: "caregiver" | "device" }` → 201 same item.
  Lifetime: caregiver 72 h, device 24 h.
- `DELETE /api/patients/:id/invites/:code` → 204

Consumers: dashboard (`/signup`, `/join`, `/join/:code`, Settings → Family circle / Set up phone),
companion app (family sign-in screen, family Settings, patient onboarding).

## Decision layer — added Session 13

`GET /api/patients/:id/decisions?days=7` (JWT, family_caregiver, own patient) →
`{ mode, model, summary: { decisions, drex_answered, drex_errors, agreement (%), median_ms },
recent: [{ log_entry_id, timestamp, entry_type, what, rules: {notify}, drex: {notify, notify_p, urgency, ms} | {error, ms}, agree, decided_by }] }`.

Side effect on ingestion (no shape change): every `POST /api/frames`, and every `POST /api/geofence-check`
that the rules flag, gets `agent_decision` set and `payload.decision = { mode, rules, drex, agree, decided_by, at }`.
In `live` mode the frame response's `requires_attention` reflects Drex's answer.


## Helper queue, journal & summaries — added Session 14 (2026-10-18)

**Changed:** `POST /api/frames` now answers **202** `{ log_entry_id: <job id>, requires_attention: false, queued: true }`
straight away; the log entry is written when a helper (or the Gemini fallback) answers. Same field names as
before so the companion app (which ignores the body) is unaffected. Frame payloads gain `activity_key`
(fixed vocabulary in `services/prompts.js`) and `analysed_by` (`claude` | `codex` | `gemini`).

**Family (JWT, family_caregiver, own patient)**
- `GET /api/patients/:id/summary?range=today|morning|week[&fresh=1]` → `{ range, summary, provider, generated_at,
  journal_lines, cached }`. Waits up to 60 s; 503 if nothing could write it. Cached until the journal changes.
- `GET /api/patients/:id/journal?date=YYYY-MM-DD` → `text/markdown`, the day journal.

**Internal (helpers on the same host only — `x-worker-token` must equal `WORKER_TOKEN`, and any request
carrying `X-Forwarded-For` (i.e. via Caddy) gets 404)**
- `POST /internal/jobs/claim` `{ provider: claude|codex, worker_id }` → `{ id, kind, prompt, image? }` or 204
  after a 20 s long-poll.
- `POST /internal/jobs/:id/result` `{ worker_id, ok, output | error, ms }` → `{ accepted }` (409 if not theirs).
- `GET /internal/stats` → jobs in the last hour by status and provider.

## Family Assistant & device commands — added Session 15 (2026-10-18)

**Family (JWT, family_caregiver, own patient)**
- `POST /api/patients/:id/assistant` `{ message, history?: [{role:'user'|'assistant', text}] }` →
  `{ reply, actions: [...], provider, sources: [...], ms }`. Runs as an `agent` job on the helper queue (Gemini
  fallback after 15 s); 503 if nothing answers in 60 s. `actions` are **proposals only**, from a closed set:
  `{type:'speak', text}` · `{type:'map'}` · `{type:'photo'}` · `{type:'summary', range}`. Unknown types are dropped.
- `POST /api/patients/:id/commands` `{ type:'speak', text (≤280) }` → 201 command (the family confirmed it).
- `GET /api/patients/:id/commands` → last 20 `{ id, type, text, status, from, created_at, delivered_at, done_at, error }`.
  `status`: `pending` → `delivered` (phone picked it up) → `done` (spoken) | `failed` | `expired` (not picked up in 10 min).

**Device (api key)**
- `GET /api/devices/commands?device_id=` → pending `[{ id, type, text, from, created_at }]`, marked `delivered`.
- `POST /api/devices/commands/:id/ack` `{ device_id, ok, error? }`.
Companion app polls every 6 s from MonitoringService, speaks "Message from X. …", posts a notification and
shows the message on the patient Home for 3 h with "Say it again".

## Phone actions, notifications, broadcasts, demo kit — Session 16 (2026-10-18)

**Commands** (`POST /api/patients/:id/commands`) now take any type from `services/commandTypes.js`:
`speak {text}`, `ask {question, quick?}`, `voice {url, seconds, text?}`, `ring`, `reminder {text, at "HH:MM"|ISO}`,
`capture`, `guide_home`, `quiz`, `calm {voice_url?}`. `GET /commands` items carry the payload fields plus results
(`reply`, `replied_at` for ask; `uploaded`, `log_entry_id`, `image_public_id`, `activity`, `person_state` for capture).
Device: `GET /api/devices/commands` returns `{id, type, from, created_at, ...payload}`; ack body adds `reply` (ask).
`POST /api/frames` accepts optional `command_id` (photo taken on request; jumps the queue, never dropped as stale).
- `POST /api/patients/:id/voice-notes {audio: base64, mime}` → `{url (mp3), seconds}` (Cloudinary, resource_type video).

**Notifications** — `GET /api/patients/:id/notifications?since=ISO` (default 24 h) →
`{items: [{id, kind: alert|reply|failed|note, severity, title, body, at, mine?, acknowledged?}], now}`.
**Broadcasts** — `POST /api/patients/:id/broadcasts {text}` → `{id, text, from, created_at, family_members}`.
Agent actions gain `ask`, `reminder`, `capture`, `ring`, `guide_home`, `calm`, `quiz`, `broadcast`.

**Demo kit** (JWT family, own patient, `DEMO_MODE=1`, else 403):
`GET /api/demo/:id` · `POST /api/demo/:id/scenes/:name/save {log_entry_id?}` · `POST /api/demo/:id/scenes/:name/play` ·
`POST /api/demo/:id/walk {direction: away|home|stop}` · `POST /api/demo/:id/answer {reply}` · `POST /api/demo/:id/reset`.
Simulated readings come from per-patient devices `demo:camera` / `demo:phone`; real phone pings are ignored while a walk runs.

## Agent on its own (Batch C) — Session 17 (2026-10-18)

- **Rules** (`services/ruleEngine.js`): `GET /api/patients/:id/rules` · `POST {kind, params, repeat: once|daily}` ·
  `DELETE /rules/:rid`. Kinds: `arrives_home {}`, `leaves_home {after?}`, `not_seen_by {activity, by}`,
  `still_at {activity, at}`, `seen {activity}`, `quiet_for {minutes}` (activity = vocabulary key or `up`).
  Checked every 30 s by `services/scheduler.js`; hits become `family_events` (kind `rule`).
- **Fall check** (`services/fallCheck.js`): a `fallen` frame or watch emergency → `ask` command with
  `fall_check: true`, quick `I'm okay / I need help`. Okay → event `fall_ok`; help or 2 min silence → event
  `fall_escalated` (act) + `ring` command.
- **Bedtime story** (`services/storyService.js`): `POST /api/patients/:id/story` → `{text, provider, command_id}`;
  `GET/PUT /api/patients/:id/story-settings {at: "HH:MM"|null}` (default 21:00). New command type `story {text, title}`.
- **Notifications** now include `family_events` (kinds `rule`, `story`; fall events as `alert`/`reply`).
- Agent actions gain `rule {kind, params, repeat}` and `story`.
- Demo kit: `POST /api/demo/:id/tick` (run the scheduler now), `POST /api/demo/:id/fall` (simulated fall).
- Schema: `watch_rules`, `family_events`, `patients.bedtime_story_at`.
