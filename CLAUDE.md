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
- **Architect builds the plan, laborer lays the bricks — the user's own framing, decided explicitly on
  2026-09-12, refined once more the same day after a first pass got the justification wrong.** The test is
  **accuracy under genuine complexity, never token-saving.** The master may do a piece of implementation
  work itself, instead of dispatching it, only when *both* hold: (1) the task is genuinely complex — not
  merely larger, but the specific kind where a worker plausibly gets it wrong because of real judgment or
  subtlety involved — and (2) the master isn't confident a worker would execute it correctly. When both
  hold, spend the extra cost of doing it directly rather than risk a wrong result from delegating something
  too hard; that is the entire justification. It is never "this is cheaper if I just do it," and it is never
  a blanket allowance for spec-shaped or plan-shaped work in general — most tasks, including most spec
  work, are routine enough to delegate, and delegating is still the default there because token-saving *is*
  the point for ordinary work. This is a narrow exception for hard cases, not a standing category.
  The master has to actually be able to tell "this is routine, delegate it" apart from "this is genuinely
  complex, doing it myself is the accurate choice" — and given this is exactly the class of judgment call
  that broke once already that same night (a master used "this is foundational, not an implementation
  task" to justify writing an entire design-system layer itself), when genuinely unsure which side
  something falls on, the master stops and asks the user rather than trusting its own confidence. Treat "I'm
  sure this one counts" with real suspicion, especially under any pressure to finish faster. Two sessions
  independently argued for making the rule absolute instead, for real reasons (a shared spec file is only
  ever authoritative for whichever repos can consume it; transcription overhead turned out to be small and
  checkable in practice) — the user heard that and chose this narrower exception over the absolute version
  anyway. Once work is dispatched, the master still verifies what came back itself — opens the dev server,
  looks at the actual rendered result, reads the real diff — rather than trusting a report of it.
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
