# Week 3 Guide — Auth, Roles, Testing

**Ships:** Rungs 6–7, Phase 1 exit criteria. **Concepts unlocked:** JWT vs sessions, claims, password hashing, unit vs integration testing, WebApplicationFactory.

## Day-by-day

### Mon — Rung 6, part 1 (identity basics)
- Concept deep-dive FIRST (auth is where blind copying hurts most): **"JWT vs session/cookie auth — how each works, tradeoffs, what's in a JWT, why the signature matters, where to store it client-side."** This is a guaranteed screener question.
- `User` gets `PasswordHash`. Register endpoint: hash with `BCrypt.Net` or ASP.NET Identity's `PasswordHasher<T>` (hasher only, skip full Identity — too much magic for week 3). Duplicate email → 409.
- Login endpoint: verify hash, issue JWT with `sub`, `email`, `role` claims, sane expiry (60 min).

### Tue — Rung 6, part 2 (locking it down)
- JWT bearer middleware config; decode one of your own tokens at jwt.io and identify every part — interviewers love "what's actually inside a JWT".
- `[Authorize]` on ticket/comment controllers; 401 vs 403 paths verified in `.http` (no token / wrong role).
- "Only assignee or Admin resolves": read the user id from claims in the service. Reporter is now taken from the token on create, not the request body — **spot why the old way was a vulnerability** (impersonation via over-posting; say it out loud, it's an interview answer).
- Signing key + connection string out of source: user-secrets locally. Confirm nothing sensitive in git history.

### Wed — Rung 7, part 1 (unit tests)
- Service-layer unit tests, real DB faked via EF InMemory/SQLite-in-memory (know the tradeoff: InMemory doesn't enforce relational behavior — that nuance is a mid-level flex in interviews).
- Cover: every legal status transition, every illegal one, resolve-permission matrix (assignee ✓, admin ✓, random agent ✗), comment-on-closed.
- Let Claude scaffold test class skeletons; **you write every assertion.** Naming: `MethodName_Scenario_ExpectedResult`.

### Thu — Rung 7, part 2 (integration tests)
- `WebApplicationFactory<Program>` + test auth handler (or issue real test JWTs). Concept deep-dive: **unit vs integration — what each catches that the other can't.** Have a one-liner ready: *"unit tests verify my logic, integration tests verify my wiring."*
- Cover: full CRUD happy path, filter+page envelope correctness, 401 without token, 403 wrong role, 404, 422 illegal transition.
- PRs #6–7 merged.

### Fri — Debugging drill #2 + full-project review
- Planted-bugs drill on the now-larger codebase.
- Then the Phase-1 boss fight: > "Review this ENTIRE repo as a hiring manager evaluating a mid-level take-home submission. Grade: code quality, REST design, EF usage, error handling, auth, tests. What would make you reject it? What's the single best improvement?"
- Fix what's fixable today; log the rest.

### Sat — Rebuild drill #3 + exit check
- From memory: register + login endpoints incl. hashing and token issuance, ≤45 min, AI off.
- Then the honest exit-criteria pass in [00-overview.md](00-overview.md). Anything unchecked → it's Monday's work before Phase 2 starts.

### Sun — Concept checklist closeout
- [05-concept-checklist.md](05-concept-checklist.md): teach-back on all 12. Under 10/12 solid → spend Monday drilling before Phase 2.

## Interview connection
Week 3 output maps 1:1 to the auth section of every screener: JWT anatomy, 401 vs 403, where tokens live, why hashing ≠ encryption, over-posting. And "how do you test your code" is answered with specifics — *"xUnit service tests for the transition rules, WebApplicationFactory integration tests for auth paths"* — instead of "um, I write tests."
