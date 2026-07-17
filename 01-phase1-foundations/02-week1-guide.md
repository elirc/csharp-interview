# Week 1 Guide — CRUD, REST Semantics, DTOs, Validation

**Ships:** Rungs 1–2 of the [project spec](01-project1-spec.md). **Concepts unlocked:** HTTP semantics, DTOs, model binding/validation, DI basics.

## Day-by-day

### Mon — Rung 1, part 1
- Solution skeleton from setup day. Create `Ticket` model + `TicketsController` with POST and GET (list + by id) against a static in-memory list.
- Attempt-first: write the controller before asking anything. Then task-breakdown prompt if flailing.
- Build a `tickets.http` file as you go — every endpoint gets a request the moment it exists.
- **Trap you should hit on purpose:** return the wrong status code somewhere, then ask Claude to review — see if it catches it.

### Tue — Rung 1, part 2
- PUT, DELETE. Decide: 204 vs 200-with-body on update (either defensible — *write your reasoning in the PR description*, that habit is the point).
- 404 handling for every `{id}` route. GET-after-DELETE returns 404.
- Strict-review prompt on the full controller → fix → ownership quiz → merge PR #1.

### Wed — Rung 2, part 1 (DTOs)
- Before coding, concept deep-dive prompt: **"why DTOs instead of exposing entities"**. You want three reasons minimum (over-posting, coupling/API stability, serialization cycles later with EF).
- Create request/response DTOs, hand-map (no AutoMapper — hand-mapping makes the boundary visible while you're learning).

### Thu — Rung 2, part 2 (validation)
- Data annotations on request DTOs; observe `[ApiController]` auto-400 behavior. Inspect the error response shape in your `.http` output — you'll be asked "what does a validation failure look like to the client" in take-home debriefs.
- Edge cases: empty body, wrong types (`"priority": "banana"`), extra unknown fields. Know what happens for each.
- PR #2: review → quiz → merge.

### Fri — Drill day
- Debugging drill starts next week; today instead: **status-code oral exam** — quiz prompt: "rapid-fire me on HTTP: when 200 vs 201 vs 204, 400 vs 404 vs 409 vs 422, 401 vs 403, PUT vs PATCH vs POST, what idempotent means and which verbs are."
- SQL drill as every day (days 1–5 of the plan).

### Sat — Rebuild drill #1
- Blank file, AI off: `Ticket` CRUD controller with correct status codes + one DTO, from memory, ≤45 min. Rules in [../drills/rebuild-drills.md](../drills/rebuild-drills.md).

## Concepts to close out this week (checklist items 1–4)
HTTP semantics · DTOs & the API boundary · model binding + validation pipeline · DI registration (`AddScoped` etc. — you used it; now be able to explain it).

Run the teach-back prompt on each. Log gaps in LEARNING-LOG.

## Interview connection
Rung 1–2 *is* the first 30 minutes of most live-coding interviews for CRUD roles: "add an endpoint, add validation, correct status codes, talk while you do it." By Saturday you should be able to do it cold. That's the bar.
