# Decisions log

Judgment calls made during the build, with rationale. Only the master session writes here (see
[../CLAUDE.md](../CLAUDE.md) working agreement). Newest at the bottom.

---

## Session 1 — 2026-09-05

### `patients.home_address` — no dedicated field
Handled entirely via `geofence_zones` instead of a separate `home_address`/default-safe-zone field on
`patients`. A home address is just a geofence zone with a conventional name (e.g. "Home") and no special
casing anywhere else. A dedicated field would duplicate data that already lives in `geofence_zones` and
could drift out of sync with it — violates the "no field duplicating data that belongs in another table"
rule the spec itself calls out for schema extensions.

### `enrolled_faces.reference_embedding` — store the descriptor vector directly
Stored as a `JSONB` array of floats (the face-api.js 128-d descriptor), not as a reference image
recomputed at match time. Recomputing from a stored image on every match would mean re-running face
detection + descriptor extraction per comparison, for no benefit — the descriptor *is* the thing matching
happens against. Storing it directly is both simpler to implement and cheaper at request time.

### Local Postgres install method — winget, not Chocolatey
The spec called for a local Postgres install via Chocolatey for dev. Chocolatey's install failed in this
environment: the shell it ran from wasn't elevated, and Chocolatey's own lock-file handling doesn't
tolerate that (`Access to the path 'C:\ProgramData\chocolatey\lib-bad' is denied`). `winget install` used
the same public EDB installer but self-elevates the way it packages installer execution, so it succeeded
without needing an elevated shell. Functionally equivalent outcome (PostgreSQL 17, official EDB binaries,
Windows service) — the tool choice doesn't affect anything downstream. First install landed a randomly
generated superuser password with no way to recover it (winget didn't surface it and it's not logged
anywhere), so that instance was uninstalled and reinstalled with the password pinned via the installer's
own silent-install override (`--superpassword`) rather than trying to reset auth on the broken instance.

