# Weeks 4–5 Guide — Core Domain + The Hard Problem

**Ships:** Features 1–7 of the [Bookable spec](01-project2-spec.md), including the concurrency centerpiece.

## Week 4 — foundations at speed, then the interesting part

### Mon — Feature 1–2 (auth + provider/services)
- This is Phase-1 material again on purpose. Target: both features in one day. If auth takes more than ~2 hours, that's signal — note it and drill the rebuild.
- New wrinkle: **ownership** — a provider edits only their own services. Enforce from claims in the service layer; add the 403 test.

### Tue — Feature 3 (availability windows)
- Overlap validation between windows is a warm-up for booking overlaps: half-open intervals `[start, end)`, two windows overlap iff `aStart < bEnd && bStart < aEnd`. Get this predicate right today with unit tests — you reuse it for the real invariant.

### Wed–Thu — Feature 4 (slot computation) — the domain-logic showpiece
- Pure function: `(windows, duration, existingBookings, date, timeZone) → slots`. No DB, no HTTP in the core — this is your best unit-testing material and a great "how do you make code testable" interview exhibit.
- Design note in PR: why slots are computed, not stored (stored slots = state to sync = bugs; computed = always consistent).
- Timezone discipline: compute in provider-local, return UTC. Test a DST transition date. **This test will probably fail the first time. Perfect — log the story.**
- Edge tests: window shorter than duration, booking mid-window splitting it, request for a past date.

### Fri — Drill day
- Planted-bug drill now runs on Bookable. New twist from this week: one of the 3 bugs should be a *domain* bug (wrong boundary in the overlap predicate, off-by-one in slot math).
- SQL drills: days 22–26.

### Sat — Rebuild
- From memory: the slot-computation core function + 3 of its unit tests. Pure logic rebuilds are the best kind — this one doubles as live-coding practice.

## Week 5 — the hard problem

### Mon — Feature 5 (naive booking)
- Check-then-insert, knowingly racy. Write the correctness tests that don't involve concurrency (fits in window, no overlap with *committed* bookings, 24h cancel rule can wait).
- **Branch `naive-booking` stays in history** — your before/after evidence.

### Tue — Prove the race
- The parallel test: N=20 simultaneous requests, one slot, `Task.WhenAll`. Assert on how many 201s came back. Watch it prove your code wrong. Screenshot/log the output — it goes in the decision doc.
- If the race won't reproduce (fast local DB), widen the window: `await Task.Delay(50)` between check and insert. Being able to *make* a race reproducible is itself a skill interviewers respect.

### Wed — Layer 1: optimistic concurrency
- Concept deep-dive first (#14): optimistic vs pessimistic, why optimistic suits web traffic (conflicts rare, retries cheap, no lock contention).
- Concurrency token on the contended row; catch `DbUpdateConcurrencyException` → 409 (retry only if you can re-validate). Parallel test improves but — key insight — **may still not be airtight depending on what you're versioning. Articulate why: two INSERTs don't conflict on a version column.** That's the argument for layer 2.

### Thu — Layer 2: the DB constraint
- Postgres exclusion constraint on `(provider_id, tstzrange(start_utc, end_utc))` for non-cancelled bookings (btree_gist extension; raw SQL in a migration — good, it shows you can leave the ORM when needed).
- Map the constraint violation to a clean 409. Parallel test: **exactly 1 success, 19 clean 409s.** Green. Commit.
- Write `docs/decisions/001-booking-concurrency.md` while it's fresh: problem → race proof → layer 1 → why insufficient alone → layer 2 → test evidence. This is the interview story; write it like you'll read it aloud, because you will.

### Fri — Feature 7 (cancellation) + drills
- 24h rule, role rules, status flow — an hour of work with the P1 patterns. Then Friday drill as usual.

### Sat — Rebuild
- From memory: check-then-act race explanation + the optimistic-concurrency handling code. If you can rebuild this cold, the 10-minute story is nearly rehearsed.

### Sun — First mock interview 🎤
- Format 1 (pair programming) from [../03-phase3-interviews/04-live-coding-drills.md](../03-phase3-interviews/04-live-coding-drills.md). Calendar it. Doing it tired/unready is the point — real interviews don't wait either.
