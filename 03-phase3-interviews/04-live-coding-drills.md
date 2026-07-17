# Live Coding / Pair Programming Drills

The most failed round, and the most trainable. Practical shops don't ask LeetCode — they ask you to **add a feature to an existing codebase, fix a bug, or build a small thing from scratch, while talking.** Three skills, trained separately then together: the coding motion (Saturday rebuilds), narration (here), and composure under observation (reps).

## The narration framework — PACE

What "thinking aloud" concretely means; interviewers grade this more than the code:

- **P — Plan aloud before typing.** "Before I write anything: I'll look at how existing endpoints handle validation so I'm consistent, then write the failing case first."
- **A — Announce decisions + alternatives.** "I'll return 409 here — could argue 422, but this is a state conflict. Happy to discuss."
- **C — Check in.** "That's the happy path. Edge cases next or is this the depth you want?" (Interviewers redirect wandering candidates all day; the candidate who invites redirection stands out.)
- **E — Expose uncertainty productively.** Never bluff, never spiral: "I don't remember the exact overload — I'd check IntelliSense/docs; the shape I want is X." Bluffing fails candidates; honest recovery passes them.

## The mock protocol (45 min, 2–3×/week from week 5's Sunday, daily-ish weeks 8–10)

Run from a scratch copy or a fresh branch of one of YOUR repos — realistic "existing codebase" material you didn't memorize yesterday:

> You are a live-coding interviewer for a mid-level .NET role. Use the codebase in this repo. Give me ONE task from [feature-add / bug-fix / from-scratch — rotate]. Then: (1) act like a real interviewer — answer clarifying questions, drop occasional hints if I stall >3 min, interject "why did you do X?" at least twice; (2) after 35 minutes call time regardless of state; (3) then grade me: communication, approach, code quality, handling of your interjections — each 1–5 with specifics, plus the one behavior to change next session. I will narrate aloud and paste what I say as commentary; treat silence gaps in my commentary as silence in the interview.

**Narrate out loud, literally** — then paste/summarize your narration as you go. Typing-only defeats the drill; the mouth is the muscle being trained.

## Task rotation (the interviewer picks within these)

**Format A — feature-add (most common):**
- Add `PATCH /tickets/{id}/priority` with an authorization rule
- Add "reassign booking to another provider slot" with validation
- Add soft-delete to an entity incl. filtering it from all reads
- Add a rate-limit-ish rule: max 3 bookings per customer per day

**Format B — bug fix:** interviewer plants one bug (your Friday drill, observed + narrated + timed at 30 min — the combo round some companies run).

**Format C — from scratch (blank-file):**
- In-memory shopping cart API: add/remove/total with quantity rules
- A `SlotFinder` class given windows + bookings (you know the domain — do it in a *new* domain too: parking lots, meeting rooms)
- CSV import endpoint: parse, validate rows, report per-row errors (surprisingly common ask)

**Format D — refactor-with-tests (occasional):** "This service method is 80 lines and untested. Make it testable, add the tests, don't change behavior."

## Failure modes and their fixes

| Symptom in mocks | Fix |
|---|---|
| Long silences while thinking | Narrate the *question* you're thinking about: "I'm deciding between X and Y" — thinking aloud ≠ knowing aloud |
| Rushing to type before understanding | Enforce your own 3-minute no-code rule; restate the task + one clarifying question first |
| Spiraling on a forgotten API | PACE-E script, rehearsed until automatic |
| Ignoring interviewer hints | Hints are steering, not judgment — "good point, let me follow that" and *actually follow it* |
| Code works, tests skipped | Say the test plan even when time-boxed out: "with 10 more minutes I'd test the cap boundary and the 403 path" |
| Freezing when observed | Only fixed by reps — this is why mocks start week 5, not week 9 |

## Scoring trajectory

Log all four grades per session. **Ready-for-real bar: two consecutive mocks with everything ≥4 and zero bluffing incidents.** Communication routinely below 3 → double narration reps (do rebuild drills narrated too — they're the same muscle).
