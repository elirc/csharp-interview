# Testing — 36 Questions (`T`)

Testing questions are a values interview disguised as a technical one. There's no single right answer to "how much do you mock" — there's a *defended position with tradeoffs*, and that's what's graded. Every answer here should sound like someone who has felt the pain, not read the pyramid diagram.

Log misses as `T##`.

---

## Strategy

**T1. ⭐ What's your testing strategy?**
Concrete beats theory: *"Unit tests on domain logic — slot computation, transition rules — because they're pure and fast. Integration tests through `WebApplicationFactory` against a real Postgres for anything involving routing, auth, serialization, or SQL, because those are the parts that actually break. And one concurrency test that spins up parallel bookings and proves the double-booking is impossible — that one is the reason I trust the feature."* Name real tests from your repo; the specificity *is* the answer.

**T2. ⭐ Unit vs integration test — where's your line?**
Unit: logic in isolation, no I/O, milliseconds, and a failure points at one function. Integration: the wiring — DI, middleware, model binding, EF's actual SQL, the constraint. The one-liner: **unit tests verify my logic, integration tests verify my wiring** — and most production bugs live in the wiring, which is why I don't over-invest in mocked unit tests of controllers.

**T3. The test pyramid — do you believe it?**
Have a nuanced take: the *shape* is right (fast tests dominate) but the classic pyramid predates cheap containers. With Testcontainers, an integration test costs seconds, and it tests the thing that actually breaks. I run a "trophy": a thick band of integration tests, a solid base of unit tests on pure logic, and a couple of end-to-end smoke tests. Rigid pyramid-quoting is a junior tell.

**T4. What makes code testable?**
Dependencies injected and interface-shaped; **logic separated from I/O**; no static/global state; no `DateTime.Now`/`Guid.NewGuid()` inline; small units. Cite the decision: extracting slot computation into a pure function so I could test DST boundaries without a database was a *testability* decision I made on purpose, and it also made the code better.

**T5. ⭐ How much do you mock?**
A stance with a reason: minimal. Mock at boundaries you don't own and can't run (the email provider, a payment API). Use the real thing where it's cheap (the database, in a container). Over-mocked tests assert *implementation* — they break on every refactor and pass while production burns. The classic failure is a test that mocks the repository, verifies `Save` was called, and proves nothing about whether the data persists.

**T6. Mock vs stub vs fake vs spy?**
Stub: returns canned data. Mock: a stub with **assertions about how it was called**. Fake: a real working implementation that's not production-grade (in-memory repo). Spy: records calls for later inspection. The distinction that matters in review: asserting on a mock's interactions couples the test to *how*, not *what*.

**T7. What do you do when a test needs the current time?**
Inject `TimeProvider` (.NET 8) or an `IClock`, and use `FakeTimeProvider` in tests. Never `DateTime.UtcNow` in domain code. Payoff you can state: it's how I test the "cancel ≥24h before start" rule at exactly 23h59m without a `Thread.Sleep`, and how I test DST transitions deterministically.

**T8. Is 100% coverage a good goal?**
No, and say why precisely: coverage measures **executed lines, not asserted behavior** — a test that calls everything and asserts nothing scores 100%. It's a decent tool for *finding* untested areas and a terrible target, because chasing the last 20% means testing getters and hitting else-branches that can't happen. I care that the risky paths are covered: money, auth, concurrency, the unhappy path.

**T9. Ever heard of mutation testing?**
Yes — Stryker.NET. It mutates your code (flips a `>` to `>=`, removes a line) and re-runs the suite; a surviving mutant means no test noticed. It measures whether your assertions actually *assert*, which is the thing coverage can't see. Nice-to-know that lands well; you don't need to have run it in CI.

---

## xUnit mechanics

**T10. ⭐ `Fact` vs `Theory`?**
`[Fact]` = one case. `[Theory]` + `[InlineData(...)]` = the same test over parameters — each row is a **separate test** with its own pass/fail, which beats a `foreach` inside one test (that stops at the first failure and hides the rest).

