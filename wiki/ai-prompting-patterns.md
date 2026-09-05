# AI prompting patterns

All AI techniques used in this project, centralized here so prompt design stays consistent across
features. Model: **Gemini 2.5 Flash** everywhere (not Pro — dropped from Gemini's free tier as of April
2026).

## 1. Frame → structured meaning

- **Structured JSON output prompting**: Gemini must return a fixed schema — `activity`,
  `objects_of_note`, `person_state`, `confidence`, `requires_attention: bool` — never free-text.
- **Few-shot prompting**: 2-3 example frame→JSON pairs specific to this domain in the system prompt.
- **Chain-of-thought, then discard**: brief reasoning before emitting JSON, reasoning itself not stored.

## 2. Short-term memory context (Phase 1)

Feed the last 3 `log_entries` into the prompt for the current event — session-scoped context
engineering, not full RAG, turning isolated observations into pattern-aware understanding.

## 3. Two-step agentic decision layer (Phase 1, centerpiece technique)

Every trigger (frame, geofence check, heart-rate reading) goes through the same shape:
- **Step 1 — Perception**: describe what's happening, incorporating §2's context.
- **Step 2 — Decision**: `log_only` / `alert_family` / `check_in`.

Same orchestrator pattern reused across all three trigger types — a real, explicit architectural talking
point for the report. Implemented once, called from `frames`, `geofence-check`, and `heartrate-events`.

## 4. Daily quiz generation (grounded mini-RAG)

Pull the day's actual `log_entries`, prompt Gemini for 3-5 questions grounded only in that data.
Retrieval = the DB query, generation = the LLM call. Never fall back to generic trivia if the day's log
is thin — that would break the "grounded in real activity" premise the feature depends on.

## 5. Watch app — scenario-driven, not live sensors

Scenario picker (`normal`/`confusion`/`emergency`) emits scripted heart-rate + motion data to the
backend, which runs the identical §3 decision layer. Live Health Services integration is explicitly
future-work, not in scope.

## 6. STT/TTS clarification step

Raw STT transcript → LLM clarifies intent (accounting for possible fragmentation/repetition patterns
common in memory impairment) → response generated → TTS out. The clarification step is the actual
technique being demonstrated, not the STT/TTS plumbing itself.

## Status

Phase 0 (today) implements §1 only, as a single Gemini call with no memory context and no decision layer
— see [api-contract.md](api-contract.md) status note. §2–§6 land in later phases per
[index.md](index.md) phasing.
