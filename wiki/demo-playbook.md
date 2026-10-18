# Demo playbook — Assist at the auction

> The website's **Demo kit** page (`/demo`, also linked from Settings) runs everything below and keeps a
> rehearsal checklist (`family-dashboard/src/lib/demoCases.js` — keep both in step). Server needs
> `DEMO_MODE=1` (default in `deploy/docker-compose.yml`; set `DEMO_MODE=0` in `deploy/.env` to turn off).

## The day before

| Check | How |
|---|---|
| Server up | `https://assist.3-107-133-23.sslip.io/health` → `{"status":"ok"}` |
| Helpers answering | Today → In a few words → Write summary → "Written by Claude/Codex" |
| Phone = Margaret's phone | App on the patient side, paired (Home shows "With your family") |
| Phone location ON | System Location on; Today shows "At Home" |
| Home place correct | Places → Home circle around the venue/your house (walk simulator starts from it) |
| Camera scenes saved | Stage each scene once (eating, TV, resting, leaving, fall, empty) → Demo kit → Save latest photo |
| Desktop alerts on | Bell → "Get desktop alerts" → Allow |
| Phone volume up, not on silent | For speech; "Ring phone" is loud by design |
| Reset | Demo kit → Reset demo |

Venue WiFi is not needed for the server (public HTTPS). The phone needs internet (mobile data is fine).
The ESP32 needs to share a network with the phone; if it can't, use **camera scenes** instead of live photos.

## Running order (≈ 6 minutes)

1. **Today** — the day at a glance; "In a few words" summary written by an AI helper.
2. **Assistant** — "How is Margaret right now?" (grounded answer, sources shown).
3. **Speak** → her phone says it in a natural voice. Then **Voice note** in your own voice.
4. **Ask her** "Have you had your dinner?" → tap Yes on her phone → answer appears + notification.
5. **Demo kit → Walk away from home** → red bell, toast, desktop alert within ~10 s.
6. **Guide home** → her phone jumps to the arrow, speaks, pings every 15 s. **Walk back home** → closes.
7. **Calm down** → breathing card + your voice note.
8. **Play scene "A fall"** → helper describes it → alert.
9. **Tell the family** → note to everyone.
9b. **"Tell me when she gets home"** → walk back → notification. **Simulate a fall** → "are you okay?" → escalation.
9c. **Bedtime story** → her phone reads her day back to her.
10. **Reset demo**.

## Test cases

| # | Feature | Steps | Judges see | Needs |
|---|---|---|---|---|
| 1 | Assistant answers | Assistant → "How is Margaret right now?" | Short grounded answer + "Claude · read location, camera, journal · Ns" | Website |
| 2 | Speak | Assistant → "Tell Margaret I'll call at 8" → Send | Sent → On her phone → Spoken aloud; spoken + on her Home | Phone |
| 3 | Voice note | Voice note → Record → Stop → Send | Her phone plays your voice; words under "Play again" | Phone, Chrome mic |
| 4 | Ask her | Ask her → question → tap Yes on phone (or Demo kit step 3) | "Margaret answered: Yes" on card + notification | Phone or kit |
| 5 | Wander alert | Demo kit → Walk away from home | Bell + toast + desktop "Margaret left the safe places"; journal | Home place |
| 6 | Guide home | Guide home → Start guiding | Phone speaks, opens arrow, 15 s pings | Phone |
| 7 | Comes back | Demo kit → Walk back home | "Back inside the safe places"; Today → At Home | — |
| 8 | Calm down | Calm down → Start | Breathing card, spoken lines, voice note | Phone |
| 9 | Reminder | Reminder → text + time = now+1 min | Spoken + shown at that minute | Phone |
| 10 | Ring | Ring phone → Ring now | 20 s at full volume, even on silent | Phone |
| 11 | Quiz | Send quiz → Answer now on phone | Invite card → questions about her day | Phone |
| 12 | Photo now | Take photo → Take photo | Sent → Photo taken → Described + photo; if camera off, a clear reason | Camera |
| 13 | Staged scene | Demo kit → Play "Eating" / "A fall" | New moment on Today/Log; "A fall" alerts | Saved scene |
| 14 | Tell the family | Tell family → text → Send to everyone | Other members get a notification | 2nd account optional |
| 15 | Summary | Today → In a few words → Write summary | Warm 3–4 sentences by a helper | Website |
| 16 | "Tell me when she gets home" | Walk away → rule → Walk back home → Run checks now | "Margaret is back inside the safe places"; rule ticks off | Home place |
| 17 | "Hasn't eaten by…" | Watch for… → not seen eating by (1 min ago) → Run checks now | "It's past … and Margaret hasn't been seen eating today" | — |
| 18 | Fall check — okay | Demo kit → Simulate a fall → tap "I'm okay" | "Possible fall — checking" then "Margaret says she's okay" | Phone or kit |
| 19 | Fall check — no answer | Simulate a fall → wait 2 min → Run checks now | "⚠ Possible fall — Margaret needs you" + phone rings | Phone |
| 20 | Bedtime story | Bedtime story → Write and send | "Today you…" story read aloud; "Your day" card | Phone |
| 21 | Reset | Demo kit → Reset demo | Demo actions/notes/rules/events/simulated readings cleared | — |

## If something goes wrong on stage

| Symptom | Do this |
|---|---|
| Camera not answering | Use Demo kit scenes; "Take photo" shows a clear failure message (that's fine to show) |
| Phone action stuck at "Sent" | Phone offline/asleep → wake it, open the app; actions expire after 10 min |
| No answer from the Assistant | Gemini fallback kicks in after 15 s; check helper log `journalctl -u assist-helpers -n 20` |
| Walk doesn't alert | Home place missing/too large — fix on Places; Stop + Walk again |
| Desktop alert missing | Browser permission blocked; the in-page toast and bell still work |
