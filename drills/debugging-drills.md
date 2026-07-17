# Friday Debugging Drills — Planted Bugs

Systematic debugging is mid-level signal #2 ("debugs systematically, doesn't guess") and the least practiced skill in tutorial-land. Every Friday, 60–90 minutes.

## The protocol

1. **Commit/stash everything.** Create branch `drill/bugs-week-N`.
2. **Plant:** run this in Claude Code from the project repo:
   > Introduce 3 realistic bugs into the code on this branch: one logic error, one async/concurrency issue, one data/EF issue. Make them subtle — the kind that pass a lazy code review and may only fail on certain inputs. Amend them into the code with an innocent-looking commit message. Tell me only which general area of the app is affected (e.g. "somewhere in tickets + auth"). Do NOT list the bugs. Keep a hidden record to verify against later.
3. **Hunt — AI off.** Tools allowed: running the app, `.http` requests, unit/integration tests, the VS Code debugger, logs, `git diff main` **only after 45 minutes** (reading a diff to spot bugs is itself an interview skill, but hypothesis-first hunting is the primary muscle).
4. **For each bug, write down before fixing:** symptom → hypothesis → experiment that confirmed it → fix. Four lines. This narration IS the "tell me about a hard bug" answer format.
5. **Verify:** > Here's what I found and fixed: [list]. Check against your record. Reveal anything I missed and rate my debugging approach.
6. **Log it** in LEARNING-LOG: which bug took longest and why.

## Escalation by week

| Week | Twist |
|---|---|
| 2 | First run — default protocol on Ticket Tracker |
| 3 | One bug must be in auth (wrong claim check, token validation gap) |
| 4 | Bookable; one *domain* bug (overlap predicate boundary, slot math off-by-one) |
| 5 | One bug reproduces **only under concurrency** — you'll need the parallel test to see it |
| 6 | One bug in React (stale closure, missing await); find it with browser devtools + network tab |
| 7 | One bug only manifests with realistic data volume ("works on my 3 seed rows") |
| 8+ | Interview mode: 30-min time limit, narrate aloud the whole time as if an interviewer is watching — because in a real bug-hunt interview round, one is |

## The debugging method (internalize the loop)

**Reproduce → localize → hypothesize → experiment → fix → verify → (write regression test).**

Anti-patterns you're training out of yourself:
- Changing code before reproducing the bug
- Changing two things at once (which one fixed it? unknowable)
- "Fixed" without understanding *why* it was broken (it'll be back)
- Shotgun-logging everywhere instead of binary-searching the failure point (breakpoint at the midpoint of the flow: is the data already wrong here? Go left/right.)

## Interview payoff

The "hardest bug" behavioral question and the live bug-hunt round both map exactly to step 4's four-liner. After 8 Fridays you'll have ~24 documented hunts; pick the 2 best stories for the story bank ([../03-phase3-interviews/06-behavioral-story-bank.md](../03-phase3-interviews/06-behavioral-story-bank.md)).
