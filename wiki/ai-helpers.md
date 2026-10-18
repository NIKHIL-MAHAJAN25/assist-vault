# AI helpers: job queue, worker pool, day journal & summaries

> **Status:** built and running in production on the user's EC2 box since 2026-10-18 (Session 14).
> **Code:** `D:\backend-server` (backend side + helper program) · consumers in `family-dashboard` and `companion-app`.
> **Read with:** [api-contract.md](api-contract.md) (routes), [db-schema.md](db-schema.md) (`ai_jobs`),
> [decisions-log.md](decisions-log.md) Session 14 (why each choice), [ai-prompting-patterns.md](ai-prompting-patterns.md).
>
> This page is written as context for another engineer or AI picking the system up cold. It describes what
> exists, exactly how it behaves, every timing constant, where things live on the server, and how to operate it.
> No secret values appear here — only the names and locations of the files that hold them.

---

## 1. What it is, in one paragraph

Camera photos and family "summarise my day" requests are turned into **jobs** in a Postgres table. A pool of
**8 helper processes** running on the EC2 host (4 using the **Claude Code** CLI, 4 using the **OpenAI Codex**
CLI, both on the user's personal subscriptions) compete to take jobs, run one fresh CLI call each, and post the
answer back. The backend validates every answer against a **fixed activity vocabulary**, retries a bad answer
once on the *other* provider, and falls back to **Gemini** (called from inside the backend) when no helper
answers in time. The backend is the **only writer** of a per-patient, per-day **Markdown journal** rendered from
the database; the journal's last lines give helpers short-term memory, and summaries are written from it.

Pattern names: asynchronous job queue (producer/consumer), competing-consumer worker pool, Postgres-as-queue
(`FOR UPDATE SKIP LOCKED`), long polling, fallback chain, structured output + controlled vocabulary, output
validation with cross-provider retry, circuit breaker with exponential backoff, load shedding, single-writer
principle, materialized view (the journal), grounded generation (summaries).

## 2. Why it exists (short — full reasoning in decisions-log Sessions 12 and 14)

- Gemini free tier could not sustain a photo every ~5 s for a live demo; the user already pays for Claude and
  ChatGPT subscriptions and is the only data subject of this prototype. **Revisit before any real users:**
  personal subscriptions are for personal use → switch to API keys.
- Before this, `POST /api/frames` called Gemini synchronously; a slow/failed AI call failed the phone's upload
  (the HTTP 500 seen in Session 13 was most likely this). Now uploads never wait on AI.
- Two providers answering free-text would describe the same scene differently and stop consecutive photos
  merging into one "moment" → the fixed vocabulary.
- Eight helpers appending to one shared file would interleave and lose ordering → backend-only journal.

## 3. Architecture

```
ESP32 cam ──▶ patient phone (companion-app MonitoringService, 1 photo / 5 s, upload only on change)
                    │ POST /api/frames  (device api_key)
                    ▼
        ┌─────────────────────────── EC2 host 3.107.133.23 ───────────────────────────┐
 Caddy ─┼─▶ assist-api (Docker)  ── 202 "queued" back to phone immediately          │
 (HTTPS)│      │  enqueueFrame: INSERT ai_jobs (photo, prompt incl. last 10 journal lines)
        │      │  starts Cloudinary upload in background                               │
        │      ▼                                                                       │
        │   assist-db (Docker, Postgres 17)  table ai_jobs  ◀── claim (SKIP LOCKED) ──┐ │
        │      ▲                                                                     │ │
        │      │ result → validate → save log_entry → Drex decision → journal        │ │
        │      │                                                                     │ │
        │   127.0.0.1:3100  /internal/*  (token + not-via-Caddy)                     │ │
        │      ▲                                                                     │ │
        │   assist-helpers (systemd, user "assist", NOT Docker)                      │ │
        │      pool.js: claude-1..4, codex-1..4 ── each: claim → CLI call → result ──┘ │
        │                                                                              │
        │   Gemini fallback runs INSIDE assist-api (no helper involved)                │
        └──────────────────────────────────────────────────────────────────────────────┘
Family website / app ──▶ GET /api/patients/:id/summary ──▶ summary job ──▶ helper ──▶ text
```

Family website currently runs on the user's laptop (Vite dev server) pointed at the server URL; it will be
deployed onto the server at the end of the project (user's decision: fast iteration first).

## 4. Files (backend-server)

| File | Role |
|---|---|
| `controllers/frameController.js` | `POST /api/frames`: enqueue and answer **202** at once. Body keeps the old shape (`log_entry_id` = the **job id**, `requires_attention: false`, `queued: true`); the companion app ignores the body. |
| `services/jobQueue.js` | The queue: `enqueueFrame`, `enqueueSummary`, `claim`, `release`, `complete`, `finishFrame`, `retryOrFallback`, `runFallback` (Gemini), `drop`, `sweep`, `waitFor`, `stats`, `start`. |
| `services/prompts.js` | `ACTIVITIES` (vocabulary), `framePrompt(journalTail)`, `summaryPrompt({firstName, range, journal})`, `validateFrame(raw)`. **Single source of truth** for what every provider is told. |
| `services/journalService.js` | Renders / reads / prunes the day journal: `renderLines`, `linesFor`, `writeDay`, `readDay`, `tail`, `refreshChanged`, `prune`, `start`. |
| `services/geminiService.js` | Adds `analyzeFrameWithPrompt` and `generateText` (thinking budget 0, one retry on 429). Older functions unchanged. |
| `services/decisionService.js` | Drex/rules alert decision (Session 13). Called by `finishFrame` via `decideFor(entryId)`. |
| `controllers/summaryController.js` | `GET /api/patients/:id/summary`, `GET /api/patients/:id/journal`. |
| `routes/internal.js` | `/internal/jobs/claim`, `/internal/jobs/:id/result`, `/internal/stats` + the `onlyHelpers` guard. |
| `routes/accounts.js` | Registers the two family routes above. |
| `app.js` | Mounts `/internal`. |
| `index.js` | Calls `journal.start()` and `queue.start()` (background timers) before `listen`. |
| `prisma/schema.prisma` + `prisma/migrations/*_ai_jobs/` | `AiJob` model / `ai_jobs` table. |
| `deploy/workers/pool.js` | The helper program (runs on host). No npm dependencies. |
| `deploy/workers/run.sh` | Loads the `assist` user's nvm, then `exec node pool.js`. |
| `deploy/workers/assist-helpers.service` | systemd unit. |
| `deploy/docker-compose.yml` | `assist-db` + `assist-api`; sets `TZ`, `AI_PROVIDER=workers`, `WORKER_TOKEN`, `JOURNAL_DIR`. |
| `deploy/update.sh` | One-command server update (see §12). |
| `.gitattributes` | Forces LF on `*.sh`, `*.js`, `*.service` (CRLF breaks them on Linux). |

Consumers: `family-dashboard/src/components/SummaryTile.jsx` (Today page), `family-dashboard/src/lib/story.js`
(`frameKey`/merging by `activity_key`); `companion-app/.../family/SummaryCard.kt` (family Today),
`.../family/Story.kt` (same merging), `data/api/ApiService.kt` (`summary()`), `ApiClient` read timeout 75 s.

## 5. Data model: `ai_jobs`

| Column | Meaning |
|---|---|
| `id` uuid | Job id (also returned to the phone as `log_entry_id` in the 202). |
| `kind` | `frame` \| `summary`. |
| `status` | `queued` → `running` → `done` \| `failed` \| `dropped`. |
| `patient_id` | Owner patient (no FK on purpose — queue rows are disposable). |
| `payload` json | frame: `{ device_id, captured_at, image (base64 JPEG), prompt }` — **`image` is deleted from the row when the job finishes or is dropped**. summary: `{ prompt, cache_key }`. |
| `result` json | frame: `{ log_entry_id, provider, ms }`. summary: `{ text, provider, ms }`. |
| `error` | Last error / reason (`stale`, `gemini throttle`, validation message, CLI error…). |
| `attempts` | Incremented on every claim (helper or Gemini); decremented by `release`. |
| `avoid_provider` | Set after a bad answer so the *other* provider gets the retry. |
| `provider` | `claude` \| `codex` \| `gemini` (who is/was working on it). |
| `worker_id` | e.g. `claude-2`, `codex-4`, or `backend` for Gemini. |
| `created_at`, `started_at`, `finished_at` | Timestamps. Index on `(status, created_at)`. |

Finished rows (`done`/`failed`/`dropped`) are deleted 24 h after `finished_at` by `sweep()`.

## 6. Job lifecycle

```
                     claim (helper or Gemini)          valid answer
   [queued] ─────────────────────────────▶ [running] ─────────────────▶ [done]
      ▲                                       │
      │  bad/failed answer, attempts < 2,     │  bad/failed again, or provider was gemini
      │  provider ≠ gemini → avoid_provider   │  → runFallback (Gemini) → [done] | [failed]
      └───────────────────────────────────────┤
      │                                       │  helper disconnected before receiving it → release → [queued]
      │                                       │  running > 120 s with a helper (helper died) → retryOrFallback
   frame queued > 90 s ───────────────────────────────────────────────────────────────▶ [dropped] "stale"
   queued > 20 s (frame) / 25 s (summary), nobody claimed ─▶ runFallback (Gemini)
   Gemini asked for a frame < 10 s after its previous frame ─▶ [dropped] "gemini throttle"
```

### Claim (the critical statement — `jobQueue.claim`)

```sql
UPDATE ai_jobs SET status='running', started_at=now(), provider=$provider, worker_id=$worker, attempts=attempts+1
WHERE id = (
  SELECT id FROM ai_jobs
  WHERE status='queued' AND (avoid_provider IS NULL OR avoid_provider <> $provider)
  ORDER BY (kind='summary') DESC, created_at DESC
  LIMIT 1 FOR UPDATE SKIP LOCKED)
RETURNING id, kind, payload;
```

- Find-and-mark is **one statement**; `SKIP LOCKED` means concurrent claimers never get the same row.
- **Summaries first** (a person is waiting), then **newest frame first** (family want "now"; old frames get
  shed rather than processed late).
- Gemini's own claim (`runFallback`) is an atomic `updateMany where {id, status:'queued'}` so it can never
  double-process a job a helper took.

### Timing constants

| Constant | Value | Where |
|---|---|---|
| Claim long-poll | up to 20 s, re-checks every 1 s, stops when the helper disconnects | `routes/internal.js` |
| Fallback to Gemini if unclaimed | frame 20 s, summary 25 s | `FALLBACK_AFTER_S` |
| Drop stale queued frame | 90 s | `FRAME_STALE_S` |
| Helper presumed dead | running > 120 s | `RUNNING_TIMEOUT_S` |
| Gemini frame throttle | ≥ 10 s between Gemini frame calls (excess dropped) | `GEMINI_FRAME_GAP_MS` |
| Sweep interval | 3 s | `jobQueue.start` |
| Finished-row retention | 24 h | `sweep` |
| CLI hard timeout in helper | frame 75 s, summary 120 s (SIGKILL) | `pool.js TIMEOUT_MS` |
| Summary request wait | 60 s, then 503 | `summaryController` |
| App HTTP read timeout | 75 s (was 20) | `ApiClient.kt` |
| Journal refresh | every 20 s for patients with new/acknowledged entries | `journalService.start` |
| Journal retention | 7 days (prune every 6 h) | `KEEP_DAYS` |
| Journal / UI merge gap | 15 min | `MERGE_GAP_MIN`, `MERGE_GAP_MS` |
| Context lines given to a frame job | last 10 journal lines of today | `enqueueFrame` |
| Helper cool-off | 3 consecutive CLI failures → pause 5, 10, 20, 40, 60 min (cap) | `pool.js health` |

### `AI_PROVIDER`

- `workers` (set in compose on the server): jobs wait for helpers; Gemini only as fallback.
- `gemini` (default; the laptop dev backend uses this): every job goes straight to `runFallback` → Gemini,
  still through the queue table (so journal, validation and throttle behave the same).

## 7. Prompts and the activity vocabulary (`services/prompts.js`)

Every provider gets byte-identical instructions; the backend builds the prompt and hands it to the helper inside
the job, so helpers contain **no prompt logic**.

**Vocabulary (`activity_key`, 19 keys):** `eating`, `drinking`, `cooking`, `taking_medication`, `watching_tv`,
`reading`, `resting`, `sleeping`, `walking_around`, `cleaning`, `on_phone`, `with_visitor`,
`getting_ready_to_leave`, `out_of_house`, `fallen`, `distressed`, `empty_room`, `not_visible`, `other`.

**Frame answer shape** (JSON only):
`{"activity_key","activity","person_state","objects_of_note":[≤4],"confidence":0..1,"requires_attention":bool}`

Key prompt rules: `activity` describes **what the person is doing** in family language (≤ 8 words) — never the
camera, photo quality or furniture; nobody there → `empty_room`, blurred/blocked/face against the lens →
`not_visible`, both with activity "no one in view"; `requires_attention` only for fall, visible distress or a
hazard; use the journal lines for continuity; never guess names.

**`validateFrame`:** extracts the first `{…}` even if the model wrapped it in prose/fences; rejects unknown
`activity_key`; clamps `confidence`; trims lengths; forces `requires_attention=true` for `fallen`/`distressed`.
The saved `log_entries.payload` = validated fields + `analysed_by` (`claude`|`codex`|`gemini`).

**Summary prompt:** warm, plain, uses the first name, no medical advice, only facts from the journal, says
"the camera saw nothing" for gaps. `today`/`morning`: 3–4 sentences. `week`: one paragraph ≤ 5 sentences,
overall pattern + only the standout days (no day-by-day list). Times as "around 7:30 am".

## 8. The helper program (`deploy/workers/pool.js`)

- Starts `CLAUDE_WORKERS` (default 4) + `CODEX_WORKERS` (default 4) async loops in one Node process.
- Loop: (skip while provider is in cool-off) → `POST /internal/jobs/claim` → if a job: write photo to a
  per-helper temp dir → run CLI → `POST /internal/jobs/:id/result` with `{worker_id, ok, output|error, ms}` →
  delete temp files. Backend unreachable → exponential retry 1 s … 30 s.
- **Claude call:** `claude -p "<prompt>" --model $CLAUDE_MODEL(sonnet) --output-format text --allowedTools Read`;
  for photos the prompt is prefixed with "First look at the photo at <path> (use the Read tool)". Only the
  Read tool is allowed. Auth: `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`).
- **Codex call:** `codex exec --skip-git-repo-check --sandbox read-only --output-last-message <file> [--image photo.jpg] "<prompt>"`;
  the answer is read from the file so banners/logs never mix in. Auth: `codex login --device-auth` (stored in
  the `assist` user's home).
- Every job is a **fresh CLI process** — no session carries over between photos.
- **Cool-off (circuit breaker):** only CLI failures (non-zero exit, timeout) count — a bad JSON answer is the
  backend's business, not a provider outage. 3 in a row → all helpers of that provider stop claiming for 5 min,
  then 10, 20, 40, 60 (cap); one success resets. Log lines: `failing repeatedly — pausing all claude helpers…`,
  `working again`.
- Log format: `HH:MM:SS [claude-2] frame 3f9a1c2b answered in 6.2s` / `… failed: <reason>`.

## 9. Security

- `/internal/*` answers **404** unless **both**: header `x-worker-token` equals `WORKER_TOKEN` (≥ 16 chars,
  constant-time compare) **and** the request has **no `X-Forwarded-For`** (Caddy always adds it, so nothing
  arriving through the public site can pass even with the token).
- The API container publishes only `127.0.0.1:3100` (host loopback); public traffic only via Caddy.
- Helpers run as the unprivileged `assist` user with `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem=full`;
  `assist` cannot read `/home/ubuntu` (other projects). Claude is limited to the Read tool; Codex to a read-only
  sandbox.
- Photos leave the box to Anthropic/OpenAI (helpers), Google (Gemini fallback) and Cloudinary (storage).
  Accepted for this single-user prototype; revisit with paid API terms before real users.

## 10. The day journal (`services/journalService.js`)

- File: `$JOURNAL_DIR/<patientId>/<YYYY-MM-DD>.md` (server: Docker volume mounted at `/app/journal`).
  Summary cache files sit beside it: `summary-<YYYY-MM-DD>-<range>.json`.
- **Rendered from the DB**, never appended by helpers. Re-rendered: after each finished frame
  (`finishFrame`), on every summary/journal request, and every 20 s for patients with new or newly
  acknowledged entries. Always time-ordered (by capture time, not finish time); rebuildable at any moment.
- Rules: consecutive frames with the same key (or, for old entries, the same text) within 15 min merge into a
  range `10:02–10:20`; `empty_room` + `not_visible` share the key `quiet` → "Camera · no one in view";
  flagged frames never merge and get `⚠` plus `(rules ✓/✗, Drex ✓/✗)` when Drex answered; geofence pings
  only produce lines on change (Left / Still outside / Back inside); heart-rate lines only when flagged;
  emergency-tag sightings; quiz attempts grouped per ~10 min ("Memory quiz · 2 of 3 right"); acknowledged
  alerts get "· seen by family HH:MM".

Example:
```
# Margaret · Sun, 18 Oct 2026
07:40–08:05  Camera · eating breakfast at the table (seated, calm)
13:21        ⚠ Left the safe places · family alerted (rules ✓, Drex ✓)
14:07        Back inside the safe places
18:51–18:52  Camera · no one in view
```
Time zone: the API container runs with `TZ=Asia/Kolkata`; "today" and all journal times are IST. (Demo data
seeded before TZ was set showed breakfast "in the afternoon" — re-seeding fixed it.)

## 11. Summaries (`controllers/summaryController.js`)

`GET /api/patients/:id/summary?range=today|morning|week[&fresh=1]` (caregiver JWT, own patient only)
1. `writeDay` (bring today's file in line with the DB).
2. Build input: today's file; `morning` = header + lines starting 00:00–11:59; `week` = the last 7 day files.
3. No lines → immediate "Nothing has been recorded for X yet today." (no AI call).
4. Cache: fingerprint = `<line count>:<last line>`; same fingerprint → cached body (`cached: true`)
   unless `fresh=1`.
5. Else enqueue a summary job, `waitFor` ≤ 60 s → `{ range, summary, provider, generated_at, journal_lines, cached }`,
   or **503** "Could not write the summary just now…".

`GET /api/patients/:id/journal?date=YYYY-MM-DD` returns the raw Markdown (useful in demos).

UI: dashboard Today → "In a few words" tile (Today / This morning / This week, "Write summary", "Written by
Claude · from N journal lines", "Write again" = `fresh=1`). App family Today → the same as a card.

## 12. Deployment & operations (server 3.107.133.23, Ubuntu 24.04, 8 GB, shared with Forgejo/OneDev)

| Thing | Where |
|---|---|
| Public URL | `https://assist.3-107-133-23.sslip.io` (sslip.io wildcard DNS; Caddy auto-HTTPS) |
| Caddy | existing container `caddy`, config `/home/ubuntu/forgejo-stack/Caddyfile` (site block `reverse_proxy assist-api:3000`), network `forgejo-stack_default`; reload: `sudo docker exec caddy caddy reload --config /etc/caddy/Caddyfile` |
| Backend code | `/home/ubuntu/assist/backend` (owner `ubuntu`); compose project `assist` in `deploy/` |
| App secrets | `/home/ubuntu/assist/backend/.env` (Gemini, Cloudinary, Drex, JWT, PUBLIC_BASE_URL) |
| Compose secrets | `/home/ubuntu/assist/backend/deploy/.env` (`POSTGRES_PASSWORD`, `WORKER_TOKEN`) |
| Containers | `assist-db` (postgres:17, volume `assist_pgdata`), `assist-api` (port `127.0.0.1:3100`, volume `assist_journal`) |
| Helpers | user `assist`; code `/home/assist/helpers/{pool.js,run.sh}`; env `/home/assist/.config/assist/workers.env` (`CLAUDE_CODE_OAUTH_TOKEN`, `WORKER_TOKEN`, optional `CLAUDE_WORKERS`, `CODEX_WORKERS`, `CLAUDE_MODEL`, `ASSIST_API`); Node 22 via nvm; unit `/etc/systemd/system/assist-helpers.service` |

**Update (from the laptop, PowerShell)** — the maintainer rebuilds the package with
`git -C D:\backend-server archive --format=tar.gz -o D:\assist-backend.tgz HEAD`, then:
```
scp -i "C:\Users\Nikhil\.ssh\forgejo-demo.pem" D:\assist-backend.tgz ubuntu@3.107.133.23:/tmp/
ssh -i "C:\Users\Nikhil\.ssh\forgejo-demo.pem" ubuntu@3.107.133.23 "bash ~/assist/backend/deploy/update.sh"
```
`update.sh`: untar over the code (keeping both `.env` files) → `docker compose up -d --build` (migrations run on
container start via `prisma migrate deploy`) → health check → install `pool.js`/`run.sh` into
`/home/assist/helpers` → restart `assist-helpers` if enabled.

**Runbook (on the server as `ubuntu`)**

| Need | Command |
|---|---|
| Helper log (live) | `journalctl -u assist-helpers -f` |
| Restart helpers | `sudo systemctl restart assist-helpers` |
| Backend log | `cd ~/assist/backend/deploy && sudo docker compose logs -f --tail 50 assist-api` |
| Queue stats (last hour) | `curl -s 127.0.0.1:3100/internal/stats -H "x-worker-token: $(grep ^WORKER_TOKEN ~/assist/backend/deploy/.env \| cut -d= -f2)"` |
| Look at the rail | `cd ~/assist/backend/deploy && sudo docker compose exec assist-db psql -U assist_app -d assistdb -c "select kind,status,provider,worker_id,attempts,error,created_at from ai_jobs order by created_at desc limit 15;"` |
| Re-seed demo data | `cd ~/assist/backend/deploy && sudo docker compose exec assist-api node prisma/seed.js` |
| Claude only / Codex only | set `CODEX_WORKERS=0` or `CLAUDE_WORKERS=0` in `workers.env` (via `sudo -iu assist`), restart helpers |
| Re-auth Claude | as `assist`: `claude setup-token`, save with the masked `read -rs` method into `workers.env`, restart helpers |
| Re-auth Codex | as `assist`: `codex login --device-auth`, restart helpers |

**Troubleshooting**

| Symptom | Likely cause / check |
|---|---|
| Moments stop appearing | `stats`: many `queued`→`dropped "stale"` = no helper claiming (service down? both providers in cool-off?). `failed` with "no GEMINI_API_KEY" = fallback not configured. |
| `claude … 401 OAuth access token is invalid` | Token mis-copied (terminal wrapping). Check length ≈108 and prefix `sk-ant-oat01-` without printing it; regenerate. |
| `pausing all <provider> helpers` in log | Outage / expired sign-in / quota. The other provider and Gemini keep the system running. |
| `/internal/*` returns 404 from the helper | `WORKER_TOKEN` differs between `deploy/.env` and `workers.env`, or request went through Caddy. |
| Summary returns 503 | No helper or Gemini answered in 60 s; check helper log. |
| Many identical-looking moments | Old entries (no `activity_key`) merge by text only; new ones merge by key. |

## 13. Verified so far

- Local (laptop): enqueue → 202; `/internal` refuses wrong token and any `X-Forwarded-For`; bad answer from
  "claude" re-queued with `avoid_provider` and taken by "codex"; valid answer saved, journal line written;
  summary job claimed and returned; cache hit; abandoned long-poll no longer swallows a job (bug found in
  testing and fixed with `res.on('close')` + `release`); cool-off logic unit-checked.
- Server: real Claude helper wrote a weekly summary in 9.1 s; real camera photos from the phone were described
  by helpers and appeared as moments; Drex/rules decision and journal run on each.

## 14. Known limitations / next steps

- Frame merging in the UI depends on `activity_key`; entries from before Session 14 merge by text only.
- `requires_attention` from a helper is trusted after validation; only `fallen`/`distressed` force it. Drex then
  records its view (shadow mode).
- No per-patient fairness in the queue (one global newest-first order) — fine for one patient.
- `ai_jobs.payload.image` holds base64 until the job ends (~100 KB per frame); fine at this scale.
- Gemini fallback frames are throttled to one per 10 s; if both providers are down, most frames are dropped
  by design.
- Not yet built on top of this: notifications to family, scheduled daily quiz via the queue, voice notes.
- Move to paid API keys before any real users (subscription terms).