### Repo split confirmed as originally scoped
Sibling repos (`AssistVault`, `backend-server`, `companion-app`, `family-dashboard`, `watch-app`) under
`D:\`, not a monorepo. Matches the Vedika precedent this project's methodology is explicitly modeled on
(see [../CLAUDE.md](../CLAUDE.md)) and keeps `api-contract.md` doing real work as the only thing
non-backend repos need to read.

### Tools evaluated and deliberately not adopted
- **Graphiti** (temporal knowledge-graph memory via Neo4j) — solves a relationship-memory problem this
  project doesn't have at this scale; Postgres foreign keys are sufficient.
- **Graphify** (codebase-to-knowledge-graph tool) — solves "rediscovering a large mature codebase every
  session," which doesn't apply to a greenfield project with nothing built yet. Worth revisiting once
  real codebases exist.
- **Munder Difflin** (multi-agent desktop harness, file-based hive coordination) — built for unattended,
  long-horizon, many-agent coordination, not this project's actual shape of sequential, human-directed
  sessions. Also carries real recent Windows-specific coordination bugs, only fixed in its most recent
  release.

Don't re-propose any of these without a genuine change in project shape (e.g. Graphify becomes relevant
once `backend-server` is a large existing codebase future sessions need to rediscover).

---

## Session 2 — 2026-09-07 — companion-app + family-dashboard stack

Full Phase 0-2 scope confirmed with the user for both components (everything short of Phase 3:
wearable app, "Lost?" widget, NFC, voice-note playback). Built via two parallel builder subagents
(master/slave split, now useful since `api-contract.md` is locked and these are independent consumers
of it — see `../CLAUDE.md` working agreement). This session's orchestration primitive is the `Agent`
tool (in-process subagents with a `model` override), not separate CLI processes with per-repo
`settings.json` grants like the Vedika precedent `CLAUDE.md` describes — read/write separation is
enforced by prompt scope (each slave gets only its own repo's relevant contract slice pasted in,
explicit instruction not to touch the vault or the sibling repo) rather than filesystem permission.
This session is running Sonnet, not Opus as the spec assumed for the master role — accepted as a
deviation, not blocking.

### `companion-app` — Kotlin/Android
- Jetpack Compose + Material3, min SDK 26. Least boilerplate, fits the persona's low-nav-complexity
  need better than XML views.
- Retrofit + OkHttp + kotlinx.serialization for the API client; `EncryptedSharedPreferences` for the
  device API key.
- `FusedLocationProviderClient` for GPS; Android `SpeechRecognizer`/`TextToSpeech` for STT/TTS — all
  on-device, no extra deps beyond Play Services.
- A foreground `Service` with a coroutine ticker drives the 5s frame-capture loop and GPS pings —
  `WorkManager` doesn't fit sub-minute intervals.
- Frame source is a `FrameSource` interface with an HTTP-polling implementation against a
  **configurable** base URL, not a hardcoded ESP32 address — the physical device isn't wired yet
  (that's build-order step 5, a separate later session). This keeps the change-detection + relay logic
  real and testable now without blocking on hardware.
- **Known limitation, not silently built around**: the TTS-on-recognition toggle exists in Settings but
  has no real trigger — Gemini's frame analysis reports whether *a* person is visible, not *which*
  enrolled face. Wiring that requires backend Phase 2 work (face-api.js against captured frames) that
  doesn't exist yet.

### `family-dashboard` — React
- Vite + React 18, plain JavaScript (matches `backend-server`, skips TS tooling overhead for a
  prototype-scale dashboard).
- `react-router-dom` + `@tanstack/react-query` + Tailwind CSS; `react-leaflet`/OpenStreetMap for the
  geofence map editor (no API key needed).
- Face enrollment bundles `face-api.js` **client-side** (it's a browser-first library) to compute the
  descriptor from an uploaded reference photo in the browser, then POSTs the resulting array to
  `/api/enrolled-faces` — makes the feature usable from a photo instead of hand-typing a 128-number
  array.
- JWT kept in memory + `localStorage` — deliberate simplicity-over-hardening for a prototype; the
  XSS-exposure trade-off against httpOnly cookies is a known, accepted one, not an oversight.
- `patient`-role logins get a "use the companion app" message instead of the dashboard — the persona
  this system is designed around isn't the dashboard's user; `family_caregiver` is the only full-access
  role.

### Contract fix: `quiz/today` and `quiz/attempts` moved from JWT to device API key
Caught while prepping the companion-app dispatch: `api-contract.md` had these marked JWT-only, but
there's no patient login flow anywhere in the spec — the companion app is device-authenticated only.
As written, the patient could never actually take the daily quiz from the app. Fixed in `backend-server`
(`middleware/auth.js` now accepts `device_id` from query as well as body, since `quiz/today` is a GET;
`quizController.getTodayQuiz` derives `patientId` from the authenticated device instead of trusting a
client-supplied `patient_id` query param — tightens the same class of routes' security model, not just a
compile fix; `recordAttempt` now 404s if the quiz question doesn't belong to the authenticated device's
patient). `quiz/history` stays JWT — that one's genuinely dashboard-only. This is a breaking change to
an already-"locked" contract page, called out here per the working agreement in `../CLAUDE.md`.

### Prisma CLI pinned to 6.19.3, not `latest`
`npm install prisma@latest` resolved to `8.0.0-rc.13`, a pre-release with a completely different CLI (no
`migrate dev` — it's built around Prisma's own cloud platform instead). Pinned both `prisma` and
`@prisma/client` to `6.19.3`, the current stable line, so `npx prisma migrate dev` (what the spec and
`db-schema.md` assume) actually exists.

### `assist_app` role granted `CREATEDB`
`prisma migrate dev` needs to create a throwaway shadow database to diff against. Without `CREATEDB` on
the app role it fails with `P3014`. Granted it via `ALTER USER assist_app CREATEDB` — fine for a local
dev-only role; would not grant this to an app role in a shared/production database.

### `enrolled_faces` Phase 0 implementation accepts a raw embedding, not an image
`api-contract.md` originally sketched `POST /api/enrolled-faces` as taking a `reference_image` with the
descriptor computed server-side. That computation is face-api.js/TensorFlow.js work, which is explicitly
Phase 2 scope (facial recognition) — installing that dependency and wiring detection now would be
building ahead of the phase it belongs to. Phase 0's route accepts `reference_embedding` (the descriptor
array) directly from the caller instead, so the table/route shape exists and is testable today; Phase 2
adds the `reference_image` → descriptor computation in front of it without changing the stored shape.

---

## Session 3 — 2026-09-12 — Standing practices for master/worker task dispatch

Not a one-off decision — these are standing rules for how the master (Opus) hands work to worker
sessions (Sonnet) from here on, under the master/slave split in `../CLAUDE.md`. Adopted specifically to
cut down on made-up ("hallucinated") behavior and to make "the task is done" mean something checkable,
after repeated sessions of eyeballing worker output and taking self-reports at face value.

1. **Paste the real spec text into every task, never a paraphrase.** The master quotes the actual
   wording from `api-contract.md` / `db-schema.md` / whichever page applies, not its own summary of it.
   A summary is where small drift enters and gets built on as if it were the rule.
2. **One small task per dispatch, not a whole feature.** "Add this one route" instead of "build the
   login feature." Bigger, vaguer tasks leave gaps a worker fills with guesses.
3. **Every task states a concrete definition of done.** A specific check — "must pass `npm test`",
   "hitting this endpoint must return this exact shape" — not "implement X" left open-ended.
4. **The worker must actually run the code, not just write it and assert it works.** Tests passing, a
   server responding, a screenshot of a rendered page — real output, not the model's own read of its own
   code.
5. **No coverage in the vault is a stop sign, not room to improvise.** If a worker can't point to the
   line in the spec covering what it's about to build, it stops and asks the master instead of inventing
   something plausible.
6. **Point workers at one real existing example to match**, not a written description of the style —
   copying a working pattern is far more reliable than following prose about the pattern.
7. **The master scrutinizes the actual diff itself before accepting a task — not the worker's summary of
   the diff.** This is not optional and not delegated further: a report of work can look right even when
   the underlying change isn't; the master reads the real change before it gets marked accepted or
   written into this log.
8. **Every mistake found, in any task, gets logged here immediately and carried into future task prompts
   that touch the same area.** Enforce this heavily — it's the only mechanism that lets these practices
   actually improve over time instead of the same class of mistake getting quietly re-made by a different
   worker later. Skipping this step defeats the entire point of keeping this log.

### Worker processes run at high thinking effort, independent of model tier
Each worker is spawned as its own `claude -p` process from the master's shell, so its thinking effort can
be set independently of both the master's effort and the model tier — the whole point of the master/slave
split is cheap model + strong output on the worker side, and effort level is the knob that buys the
"strong output" half without switching the worker off Sonnet. Set via the `--effort` flag at launch
(confirmed against Claude Code's own docs, 2026-09-12):

```powershell
claude --model sonnet --effort high -p "task text here"
```

Use the flag at launch, not the in-session `/effort` command — on some models `/effort` doesn't take
effect in non-interactive (`-p`) mode. For dispatching several workers in a row, set
`CLAUDE_CODE_EFFORT_LEVEL=high` once in the shell instead of repeating the flag on every call. Levels
available: `low`, `medium`, `high`, `xhigh`, `max`.
