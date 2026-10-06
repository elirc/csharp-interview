# Technical Question Bank — Screener & Q&A Rounds

The 56 core questions for practical mid-level .NET screeners. ⭐ (18 of them) = highest frequency, drill first. Answers are the *shape* of a mid-level response — one confident paragraph with a why and a tradeoff — not essays to memorize. Personalize each with your project experience: "in my booking app, I…" beats any textbook answer.

**Drill protocol (20 min, 4×/week):** > "Quiz me from these categories: [pick 2]. Ask one at a time, wait for my spoken-style answer, grade at a mid-level bar, push back with 'why' on anything shallow." Track misses; re-drill misses first next session.

> **This file is the core. [../04-question-vault/](../04-question-vault/) is the depth layer** — ~490 more Q&A, the follow-up chains that come after "why?", 270 rapid-fire flashcards, code-output gotchas, and spot-the-bug snippets. Category links are inline below. **Get fluent here first**; breadth without the core is how candidates fail.

---

## C# / .NET language & runtime
*Depth: [`L` — 72 language/runtime questions](../04-question-vault/01-csharp-language.md) · [`A` — 46 async/concurrency questions](../04-question-vault/02-async-concurrency.md) · [`G` — code-output gotchas](../04-question-vault/12-code-output-gotchas.md)*

**1. ⭐ async/await — why in a web API, and does it speed up requests?**
Throughput, not latency: during I/O awaits the thread returns to the pool to serve other requests. Same request takes the same time; the server handles far more concurrent load. Sync-over-async (`.Result`/`.Wait()`) blocks threads → pool starvation. Async all the way down. CPU-bound work gains nothing.

**2. Value vs reference types; where do structs bite you?**
Copy semantics vs shared reference. Bites: mutable structs (mutating a copy, change lost), boxing when a struct hits `object`/non-generic interfaces. `record` vs `record struct` shows you're current.

**3. ⭐ IEnumerable vs IQueryable?**
Both lazy; IQueryable builds an expression tree a provider translates (to SQL); IEnumerable executes delegates in memory. Footgun: early `.ToList()` materializes the table then filters in C#. Also: `Where` on IQueryable → SQL WHERE; after `.AsEnumerable()` → in-memory scan.

**4. Exceptions — when to catch, when to let fly?**
Catch only where you can add value: translate (concurrency exception → 409), retry, or enrich. Otherwise let global middleware handle it. Catching-and-swallowing hides bugs; exceptions for exceptional flow, not control flow (rule violations → result types or domain exceptions mapped to 4xx).

**5. `string` interning/immutability — why does building strings in a loop matter?**
Strings immutable → each concat allocates. Loops → StringBuilder. Rarely load-bearing in web code, common warm-up.

**6. What is LINQ deferred execution and when has it hurt you?**
Query executes at enumeration, not definition. Hurts: enumerating twice = querying twice (cache with ToList when reusing); capturing a loop variable in a Where; querying after the DbContext is disposed. Real story from your repo beats theory.

**7. `null` handling — how do nullable reference types change your code?**
Compile-time flow analysis on possible nulls. Mid answer: enabled project-wide, warnings-as-errors, `?` only where null is a *meaningful* state (Ticket.AssigneeId), not a habit.

**8. Records vs classes?**
Value-based equality, `with` expressions, concise init-only shape — ideal for DTOs. Entities stay classes (identity by Id, EF change tracking mutates).

## ASP.NET Core
*Depth: [`W` — 62 questions](../04-question-vault/03-aspnetcore.md), incl. the `IOptions` trio, `IHttpClientFactory`, filters vs middleware, and the full request narration (`W62`)*

**9. ⭐ DI lifetimes — and a bug from misusing them?**
Transient per-resolve, scoped per-request, singleton per-app. The classic: scoped (DbContext) captured by a singleton → captive dependency, possibly shared non-thread-safe context across requests. .NET's container throws on this in development — know that it's validating scopes.

**10. ⭐ Walk a request through the pipeline.**
Kestrel → middleware in registration order (exception handler → HSTS/HTTPS → static files → routing → CORS → authN → authZ → endpoint) → model binding → validation → filters → action → result serialization → back out the chain. Know *why* order matters: authN before authZ; exception handler outermost to catch everything below.

**11. Middleware vs action filter?**
Middleware: whole-pipeline, no MVC context (logging, auth, CORS). Filters: after routing, MVC context available (model state, action args) — validation, per-action policies.

