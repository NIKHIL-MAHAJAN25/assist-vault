# AI helpers: job queue, worker pool, day journal, summaries — and everything built on them

> **Status:** running in production on the user's EC2 box since 2026-10-18 (Session 14). Extended the same day
> with the family Assistant and phone actions (Session 15), demo kit + notifications (16), the autonomous agent
> — rules, fall check, bedtime story (17) — and natural voice (17b). Last updated at backend `1b686e0`.
> **Code:** `D:\backend-server` (backend side + helper program) · consumers in `family-dashboard` and `companion-app`.
> **Read with:** [api-contract.md](api-contract.md) (routes), [db-schema.md](db-schema.md) (`ai_jobs`),
> [decisions-log.md](decisions-log.md) Sessions 14–17b (why each choice), [ai-prompting-patterns.md](ai-prompting-patterns.md),
> [demo-playbook.md](demo-playbook.md) (how to show all of it on stage).
>
> Part A (§1–§14) is the queue/helper/journal core. Part B (§15–§23) is what was built on top of it.
>
> This page is written as context for another engineer or AI picking the system up cold. It describes what
> exists, exactly how it behaves, every timing constant, where things live on the server, and how to operate it.
> No secret values appear here — only the names and locations of the files that hold them.

---

## 1. What it is, in one paragraph

Camera photos, family "summarise my day" requests, Assistant questions and bedtime stories are turned into **jobs** in a Postgres table. A pool of
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
| `services/jobQueue.js` | The queue: `enqueueFrame`, `enqueueSummary`, `enqueueAgent`, `enqueueStory`, `claim`, `release`, `complete`, `finishFrame`, `retryOrFallback`, `runFallback` (Gemini), `drop`, `sweep`, `waitFor`, `stats`, `start`. |
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
| *Added in Sessions 15–17b* | see Part B §15 for the full list (Assistant, commands, rules, fall check, story, voice, demo, notifications). |

Consumers: `family-dashboard/src/components/SummaryTile.jsx` (Today page), `family-dashboard/src/lib/story.js`
(`frameKey`/merging by `activity_key`); `companion-app/.../family/SummaryCard.kt` (family Today),
`.../family/Story.kt` (same merging), `data/api/ApiService.kt` (`summary()`), `ApiClient` read timeout 75 s.

## 5. Data model: `ai_jobs`

| Column | Meaning |
|---|---|
| `id` uuid | Job id (also returned to the phone as `log_entry_id` in the 202). |
| `kind` | `frame` \| `summary` \| `agent` (Assistant question) \| `story` (bedtime story). |
| `status` | `queued` → `running` → `done` \| `failed` \| `dropped`. |
| `patient_id` | Owner patient (no FK on purpose — queue rows are disposable). |
| `payload` json | frame: `{ device_id, captured_at, image (base64 JPEG), prompt, command_id? }` (`command_id` = a "take a photo now" request) — **`image` is deleted from the row when the job finishes or is dropped**. summary: `{ prompt, cache_key }`. agent / story: `{ prompt }`. |
| `result` json | frame: `{ log_entry_id, provider, ms }`. summary / agent / story: `{ text, provider, ms }`. |
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
   queued > 20 s (frame) / 25 s (summary, story) / 15 s (agent), nobody claimed ─▶ runFallback (Gemini)
   Gemini asked for a frame < 10 s after its previous frame ─▶ [dropped] "gemini throttle"
```

### Claim (the critical statement — `jobQueue.claim`)

```sql
UPDATE ai_jobs SET status='running', started_at=now(), provider=$provider, worker_id=$worker, attempts=attempts+1
WHERE id = (
  SELECT id FROM ai_jobs
  WHERE status='queued' AND (avoid_provider IS NULL OR avoid_provider <> $provider)
  ORDER BY (kind <> 'frame') DESC, (payload->>'command_id') IS NOT NULL DESC, created_at DESC
  LIMIT 1 FOR UPDATE SKIP LOCKED)
