# Architecture, Patterns & Judgment — 44 Questions (`D`)

This is the mid-level differentiator. A junior knows how; a mid knows **why, and what it costs**. Nearly every question here is graded on whether you name the tradeoff without being asked — and on whether you resist reaching for the biggest tool.

The most dangerous failure mode in this round is *enthusiasm*: proposing microservices, Kafka, and event sourcing for a booking app tells the interviewer you'd do exactly that to their codebase.

Log misses as `D##`.

---

## SOLID & OO design

**D1. ⭐ SOLID — but with real examples, not the words.**
**S**ingle responsibility: one reason to change — my `BookingService` doesn't format emails. **O**pen/closed: extend without editing — new notification channels implement an interface instead of adding a `case` to a switch. **L**iskov: a subtype must honor the base's contract — a `ReadOnlyRepository` that throws on `Add` violates it and will break callers. **I**nterface segregation: small interfaces — `IEmailSender`, not `INotificationEverything`, so tests don't implement six unused methods. **D**ependency inversion: depend on abstractions — my service takes `IClock`, not `DateTime.Now`. Reciting the acronym is a junior answer; one concrete example each is a mid answer.

**D2. Is SRP just "small classes"?**
No — and this is the good follow-up. It's "one **reason to change**," i.e. one stakeholder/axis. A 300-line class serving exactly one concern can be fine; a 20-line class that formats currency *and* talks to the DB has two. Splitting by line count gets you a hundred anemic classes and worse cohesion.

**D3. Give me a real Liskov violation.**
The classic: `Square : Rectangle` — set width on a Square and height changes too, so any code that assumes `SetWidth` leaves height alone breaks. Realistic version: an implementation that throws `NotSupportedException` for part of the interface, so callers must type-check to know if the method works. If callers need `if (x is RealOne)`, the abstraction is a lie.

**D4. Composition over inheritance — why?**
Inheritance couples you to a base's implementation forever, is single-slot, and breaks encapsulation (the fragile base class problem: a base change silently breaks derived types). Composition swaps behavior at runtime and is testable. Rule: inheritance for genuine "is-a with substitutability," composition for "has-a" or "uses-a" — which is nearly everything.

**D5. DI vs IoC vs DIP?**
DIP is the **principle** (depend on abstractions). IoC is the **pattern** (something else controls construction/flow). DI is the **technique** (pass dependencies in). A container is just a tool that automates DI. People use all four words interchangeably; distinguishing them is a nice precision signal.