**12. ⭐ 401 vs 403? 400 vs 404 vs 409 vs 422?**
Unauthenticated vs unauthorized. Malformed/failed validation (400) · no such resource (404) · state conflict — duplicate email, concurrent edit (409) · well-formed but semantically invalid, e.g. illegal status transition (422; or defend 400 — the *distinction awareness* is what's tested).

**13. How does model validation work and where do complex rules live?**
Annotations/FluentValidation run after binding; `[ApiController]` auto-400s. Shape rules on DTOs; *business* rules (24h cancellation) in the domain/service layer returning domain errors → 422. The split is the mid-level part.

**14. CORS — what problem does it solve?**
Browser-enforced same-origin policy; CORS is the server opting in to cross-origin callers via headers (preflight OPTIONS). Key nuance: protects users-in-browsers, not the API — curl ignores it; it is not an auth mechanism.

**15. How do you version an API?**
URL segment (`/v1/`) most common/visible; header/media-type cleaner but subtler. Mid answer: additive changes don't need versions — add fields, don't rename/remove; version on breaking change only.

**16. Kestrel vs IIS/reverse proxy?**
Kestrel is the app server; a proxy (nginx/IIS/cloud LB) typically fronts it for TLS termination, static assets, header hygiene, multiple apps per host.

## EF Core & data access
*Depth: [`E` — 52 questions](../04-question-vault/04-efcore.md), incl. cartesian explosion, `AsSplitQuery`'s hidden cost, `ExecuteUpdate`, and why the version token doesn't stop double-booking (`E35`)*

**17. ⭐ What is a migration; how do schema changes reach prod?**
Generated Up/Down diff, ordered, tracked in `__EFMigrationsHistory`. Dev: CLI. Prod: CI deploy step or startup-migrate (fine single-instance; racy multi-instance). Review generated SQL for destructive ops before shipping.

**18. ⭐ N+1 — detect and fix?**
List query + per-row lazy queries. Detect: EF SQL logging / profiler; symptom is "fast locally, slow with data." Fix: `Include`, or better, projection to exactly the needed columns. Tell YOUR week-2 story with query counts.

**19. Include vs projection (Select) — when each?**
Include loads full related entities (tracked, updatable). Projection fetches only needed columns into DTOs — lighter, no tracking, usually right for reads. Mid line: "project for queries, include when I need to *modify* the graph."

**20. Change tracking; AsNoTracking?**
Tracked entities → SaveChanges computes diffs. AsNoTracking for read-only: less memory/CPU, no accidental writes. Default reads to AsNoTracking in query-heavy paths.

**21. ⭐ Transactions in EF — when explicit?**
SaveChanges is already atomic for one call. Explicit `BeginTransaction` when multiple SaveChanges/operations must commit together — e.g. booking + notification-outbox row. ACID one-liner ready.

**22. ⭐ Optimistic vs pessimistic concurrency?**
Your centerpiece — answer with the Bookable story: version-token + catch-and-409/retry vs locking; optimistic fits web (rare conflicts, no held locks); AND the insight that two INSERTs don't collide on a version column, hence the DB exclusion constraint as the real guarantee. This answer alone moves you into "hire" territory in many loops.

**23. Why can't app code alone prevent duplicates/overlaps?**
Any check-then-act outside the DB has a race window between check and write. Only a database constraint (unique/exclusion) is atomic with the write. App checks = good UX; constraint = correctness.

**24. Repository pattern — yes or no?**
Have an opinion with tradeoffs: DbContext already is unit-of-work + repositories; a thin extra layer often adds ceremony; genuinely useful for swap-ability/test seams in bigger codebases. "I used services over DbContext directly in a small app, and here's when I'd add the abstraction" is a great mid answer.

## SQL & databases
*Depth: [`S` — 56 questions](../04-question-vault/05-sql-databases.md), incl. logical execution order, SARGability, window frames, isolation anomalies, and `SKIP LOCKED`. Hands-on: [../drills/sql-30-day-plan.md](../drills/sql-30-day-plan.md)*

**25. ⭐ INNER vs LEFT JOIN; "customers with zero orders"?**
LEFT keeps unmatched left rows with NULLs. The classic: LEFT JOIN + `COUNT(o.id)` (counts non-null only) — `COUNT(*)` would return 1 for zero-order customers. Whiteboard it cold.

**26. WHERE vs HAVING?**
Rows before grouping vs groups after aggregation. "Customers with more than 5 orders" → HAVING.

**27. ⭐ What is an index; why not index everything?**
B-tree → O(log n) seeks instead of scans. Costs: every write updates every index; storage; the planner may ignore low-selectivity ones. Index FKs and frequent filter/sort columns; measure with EXPLAIN. Cite your dashboard-query before/after.

**28. Composite index on (a, b) — which queries can use it?**
Leftmost prefix: filters on `a` or `a AND b` yes; `b` alone no. Column order follows query patterns.

**29. ⭐ Top-N per group?**
`ROW_NUMBER() OVER (PARTITION BY grp ORDER BY metric DESC)` in a CTE, filter rn ≤ N. THE mid-level SQL question — be able to type it.

**30. The NOT IN + NULL trap?**
`x NOT IN (subquery)` returns no rows if the subquery yields any NULL (three-valued logic). Use `NOT EXISTS`.

**31. Transaction isolation — what do you actually know?**
Honest mid answer: read committed default; dirty/non-repeatable/phantom exist on a spectrum up to serializable; higher isolation = more safety, less concurrency. "I solved my booking overlap with a constraint rather than serializable, because retry-on-conflict beats lock contention" — grounded beats encyclopedic.

**32. OFFSET pagination at page 1000?**
DB still walks all skipped rows. Keyset/cursor pagination: `WHERE (sort_key, id) > (last_seen) LIMIT n`. Knowing this cold is a differentiator.

**33. SQL injection — and is EF safe?**
Untrusted input concatenated into SQL. Parameterized queries fix it; EF parameterizes LINQ automatically — but raw SQL APIs still need parameters, and dynamic ORDER BY needs an allowlist (cite your sort-field allowlist).

## Auth & security
*Depth: [`X` — 42 questions](../04-question-vault/08-security-auth.md), incl. OAuth flows, IDOR, SSRF, timing attacks, Data Protection keys, and the file-upload threat model (`X42`)*

**34. ⭐ JWT — anatomy, and why can't clients forge one?**
header.payload.signature; signature = HMAC/RSA over the first two with the server's key — tamper-evident, not encrypted (never put secrets in claims). Stateless tradeoff: no server-side revocation before expiry → short-lived access + refresh tokens.

**35. JWT vs session cookies?**
Session: server-side state, instantly revocable, cookie transport (CSRF to manage). JWT: stateless, scales across services, revocation hard. Mid answer names storage tradeoff: localStorage (XSS-readable) vs httpOnly cookie (CSRF to manage).

**36. Password storage?**
Slow adaptive hash (bcrypt/argon2/PBKDF2), per-password salt. Never encryption (reversible), never fast hashes (GPU brute-force). Know *why* slow is the feature.

**37. ⭐ Over-posting / mass assignment?**
Binding request JSON straight to entities lets clients set fields they shouldn't (`"role": "Admin"`). Fix: input DTOs with only settable fields; server-controlled values (reporter id) from claims, never the body. Cite your week-3 fix.

**38. OWASP top items you actively handle?**
Injection (parameterized), broken auth (hashing, expiry), broken access control ⭐ (ownership checks — *the* most common real-world API vuln: "can user A fetch user B's booking by ID?"), sensitive-data exposure (secrets out of git, HTTPS). Four with concrete mitigations beats ten names.

**39. Where do secrets live?**
Dev: user-secrets/env. Prod: platform env vars / secret manager. Never git — history is forever; a leaked key means rotation, not deletion.

## Testing & quality
*Depth: [`T` — 36 questions](../04-question-vault/06-testing.md), incl. xUnit mechanics, `WebApplicationFactory` service swapping, Testcontainers vs InMemory, and how to test a race (`T26`)*

**40. ⭐ Your testing strategy?**
Concrete, not pyramid-theory: "xUnit unit tests on domain logic — slot computation, transition rules; integration via WebApplicationFactory against real Postgres for routing/auth/persistence; the race-condition regression test that proves double-booking is impossible. Behaviors, not implementation details."

**41. What makes code testable?**
Dependencies injected & interface-shaped; logic separated from I/O (pure slot function!); no static/global state; small units. Cite extracting slot logic into a pure function as a deliberate testability decision.

**42. Mocks/fakes/stubs — how much mocking?**
Answer with a stance: minimal — mock at boundaries you don't own (email sender), prefer real/in-memory infrastructure where cheap; over-mocked tests assert implementation, break on refactor, catch nothing.

**43. A test caught a real bug — story?**
From your repo — the DST slot test or the parallel booking test. If drills went as planned, this is a highlight reel, not a stretch.

## Debugging & operations
*Depth: [`O` — 38 git/Docker/CI/ops questions](../04-question-vault/10-devops-git-docker-cicd.md) · [`H` — 38 HTTP/REST questions](../04-question-vault/07-http-rest-api-design.md) · [`B` — 24 spot-the-bug snippets](../04-question-vault/13-spot-the-bug.md)*

**44. ⭐ Endpoint slow in production — walk me through it.**
Sequence, not vibes: reproduce/scope (which endpoint, since when, all users?) → logs + timings to localize (app vs DB vs external) → if DB: EF SQL logging / EXPLAIN → classic culprits: N+1, missing index, early materialization → fix, measure again, add regression check. Your week-2/feature-9 stories slot right in.

**45. ⭐ Browser to server — what happens?**
90-second version, rehearsed: DNS → TCP + TLS handshake → HTTP request → proxy/LB → Kestrel → middleware → routing → your action → EF → SQL → response back out → render. Depth on request; breadth first.

**46. Structured logging — why, and what do you log?**
Named properties (`{TicketId}`) → searchable/aggregatable, vs string mush. Log: request summaries, domain events, warnings on rule violations, errors with context. Never: passwords, tokens, PII. Correlation ids for tracing a request across logs.

**47. How would you find out why the app crashed at 3am?**
Logs first (persisted where? platform logs) → last requests before death → unhandled exception? OOM? host restart? Mid answer admits the gap honestly: "my demo logs to console + platform capture; production I'd want centralized logs + alerts — Seq/ELK/App Insights."

**48. What's a health check / why?**
Endpoint reporting liveness/readiness (incl. DB reachability) for LBs and orchestrators to route/restart on. Trivial to add to Bookable — do it, then you "have used" them.

## Architecture & design judgment
*Depth: [`D` — 44 questions](../04-question-vault/09-architecture-patterns.md), incl. SOLID with real examples, the outbox in 30 seconds, CAP stated correctly, and the URL-shortener design (`D44`)*

**49. ⭐ Why DTOs everywhere in your API?**
(Checklist #2 — over-posting, contract stability, cycles, per-endpoint shaping. With the JSON-cycle war story.)

**50. Where do business rules live and why not the controller?**
Service/domain layer: controllers = transport (bind, call, map result). Rules in services are unit-testable without HTTP and reusable across endpoints/jobs. Cite the transition rules.

**51. When would you add caching; what's hard about it?**
When reads dominate and staleness is tolerable (provider list — yes; slot availability — dangerous, staleness = double-booking UX). Hard part: invalidation. IMemoryCache single-instance; Redis when instances multiply. Knowing when NOT to cache is the mid signal.

**52. Why a background worker for email instead of sending in the request?**
Latency decoupling + failure isolation + retries. Plus the outbox insight: write intent transactionally with the booking, or a crash between commit and send silently drops the email. Your feature 8 verbatim.

**53. Monolith vs microservices for a new product?**
Confident mid answer: monolith, modular internally — services add distributed-systems tax (network failures, versioned deploys, data consistency) that a small team/product can't amortize. "I'd split when a boundary proves itself (scaling or team friction), extracting around the seams my service layer already defines."

**54. How do you decide what to build vs cut when time-boxed?**
Take-home muscle: core happy path + correctness invariants + tests first; polish/optional features cut with a note in the README ("cut X because Y, would add by Z"). Explicit tradeoffs beat silent gaps.

## AI-tooling questions (increasingly common — have real answers)

**55. How do you use AI tools in your workflow?**
Honest and disciplined: "Completions while I drive; review passes before PRs; test scaffolding; drilling concepts. I keep authorship — everything merged I can explain and rebuild without it. My repo history and my ability to whiteboard any part of it back this up."

**56. When has AI been wrong and how did you catch it?**
Have a real example (you'll accumulate several — log them). Shape: plausible-looking suggestion → test/review caught it → what it taught you about verification.

---

## Coverage tracker

Drill sessions should hit every category weekly. Misses list: keep at the bottom of LEARNING-LOG; a question missed twice gets a 15-min deep-dive + re-quiz within 48h.

## When you're fluent here

Fluent = every ⭐ answered in under 90 seconds, spoken, with the tradeoff beat included, no reconstruction. Then move up a layer:

1. **Daily, 10 min:** [rapid-fire trivia](../04-question-vault/11-rapid-fire-trivia.md) — out loud, <3 sec per answer
2. **4×/week, 20 min:** two vault categories (rotation in [00-overview.md](00-overview.md))
3. **Weekly:** the [gotcha gauntlet](../04-question-vault/12-code-output-gotchas.md) + [spot-the-bug](../04-question-vault/13-spot-the-bug.md), narrated
4. **Weekly from week 7:** one full [mock screener script](../04-question-vault/14-mock-screener-scripts.md), timed, single sitting

The vault's numbering (`L12`, `E35`, `S17`…) is built for your miss-log: log the number, not "something about EF." Precise misses are cheap to re-drill; vague ones never get re-drilled at all.
