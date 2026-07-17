# Behavioral Story Bank

Behavioral rounds fail more candidates than technical ones, and they're the *most* preparable. Six rehearsed stories cover ~every question; vague, rambling, or invented answers are the failure mode — specific and rehearsed is the fix.

## Format: STAR, tightened

**S/T (2 sentences max)** — context + the problem. **A (the bulk)** — what *you* did, first person, specific decisions. **R** — outcome with something concrete (a number, a behavior change, a merged fix) + one-line lesson.
Two lengths per story: **90 seconds** (the answer) and **4–5 minutes** (the deep-dive when they say "tell me more").

## The six stories — mined from your 10 weeks

Your LEARNING-LOG and decision docs are the source. Draft all six in week 8; the raw material will genuinely be there if the drills happened.

### 1. The hardest bug ⭐ (asked in ~every loop)
**Candidates:** the DST slot-boundary failure · reproducing the double-booking race and making it deterministic · the best Friday planted-bug hunt (an AI planted it; you still genuinely hunted it — tell it as "a subtle bug in the codebase," which is true).
**Must include:** the symptom→hypothesis→experiment→fix loop, explicitly — the *method* is what's being graded, not the bug.

### 2. A technical decision with tradeoffs ⭐ (the Bookable anchor story)
Double-booking prevention: naive → proved the race with a parallel test → optimistic concurrency → *why that alone was insufficient* → DB exclusion constraint → exactly-one-success test. This is simultaneously your "hardest problem," "proudest work," and "decision you defended" answer. **Rehearse this one the most — it's the anchor.**

### 3. Receiving critical feedback
Honest and strong version: strict review passes (AI or human) repeatedly flagged something in your work — e.g., early weeks of leaking entities, or thin error handling — and you changed your default behavior, not just the instance. R: "it stopped appearing in reviews; it's now how I build by default."

### 4. A deadline / scope call
Take-home sim material: the T-45-minute feature-stop, what you cut, how you documented it — or the week-6 "2 days for the frontend, functional over pretty" call. Shows judgment, the actual thing being probed.

### 5. A disagreement (handled well)
Solo-project honest version: your two AI reviewers disagreed (Claude vs Codex on a design point), or you disagreed with a review finding — how you evaluated arguments on merits, tested the claim, and picked. Frame: "I don't resolve disagreements by authority or stubbornness; I resolve them by evidence." If you have ANY real prior-job/team story here, prefer it.

### 6. Something you'd do differently
Real material by week 8, guaranteed (check LEARNING-LOG). Strong shape: a process lesson, not a syntax lesson — "I built the naive booking path before thinking about concurrency; the rework taught me to identify the correctness-critical invariant *first*. In Bookable's later features I started from the invariant." Self-awareness + changed behavior = the whole point of the question.

## Question → story routing

| They ask… | Story |
|---|---|
| Hardest bug/problem | 1 (or 2 for "problem") |
| Proudest work / deep-dive a project | 2 |
| Tell me about a failure | 6 (or 1 if the bug was self-inflicted — even better) |
| Conflict / disagreement | 5 |
| Tight deadline / prioritization | 4 |
| Feedback you received | 3 |
| Why should we hire you / strengths | 2's method + 3's coachability, composed as traits |

## The "junior with AI training wheels?" question — be ready

Some interviewer will probe whether AI tools built your portfolio. This is a *gift* if prepared:
> "AI is in my workflow the way it's in your team's — completions, review passes, test scaffolding. I deliberately trained authorship: everything merged, I can rebuild without it, and I do weekly closed-book rebuild drills to keep that true. Pick any part of the booking system and I'll whiteboard it right now."
Offering the whiteboard *converts the doubt into your best moment* — you've rehearsed exactly that motion every Saturday.

## Rehearsal protocol (weekly from week 8)

> Ask me 5 behavioral questions typical for a mid-level engineering interview, one at a time. After each answer: grade concreteness and structure; flag anything vague ("we improved things" — how much? what changed?), any missing R, anything over 2.5 minutes. Then ask ONE skeptical follow-up per story like an interviewer who suspects it's embellished.

Out loud, always. The follow-up drill matters most: rehearsed candidates fail on the *second* question about a story, not the first.

## Questions to ask THEM (prepare 3, they're graded too)

- "What does the path from mid to senior look like on this team — what's the difference in expectations?"
- "How does code get from a laptop to production here — review, CI, deploy cadence?"
- "What's the hardest part of the codebase right now, and is that where I'd start?"
Signal: you think about growth, process, and real work — not perks.
