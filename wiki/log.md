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

---

## Session 6 — 2026-09-13 — Governance settled; tooling updated; a roadmap proposed

**Scope**: no new application code. This session picked up where Session 5's entry stopped —
two master sessions were both mid-flight on the same vault — and spent most of its time closing
real gaps in the working agreement that Session 5's own UI dispatch surfaced, updating the two
shared tools (`D:\agent-office`, `D:\vault-template`) to match, and verifying claims rather than
taking them on report. **If you're picking this up cold: read `../CLAUDE.md` in full before
dispatching anything — the rules below are the current, converged version, not a draft.**

### The governance rules, final form (all in `../CLAUDE.md`, cross-referenced from here rather
than restated in full)

- **Architect builds the plan, laborer lays the bricks — the master always dispatches to a
  worker first, on every task, with no exception on a first attempt, no matter how complex the
  master believes the task to be.** Direct authorship is reachable only after a *demonstrated
  failure*, and "demonstrated failure" is precisely defined: the dispatch itself must have been
  sound (right tool-permission names, the non-interactive-worker clause present, real spec text,
  an executable done-check) or it doesn't count — it's a bad dispatch to fix and re-run, not
  evidence the task needed the master. This took **four passes in one day** to get right; earlier
  versions gated on the master's own confidence that a worker could do something, which is
  exactly the prediction the master has proven bad at (it was confident workers couldn't hold
  visual consistency across seven screens; they held it perfectly). The strongest reason for the
  rule isn't cost, it's that this makes the master's "this one needs me" instinct falsifiable and
  logged instead of self-certifying with no trace.
- **One narrow, checkable exception**, added the same day on the user's direct instruction after
  a real 4-line `Card.jsx` fix made the absolute version look silly: the master may act directly,
  no dispatch, only when *both* hold — the complete diff is known before starting (nothing
  discovered while writing it) **and** it's confined to one file. Deliberately not framed as
  "small changes" — that's the same self-assessment shape that caused the original failure.
