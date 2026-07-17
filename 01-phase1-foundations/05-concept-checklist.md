# Phase 1 Concept Checklist — The 12 You Must Own

These are the highest-frequency .NET screener topics. For each: the bar is the **teach-back prompt** — explain it aloud to Claude-as-junior-teammate and get rated interview-ready. A ✅ requires that, not "I read about it."

Format below: the concept → the interview question it appears as → what a *mid-level* answer includes (the junior answer is always the first sentence only).

---

### 1. HTTP semantics
**Asked as:** "When would you return 422 vs 400? Is DELETE idempotent?"
**Mid answer includes:** 400 malformed/validation vs 422 semantically-invalid (or a defended house style); idempotency defined as "same effect if repeated" — GET/PUT/DELETE yes, POST no; 401 unauthenticated vs 403 unauthorized; 201+Location on create.

### 2. DTOs & the API boundary
**Asked as:** "Why not return your EF entities directly?"
**Mid answer includes:** over-posting/mass-assignment risk on input; API contract decoupled from schema (can migrate DB without breaking clients); serialization cycles with navigation properties; different shapes for different endpoints.

### 3. Dependency injection & lifetimes ⭐ (the single most-asked .NET question)
**Asked as:** "Transient vs scoped vs singleton — and what goes wrong if you misuse them?"
**Mid answer includes:** definitions (per-resolve / per-request / per-app); DbContext is scoped and *why* (unit-of-work per request, not thread-safe); the classic failure — scoped service injected into a singleton (captive dependency); DI enables testability via interface substitution.

### 4. Model binding & validation
**Asked as:** "What happens between the JSON hitting Kestrel and your action running?"
**Mid answer includes:** routing → binding (body/route/query) → validation → auto-400 via `[ApiController]`; where custom validation lives; validation errors as field-level ProblemDetails.

### 5. EF Core migrations
**Asked as:** "How do schema changes reach production?"
**Mid answer includes:** migration = generated diff (Up/Down) applied in order, tracked in `__EFMigrationsHistory`; generated SQL should be reviewed; migrate via CI step or startup (and the tradeoff); why `EnsureCreated` is not migrations.

### 6. Change tracking
**Asked as:** "What does SaveChanges actually save? When AsNoTracking?"
**Mid answer includes:** context tracks loaded entities, computes diffs on SaveChanges; `AsNoTracking` for read-only paths (faster, less memory); detached/attached states exist and update-without-load uses them.

### 7. N+1 queries ⭐
**Asked as:** "This endpoint is slow. Go." (also: "what's the most common ORM performance bug?")
**Mid answer includes:** definition (1 list query + N per-row queries); found via SQL logging/profiler; fixed via `Include` or — often better — projection (`Select` into DTO, fetch only needed columns); your own war story from week 2.

### 8. IQueryable & deferred execution
**Asked as:** "Difference between IQueryable and IEnumerable? When does the query run?"
**Mid answer includes:** IQueryable builds an expression translated to SQL at enumeration; composable Where/OrderBy/Skip/Take = one SQL statement; the footgun — `.ToList()` too early pulls the table into memory then filters in C#.

### 9. Middleware pipeline
**Asked as:** "Walk me through a request through ASP.NET Core."
**Mid answer includes:** ordered chain, each can short-circuit; canonical order (exception handler → HTTPS → routing → CORS → authN → authZ → endpoint); why exception middleware is first/outermost; where your custom middleware slots and why.

### 10. async/await in a web server ⭐
**Asked as:** "Why async controllers? Does async make the request faster?"
**Mid answer includes:** it's about *throughput*, not per-request speed — thread released back to the pool during I/O waits; sync-over-async (`.Result`) risks thread-pool starvation/deadlocks; async all the way down; CPU-bound work doesn't benefit.

### 11. JWT auth
**Asked as:** "How does the server know who's calling? Why can't the client forge a token?"
**Mid answer includes:** header.payload.signature; signature = HMAC/RSA over header+payload with server key → tamper-evident; stateless (no session store) with the tradeoff — hard to revoke before expiry, hence short expiry + refresh tokens; claims → authorization; storage tradeoffs (localStorage XSS vs cookie CSRF).

### 12. Unit vs integration testing
**Asked as:** "How do you test an API? What's your testing strategy?"
**Mid answer includes:** unit = logic in isolation (fast, precise failures); integration = wiring — routing, filters, auth, serialization, DB (WebApplicationFactory); the one-liner: *unit tests verify my logic, integration tests verify my wiring*; test behaviors not implementation; name tests as specs.

---

## Scoring yourself

| ✅ | Teach-back passed, rated interview-ready |
|---|---|
| 🟡 | Can explain with hesitation — re-drill within 3 days |
| ❌ | Can't explain without notes — concept deep-dive + rebuild something using it |

Track: `1[ ] 2[ ] 3[ ] 4[ ] 5[ ] 6[ ] 7[ ] 8[ ] 9[ ] 10[ ] 11[ ] 12[ ]`

**Phase 2 adds eight more** (transactions, concurrency, caching, background jobs, config/secrets, browser-to-server, indexes, REST design at scale) — listed in [../02-phase2-portfolio/00-overview.md](../02-phase2-portfolio/00-overview.md). Master these 12 first; they're the foundation the next 8 stand on.
