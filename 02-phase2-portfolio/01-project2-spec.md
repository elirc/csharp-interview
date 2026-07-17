# Project 2 Spec — "Bookable" (Appointment Booking System)

A booking platform for service providers (think: barbershop / clinic / tutoring). Chosen because the domain forces the best interview material in CRUD-land: **time, conflicts, and concurrency**. Every interviewer understands double-booking, and preventing it correctly is genuinely mid-level work.

**Repo:** `bookable` (public — this is the portfolio piece). Stack: ASP.NET Core API + EF Core + PostgreSQL + minimal React frontend + GitHub Actions + cloud deploy.

## Domain model

```
Provider     (Id, UserId → User, DisplayName, Bio, TimeZone)
Service      (Id, ProviderId, Name, DurationMinutes, PriceCents [int — money rule below])
Availability (Id, ProviderId, DayOfWeek, StartTime, EndTime)      -- weekly recurring windows
Booking      (Id, ServiceId, ProviderId, CustomerId → User,
              StartUtc, EndUtc, Status [Pending|Confirmed|Cancelled|Completed],
              CreatedAt, RowVersion)
User         (Id, Email, PasswordHash, DisplayName, Role [Customer|Provider|Admin])
Notification (Id, BookingId, Type, Status [Pending|Sent|Failed], AttemptCount, CreatedAt)
```

**Money rule:** store cents as integers. Never float/double for money — and be ready to say why (binary floats can't represent 0.1; rounding drift). Free interview point.

## Business rules
- Slots are derived: availability windows ÷ service duration, minus existing bookings. Slots are *computed*, never stored.
- A booking must fit entirely inside an availability window and **must not overlap** any non-cancelled booking for that provider. ← the invariant everything protects
- Customers cancel ≥24h before start; providers can cancel anytime (triggers notification).
- All times stored UTC; provider timezone used only at the edges (display, availability interpretation). Expect at least one painful timezone bug — it's a story, log it.

## The Hard Problem ⭐ (your interview centerpiece)

**Two customers request the last slot at the same moment. Exactly one succeeds.**

Naive code (check-then-insert) has a race: both requests read "slot free," both insert, double-booking. You will:

1. **Build the naive version first** and prove it fails: a test firing N parallel booking requests at one slot (`Task.WhenAll` against `WebApplicationFactory`) — assert >1 succeeded. **Commit this failing state on a branch — it's the "before" of your story.**
2. **Fix layer 1 — optimistic concurrency:** `RowVersion`/`xmin` concurrency token, catch `DbUpdateConcurrencyException`, retry-or-409.
3. **Fix layer 2 — DB constraint as backstop:** Postgres exclusion constraint (`EXCLUDE USING gist (provider_id WITH =, tstzrange(start_utc, end_utc) WITH &&)` where status ≠ cancelled) — or a serializable-transaction/unique-slot-key design if you choose discrete slots. **Know why the app-level check alone is never sufficient** (any check outside the DB is a race window; the constraint is the only true guarantee).
4. **Prove it:** same parallel test now asserts exactly 1 success and the rest get 409 + a clean error body.
5. **Write it up:** `docs/decisions/001-booking-concurrency.md` — the problem, both layers, why each exists, test evidence. This doc IS the 10-minute interview story.

## Feature ladder

| # | Feature | Notes |
|---|---|---|
| 1 | Auth + roles (port pattern from P1 — should take a day, that's the point of P1) | Customer/Provider registration |
| 2 | Provider profile + services CRUD | Provider-only, ownership enforced from claims |
| 3 | Availability windows CRUD | Validate: no overlapping windows, end > start |
| 4 | **Slot computation endpoint** `GET /api/providers/{id}/slots?serviceId=&date=` | Pure domain logic → unit-test heavy; timezone edges live here |
| 5 | Booking create (naive) + list (mine/provider's) | Commit the race-condition version on a branch |
| 6 | **The hard problem** (above) | Its own PR with the war-story description |
| 7 | Cancellation rules + status flow | 24h rule, role rules → 422s |
| 8 | Background notifications | `Notification` row (transactional outbox-lite) + `BackgroundService` polling → fake email sender w/ retry + AttemptCount. Explain why "send email inside the request" is wrong (latency, failure coupling) — and why writing the row transactionally beats fire-and-forget. |
| 9 | Provider dashboard query | Bookings today/this week, revenue sum, utilization % — one honest SQL/LINQ reporting query; EXPLAIN it, index what needs indexing, before/after in the PR |
| 10 | React frontend slice | Login, browse providers, pick slot, book, "my bookings" + cancel. Functional > pretty. Vite + fetch; no Redux, no component library rabbit holes |
| 11 | CI/CD + deploy | [05-ci-cd-deploy-guide.md](05-ci-cd-deploy-guide.md) |
| 12 | README + architecture diagram + decisions docs | The most-read file in the repo — treat as a feature |

## Explicitly out of scope
Payments (fake it: bookings just have a price) · multi-tenant orgs · websockets/live updates · k8s. Say no; ship.