**T11. `InlineData` vs `MemberData` vs `ClassData`?**
InlineData for compile-time constants. MemberData for a static property/method returning `IEnumerable<object[]>` — needed for `DateTime`s, objects, computed sets. ClassData for a reusable data source class. The trigger is simple: InlineData can't hold anything that isn't a constant.

**T12. How does xUnit do setup/teardown?**
Constructor = setup, `IDisposable.Dispose`/`IAsyncLifetime` = teardown, and xUnit creates **a new instance of the test class per test** — that's the isolation guarantee, and it's why there's no `[SetUp]` attribute. Shared expensive state: `IClassFixture<T>` (once per class) or `ICollectionFixture<T>` (once across a collection).

**T13. What runs in parallel in xUnit?**
Test **collections** run in parallel; tests within a collection run sequentially. By default each class is its own collection. So two test classes hitting the same database can and will collide — which is why you either give each a schema/database or put them in one collection to serialize them. This is the answer to "why are my tests flaky in CI but fine locally."

**T14. `IAsyncLifetime`?**
Async setup/teardown (`InitializeAsync`/`DisposeAsync`) — what you need for starting a container or seeding a DB, since a constructor can't await.

**T15. How do you assert an exception?**
`await Assert.ThrowsAsync<DomainException>(() => sut.CancelAsync(id))`, then assert on the returned exception's message/properties. Point to make: assert the *type and the meaningful detail*, not the message string — messages change, contracts shouldn't.

**T16. How do you name a test?**
`Method_Scenario_ExpectedResult`: `Cancel_WhenLessThan24HoursBeforeStart_ThrowsDomainException`. The bar: a failing test name in CI output should tell you what broke **without opening the file**. That's the entire justification.

**T17. AAA?**
Arrange / Act / Assert, visually separated. One Act per test. If you need two Acts you probably have two tests — or you're testing a sequence, in which case say so in the name.

**T18. FluentAssertions vs `Assert`?**
Fluent reads better and gives far better failure messages on collections/objects (`should be equivalent to`, with a diff). `Assert` has no dependency. Note the licensing wrinkle if it comes up — FluentAssertions moved to a paid license for commercial use from v8, which pushed some teams to Shouldly or plain xUnit asserts. Knowing that is very current and lands as "pays attention."

---

## Integration testing

**T19. ⭐ What is `WebApplicationFactory`?**
It boots your real app in-memory — real DI, real middleware, real routing, real model binding — and gives you an `HttpClient` that talks to it **without a network socket**. So a test does `await client.PostAsJsonAsync("/bookings", dto)` and exercises the actual pipeline. It's the highest value-per-line testing tool in ASP.NET Core.

**T20. How do you swap a dependency in it?**
`WithWebHostBuilder(b => b.ConfigureTestServices(s => { s.RemoveAll<IEmailSender>(); s.AddSingleton<IEmailSender, FakeEmailSender>(); }))`. `ConfigureTestServices` runs **after** the app's own registration, which is the whole trick — it's why you can override without touching `Program.cs`.

**T21. Why does `WebApplicationFactory` need `Program` to be public?**
It needs the entry-point type as its generic argument. With top-level statements, `Program` is internal — hence either `public partial class Program { }` at the bottom of Program.cs or `InternalsVisibleTo`. Everyone hits this once; knowing it instantly reads as "has done this."

**T22. ⭐ Which database do you test against?**
The real engine, via **Testcontainers** — spin up Postgres in Docker per test run, migrate, seed. Reason: constraints, transactions, concurrency behavior, and provider-specific SQL are exactly what I want tested, and those are exactly what a fake doesn't have. My double-booking test is meaningless without a real database.

**T23. Why not the EF InMemory provider?**
It isn't relational: no constraints (so your unique index never fires), no real transactions, no raw SQL, no concurrency tokens. It will pass a test for the bug you most care about. Microsoft's own docs now steer people away from it. SQLite in-memory is closer but still differs (types, some SQL). Say this crisply — it's a well-known distinction and a clean way to show judgment.

