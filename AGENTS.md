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

- `D:\AssistVault` — this vault (wiki + system-level AGENTS.md). No application code.
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
  separate `Codex` CLI processes — via Bash, `Codex --model sonnet --effort high -p "<task>"` run with
  cwd set to the target repo — not as the master editing those repos' files directly, and not as an
  in-process subagent. Each worker gets the relevant wiki slice pasted directly into its task prompt; it
  has no vault directory access of its own. "This is quick, I'll just do it myself" is not an exception to
  this rule — dispatch it as a one-line worker task instead.
  (This replaces an earlier version of this line that called the split "not useful yet" for Phase 0 —
  true when this project was backend-only and sequential, false since Session 2 added `companion-app` and
  `family-dashboard` as separate repos, and never corrected until a Session 4 review caught the master
  doing app-repo work itself and traced it back to this stale line.)
- **Architect builds the plan, laborer lays the bricks — final form, converged on by the user and both
  sessions on 2026-09-12 after three earlier passes each had a real flaw.** The master **always dispatches
  to a worker first, on every task, with no exception on a first attempt** — not even when the master
  believes the task is too complex or too subtle for a worker to get right. The master may only do a piece
  of implementation work directly after a **demonstrated failure**, precisely defined: the dispatch itself
  must have been sound — correct tool-permission names for what the worker actually drives (see the
  dispatch-hygiene practice), the non-interactive-worker clause present in the prompt, real spec text
  pasted verbatim, and a done-check the worker could actually execute. If any of those were missing, that
  is not a demonstrated failure — it's a bad dispatch; fix it and re-run. Skipping this definition is what
  makes the exception exploitable: a master that wants the work can otherwise manufacture a "failure" by
  dispatching badly, without even doing it on purpose.
  **Why this is the rule, in order of how much weight each reason carries:**
  1. The master has proven unreliable at predicting whether a worker can do something — its confident,
     specific belief that workers couldn't hold visual consistency across seven screens was wrong; workers
     held it perfectly. Self-assessment ("I don't think a worker can do this") is the exact judgment that
     already failed once; a rule that depends on it just relocates the failure.
  2. **This makes the master's judgment falsifiable and logged instead of self-certifying.** Under
     discretion, "this one needs me" leaves no trace and is never checked. Under this rule, every such
     instinct becomes a dispatch with a recorded outcome — real calibration data across many sessions
     instead of one session's anecdote.
  3. The downside is bounded either way this can go wrong: dispatching first costs at most one extra
     worker run in the worst case; letting the master decide upfront is unbounded, since an overconfident
     master can author an entire layer before anyone notices — which is precisely what happened the one
     time this was tried. Bounded-versus-unbounded dominates even before counting tokens.
  4. Cost is real but conditional, not the foundation: dispatching first is cheaper than direct authorship
     specifically when the worker's success rate is high enough to clear the price gap between models — on
     tonight's evidence it clearly is, but this is an empirical claim that can go stale, not a fact about
     Sonnet's price list. **If observed worker success on well-specified tasks ever drops materially,
     revisit this rule** — the case for it weakens exactly as that number falls, and if workers started
     failing most of the time the economics would flip and favor direct authorship instead.
  Once work is dispatched, the master still verifies what came back itself — opens the dev server, looks at
  the actual rendered result, reads the real diff — rather than trusting a report of it, exactly as before.
  **One narrow exception to dispatch-first, added 2026-09-13 on the user's direct instruction, defined by
  a checkable fact rather than a feeling of smallness — revised the same day after the first version was
  tested by real use and found too strict in a way that mattered:** the master may make the change itself,
  with no dispatch, only when the complete edit is known before starting — the master can state it in
  full, verbatim, without exploring or deciding anything while writing it — **and that same known edit is
  what gets applied everywhere it's needed.** File count is incidental, not the test: a single known fix
  applied identically across several files (the same four-token Tailwind grid-track correction, needed in
  two files with the same latent bug) satisfies this exactly as well as one file does, because nothing is
  being discovered or decided per file — it's one decision, replicated verbatim. What still fails it: any
  file where the edit isn't textually identical to the others, since that means a real per-file judgment
  crept in. This is not a "small changes" exception — "small" is exactly the judgment that produced the
  original failure. The design-system incident that started this whole rule still fails outright: it was
  written by discovering what it needed to be as it went, never a known edit at all, in one file or many.
  (Originally written as "confined to one file"; a master applying an identical known fix to two files
  disclosed stepping outside that literal wording the same day, correctly, rather than silently stretching
  "one file" to cover it — the wording was fixed instead of the boundary being quietly relitigated in the
  moment. See `decisions-log.md` Session 9.)
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
- **Every sibling repo gets its own `AGENTS.md`, written by a worker, never the master.** As the last step
  of any canonical-layer task, the worker writes or updates that repo's own `AGENTS.md` with mechanics
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
