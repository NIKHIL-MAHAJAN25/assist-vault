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
- **For UI/visual work specifically, the master's job stops at spec, dispatch, and verification — never
  authorship.** This is the exact case that broke the rule above the same night it was written: a master
  reasoned that building the whole design-system layer itself first was "foundational, so workers have
  something to match," not "an implementation task," and wrote it anyway. There is no such carve-out.
  Concretely: the master writes literal design-token values, requirements, and (once one exists) the path
  to a real reference file — that's spec, and it's the master's job. If no reference example exists yet,
  the master dispatches **one** worker to build the canonical layer plus one reference screen from that
  literal spec, then dispatches the remaining screens against that real file, each pointed at it
  explicitly — never builds the canonical layer itself "to save a round trip" or "so it's consistent."
  Once work is dispatched, the master's own role is to verify what came back — open the dev server, look
  at the actual rendered result, read the real diff — not to touch the application files itself under any
  framing. The goal is that a human never has to catch this by watching the activity viewer and telling
  the master to stop; the master should self-govern to this rule without needing to be told twice.
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
