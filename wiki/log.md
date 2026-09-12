# Session log

Session-by-session handoff notes. Only the master session writes here. Newest at the bottom.

---

## Session 1 — 2026-09-05

**Scope**: Phase 0 plumbing per [index.md](index.md) — backend structure only. No companion app,
dashboard, watch app, or ESP32 integration work this session.

**Done**:
- Split the original handoff doc into this wiki (`index.md`, `architecture.md`, `tech-stack.md`,
  `db-schema.md`, `api-contract.md`, `ai-prompting-patterns.md`, `decisions-log.md`, this file) plus a
  system-level `CLAUDE.md`.
- `D:\AssistVault` and `D:\backend-server` created as git repos.
- PostgreSQL 17 installed locally (see [decisions-log.md](decisions-log.md) for why winget instead of
  Chocolatey). Database `assistdb`, app role `assist_app` created.
- `backend-server` scaffolded: Express app, layer-based `routes/`/`controllers/`/`services/`/`models/`
  structure, Prisma schema matching [db-schema.md](db-schema.md), initial migration run against
  `assistdb`, JWT user auth + device API key auth implemented, all Phase 0 routes from
  [api-contract.md](api-contract.md) stubbed/implemented against a single Gemini call (no agentic
  decision layer, no memory context yet — Phase 1).

**Verified working** (smoke-tested against the real local Postgres instance): login, device
registration, device-API-key auth, geofence zone creation, geofence-check writing a `log_entries` row,
JWT-protected log-entries listing. `/api/frames` was exercised too — it correctly reaches the Gemini
call and fails only on the placeholder API key (`API key not valid`), confirming the whole
route → middleware → service → DB chain is wired correctly end to end.

**Outstanding / next session**:
- Real credentials for `GEMINI_API_KEY`, `CLOUDINARY_*`, and (when notification features are built)
  Twilio/Resend/FCM are not yet in `backend-server/.env` — placeholders only. Fill in before testing the
  `/api/frames` pipeline end-to-end.
- `D:\AssistVault\.claude\settings.json` (the `additionalDirectories` link to sibling repos described in
  `../CLAUDE.md`) could not be created this session — writing Claude Code's own settings file was blocked
  by the tool's safety classifier. Create it by hand next session if the master/slave subagent pattern is
  needed (not yet — Phase 0 was single-repo, sequential work).
- Companion app, React dashboard, watch app, ESP32 firmware integration — all later sessions per
  [index.md](index.md) phasing.
- Phase 1 work (two-step agentic decision layer, short-term memory context, geofencing logic beyond the
  raw endpoint, real notification delivery) not started.

---

## Session 2 — 2026-09-07

**Scope**: `companion-app` (Kotlin/Android) and `family-dashboard` (React) — full Phase 0-2 feature set
(everything short of Phase 3: wearable app, "Lost?" widget, NFC, voice-note playback), per user request
to build both using the master/slave subagent pattern from `../CLAUDE.md` §13, now that
`api-contract.md` is locked. Plan approved before coding; see `decisions-log.md` for the stack decisions
and the orchestration adaptation (this session's `Agent` tool, not separate CLI processes per repo).

**Done**:
- Fixed a real bug in the "locked" `api-contract.md`: `quiz/today`/`quiz/attempts` were JWT-only, but
  there's no patient login flow anywhere in the spec — the companion app is device-only. Switched both
  to device API key auth in `backend-server`, verified against the live DB (correct auth accepted,
  missing/wrong auth rejected, `quiz/history` unaffected).
- `D:\AssistVault\.claude\settings.json` created successfully this session (blocked by the safety
  classifier in session 1 — worked this time with no changed circumstances noticed).
- Dispatched two parallel builder subagents. Both finished clean, with real (not just claimed)
  verification:
  - **`family-dashboard`**: all 7 screens (login w/ patient-role block, log timeline, alerts, geofence
    map editor, face enrollment via client-side face-api.js, quiz history, device registration).
    `npm run build` succeeds; live-smoke-tested against the real running backend (login, real data
    rendering, an actual device-registration round-trip), zero console errors.
  - **`companion-app`**: onboarding/registration, static orientation home, recall (STT → clarify →
    recall query → TTS), daily quiz, local medication reminders, settings. `./gradlew clean
    assembleDebug` → BUILD SUCCESSFUL (APK produced), lint clean.
  - Both spot-checked by the master afterward: clean git history, correct `.gitignore` (no
    `node_modules`/`build`/`dist` committed).

**Infra note**: hit a real Postgres outage mid-session — Windows **Smart App Control** (surfaced via a
`pg_ctl` error as "blocked by your organization's Device Guard policy") was blocking `postgres.exe`
outright. Neither restarting nor even stopping the service was possible from this session's
non-elevated shell. User disabled Smart App Control themselves (a one-way toggle — noted the tradeoff
before they did it) and restarted the service from an elevated PowerShell; `assistdb`'s data survived
untouched. Not a code issue, but worth knowing if Postgres mysteriously dies again on this machine.

**Known limitations, documented rather than silently built around** (see `decisions-log.md` for full
detail): companion app's face-recognition-announcement toggle has no real trigger yet (backend doesn't
identify *which* face, Phase 2); medication reminders don't correlate with camera confirmation yet
(same reason); dashboard's Cloudinary images fall back gracefully since no real cloud name is
configured yet; quiz/recall quality on both will stay thin until the ESP32 is actually wired and
producing real log entries (build-order step 5, still not started).