RETURNING id, kind, payload;
```

- Find-and-mark is **one statement**; `SKIP LOCKED` means concurrent claimers never get the same row.
- **Text jobs first** (summary, agent, story: a person is waiting), then **photos someone asked for**
  ("take a photo now"), then **newest frame first** (family want "now"; old frames get shed rather than
  processed late).
- Text jobs are accepted when the answer is longer than 20 characters (agent: 2); shorter = "empty summary"
  → retry on the other provider.
- Gemini's own claim (`runFallback`) is an atomic `updateMany where {id, status:'queued'}` so it can never
  double-process a job a helper took.

### Timing constants

| Constant | Value | Where |
|---|---|---|
| Claim long-poll | up to 20 s, re-checks every 1 s, stops when the helper disconnects | `routes/internal.js` |
| Fallback to Gemini if unclaimed | frame 20 s, summary 25 s, agent 15 s, story 25 s | `FALLBACK_AFTER_S` |
| Drop stale queued frame | 90 s | `FRAME_STALE_S` |
| Helper presumed dead | running > 120 s | `RUNNING_TIMEOUT_S` |
| Gemini frame throttle | ≥ 10 s between Gemini frame calls (excess dropped) | `GEMINI_FRAME_GAP_MS` |
| Sweep interval | 3 s | `jobQueue.start` |
| Finished-row retention | 24 h | `sweep` |
| CLI hard timeout in helper | frame 75 s, summary 120 s, agent 60 s, other (story) 90 s (SIGKILL) | `pool.js TIMEOUT_MS` |
| Summary request wait | 60 s, then 503 | `summaryController` |
| Assistant question wait | 60 s, then 503 | `assistantController.ask` |
| Story wait | 90 s, then 503 | `storyService.sendStory` |
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
  alerts get "· seen by family HH:MM"; **family actions on the phone** (Session 15+) appear as
  `Priya → phone · "Lunch is in the fridge"` and answers as `Answered Priya: "Yes"` (expired actions are left out).
  The journal is also refreshed for patients with new device commands.

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
- ~~Not yet built on top of this: notifications to family, voice notes~~ — built in Sessions 15–16 (Part B).
  Still not built: a scheduled daily quiz (the quiz can be *sent* on demand), replay-a-walk, doctor's report.
- Move to paid API keys before any real users (subscription terms).

---

# Part B — built on top of the queue (Sessions 15–17b)

The same rail (queue → helpers → Gemini fallback → journal) now powers a family **Assistant**, a set of
**phone actions**, an **autonomous agent** that watches on its own, a **natural voice**, a **notification feed**
and a **demo kit**. None of it added a new AI path: every model call that writes text is still a job in `ai_jobs`
(the only exception is speech, §21, which calls Gemini TTS directly from the backend).

## 15. Files added (backend-server)

| File | Role |
|---|---|
| `services/assistantService.js` | Assistant: `ACTIONS` registry (closed list the agent may propose), `gatherContext`, `buildPrompt`, `parseAnswer`. |
| `services/commandTypes.js` | `TYPES`: every action the phone can carry out, each with `clean(body)` validation + a label; `describe(cmd)` for journal/context; `hhmmToDate`. |
| `controllers/assistantController.js` | `ask`, `createCommand`, `listCommands`, `pullCommands` (phone), `ackCommand` (phone), `uploadVoiceNote`, rules CRUD, `sendStoryNow`, `storySettings`. |
| `services/ruleEngine.js` | "Tell me when…" rule kinds, `cleanRule`, `evaluate`, `runRules`, `describeKinds`. |
| `services/fallCheck.js` | `start` (ask "are you okay?"), `answered`, `escalate`, `sweep`. |
| `services/storyService.js` | `storyPrompt`, `sendStory`, `sweep` (bedtime). |
| `services/scheduler.js` | Every 30 s: `runRules` → `fallCheck.sweep` → `story.sweep`; never overlaps itself. |
| `services/voiceService.js` | Gemini TTS: `speechFor` (cached), `phraseFor`, `warmUp`, `prepareCommand`, `wantsVoice`, `holdFor`. |
| `controllers/notificationController.js` | `list` (one merged feed), `broadcast` ("tell the family"). |
| `services/demoService.js` + `controllers/demoController.js` | Demo kit: `demo:` devices, saved scenes, walk simulator, answer-as-patient, tick, fall, reset. |
| `controllers/geofenceController.js` | Now exports `recordPing` (shared by the phone route and the walk simulator); ignores the real phone's pings while a demo walk runs. |
| `controllers/frameController.js` | Accepts `command_id` on an upload (the answer to "take a photo now"). |
| `app.js` | Adds `/tts` static (natural-voice audio, `maxAge 30d, immutable`). |
| `index.js` | Starts journal, queue, **scheduler**, and the **TTS warm-up** of fixed phrases. |
| Migrations | `20261018140052_device_commands`, `20261018152021_family_broadcasts`, `20261018154616_batch_c_rules_events` (`watch_rules`, `family_events`, `patients.bedtime_story_at`). |
| `deploy/docker-compose.yml` | Adds `DEMO_MODE` (default `1`). |

Front-end counterparts: dashboard `pages/Assistant.jsx`, `pages/Demo.jsx`, `components/PhoneActions.jsx`
(PhoneActionCard, Recorder, BroadcastCard, quick chips), `components/AgentCards.jsx` (RuleCard, StoryCard),
`components/Notifications.jsx`, `components/SettingsPanels.jsx` (BedtimeStory), `lib/demoCases.js`.
App: `service/MonitoringService.kt` (command loop), `messages/PatientInbox.kt`, `ui/home/InboxCards.kt`
(AskCard, QuizInviteCard, CalmCard, StoryCard), `voice/TtsManager.kt`, `family/FamilyAlertsService.kt`.

## 16. The family Assistant (agent jobs)

`POST /api/patients/:id/assistant { message, history }` → `{ reply, actions[], provider, sources[], ms }`.

1. **Grounding** (`gatherContext`): now; age of the last device reading (flags STALE); phone location (which
   safe place / OUTSIDE); latest camera activity; unacknowledged alerts; safe-place names; today's quiz score;
   active "tell me when" rules; the last 5 phone actions with status and sender; **today's full journal**.
   `sources` (location, camera, journal, alerts, messages) are shown as chips under the answer.
2. **Prompt** (`buildPrompt`): answer ONLY from the context, otherwise say "I don't know"; 1–3 warm sentences;
   no medical advice; say plainly if something is unsafe; last 8 turns of history; the whole `ACTIONS` list with
   JSON shapes; reply with ONE object `{"reply":"…","actions":[…]}`.
3. **Queue:** `enqueueAgent` → helper (jumps ahead of frames) → `waitFor` 60 s → 503 if nobody answered.
4. **Parse** (`parseAnswer`): first `{…}` in the text; each action must be a known type and pass its own
   `validate`; max 4; anything unparseable becomes plain text with no actions.
5. **The agent never acts on the patient.** Actions come back as proposal cards; the family member taps to
   confirm, which calls the normal REST route (`POST /commands`, `/rules`, `/story`, `/broadcasts`) or just
   renders something (`map`, `photo`, `summary`).

**Action registry (`ACTIONS`):** `speak`, `ask`, `reminder`, `capture`, `ring`, `guide_home`, `calm`, `quiz`,
`rule`, `story`, `broadcast`, `map`, `photo`, `summary`. Adding a capability = one entry here + a card in
`PhoneActions.jsx` / `AgentCards.jsx`.

UI: dashboard **Assistant** tab (nav right after Home): chat with an animated orb, quick chips, live
speech-to-text input, proposal cards, live status of each action ("Sent → On her phone → Answered 'Yes'"),
voice-note recorder, broadcast card. Exact-height layout, hidden scrollbars.

## 17. Phone actions (device commands)

Table `device_commands`: `id, patient_id, type, payload json, status, created_by, sender_name, error,
created_at, delivered_at, done_at, expires_at`.

```
family confirms ─▶ POST /api/patients/:id/commands {type,…}   (TYPES[type].clean validates)
                     │ status=pending, expires in 10 min (COMMAND_TTL_MIN); spoken types get voice_pending=true
                     ▼
