# Phase 1 — Foundations Under a Real Project (Weeks 1–3)

**Output:** a complete, tested Ticket Tracker API you can explain line-by-line, plus fluency in the concepts every .NET screener call probes.

**Method:** no courses. Build the feature ladder in [01-project1-spec.md](01-project1-spec.md); when you hit a gap (what's a DTO? why is this async?), fill it *that day* with the concept deep-dive prompt, then log it.

## Files in this phase

| File | What |
|---|---|
| [01-project1-spec.md](01-project1-spec.md) | Full spec: domain, entities, endpoints, acceptance criteria |
| [02-week1-guide.md](02-week1-guide.md) | CRUD, REST semantics, DTOs, validation |
| [03-week2-guide.md](03-week2-guide.md) | EF relationships, migrations, filtering/paging, errors, logging |
| [04-week3-guide.md](04-week3-guide.md) | Auth (JWT), roles, unit + integration tests |
| [05-concept-checklist.md](05-concept-checklist.md) | The 12 concepts you must be able to *explain*, with quiz questions |

## Rules for this phase

1. **Each rung of the feature ladder = one branch = one PR** (even solo). Real PR description: what, why, how tested.
2. **The 4-step loop applies to every rung.** Attempt → review → quiz → (Saturday) rebuild.
3. **SQL drills start day 1** — [../drills/sql-30-day-plan.md](../drills/sql-30-day-plan.md), 30 min daily, non-negotiable. Phase 1 covers days 1–21.
4. **Friday debugging drill starts week 2** (you need working code to break first).
5. **Don't gold-plate.** No AutoMapper, no MediatR, no clean-architecture folder origami in Project 1. Controllers → services → EF. Complexity budget is spent in Phase 2.

## Exit criteria (be honest — Phase 2 assumes these)

- [ ] All ladder rungs merged, integration tests green
- [ ] Passed a full strict Claude review with no major findings
- [ ] Concept checklist: all 12 explained back via teach-back prompt, rated interview-ready
- [ ] Can write a controller + service + test from a blank file, no AI, narrating aloud, in under 30 min
- [ ] SQL drill days 1–21 complete
