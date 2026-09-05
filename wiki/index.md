# Index — Problem, Persona, Scope, Phasing

## Problem statement

Individuals with Alzheimer's and related memory impairments face escalating risks — wandering,
forgetting recent activities, difficulty communicating needs — while family caregivers face constant
uncertainty about their loved one's safety and wellbeing. Existing solutions are typically single-purpose:
medical alert buttons are purely reactive, GPS trackers offer location without context, and none provide
passive activity understanding, safety intervention, and cognitive engagement in one system.

This project is an AI-assisted safety, navigation, and cognitive engagement system that passively
observes daily activity via IoT vision, understands it using AI, intervenes when safety is at risk, and
reinforces memory through personalized daily engagement.

## Primary persona (design lens)

Center all UX and design decisions on **a person with mild-to-moderate Alzheimer's/memory impairment
living semi-independently**. If any feature's behavior is ambiguous, default to what serves this persona
specifically — low cognitive load, minimal navigation, familiar/simple interaction patterns — rather than
a generic "assistive tech" user.

### Research grounding (cite plainly, don't overclaim)

- Wandering is a well-documented, major safety risk in Alzheimer's — commonly cited estimates put
  lifetime wandering risk at roughly 60% of people with the disease, with study-to-study variation
  (11-50% in various methodologies, up to ~70% in some community-dwelling samples). This justifies
  geofencing as a core feature — cite plainly, don't inflate further.
- Emotionally significant/long-consolidated memories are somewhat better preserved than recent episodic
  memory in Alzheimer's — supports the "reinforce via personalized daily content" framing for the quiz
  and recall features.
- **Do not claim** "familiar voices/faces are always recognized regardless of memory loss" — the research
  is genuinely mixed (some studies show impaired emotional-prosody recognition from voice in Alzheimer's;
  voice/identity recognition itself degrades progressively, just often later than other functions). The
  defensible framing: voice-based, low-cognitive-load interaction is a *reasonable design bet*, not a
  guarantee — pitch it that way in the report, not as settled fact.

## Full feature scope

**Core pipeline (the spine)**
- ESP32-CAM (done) → throttled frame capture (1 frame/5s) → companion app local change-detection
  (histogram/pixel-diff heuristic, not ML Kit) → backend → Gemini 2.5 Flash vision call → structured JSON
  → `log_entries` → two-step agentic decision layer (log / alert family / check-in)
- Short-term memory context: last N log entries fed into prompts for continuity-aware understanding

**Safety**
- Geofencing (GPS from companion app, safe zones configured via dashboard)
- Wear OS app (Galaxy Watch 6 Classic compatible): scenario picker (normal / confusion / emergency),
  each emitting scripted heart-rate + motion-pattern data — NOT live Health Services sensor polling
- "Lost?" Android home screen widget: one-tap, no app navigation required — opens a simplified,
  dementia-friendly map view (large single directional indicator, not full interactive maps UI) and/or
  plays a single designated caregiver's pre-recorded voice message
- NFC emergency-contact sticker on the device box — static, no AI, no backend dependency, programmed
  once via any NFC-writer app with a vCard/contact URL. Not a software build task.

**Recognition & memory support**
- Facial recognition, up to 6 enrolled family members (face-api.js)
- TTS-only guidance on recognition (e.g. "He is your son") — user-toggleable, off by default
- Routine/pattern learning + deviation flagging (compares current event against the patient's established
  pattern, not just judging each event in isolation)
- Object recall ("where are my glasses") — grounded query over `log_entries`
- "What did I do today" recall — conversational, grounded over the day's log

**Engagement**
- Daily quiz generated from that day's actual `log_entries` (grounded mini-RAG, not generic trivia)
- Day/time orientation display (non-AI — static "Today is Tuesday, it's afternoon, next: lunch at 1pm")
- Medication reminders + visual confirmation via camera log (did the patient appear to take medication)
- Family voice-note playback (family records a message via dashboard, played back on trigger/schedule)

**Family-facing**
- React dashboard: RBAC login (`patient` / `family_caregiver`), log timeline, geofence zone editor, face
  enrollment, quiz history, alert view
- Notifications: attempt FCM push first; if implementation friction is too high, fall back to in-dashboard
  notifications + email (Resend) / SMS (Twilio, pre-verified numbers) for emergency-severity alerts only

## Phasing

- **Phase 0 — plumbing:** ESP32 → companion app → backend → DB, one frame all the way to a log entry.
  Prove the pipe works before adding intelligence. *(Backend structure built first — see
  [decisions-log.md](decisions-log.md) session 1 entry.)*
- **Phase 1 — core intelligence:** two-step agentic decision layer, short-term memory context, geofencing,
  notifications.
- **Phase 2 — engagement + recognition:** facial recognition, routine learning, object/day recall, daily
  quiz, TTS guidance toggle, medication reminders, orientation display. Needs a healthy log of real data
  to be meaningfully grounded — don't build before Phase 0/1 produce real entries.
- **Phase 3 — wearable + physical:** Wear OS app, "Lost?" widget, NFC sticker, family voice-note playback.

## Explicitly out of scope for this prototype

- Live Wear OS Health Services sensor polling (scripted scenarios only)
- ML Kit or any on-device model for change detection (cheap heuristic only)
- Full auth/roles beyond the two RBAC roles specified
- NFC content beyond a static contact card
- Continuous/unthrottled frame processing (companion app change-detection gate is mandatory)
- Graphiti, Graphify, and Munder Difflin (see [decisions-log.md](decisions-log.md) — considered, deferred)
