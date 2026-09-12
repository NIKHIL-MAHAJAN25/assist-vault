# AssistVault — system-level guide

This is the wiki/vault for the **AI-Assisted Safety, Navigation & Cognitive Engagement System**
(working name: **Assist**). It is the spec-of-record for a multi-repo project. Read the wiki pages
under `wiki/` before making architectural decisions in any sibling repo.

## What this project is

A passive, IoT-vision-based safety and cognitive-engagement system for people with Alzheimer's/memory
impairment living semi-independently, with a family-facing dashboard. See [wiki/index.md](wiki/index.md)
for the full problem statement and design lens.

## Wiki map

| Page | Purpose |
|---|---|
| [wiki/index.md](wiki/index.md) | Problem statement, persona, research grounding, feature scope, phasing |
| [wiki/architecture.md](wiki/architecture.md) | System diagram, component responsibilities, repo layout |
| [wiki/tech-stack.md](wiki/tech-stack.md) | Stack choices and *why*, infra/credential status |
| [wiki/db-schema.md](wiki/db-schema.md) | Postgres/Prisma schema, normalization rules for extending it |
| [wiki/api-contract.md](wiki/api-contract.md) | **The critical shared page** — every route, every consumer |
| [wiki/ai-prompting-patterns.md](wiki/ai-prompting-patterns.md) | All AI techniques (structured output, agentic decision layer, RAG, etc.) |
| [wiki/decisions-log.md](wiki/decisions-log.md) | Judgment calls made during the build, with rationale |
| [wiki/log.md](wiki/log.md) | Session-by-session handoff notes |

## Repo layout (siblings under `D:\`)

- `D:\AssistVault` — this vault (wiki + system-level CLAUDE.md). No application code.
- `D:\backend-server` — Node/Express + Prisma + Postgres backend. **Built first (Phase 0).**
- `D:\companion-app` — Kotlin/Android companion app. Later session.
- `D:\family-dashboard` — React dashboard. Later session.
- `D:\watch-app` — Kotlin/Wear OS app. Later session.
- ESP32-CAM firmware already exists and works; not tracked as a repo here yet.

## Working agreement for AI agents

- **`api-contract.md` is locked-contract territory.** Once a route is in that page, changing its shape
  is a breaking change for every consumer repo — update the page and flag the affected repos in the
  same session.
- **Schema changes go through Prisma migrations only** (`npx prisma migrate dev` in `backend-server`),
  never hand-edited against the live database. Follow the normalization rules in `db-schema.md`.
- **Master/slave subagent split — active now, not deferred.** The master (Opus, holds vault access) is
  the only writer to `decisions-log.md` and `log.md`, and does **not** write application code in any
  sibling repo itself, no matter how small the change looks. Work is dispatched to workers as genuinely
  separate `claude` CLI processes — via Bash, `claude --model sonnet --effort high -p "<task>"` run with
  cwd set to the target repo — not as the master editing those repos' files directly, and not as an
  in-process subagent. Each worker gets the relevant wiki slice pasted directly into its task prompt; it
  has no vault directory access of its own. "This is quick, I'll just do it myself" is not an exception to
  this rule — dispatch it as a one-line worker task instead.
  (This replaces an earlier version of this line that called the split "not useful yet" for Phase 0 —
  true when this project was backend-only and sequential, false since Session 2 added `companion-app` and
  `family-dashboard` as separate repos, and never corrected until a Session 4 review caught the master
  doing app-repo work itself and traced it back to this stale line.)
- **Architect builds the plan, laborer lays the bricks — this is the user's own framing for where the
  master/worker line sits, decided explicitly by the user on 2026-09-12, and it supersedes the stricter
  "master never touches a sibling-repo file, no exceptions" version written earlier the same night.** The
  master (architect) may author plans, specs, and literal-value artifacts — requirements, a path to a real
  reference file, and specifically a plain, non-executable data file of literal values (e.g. a token/color
  values file with no imports, no computed values, no logic) — for token-saving reasons: writing values
  once as the master is cheaper than having a worker transcribe a wall of numbers it didn't choose. The
  master never authors anything that is itself part of the build or render path — no `.jsx`/`.tsx`/`.kt`,
  no `.css` a component actually imports, no page, no wiring of one thing to another. That "lay the bricks"
  work is always a worker's job, dispatched with the master's literal values as its input, never
  transcribed loosely from memory. When only one canonical example exists yet, the master dispatches
  **one** worker to build it plus one reference screen, then dispatches the rest against that real file.
  Once work is dispatched, the master verifies what came back — opens the dev server, looks at the actual
  rendered result, reads the real diff — it doesn't just read a report of it.
  (This line replaced an earlier one the same night that made the rule absolute with no exceptions, after
  a master had used "this is foundational, not an implementation task" to justify writing an entire
  design-system layer itself. Two sessions independently argued back to the absolute version and
  recommended keeping it; the user weighed both arguments and explicitly chose to keep discretion for the
  master instead, using the architect/laborer split as the standing test rather than a fixed file-type
  boundary. If the master is ever unsure which side of that line something is on, it stops and asks the
  user — this is exactly the class of judgment call that broke once already, so treat "I'm confident this
  counts as architecture" with real suspicion, especially under any pressure to finish faster.)
- **Session handoff discipline**: update `log.md` and `decisions-log.md` at the end of every real
  session, starting with session one.
- Tools deliberately evaluated and not adopted for this project (Graphiti, Graphify, Munder Difflin) are
  recorded in `decisions-log.md` — don't re-propose them without re-reading why they were deferred.
- **Standing dispatch practices** for how the master hands work to workers (real spec text not
  paraphrase, small tasks, concrete done-checks, workers must actually run the code, no vault coverage
  is a stop sign not a guess, match a real example, master scrutinizes the real diff, log every mistake
  found) are recorded in `decisions-log.md` Session 3 — apply them to every task dispatch, not just that
  session's. A live view of master/worker activity is available via the reusable viewer at
  `D:\agent-office` (`node D:\agent-office\watch.js "D:\AssistVault"`, then open `http://localhost:4545`)
  — it self-configures off this vault's own `additionalDirectories`, nothing to edit per project.
- **Self-improvement of these practices.** If the master notices a flaw or a missing case in this file,
  in `decisions-log.md`'s standing practices, or in the working agreement itself — as opposed to a
  decision about the app being built — it edits the file directly in the same session rather than
  waiting to be asked, and notes briefly why. This applies only to how the agents work, never silently to
  what the app does; app-level judgment calls still go through the normal decision-log process above.
- **Every sibling repo gets its own `CLAUDE.md`, written by a worker, never the master.** As the last step
  of any canonical-layer task, the worker writes or updates that repo's own `CLAUDE.md` with mechanics
  only — stack, build command, file layout, component signatures, what not to touch. It never restates a
  decision or the rationale behind it; that belongs in this vault's `decisions-log.md`, and duplicating it
  in a repo file is exactly how the two copies drift apart. Approved by the user 2026-09-12.
- **Before either of two concurrently-running master sessions writes to `decisions-log.md` or `log.md`,
  call `ListAgents` first.** If another vault-scoped session is live, message it and agree who writes
  before touching the file. Cheap (one tool call) and it has an actual track record — this is the specific
  thing that prevented a real collision the night it was adopted, not a theoretical safeguard. Running two
  master sessions on this vault at once is itself a deliberate, temporary choice by the user while this
  process is still being iterated on (2026-09-12) — not a standing recommendation — precisely so the two
  sessions can compare notes and surface disagreements like this one to the user directly.