- **Dispatch hygiene** (standing practice #10) grew substantially this session, all from real
  waste, not theory: permissions belong in each repo's own `.claude/settings.json`, inherited
  automatically, not hand-typed fresh into every dispatch command (none of the four sibling repos
  have this file yet — see Outstanding below); every tool the worker might actually drive must be
  named (a rule written for `Bash` silently never matches a worker driving `PowerShell`, and the
  failure is invisible — the worker writes correct code, never verifies it, exits 0); prefix
  patterns only; the real security boundary is directory scope and reversibility, not which
  binary is nominally allowed, since granting any language runtime is already arbitrary code
  execution — allow the build toolchain and read-only git, withhold `git push`/`commit`,
  deploys, `rm -rf`, credentials; a worker that can't run its own verification is worse than one
  never dispatched; probe first with a throwaway worker before a real batch; environment facts a
  worker pays context to discover go into the next prompt and the repo's own `CLAUDE.md` the same
  session; seed realistic demo data before building screens against it, not after (a
  grouping-logic bug only appeared once real data density existed — no review against empty
  state would have caught it). **A live misdiagnosis was corrected this session**: none of
  tonight's stuck workers were a Windows/OS permission problem — every block was Claude Code's
  own tool-approval layer, which sits above the OS entirely; granting workers administrator
  rights would fix nothing and add real risk. Don't reach for it if the same symptom recurs.
- **Standing practice #9**: for consistency across several files, one worker builds the canonical
  layer plus one reference screen from the master's literal spec values; everyone else dispatches
  against that real file. This is what actually produced uniformity in Session 5's UI work, not
  who authored the canonical layer.
- **Standing practice #4** now has a UI rider (a green build proves nothing about how something
  looks) with a second-order lesson bolted onto practice #7: **looking is not measuring**. The
  same session that added "render it and look" broke that rule's own reliability within hours — a
  real spacing bug (`pl-1` losing to `p-5` in Tailwind's cascade, collapsing a 24px gutter to 4px)
  survived several of the master's own screenshot reviews and was caught by the user, not by any
  agent. In the other direction, the master separately invented a defect that never existed by
  misreading a "0" as a "6" on a downscaled screenshot, dispatched a worker to "fix" it, and wrote
  the result up as a success — retracted, see `decisions-log.md`. Text claims need `textContent`
  or source, never an image read; spacing claims worth acting on need computed styles.
- **Every sibling repo gets its own `CLAUDE.md`, written by a worker, never the master** —
  mechanics only, no restated rationale. Both `family-dashboard` (120 lines) and `companion-app`
  (81 lines) have one already, independently spot-checked against real source.
- **Two master sessions are being run at once, deliberately, for this iteration phase only** —
  not a standing recommendation. Rationale: the two sessions catching each other's mistakes in
  real time (a rule violation on one side, a factual error about log data on the other) is
  genuinely how several of tonight's fixes were found. Cost, named honestly: doubled spend on the
  expensive model tier for as long as it runs, plus a real coordination tax (every correction
  needs a round-trip through a peer message). Standing safeguard either way: **call `ListAgents`
  before writing to `decisions-log.md` or `log.md`**; if another vault-scoped session is live,
  agree who writes before touching the file.

### Tools

- **`D:\agent-office`** (the live activity viewer) rewritten this session to show *every*
  concurrently-active session as its own desk, not just the single most-recently-modified one per
  repo — verified against real data mid-session (2 masters + 3 companion-app workers + 5
  family-dashboard workers, all simultaneously, correctly laid out). Cost/token tracking, built
  and then fully removed earlier the same day after the user correctly identified it as
  misleading on a Pro/Max subscription (not your bill, and not even a fair token comparison
  against the account's own usage meter) — what's left is plain, unambiguous tool-call/message/
  escalation counts, click-to-expand full detail (including real tool output, not a placeholder),
  and green "You" color-coding for genuine human input vs. agent activity. Run:
  `node D:\agent-office\watch.js "D:\AssistVault"`, then open `http://localhost:4545`.
- **`D:\vault-template`** kept in sync with every rule change above — a fresh project started
  from it today gets the full, hard-won ruleset for free instead of rediscovering any of it.

### App status

- **`family-dashboard`**: UI overhaul complete and verified (Session 5), governance/tooling only
  since. Nothing new to build here right now.
- **`companion-app`**: UI overhaul complete and independently re-verified — a from-scratch
  `clean assembleDebug` (not the worker's incremental build), 44s, BUILD SUCCESSFUL, 11.58MB APK,
  lint clean, plus a real audit (zero raw Compose components across all six screens) and explicit
  preservation checks on two fragile pieces of prior behavior. Both repos' UI work is genuinely
  done, not just claimed.
- **`backend-server`**, **`watch-app`**: untouched this session and Session 5. Still carrying
  everything listed as outstanding in Sessions 1/2 below.

### Outstanding / next session

- **No `.claude/settings.json` exists yet in any of the four sibling repos.** The dispatch-hygiene
  rule above says permissions should live there instead of being hand-typed per dispatch — this
  hasn't been built yet, on purpose, since it's real infrastructure across four repos and needs
  the user's go-ahead before either master session touches it.
- **A proposed improvement roadmap is sitting unstarted** (see the dedicated section below) — the
  user hasn't picked a starting point yet.
- Two master sessions are still concurrently live as of this entry — resolve or continue
  deliberately, don't let it become the accidental default.
- Everything still outstanding from Sessions 1/2/5: real `GEMINI_API_KEY`/`CLOUDINARY_*` values,
  ESP32 firmware integration, the watch app (Phase 3), the Phase 1 agentic decision layer, an
  actual emulator walkthrough of `companion-app` (compile/lint/audit only so far, never run on a
  device), and the pre-existing uncommitted `MonitoringService.kt` fix in `companion-app` from
  before Session 5 — don't lose it in any future `git` operation on that repo.

### Proposed improvement roadmap (not started — a plan, not a commitment)

Came out of a separate research pass evaluating whether this master/worker pattern could become
something worth open-sourcing. Verdict: don't compete on "run more agents" (that space is
crowded — OpenHands, Stoneforge, and others are already there); the more interesting, less
crowded problem this project already organically has is **cross-repository architectural
coherence** — keeping several agents working on one system without losing the plot. Tiered by
what's actually buildable:

**Tier 1 — cheap, no new dependencies, buildable immediately:**
1. A machine-readable system graph (which repo owns which API route, which repos consume it) —
   turns `api-contract.md`'s prose into something a script can read.
2. Automatic contract-impact detection — a script diffing `api-contract.md` against its previous
   version and listing which repos' consumer lists reference a changed route. No external tool
   needed for this version.
3. Evidence-based completion as an actual checklist template, not prose — turns practices
   #4/#7/#10 into a form a task isn't marked done without.
4. Failure classification as a documented decision tree (trivial → retry, missing context →
   enrich prompt, wrong approach → replan, conflict → escalate, ambiguous → ask the user).
5. An explicit, named uncertainty/escalation state, formalizing behavior that's currently
   scattered implicit language across several rules.

**Tier 2 — a bit more real engineering:**
6. Lightweight task dependencies ("don't dispatch the Android client-update task until the API
   route it needs actually exists") — practice #9 already does a two-level version of this
   informally; this would generalize it slightly, not build a full scheduler.

**Tier 3 — verified real, but a genuine commitment, needs explicit sign-off first:**
7. **Graphify** (`github.com/Graphify-Labs/graphify`) — independently verified this session, not
   taken from an unverified paste: 117k stars, Apache-2.0/MIT, actively developed, real MCP tools
   (`query_graph`, `get_node`, `get_neighbors`, `shortest_path`, `get_pr_impact`). Note for
   whoever installs it: the actual PyPI package is `graphifyy` (double-y). Would sit alongside
   Tier 1's system graph as a *code-level* impact tool (what the implementation actually
   connects to) complementing the *system-level* one (what the architecture says should connect)
   — optional, never a hard dependency, and specifically the same "Graphify" this vault's own
   Session 1 already evaluated once and deferred with the condition "revisit once real codebases
   exist" — that condition is now met.
8. Model-agnostic master (Claude/Codex/OpenCode abstraction) — only relevant if actually
   open-sourcing for people not on Claude; deprioritized until that decision is made.

**Deliberately not planned**: worktree isolation (no worker collisions in the same repo yet to
solve) and hibernating/reviving workers (doesn't apply — `claude -p` workers are already
one-shot and disposable).

Recommended starting point, not yet confirmed by the user: Tier 1 items #1 and #2 together, since
#2 depends on #1's data and is the single most concretely useful thing on the list given this
project already has a locked `api-contract.md` and three consumer repos.


---

## Session 10 — 2026-10-02

**Done**
- `family-dashboard`: full redesign (see `decisions-log.md` Session 10) + new login. Verified by
  screenshotting every page against live data; demo login lands on Today with no console errors.
  Dashboard API host now follows the page's host, so `http://<pc-ip>:5173` works from a phone.
- `companion-app`: UI rebuilt twice; look still rejected (pending user inspiration). New: Get home
  screen, voice quiz, tappable Ask suggestions, auto re-setup on revoked device key. Installed and
  registered on the user's Galaxy S24+.
- `backend-server`: `GET /api/devices/me`.
- DB re-seeded. Demo patient id: `863e9223-56d5-4a3d-b9dd-4cc98937b73c`.
- PC LAN IP is currently `192.168.1.13` (it changes between networks — check `ipconfig`).

**Pending features (spec vs. what's built)**
1. Phase 1 two-step agentic decision layer (`agent_decision` is only faked in seed data).
2. Short-term memory context (last N entries fed into prompts).
3. Notifications: FCM / Resend email / Twilio SMS — none built.
4. Face recognition on camera frames + "this is your son" announcement (only enrolment exists).
5. Routine learning / deviation flagging.
6. Medication confirmation via camera (phone reminders are local-only).
7. Family voice notes (record on dashboard, play on phone).
8. "Lost?" home-screen widget (the in-app Get home screen now exists; the widget doesn't).
9. Wear OS watch app.
10. Scheduled jobs: weekly photo cleanup (UI already promises 7-day expiry — make it true) and daily
    quiz generation.
11. "Next: lunch at 1pm" orientation line (no schedule data model).
12. Patient creation / caregiver invite flow (no API creates patients or users).
13. NFC sticker (physical, no code).
14. Companion app visual design (awaiting user inspiration).

## Session 11 — 2026-10-03: Linen design + dual-mode app

- Dashboard: "Linen" palette (warm tinted neutrals, earthy status colours), painted time-of-day
  backdrop with frosted-glass cards, floating light header, day-picker pill, "later today"
  sky placeholders, alert said once (incidents inside the Today hero), fully responsive
  (bottom dock nav below 1024px).
- Companion app rebuilt as two sides behind a welcome screen ("Who's using this phone?"):
  - Patient side: same Linen language, painted sky (dark dusk at night), big clock + orientation
    sentence, one due item, "Ask me anything", three tiles.
  - Family side: caregiver sign-in (same account as the web), and every dashboard feature —
    Today, Log, Places (osmdroid editor), People (face-api.js in a WebView, fingerprints match
    the website's), Memory, Settings & devices. `family/Story.kt` ports `story.js`.
- Pending list unchanged except item 14 (app visual design) is done; the app is installed on
  the S24+ (currently left in family mode after testing — switch via the welcome screen).

**Session 11 addendum (later the same day)**
- Family side of the app was rethought as a phone app, not a website port: hub-and-spoke instead of five tabs.
  `FamilyHome` answers "is she okay?" at a glance (status card + one action, camera line, one-sentence day,
  four doors: Places / People / Memory / Devices); Alerts, Day, Places, People, Memory, Settings are one tap
  deeper; moment details open in a bottom sheet. Family side uses a smaller type scale (`FamilyTypography`)
  and slimmer buttons (`LocalCompact`); the patient side keeps large type on purpose.
- Fixed an ANR: osmdroid `zoomToBoundingBox` loops forever on a map with no size yet (see `companion-app/CLAUDE.md`).
- Installed on the S24+ (family mode; Margaret's patient registration is kept).
- Research (no code): V-JEPA 2 judged a poor fit (clip-based, no text output, no data yet; only revisit for
  routine learning once the ESP32 sends bursts). Jev (TypeSafe, text-only closed classifier, launched
  2026-09-15) judged a good fit for pending items 1/5-ish: alert probability, activity category at ingest,
  quiz-answer judging, recall intent routing. Open: API key + privacy sign-off before sending activity text.
- Gemini cost/limits: one `GEMINI_MODEL` env var drives all four calls (default `gemini-2.5-flash`). Frame call
  does not set a thinking budget (thinking tokens bill as output). With a photo every 5-10 s the free tier is
  the risk (per-minute and per-day caps), not price. Proposed, not built: thinking budget 0 on frames,
  server-side throttle (<= 1 analysis / 10 s), save-without-description on 429.
- Still open: nothing is pushed to GitHub (no remotes); ESP32 not reaching the server; Home zone is ~15 km from
  the phone; add-a-person, place editing and "I've checked" not yet exercised on the phone.

## Pending features — re-categorised 2026-10-17 (NFC parked)

**Alerts & safety**
- Notifications to family (FCM push / Resend email / Twilio SMS) — none built.
- Agentic decision layer ("does the family need to know?") — rules only; Jev candidate (needs key + privacy OK).
- Medication confirmation via camera.
- "Lost?" home-screen widget on the patient phone.
- PARKED: emergency NFC tag — software done (Session 11); hardware (NTAG216 stickers), contacts, public URL and writing the tag deferred.

**AI & understanding**
- Short-term memory context (last N entries into prompts).
- Face recognition on camera frames + "this is your son".
- Routine learning / deviation flagging (V-JEPA 2 only worth revisiting once the camera sends clips).
- Gemini cost/limits for live demo: Flash-Lite or paid key, thinking budget 0 on frames, 1 analysis / 10 s throttle, retry on 429.

**Patient experience**
- Family voice notes (record on dashboard, play on phone).
- "Next: lunch at 1pm" orientation line (needs a schedule data model).
- Wear OS watch app.

**Accounts & platform**
- Patient creation / caregiver invite flow (all accounts are seeded today).
- Scheduled jobs: 7-day photo cleanup (UI already promises it) + daily quiz generation → Vercel Cron.
- Deploy backend to Vercel: hosted Postgres + Prisma pooling, bundle public/tag.html, PUBLIC_BASE_URL.
- Push the four repos to GitHub.

**Hardware & setup (user)**
- Reconnect ESP32 camera to WiFi; set its /capture URL in the app.
- Move the Home zone to the real location.
- Install latest app build; exercise add-person, place editing, "I've checked" on the phone.