phone (MonitoringService, every 6 s) ─▶ GET /api/devices/commands   (device key)
                     │ up to 5 pending → status=delivered (held back while voice_pending, see §21)
                     ▼
phone carries it out ─▶ POST /api/devices/commands/:cid/ack {ok, error?, reply?}
                     → done | failed;   still pending after expires_at → expired
```

| Type | Payload | What the phone does | Ack |
|---|---|---|---|
| `speak` | `text` ≤ 280 | Notification + "Message from X" card, says it aloud | when spoken |
| `ask` | `question` ≤ 200, `quick` ≤ 3 | Speaks the question, big tap-to-answer buttons (AskCard) | **when she answers**: `reply` + `replied_at` stored |
| `voice` | Cloudinary `url`, `seconds`, `text` | Plays the family's recorded voice note; "Play again" | when played |
| `ring` | — | Max-volume ringtone for 20 s, overrides silent | when rung |
| `reminder` | `text`, `at` (ISO; HH:MM = today, or tomorrow if already past) | Stores it; at the time: notification + spoken | when stored |
| `capture` | — | Takes an ESP32 photo now, uploads it with `command_id` | `uploaded`; **done once a helper has described it** (`finishFrame` copies activity/objects/photo onto the command) |
| `guide_home` | — | Says "time to head home", opens the way-home arrow, location ping every 15 s for 15 min | when started |
| `quiz` | — | QuizInviteCard + spoken invite | when shown |
| `calm` | `voice_url?` | CalmCard + slow spoken breathing script, then the family's latest voice note | when done |
| `story` | `text` ≤ 1500, `title` | StoryCard ("Read it again") + spoken | when spoken |

Failures are explicit: e.g. the camera not answering → `failed` "home camera didn't answer", shown on the card
and in the notification feed. Voice notes: `POST /api/patients/:id/voice-notes {audio base64, mime}` →
Cloudinary (`resource_type: video`, folder `assist/voice`) → mp3 URL; the `;codecs=opus` part of the browser's
mime type is stripped first (it caused a 500). Every action also appears in the day journal (§10).

## 18. The autonomous agent: scheduler, rules, fall check, bedtime story

**Scheduler** (`scheduler.js`): `setInterval(tick, 30 s)`; a `running` flag stops overlapping ticks. Demo:
`POST /api/demo/:id/tick` runs one immediately.

**"Tell me when…" rules** (`ruleEngine.js`; table `watch_rules`: kind, params, label, repeat `once|daily`,
active, fired_count, last_fired_at, last_checked_at, creator_name). The agent maps plain English onto a closed set:

| Kind | Params | Fires when |
|---|---|---|
| `arrives_home` | — | a location ping goes from outside to inside the safe places (the earlier ping only counts if < 10 min old, so a stale reading can't fake an arrival) |
| `leaves_home` | `after?` HH:MM | inside → outside (optionally only after a time) |
| `not_seen_by` | `activity`, `by` | at `by` the camera has not seen that activity today (`up` = any activity except sleeping / no one in view) |
| `still_at` | `activity`, `at` | at `at` the latest camera activity is still that one (e.g. sleeping at 09:00) |
| `seen` | `activity` | the camera sees it (e.g. taking_medication) |
| `quiet_for` | `minutes` 30–480 | no person on camera for N minutes, 08:00–21:00 only |

A hit writes a `family_events` row (kind `rule`) → it appears in the notification feed. `once` rules switch
off after firing; `daily` rules stay active. Labels are plain English ("Tell me when Margaret gets home").

**Fall check** (`fallCheck.js`): triggered by a validated frame with `activity_key = fallen` (in `finishFrame`)
or by `POST /api/demo/:id/fall`.
1. One at a time: skipped if a fall-check `ask` already exists from the last 10 min.
2. Creates an `ask` command "Margaret, are you okay?" with buttons `I'm okay` / `I need help`
   (`fall_check: true`) + an **act-now** family event "Possible fall — checking with Margaret".
