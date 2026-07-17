# Saturday Rebuild Drills — From Memory, AI Off

The retention step. If AI-assisted building is reading with a highlighter, rebuilding from memory is the closed-book exam — and it's a near-exact simulation of live-coding interviews (blank editor, someone watching, no autocomplete).

## The protocol

1. Pick this week's feature (the weekly guides name one; override if something else felt shakier).
2. Blank folder or file **outside** the project (scratch console/webapi project is fine).
3. **AI completely off:** Codex disabled (VS Code: disable completions for the workspace), no Claude, no Google, no peeking at the real repo. Official docs via IntelliSense hover is the only lifeline.
4. Timebox 45 min. **Narrate aloud the entire time** — what you're doing and why. Feels ridiculous; it's the single highest-leverage interview habit and it only becomes natural through reps.
5. Done or time-up → compare with the real implementation. Note what you forgot.
6. Grade yourself:

| Grade | Meaning | Action |
|---|---|---|
| A | Compiles, correct, minor naming diffs | Ratio is right, carry on |
| B | Right structure, forgot details (an attribute, a using, an edge case) | Note the details, re-quiz Monday |
| C | Needed the docs lifeline a lot, structure shaky | This feature was AI-authored more than you admitted — redo the ownership quiz on the original |
| F | Couldn't produce the skeleton | Red flag: shift next week hard toward attempt-first; rebuild this same feature next Saturday |

7. Log the grade in LEARNING-LOG. **Two C-or-worse weeks in a row = the AI ratio is broken; fix the ratio, not the symptom.**

## Rebuild schedule (matches the weekly guides)

| Week | Rebuild target |
|---|---|
| 1 | Ticket CRUD controller + one DTO, correct status codes |
| 2 | Filtered/sorted/paged list endpoint with envelope |
| 3 | Register + login incl. hashing + JWT issuance |
| 4 | Slot-computation pure function + 3 unit tests |
| 5 | Race-condition explanation + optimistic concurrency handling |
| 6 | Outbox write + BackgroundService poller skeleton |
| 7 | Booking-create service method, both concurrency layers |
| 8–10 | Interview-mode: whatever a mock exposed as weak that week |

## Why this works (so you don't skip it when tired)

Retrieval practice is the most robust finding in learning science: the *struggle to recall* is what writes to long-term memory — reading/re-generating with AI never triggers it. Every rebuild is also one more rep of the exact motion an interview demands: producing working code from an empty buffer while talking. Twenty minutes into your first live-coding round, you'll be running a motion you've done ten times.
