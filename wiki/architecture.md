# Architecture

```
ESP32-CAM ──(1 frame/5s, WiFi, already working)──> Companion App (Kotlin, Android — LATER SESSION)
                                                          │
                                                          ├─ local histogram/pixel-diff change detection
                                                          │
                                                          ├─ qualifying frames ──> Backend (Node/Express)
                                                          │                              │
                                                          │                              ├─> Gemini 2.5 Flash
                                                          │                              │   (structured JSON out)
                                                          │                              ├─> upload image to Cloudinary
                                                          │                              ├─> [Phase 1] agentic decision
                                                          │                              └─> write log_entries (Postgres)
                                                          │
                                                          ├─ GPS ──> Backend (geofence check)
                                                          ├─ mic (STT) ──> Backend (clarify) ──> TTS out
                                                          └─ pulls today's log ──> daily quiz

Wear OS App (LATER SESSION) ──(scripted scenario vitals)──> Backend (same agentic decision layer)

React Dashboard (LATER SESSION) ──(reads + configures)──> Backend ──> Postgres

NFC sticker — static, physical setup task, no software dependency.
```

**`log_entries` is the spine.** Every feature is either a writer to it or a reader of it. No component
holds separate state.

## Repo layout

Sibling repos under `D:\`, not nested inside the vault:

- `D:\AssistVault` — this wiki + system-level `CLAUDE.md`. No application code.
- `D:\backend-server` — Node/Express + Prisma + Postgres. Layer-based: `routes/`, `controllers/`,
  `services/`, `models/`. Built first (Phase 0).
- `D:\companion-app` — Kotlin/Android. Later session.
- `D:\family-dashboard` — React. Later session.
- `D:\watch-app` — Kotlin/Wear OS. Later session.

The vault links to sibling repos via `additionalDirectories` in `.claude/settings.json`
(`CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` set so each repo's own `CLAUDE.md` is also picked up),
so a master session working from the vault can read across all repos. Component-scoped subagents get no
such link (see [../CLAUDE.md](../CLAUDE.md) working agreement) — only the master pastes relevant wiki
slices into their prompts.

## Component responsibilities

| Component | Responsibility | Talks to |
|---|---|---|
| ESP32-CAM | Captures frames, serves them over WiFi | Companion app |
| Companion app | Change-detection gate, GPS relay, mic capture, quiz UI, recall UI | Backend, ESP32-CAM |
| Backend | All AI calls, all persistence, all business logic, auth for both users and devices | Postgres, Cloudinary, Gemini, Twilio, Resend, FCM |
| React dashboard | Family-facing config + monitoring, reads/writes via JWT-authed routes | Backend |
| Watch app | Emits scripted scenario vitals only | Backend |
| NFC sticker | Static vCard/contact URL, no runtime dependency | — |

No component other than the backend talks to Postgres, Gemini, Cloudinary, Twilio, Resend, or FCM
directly — this keeps `api-contract.md` the single integration surface between components.