3. "okay / fine / yes…" (without "help") → info event "says she's okay" (a call later is still suggested).
4. "I need help", any other answer, or **no answer within 2 min** (`sweep`) → escalate once: act-now event
   "⚠ Possible fall — Margaret needs you" + a `ring` command on her phone.
Fall-check answers are filtered out of the plain "reply" notifications so the family isn't told twice.

**Bedtime story** (`storyService.js`; `patients.bedtime_story_at` HH:MM, set in Settings → Bedtime story):
journal → `story` job ("Today you…", 4–6 warm sentences, only facts from the journal, no alerts/falls/times;
a quiet day is "a quiet, restful day"; ends with goodnight and that the family loves them) → `story` command +
info event. Sent automatically within 10 min after bedtime, once a day, only when a real (non-`demo:`) phone is
paired; or on demand via `POST /api/patients/:id/story` / the agent's `story` action.

## 19. Notifications (polled, no FCM)

`GET /api/patients/:id/notifications?since=ISO` (default: last 24 h) merges one time-sorted feed:
camera/location **alerts** (`requires_attention`), Margaret's **answers** to questions, actions that **failed or
expired**, **family_events** (rules, fall checks, stories), and **notes** from the family circle
(`family_broadcasts`, with a `mine` flag). Each item: `{id, kind, severity: info|watch|act, at, title, body, …}`.
`POST /api/patients/:id/broadcasts {text}` = "Tell the family".

