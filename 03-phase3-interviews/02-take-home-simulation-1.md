# Take-Home Simulation #1 — "Library Loans API" (Week 8)

Run this as a real take-home: **one 4-hour block, timer visible, no phone.** Ambiguities below are intentional — real take-homes have them; how you handle them (decide + document, or ask) is graded.

## The brief (as a company would send it)

> **Backend take-home — Library Loans**
> Build a small REST API for a library's loan system. We'd like to see how you structure a real service: correctness, tests, and clear decisions matter more than feature count. Timebox: ~4 hours. Use .NET; any database (in-memory acceptable if justified). Include a README with setup and your decisions.
>
> **Requirements**
> - Manage books (create, list, get). Books have title, author, ISBN, and number of copies.
> - Members can borrow an available copy and return it.
> - A member may have at most 3 active loans.
> - A loan is due 14 days after borrowing; the API should expose overdue loans.
> - List endpoints should support basic filtering and pagination.
> - Include tests for what you consider the important behavior.
>
> Submit a git repo (history included). Note anything you cut for time in the README.

## Planted ambiguities (don't read the list until AFTER — spot them yourself)

<details><summary>Reveal after submission</summary>

1. Duplicate ISBN — reject, or same book more copies? (Decide, document.)
2. "Available copy" — copies as rows or a counter? (Counter is fine at this scope; say why.)
3. Borrow when 0 copies available — 409? 422? (Consistency + documentation graded, not the pick.)
4. Can a member borrow the same book twice simultaneously? (Unspecified. Decide.)
5. "Overdue" — as-of when? Server clock? Timezone? (Inject a clock abstraction = testability win.)
6. Members aren't given a management endpoint — seed them? Create-on-demand? (Cheapest defensible: seed + document.)
7. Concurrent borrows of the last copy — your Bookable radar should ping. At minimum name the race in the README; constraint/transaction if time allows. **This is the "senior bait" — most submissions miss it entirely.**
</details>

## Grading rubric (have Claude grade you with this, then re-grade yourself in 2 days)

| Area | 1 (junior) | 3 (mid — the bar) | 5 (strong mid) |
|---|---|---|---|
| Correctness | Happy path only | Rules enforced (3-loan cap, availability), sane errors | + race named or handled, clock injected |
| API design | Verbs/codes inconsistent | RESTful, correct codes, paginated lists | + envelope, filter combos, ProblemDetails |
| Code structure | Logic in controllers | Thin controllers, service layer, DTOs | + domain rules isolated & unit-tested pure |
| Tests | None/trivial | Core rules unit-tested + a couple integration | + edge cases (cap boundary, overdue boundary), race test |
| README & decisions | Setup only | Setup + decisions + cuts documented | + tradeoffs argued, "what I'd do next" |
| Git history | One blob commit | Coherent incremental story | + history mirrors the plan, messages explain why |

**Passing = 3s everywhere, one 4+.** Any 1 → that area is the week's drill focus.

## Protocol

1. **First 20 min, no code:** read twice → list ambiguities + decisions in README skeleton → order the work: entities/persistence → borrow/return rules → list/filtering → overdue → tests as you go, not at the end.
2. **AI policy — decide before starting, hold yourself to it.** Recommended: simulate modern-allowed policy (most companies now permit AI but interrogate you on the result afterward — that's your quiz step anyway). One run of the two simulations should be AI-free for calibration.
3. At 3h15m: stop features. Final 45 min = README, cut notes, history cleanup, one full clean-clone test run.
4. **Grade:** > "Grade this repo against this rubric [paste]. Justify every score with file/line references. List what a reviewer at a good company would flag."
5. **The debrief interview (don't skip — most companies do this):** > "Interview me for 25 minutes about this submission as the reviewing engineer: why these choices, what breaks under load, how I'd extend it to multi-copy reservations. Skeptical but fair."
6. Log: score, time overruns, which ambiguities you missed.
