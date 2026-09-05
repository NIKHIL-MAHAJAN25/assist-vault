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