Consumers poll it: website `components/Notifications.jsx` (bell with unread count, toasts, desktop
notifications; **act-level toasts stay until dismissed**); the app's family side `family/FamilyAlertsService.kt`
(foreground service, phone notifications). The app's family side is **not yet tested on a real phone** (the
user's phone plays Margaret).

## 20. Demo kit (`DEMO_MODE=1`, website `/demo`)

All are family-authenticated routes under `/api/demo/:id/*`; 403 when demo mode is off.

| Route | Does |
|---|---|
| `GET /demo/:id` | State: saved scenes, safe places, open question, latest photo, whether a walk is running |
| `POST /demo/:id/scenes/:name/save` | Keep the latest (or a chosen) camera photo as a scene: `eating`, `tv`, `resting`, `leaving`, `fall`, `empty` (saved in `<journal>/<pid>/demo-scenes.json`) |
| `POST /demo/:id/scenes/:name/play` | Replay it as a new frame from device `demo:camera` through the **real** pipeline (helper → validation → Drex → journal → fall check) |
| `POST /demo/:id/walk {away\|home\|stop}` | Simulated walk: 9 steps of ~75 m every 4 s on the bearing that leaves **every** safe place soonest (slight zig-zag); `home` walks back, then hands over to the real phone after 30 s. Real phone pings are ignored while it runs |
| `POST /demo/:id/answer {reply}` | Answer the latest open question as Margaret (no phone needed) |
| `POST /demo/:id/tick` | Run the scheduler now |
| `POST /demo/:id/fall` | A fall "seen by the camera" without staging one (no photo) → fall check |
| `POST /demo/:id/reset` | Deletes today's commands, notes, events and rules, all `demo:` readings, and this patient's whole queue |

`family-dashboard/src/lib/demoCases.js` holds **21 scripted test cases** (step, what to click, what judges
should see), rendered on the Demo page; the stage version is [demo-playbook.md](demo-playbook.md).

## 21. Natural voice (Gemini TTS), Session 17b

**Why:** the phone's built-in voice sounded robotic. **Rule:** natural when possible, never silent.