**D6. Design patterns you've actually used?**
Answer honestly with the ones that show up in real .NET, not a catalog recital: **Strategy** (`IEnumerable<IValidator>` injected and looped), **Decorator** (a caching or logging wrapper over a service, registered in DI), **Factory** (`IHttpClientFactory`, `IDbContextFactory`), **Options/Builder** (`services.AddX(o => ...)`), **Adapter** (wrapping a third-party SDK behind my interface so it's mockable and swappable), **Mediator** (MediatR, with opinions). The best answer names patterns already in the framework — it shows you *recognize* them rather than perform them.

**D7. Singleton — the pattern vs the lifetime?**
The GoF pattern (a static `Instance` property, private ctor) is essentially an anti-pattern now: global state, untestable, hidden dependency. A **singleton lifetime in DI** is fine and different — it's still injected, still an interface, still substitutable in tests; the container just hands out one instance. Distinguishing these is a genuinely good answer.

**D8. What's a code smell you fix on sight?**
Pick a real one and defend it: a boolean parameter (`Save(true)` — unreadable at the call site, and it's usually two methods), primitive obsession (`string email` everywhere instead of an `Email` type that can't be constructed invalid), a method with five parameters, nested ternaries, or `catch (Exception) { }`. One conviction beats a list.

---

## Structure & layering

**D9. ⭐ How do you structure a project?**
Have a shape and defend it: API (controllers, DTOs, mapping) → Application/Services (use cases, orchestration, transactions) → Domain (entities, rules, invariants) → Infrastructure (EF, HTTP clients, email). **Dependencies point inward**: the domain knows nothing about EF or HTTP, which is what makes it unit-testable without a database. Then the honest bit: *"for a project of my size that's four projects and it's slightly ceremonial — but the layering discipline is what let me test slot logic in isolation."*

**D10. Layers vs vertical slices?**
Layers organize by *technical* concern; you touch four folders to add one feature. Vertical slices organize by *feature* (all of `Bookings/Create` in one folder) — higher cohesion, less indirection, easier deletion. My take: slices scale better with team size and feature count; layers are more familiar. Either is fine; **mixing both badly** is the actual problem.

**D11. ⭐ Where do business rules live, and why not the controller?**
Domain/service layer. Controllers are transport: bind, call, map the result to a status code. Rules in services are unit-testable without HTTP, reusable from a background job or a CLI, and don't get duplicated when a second endpoint needs them. The tell that you got it wrong: your rule is untestable without spinning up a web server.

**D12. What's an anemic domain model, and is it always wrong?**
Entities as property bags with all logic in services. Purists call it an anti-pattern (it's procedural code with objects as records). Pragmatic answer: for CRUD it's fine and honest; it becomes a problem when **invariants get scattered** — three services each remembering to check the same rule, and the fourth one forgets. The moment I have real invariants, I push them into the entity so they can't be violated.

**D13. ⭐ Repository pattern — yes or no?**
An opinion with tradeoffs, always: `DbContext` **is already** a unit of work and `DbSet<T>` **is already** a repository, so a thin passthrough repo adds ceremony and usually leaks `IQueryable` anyway (which defeats the abstraction it claims to provide). I used services over `DbContext` directly in Bookable. I'd add repositories when I need a genuine seam — swapping persistence, a team convention, or hiding a gnarly query behind a domain-shaped method like `FindOverlapping(providerId, window)`. That last one is real value; `GetById` is not.

**D14. CQRS — what is it really, and would you use it?**
At minimum: **separate the read model from the write model** — reads use projections/DTOs, writes go through the domain. That's cheap and I already do it. The heavy version (separate databases, eventual consistency, a sync pipeline) is for genuinely asymmetric scale, and it buys you a whole consistency problem. Say the distinction; people conflate "CQRS" with "MediatR" and both with "microservices."

**D15. MediatR — worth it?**
A defensible opinion either way, and this is a real .NET culture question. For: consistent handler shape, pipeline behaviors are a clean place for validation/logging/transactions, decouples controllers from services. Against: indirection (F12 goes nowhere), a "handler" per action is often a one-line wrapper, and it's not the mediator pattern doing any real work. Note the currency: **MediatR moved to a commercial license in 2025**, which pushed teams to alternatives or to just... calling services. Knowing that is a nice, very current signal.

**D16. What's an aggregate (DDD)?**
A cluster of objects with one **root** as the only entry point, forming a consistency boundary: the root enforces the invariants, and you save the whole thing atomically. `Booking` is a root; its `BookingLines` aren't reachable except through it. The practical payoff: the aggregate boundary is your **transaction boundary** — one aggregate per transaction is the rule of thumb, and needing two usually means the boundary is drawn wrong.

**D17. Entity vs value object?**
Entity: identity that persists through change (Booking #42 is the same booking after you reschedule it). Value object: defined **entirely by its values**, immutable, interchangeable (`Money(10, "USD")`, a `DateRange`). C# `record` was made for value objects; EF maps them as owned/complex types. Modeling `DateRange` as a value object with the overlap logic *inside it* is exactly how you make your booking rules testable.

**D18. Do you need a domain event?**
"When a booking is confirmed, send an email and update the dashboard." Domain events decouple the rule from the reactions. But the honest caveat: an in-memory event that fires **before** commit can act on data that then rolls back; firing **after** commit can be lost on a crash. That's precisely why the outbox exists — the event goes in the same transaction. You've built this; say it that way.

---

## Distributed & reliability patterns

**D19. ⭐ The outbox pattern — explain it in 30 seconds.**
You can't atomically write to your database *and* send an email/message — two systems, no shared transaction, so a crash between them either loses the email or sends one for a booking that rolled back (the **dual-write problem**). Fix: write the booking **and a row describing the intent** in one transaction. A background worker polls unsent rows, sends, marks sent. You've converted an impossible atomic-two-systems problem into an ordinary single-database transaction. Consequence to volunteer: delivery becomes **at-least-once**, so the consumer must be idempotent.

**D20. Why is at-least-once the price?**
The worker can send and then crash before marking the row sent — so on restart it sends again. To make it at-most-once you'd mark first, and then a crash loses it. **You must pick which failure you prefer**, and for email/notifications duplicate beats missing. Hence idempotent handlers. This "pick your failure" framing is the mid-level insight.

**D21. How would you scale the outbox poller past one instance?**
`SELECT ... FOR UPDATE SKIP LOCKED LIMIT n` — each worker claims a disjoint batch, no coordination, no distributed lock. (SQL Server: `READPAST`.) Alternatively a leader election or a partition key per worker. Being able to answer "how does this scale" about your *own* project with a specific SQL clause is a standout moment.

**D22. Idempotency — how do you make an operation safe to repeat?**
A natural key + a unique constraint (the same shape as everything else you've built): an idempotency key on the request, a message id on the consumer, `ON CONFLICT DO NOTHING`. The pattern generalizes: **make the repeat collide with a constraint, and let the database be the arbiter.**

**D23. What's a saga, in one sentence — and would you use one?**
A long-running, multi-service transaction implemented as a sequence of local transactions with **compensating actions** on failure (no distributed rollback — you refund rather than un-charge). Honest: I haven't needed one; a monolith with a real transaction is simpler, and I'd only reach for a saga once the steps genuinely live in different services with different databases.

**D24. Event sourcing?**
Store the events, derive current state by replaying. Buys: a perfect audit log, time travel, and rebuildable projections. Costs: everything — versioning old events forever, snapshots for performance, no ad-hoc SQL against current state, and a much steeper team learning curve. My take: it's a specialist tool for audit-critical domains, not a default. Knowing why *not* is the answer here.

**D25. Two-phase commit — why don't we use it?**
It's blocking: if the coordinator dies after "prepare," participants hold locks indefinitely. It couples availability across all participants (any one down = nobody commits). The industry answer became: avoid distributed transactions, use outbox + idempotency + eventual consistency. Nice depth if it comes up; don't volunteer it.

**D26. CAP theorem — say it correctly.**
When a **network partition** occurs, you must choose availability or consistency — that's it. It says nothing about the normal case, and "pick two" is a misleading pop version (you don't get to *choose* P; the network chooses for you). Practical translation: a synchronous replica is CP (unavailable when the replica is unreachable), an async read replica is AP (available, possibly stale). Say the precise version and stop; CAP is where people love to show off.

**D27. Eventual consistency — where would you accept it?**
Where staleness is invisible or harmless: dashboards, search indexes, counts, notification delivery, reports. Where I wouldn't: availability of a slot at booking time, account balance at withdrawal, anything with a uniqueness invariant. The skill is drawing that line per-feature rather than per-app.

**D28. Retry, timeout, circuit breaker — how do they fit together?**
One thought: **timeout** bounds the wait so a slow dependency can't consume your threads; **retry** (exponential backoff + **jitter**, only for idempotent ops) survives a blip; **circuit breaker** stops retrying once it's clearly down, failing fast so you don't queue up a thundering herd for their recovery. Without a timeout, the others are decoration. Retrying non-idempotent operations is how you double-charge.

**D29. What's a thundering herd / retry storm?**
Everyone retries at the same instant (or every cache key expires at once), and the recovering service is knocked over by the retry wave — an outage that *sustains itself*. Jitter, backoff, circuit breakers, and TTL randomization exist for exactly this. It's the systems version of "the fix caused the outage."

**D30. Bulkhead?**
Isolate resource pools so one failing dependency can't consume everything — a separate connection/concurrency limit per downstream, so the flaky reporting API can't starve the booking path. Named after ship compartments: the point is that the flood stays local.

---

## Caching & performance

**D31. ⭐ When do you add a cache, and what's hard about it?**
When reads dominate, the data is expensive to compute, and **staleness is tolerable** — a provider list, yes. Slot availability, no: stale means showing a slot that's gone, which is worse than slow. The hard part is invalidation and the fact that a cache **hides** the underlying problem — I'd add the index first and cache only what stays slow. Knowing when *not* to cache is the mid signal.

**D32. Cache-aside vs write-through vs write-behind?**
Cache-aside (lazy): read → miss → load → populate. Simple, the default, but every miss hits the DB and the first hit is slow. Write-through: write to cache and DB together — consistent, slower writes. Write-behind: write to cache, flush later — fast, and you can **lose data** on a crash. Cache-aside covers 95% of real apps.

**D33. `IMemoryCache` vs Redis — when does it switch?**
The instant there's more than one instance: in-memory caches diverge per node, so users get different answers depending on which server they hit, and invalidation only clears one node's copy. Redis when you need a shared cache, cross-instance invalidation, or to survive a restart. `HybridCache` (.NET 9) does L1+L2 with stampede protection. Also worth saying: in-memory is free and fine for a single instance, and premature Redis is real complexity for zero benefit.

**D34. What are the two hard problems?**
"Cache invalidation and naming things" — and the joke's third item, off-by-one errors. Say it, then say the serious version: invalidation is hard because the cache doesn't know when the truth changed, so every strategy is a guess (TTL guesses time, events guess you'll never miss one).

**D35. p50 vs p95 vs p99 — why not averages?**
An average hides the tail: 99 requests at 10ms and one at 10s averages 110ms and looks fine, while a real user waited 10 seconds. p99 is the experience of your **most engaged** users (more requests = more chances to hit the tail) and the thing that pages you. And p99 is where GC pauses, cold caches, and lock contention show up — the average never sees them.

**D36. First bottleneck in a typical CRUD app?**
The database, essentially always — and specifically N+1, missing indexes, and unbounded queries, not raw hardware. Order of moves: measure → fix the query → index → cache → read replica → *then* consider architecture. The mid-level answer explicitly refuses to start at the end.

---

## Judgment

**D37. ⭐ Monolith or microservices for a new product?**
Monolith, modular inside. Reason: services add a **distributed-systems tax** you pay from day one — network failures, versioned contracts, no cross-service transactions, distributed tracing, N deploy pipelines — and a small team amortizes none of it. I'd split when a boundary proves itself: a component with a genuinely different scaling profile, or a team that's blocked on someone else's deploy. And I'd extract around the seams my service layer already defines. **Confidence here is the signal**; the wrong answer isn't "microservices," it's not being able to name the tax.

**D38. Then how do you keep a monolith from rotting?**
Enforce boundaries with something other than good intentions: separate projects/assemblies per module with explicit references (and an architecture test — NetArchTest/ArchUnitNET — failing the build when the domain references EF), no shared "Common" dumping ground, and no cross-module DB access — go through the module's public API. A modular monolith is a microservices architecture with the network removed, which is the *good* half.

**D39. ⭐ How do you decide what to build vs cut under a deadline?**
Core happy path + the correctness invariants + tests, first. Cut polish, optional features, and gold-plating — **with an explicit note**: "cut X because Y, would add it by Z." Explicit tradeoffs read as engineering; silent gaps read as sloppiness. This is take-home muscle and the answer to half of the behavioral round.

**D40. When would you take on technical debt deliberately?**
When the deadline is real and the debt is **contained and reversible** — a hardcoded rule behind an interface, a manual step with a runbook. What I won't cut: data model mistakes (they migrate badly and forever), security, and correctness invariants — those aren't debt, they're a bug with a payment plan. The framing: debt is fine when you know the interest rate and where you wrote it down.

**D41. A senior wants a pattern you think is overkill. What do you do?**
Ask what problem it solves and what it costs, in that order and genuinely — often they know something I don't (a plan I can't see, a scar I don't have). If it's still overkill, I'd make my case once with the specific cost ("this adds three files per endpoint and we have two developers"), then commit to the decision either way and not relitigate it in code review. Disagree-and-commit, plus the humility to be persuaded, is exactly what's being measured.

**D42. How do you approach a codebase you've never seen?**
Run it first (getting it running teaches you the dependencies). Then trace **one** request end to end — the map of the whole app. Read the tests to learn the intended behavior. Look at recent commits to see where the churn is. Make the smallest useful change and put a PR up early. **What I don't do is read it top to bottom** or refactor before I understand why it's shaped that way — Chesterton's fence.

**D43. Feature flags — what are they for and what's the catch?**
Decouple deploy from release: ship dark, enable for 1%, kill instantly without a rollback. Catch: every flag is a branch → 2^n untested combinations, and flags that never get removed become permanent, undocumented config. Policy: a removal date at creation.

**D44. ⭐ Design a URL shortener. Ninety seconds.**
The rehearsal target, because it's the most-asked design-lite prompt: table `(code PK, target_url, created_at, owner_id)` · `POST /links` returns the code, `GET /{code}` → **301/302 to target**. The hard part is **code generation under concurrency**: random base62 + a unique constraint + retry on collision (a check-then-act pre-check races), or pre-generate a pool, or encode a sequence id (short but enumerable — which leaks your growth rate and lets people crawl every link). The read path is ~100:1 reads, all by primary key, and hot links repeat — so cache the code→URL map (immutable, so invalidation is trivial: the *easy* cache). Scale: this shards perfectly by code; the DB is one indexed lookup; a CDN/cache absorbs the hot tail. Note the pattern — **uniqueness under concurrency, solved with a constraint** — is the same shape as your booking overlap. Say that out loud; it shows you've generalized your own experience rather than memorized one story.

---

## Coverage tracker

`D1–D8` SOLID/OO · `D9–D18` structure/DDD ⭐ · `D19–D30` distributed ⭐ · `D31–D36` caching/perf · `D37–D44` judgment ⭐

**The five that decide the round:** D37 (monolith, with the tax named), D19 (outbox in 30 seconds — your feature 8), D13 (repository opinion with both sides), D31 (when *not* to cache), D39 (build vs cut, with the explicit-note habit).

**The meta-rule for this whole file:** every answer ends with a cost. If you describe a pattern and haven't said what it takes away, you've given a junior answer to a mid-level question.
