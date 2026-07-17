# Take-Home Simulation #2 — "Expense Approvals API" (Week 9)

Second rep, different domain, harder rules, tighter time: **3.5 hours.** The goal this round: beat your sim-1 rubric scores while cutting scope *more deliberately*.

## The brief

> **Backend take-home — Expense Approvals**
> Build a REST API for submitting and approving employee expenses. ~3.5 hours. .NET + relational DB preferred. Tests and a README with your decisions are required; we care more about correctness and clarity than completeness.
>
> **Requirements**
> - Employees submit expenses: amount, currency, category, description, receipt URL (string is fine).
> - Expenses flow: `Draft → Submitted → Approved | Rejected`. Only managers approve/reject; rejections require a reason.
> - Expenses over $500 require approval by a *senior* manager.
> - Employees see their own expenses; managers see their team's pending ones.
> - Monthly totals per employee, by category.
> - Auth can be simplified (e.g., a header identifying the user) — but document what production auth would look like.
>
> Submit a git repo. Note assumptions and cuts in the README.

## What this one is really testing (vs sim 1)

- **State machine rigor** — transitions + role gates + the amount threshold interact; matrix-test them (you have the pattern from Ticket transitions).
- **Money** — cents-as-integers + currency handling ("over $500" in *which* currency? Ambiguity: simplify to USD-only, document, mention real answer = per-currency thresholds or FX normalization).
- **Authorization shape** ⭐ — "own expenses" vs "team's expenses" is row-level access control: the #1 real-world API vulnerability class. Employee A requesting expense B-owned → 404 or 403 (pick, defend). Manager sees only *their team* — needs a team relationship you must invent (simplest: `ManagerId` on employee).
- **Reporting query** — monthly totals = GROUP BY with date truncation. Your SQL drills, cashing in.
- **The simplified-auth trap** — header auth is *fine*, but the README must show you know it's theater: "production: JWT as in my booking project — [link]."

## Planted ambiguities

<details><summary>Reveal after</summary>

1. Can a rejected expense be edited & resubmitted? (Unspecified — decide; resubmit-from-rejected is the friendlier call.)
2. Who is a "senior" manager? (Invent minimal: role enum. Document.)
3. Exactly $500 — over means > or ≥? (Trivial but graded: boundary test required either way.)
4. Can managers submit their own expenses? Who approves the manager's? (Name it in README even if unhandled — spotting it is the point.)
5. Currency of the threshold (above).
6. Delete expenses? (Unspecified — Draft-only delete is the defensible pick.)
</details>

## Rubric

Same table as sim 1 plus two rows:

| Area | 1 | 3 (bar) | 5 |
|---|---|---|---|
| Authorization | Any user does anything | Ownership + role checks on every endpoint | + IDOR test proving A can't read B's expense; 404-vs-403 reasoned |
| State machine | If-else soup | Transition rules centralized + tested | + full matrix test (state × action × role), threshold boundary test |

**Target: beat sim 1 in every area; nothing below 3.**

## Protocol

Same as sim 1 (20-min planning cap, feature-stop at T-45m, Claude rubric grading, 25-min debrief interview). Differences:
- **This run: AI-free if sim 1 wasn't** (calibration), or vice versa.
- Debrief focus areas to request: the authorization model, the state machine design, "how would this handle 10 managers approving concurrently" (your concurrency story generalizes — let them see you generalize it).
- Compare score sheets side by side; the deltas — better and worse — go in LEARNING-LOG. Worse-under-time-pressure areas get Saturday's rebuild slot.

## If a REAL take-home arrives during weeks 8–10

Skip whichever simulation you haven't done and treat the real one with this exact protocol (planning cap, feature-stop, README ritual). After submitting, still run the self-grade — calibrating your own predicted score vs their verdict is the most valuable data point you can collect.
