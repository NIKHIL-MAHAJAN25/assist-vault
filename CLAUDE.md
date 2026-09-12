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
- **Master/slave subagent split** (from Phase 1 onward, once the API contract is locked): a master
  session (Opus, holds vault access) is the only writer to `decisions-log.md` and `log.md`; slave
  subagents are scoped per-repo (`model: sonnet` set explicitly in their frontmatter — it does not
  inherit), have no vault directory access, and receive the relevant wiki slice pasted directly into
  their task prompt. Not useful yet — Phase 0 backend work is sequential, single-repo.
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
