# Week 2 Guide — EF Core, Relationships, Filtering/Paging, Errors, Logging

**Ships:** Rungs 3–5. **Concepts unlocked:** migrations, change tracking, N+1, IQueryable composition, middleware pipeline, structured logging. This is the densest week — the EF concepts here are the most-probed technical area in .NET screeners.

## Day-by-day

### Mon — Rung 3, part 1 (EF plumbing)
- Add `User`, `Comment` entities + `AppDbContext`. Relationships via Fluent API in `OnModelCreating` (unique index on email, required FKs, delete behaviors — decide and document what happens to comments when a ticket is deleted).
- First migration. **Open the generated migration file and read it.** Then look at the actual tables in DBeaver. Interviewers ask "what does a migration actually do" — the answer is in that file.
- Replace the in-memory list with the DbContext. Seed data.

### Tue — Rung 3, part 2 (the two classic failures)
Engineer both failures on purpose — debugging them is the lesson:
1. **JSON cycle:** return an entity with navigation props → serializer explodes or emits a monster. Fix = response DTOs everywhere (you have them; now you know *why*).
2. **N+1:** load tickets, then touch `ticket.Reporter.DisplayName` in a loop with lazy loading semantics — or `.Include()` missing. Turn on EF SQL logging (`LogTo(Console.WriteLine)` or Serilog), count the queries, fix with `Include`/projection, count again. **Write the before/after query counts in the PR description.** This story — "I found an N+1 by logging SQL" — is interview gold.
- Comments endpoints + business rules (transitions, closed-ticket) in a `TicketService`. Controller stays thin.

### Wed — Rung 4 (filtering/sorting/paging)
- Build the query as composable `IQueryable` — `if (status != null) query = query.Where(...)` — and understand *why* this executes as one SQL query (deferred execution; concept deep-dive if fuzzy).
- Verify in the SQL logs: one query per request, `LIMIT/OFFSET` (or `TOP`) actually in it, filters translated to `WHERE`.
- Envelope response, pageSize cap, sort-field allowlist (`sortBy=lol` → 400; also ask yourself why an allowlist and not reflection — that's an injection-adjacent interview question).

### Thu — Rung 5 (errors + logging)
- Global exception-handling middleware → ProblemDetails. Concept deep-dive: **the middleware pipeline** — order matters; be able to draw request → middleware chain → endpoint → back.
- `DomainRuleException` → 422 mapping. Grep your services: any rule violation still throwing generic exceptions? Fix.
- Serilog: console sink, request logging, structured properties. Deliberately break the DB connection string and watch what a failure looks like in the logs — that's your "how would you diagnose X in production" material.
- PRs #3–5 merged by tonight (review → quiz → merge, every one).

### Fri — First debugging drill 🐛
- Full instructions: [../drills/debugging-drills.md](../drills/debugging-drills.md). Claude plants 3 bugs in a branch; you find them with debugger + logs only. Budget 60–90 min.
- SQL drills continue (days 8–12).

### Sat — Rebuild drill #2
- From memory: the filtered/sorted/paged list endpoint, including the envelope. This is THE most common take-home component; make it automatic.

## Concepts to close out (checklist 5–9)
Migrations · change tracking (tracked vs `AsNoTracking`) · N+1 + Include vs projection · `IQueryable`/deferred execution · middleware pipeline.

## Interview connection
"How do you find and fix a slow endpoint?" — after this week your answer is concrete: *logs → EF SQL logging → spot N+1 or missing index → fix with Include/projection/index → verify query count and timing.* Rehearse saying it in under 60 seconds.
