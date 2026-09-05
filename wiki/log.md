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