- **Model and voice:** `gemini-3.1-flash-tts-preview` first, `gemini-2.5-flash-preview-tts` as fallback (the
  first that works is remembered); voice **Sulafat** (`TTS_VOICE`); style "gently, warmly and a little slowly,
  like a caring family member". Output is 24 kHz 16-bit PCM, wrapped as WAV.
- **Cache:** key = sha256(voice | style | text), 24 hex chars; file `<JOURNAL_DIR>/tts/<key>.wav`, index
  `tts-index.json`; served at `PUBLIC_BASE_URL/tts/<key>.wav` with 30-day immutable cache headers. The same
  words are generated once, ever; simultaneous requests for the same words share one generation (`inflight`).
  The first version uploaded to Cloudinary (~15 s); serving from the server's own disk made it ~6–7 s for new
  words and ~1 s cached.
- **What gets voiced** (`phraseFor`, kept identical to the app's fallback lines): speak, ask (incl. the fall
  check), reminder, story, guide_home, quiz, calm. Not voiced: ring, capture, voice notes (already a human).
- **Waiting:** spoken commands are created with `voice_pending: true`; `prepareCommand` makes the audio and
  sets `audio_url` + `voice_pending: false`. `pullCommands` holds a pending-voice command back for at most
  10 s (story 45 s) + 2 s, then releases it **without** audio.
- **Warm-up:** at startup "are you okay?" and "time to head home" are generated for every patient, so the
  urgent lines are always cache hits.
- **Phone side:** `MonitoringService.sayOrPlay(audioUrl, fallback)` plays the WAV and waits until it ends; no
  URL or a playback error → `TtsManager.speakQueued(fallback)` (best installed natural voice, rate 0.9).
  Messages show "Say it again"; the story's "Read it again" replays the same audio.
- **Off switch:** `TTS=off`, or no `GEMINI_API_KEY` → the phone's voice for everything. `TTS_TIMEOUT_MS`
  (default 12 s) caps a single generation.
- **Limits:** preview TTS models have low free-tier daily caps; the cache means a demo needs only a handful
  of new generations. Check the AI Studio quota before the auction.

## 22. Verified live (server + the user's phone acting as Margaret's)

Assistant answers grounded in the journal (by Claude); speak, ask → reply shown on the card, quiz, calm,
reminder, guide home with the arrow, voice notes; capture's failure path; demo walk → alert → answer as
Margaret → scene replay via Claude → walk home → reset; rule `arrives_home` fired; fall check escalated with a
ring; bedtime story written by Claude and delivered. **Natural voice: verified on the laptop backend only.** It
needs the server update (`1b686e0`) and the new app (`461ada3`) installed, then a phone test.

## 23. Open items around this system

- Deploy `1b686e0` and install app `461ada3`; test the natural voice on the phone.
- ESP32 camera currently not answering (capture fails cleanly); 5 demo scenes still to stage and save.
- The app's family-side notifications need a second phone to test.
- Batch D (replay her walk, "where did she leave it", doctor's report, reminiscence) and Batch E (Assistant
  speaks its answers, evidence photos, "how it decided").
- The website still runs on the laptop; deploy it onto the server last. Rotate the Drex key; move to paid
  API keys before real users.

## Changelog

| Session | Backend commit | Change |
|---|---|---|
| 14 | `228e4af`, `918804d`, `211788f` | Queue, helpers, journal, summaries; cool-off; tighter summary prompt |
| 14 | `e8b7ea7` | Prompt describes the person, not the camera; "no one in view" merges |
| 15 | `0995bdb`, `9e1fb75`, `0497ed0` | Assistant (agent jobs), device commands, all phone actions, voice notes |
| 16 | `5a885a4` | Demo kit, notification feed, family broadcasts |
| 17 | `62409f5` | Scheduler, rules, fall check, bedtime story, family events |
| 17b | `1b686e0` | Natural voice (Gemini TTS, cached, phone fallback) |