**T24. How do you keep tests isolated when they share a database?**
Options: **transaction per test + rollback** (fast, but you can't test your own transactions), **Respawn** to wipe tables between tests (fast, realistic — my default), a fresh container per class (slow, bulletproof), or unique data per test (fragile as it grows). Name the tradeoff rather than the tool.

**T25. How do you test an authenticated endpoint?**
Either issue a real token against test signing keys (tests the real auth pipeline — my preference for a few key tests), or register a test authentication handler that injects a `ClaimsPrincipal` (fast, and fine for the other 50 tests where auth isn't the subject). The mistake is disabling auth in test config: then nobody ever tests the 401/403 paths, which are the ones that get you in the news.

**T26. ⭐ How would you test a race condition?**
Deterministically-ish: `Parallel.ForEachAsync` / `Task.WhenAll` firing N concurrent requests at the same slot through `WebApplicationFactory`, then assert **exactly one 201 and N-1 409s**, and assert the row count in the DB is 1. Say the honest part: it's probabilistic — it can pass on a slow machine even when broken — so I also assert the constraint exists, and I've watched it fail with the constraint removed. *That last clause is the credibility move: you've seen your own test fail for the right reason.*

---

## Practice

**T27. Test-first or test-after?**
An honest, defensible answer beats dogma: test-first when the shape is clear and the rules are the point (domain logic, bug fixes — a failing test that reproduces the bug first, always). Test-after when I'm exploring an API or the design is in flux. What I don't do is ship untested logic and promise to come back.

**T28. What's the first thing you do when a bug is reported?**
Write a failing test that reproduces it. It proves I understand the bug, it proves the fix, and it stops the regression forever. This is also the best answer to the behavioral "how do you handle bugs" question, because it's a process, not a vibe.

**T29. What's a flaky test and what causes them?**
A test that passes and fails without a code change. Causes: **time** (`DateTime.Now`, timeouts, sleeps), **shared state** between parallel tests, **ordering** dependencies, real network calls, randomness, and async not awaited. Policy: a flaky test is a broken test — fix it or delete it, never `[Retry]` it, because a suite people don't trust is worse than no suite.

**T30. Testing private methods?**
Don't. Test through the public API; a private method is an implementation detail, and testing it welds the test to the design. If it's complex enough to need its own tests, it's a class of its own asking to come out — that's the design feedback the urge is giving you.

**T31. What don't you test?**
Framework code, getters/setters, DTOs, EF's ability to map, third-party libraries. Testing that `List.Add` works is theater. I test **my** decisions.

**T32. What are you testing when you test a controller?**
Routing, binding, status codes, auth — which are wiring, so **integration**, not a mocked unit test. A controller with mocked services asserting "returns OkResult" tests the C# compiler. If a controller needs heavy unit tests, there's logic in it that belongs in a service.

**T33. Snapshot/approval testing?**
Verify (or similar): assert output against an approved file. Great for complex payloads, generated SQL, or error contracts where handwriting 40 assertions is madness. Risk: approving a wrong snapshot is one keystroke, so diffs need real review.

**T34. Where do tests fit in CI?**
Build + unit + integration on **every PR**, must be green to merge, containers spun up in the workflow. The rules: fail fast, no manual steps, and if it's flaky it blocks everyone — so flakes are P1. Then the honest addendum: I'd add coverage reporting as information, not as a gate.

**T35. Suite takes 20 minutes. What do you do?**
Measure first (which tests?). Then: parallelize collections, share containers per run instead of per class, replace slow end-to-end tests that duplicate cheaper coverage, split the suite (fast on PR, full on main). What I don't do is delete assertions to make it green faster.

**T36. Would you test a private constructor / 100% of a DTO to hit a coverage gate?**
No — and this is a values question. If the gate demands tests that assert nothing, the gate is wrong and I'd say so with the data. Writing fake tests to satisfy a metric is worse than the untested line: it *hides* the fact that it's untested.

---

## Coverage tracker

`T1–T9` strategy ⭐ · `T10–T18` xUnit mechanics · `T19–T26` integration ⭐ · `T27–T36` practice

**The four that decide the round:** T1 (your strategy, with real test names), T5 (mocking stance + why over-mocking fails), T23 (why not InMemory), T26 (how you test a race, including the honest limitation).
