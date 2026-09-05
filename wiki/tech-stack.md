# Tech stack

| Component | Choice | Notes |
|---|---|---|
| ESP32-CAM firmware | Arduino/C++ | Already built and working — integrate against it, don't rebuild |
| Companion app | Kotlin, native Android | Later session |
| Backend | Node.js + Express | Layer-based folder structure: `routes/`, `controllers/`, `services/`, `models/` |
| ORM | **Prisma** | Schema-first, auto-migrations, type-safe client, native `JSONB` support |
| Database | PostgreSQL | Local install for dev (installed via winget on this machine — see below); Neon/Vercel Postgres flagged for future production deploy (connection pooling needed for serverless) |
| Dashboard | React | Later session |
| Watch app | Kotlin, Wear OS | Later session — scripted scenario vitals, not live sensors |
| AI | **Gemini 2.5 Flash** (not Pro — Pro dropped from Gemini's free tier as of April 2026) | Vision calls, agentic decisions, quiz generation, STT clarification |
| Change detection | Local histogram/pixel-diff heuristic in companion app | Deliberately not ML Kit — no model overhead needed |
| Face recognition | **face-api.js** (Node/TensorFlow.js) | Uses the same underlying recognition model as dlib (99.38% LFW, trained by the same author) — chosen to avoid Python/dlib/CMake build friction on Windows, not as an accuracy compromise |
| Image storage | **Cloudinary** | No card required; store `image_public_id`, not just the URL — needed for deletion later |
| Auth (users) | JWT, 2 RBAC roles: `patient`, `family_caregiver` | For dashboard/human logins |
| Auth (devices) | Device API key, issued at registration | ESP32/companion app/watch app are not "users" — don't reuse the JWT scheme for them |
| Push notifications | Firebase Cloud Messaging (FCM) | Free, unlimited, no card |
| SMS fallback | Twilio | No card if limited to pre-verified numbers (fine for a demo) |
| Email fallback | **Resend** | SendGrid's free tier was discontinued (May 2025) — do not use SendGrid |
| Deployment | Vercel | Cron Jobs needed for scheduled tasks (daily quiz generation, weekly image cleanup) since serverless functions don't run persistent background jobs |

## Infrastructure status

**Local dev Postgres**: PostgreSQL 17 installed via `winget` on this machine (Chocolatey's install
failed here due to an unelevated shell / lib lock issue — winget succeeded because it self-elevates the
installer). Service name `postgresql-x64-17`, port 5432.

- Superuser `postgres` / password `postgres` (dev machine only — never reuse this password anywhere
  real).
- App database `assistdb`, app role `assist_app` / password `assist_dev_pw`, granted on `assistdb` and
  its `public` schema. `backend-server`'s `DATABASE_URL` uses this role, not the superuser.

**Cloud accounts/credentials** (Gemini API key, Cloudinary, Vercel, Twilio, Resend, FCM) are provisioned
by the human but **not yet present as env vars on this machine** — `backend-server/.env` ships with
placeholders (see its `.env.example`); real values need to be filled in before Gemini/Cloudinary-dependent
routes will work end-to-end. Do not attempt to sign up for or provision these services.