**Outstanding / next session**:
- Real `GEMINI_API_KEY`/`CLOUDINARY_*` values still not filled in (from session 1).
- ESP32 firmware integration (build-order step 5), watch app (Phase 3), Phase 1 agentic decision layer
  — none started.
- No full emulator UI walkthrough was done on `companion-app` (compile + lint only) — worth a manual
  pass in Android Studio if UI-level bugs are a concern before relying on it.

---

## Session 5 — 2026-09-12 (into 09-13)

**Scope**: UI/UX overhaul only. One shared visual language across `family-dashboard` and
`companion-app`, derived from the user's existing `SellerApp` Android theme. No backend work, no
contract changes, no new features. See [decisions-log.md](decisions-log.md) Session 5 for the
palette provenance, the typeface reasoning, standing practice #9, and the mistakes found.

**Done**:
- **`family-dashboard`** — all 7 screens rebuilt on the "Assist Sage" system (login, timeline,
  alerts, safe zones + map editor, face enrolment, daily quiz, device registration), plus the
  shell (desktop sidebar / mobile rounded-top bottom bar) and the patient-role block screen.
  Design layer: `tailwind.config.js` tokens, `src/index.css`, `src/ui/*` primitives,
  `src/utils/datetime.js`.
- **`companion-app`** — theme layer split into `ui/theme/{Color,Type,Shape,Theme}.kt`, five
  shared components added under `ui/common/` (`AssistScaffold`, `AssistCard`, `StatusPill`,
  `AssistTextField`, plus a rewritten `BigButton`), Atkinson Hyperlegible wired in as four
  static TTFs in `res/font/`, and all 6 screens restyled.
- **Per-repo `CLAUDE.md` created in both repos** (user decision A2, Session 6 of the decisions
  log) — worker-authored, mechanics only, no rationale. `family-dashboard` 120 lines,
  `companion-app` 81. The companion one carries this machine's build traps so no future worker
  rediscovers them.

**Verified — by the master directly, not from worker reports**:
- `family-dashboard`: `npm run build` exits 0; every screen opened in a browser against the
  running backend and the real Postgres data; times checked against raw ISO values
  (`2026-09-07T05:13:14Z` → "10:43 AM" IST) after a formatting bug; console checked.
- `companion-app`: `gradlew clean assembleDebug` from scratch (not incremental) → BUILD
  SUCCESSFUL, 37 tasks executed, `app-debug.apk` 11.58 MB. `lintDebug` clean — 26 warnings, all
  pre-existing. Grep audits: no bare `Card(`/`OutlinedTextField(`/`TextField(`/`Button(` in any
  screen, all 6 screens open with `AssistScaffold`, `Color(0x` nowhere outside `theme/Color.kt`.
- Preservation spot-checks: the Settings announce-toggle kept both its "Not available yet"
  string and the comment explaining the backend can't identify *which* face; `RecallScreen`'s
  `SpeechToText` / `DisposableEffect` / permission-launcher chain unchanged in behaviour.

**Worker dispatch record** (7 workers, all as separate `claude -p` processes per the working
agreement): FD-1 timeline+cards+alerts · FD-2 zones+map+faces · FD-3 quiz+devices · FD-4
six-defect fix pass · FD-5 dashboard CLAUDE.md · CA-1/CA-1b companion theme + HomeScreen +
fonts + build · CA-2 five screens + two Compose defects + companion CLAUDE.md. The first batch
of four was silently gated by a wrong `--allowedTools` pattern and had to be re-verified by the
master; see decisions-log Session 5, mistake 2.

**Governance changes this session** (user decisions, recorded in `../CLAUDE.md` and
decisions-log Session 6): two masters kept deliberately for this iteration phase;
`ListAgents`-before-vault-write adopted as a standing practice; per-repo `CLAUDE.md` approved;
and the architect/laborer rule replaced the absolute "master never writes sibling-repo code"
line — the trigger is **accuracy under genuine complexity, never token-saving**. Both master
sessions had argued for keeping it absolute; the user heard that and chose the narrower
exception anyway.

**Outstanding / next session**:
- **Nothing is committed.** Both repos are entirely working-tree changes, deliberately left for
  the user to review. `companion-app` also still carries a *pre-existing* uncommitted fix to
  `service/MonitoringService.kt` from an earlier session (foreground-service-type handling so
  frame capture survives a missing location permission) — untouched this session, don't lose it.
- No emulator walkthrough of `companion-app` was done — compile, lint and code audit only. The
  dashboard was exercised in a real browser; the Android app was not run on a device.
- `RecallScreen`'s mic button has no leading icon: `Icons.Default.Mic` needs
  `material-icons-extended`, which is not a dependency. Left text-only rather than adding a
  dependency mid-restyle.
- Still carried from earlier sessions: real `GEMINI_API_KEY` / `CLOUDINARY_*` values,
  ESP32 firmware integration (build-order step 5), the watch app (Phase 3), and the Phase 1
  agentic decision layer. None touched.
- Environment, for whoever builds `companion-app` next: this machine's `JAVA_HOME` points at
  JDK 25, which Gradle 8.7 rejects — build with `JAVA_HOME=D:\Java\JDK21`. Now documented in
  that repo's own `CLAUDE.md`.
