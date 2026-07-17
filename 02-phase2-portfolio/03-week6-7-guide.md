# Weeks 6–7 Guide — Jobs, Frontend, CI/CD, Deploy, Ship

**Ships:** Features 8–12; Bookable public; applications out.

## Week 6

### Mon — Feature 8 (background notifications)
- Concept deep-dive #16 first. The pattern you're building (outbox-lite): booking transaction writes a `Notification` row **in the same transaction** → `BackgroundService` polls Pending rows → "sends" (log-based fake sender) → marks Sent/Failed with AttemptCount, retry ≤3.
- Be able to say *why* this beats both alternatives: send-in-request (slow, failure-coupled) and fire-and-forget `Task.Run` (lost on crash/deploy). One sentence each. That comparison is a system-design-lite answer verbatim.
- Test: booking created → notification row exists (same transaction — kill the send and verify the booking still stands).

### Tue — Feature 9 (dashboard/reporting)
- One real reporting endpoint: today's bookings, week revenue, utilization %. Write it in LINQ, **read the generated SQL**, EXPLAIN it in DBeaver.
- Add the indexes the query wants (probably `(provider_id, start_utc)`); EXPLAIN again; put before/after plans in the PR. Concept #19 gets its ✅ here, hands-on.

### Wed–Thu — Feature 10 (React slice)
- Vite + React + fetch. Pages: login, provider list, provider detail w/ date picker + slots, book button, my-bookings + cancel. Plain CSS or one utility sheet. **Timebox: 2 days.** You're demonstrating "can work across the stack," not frontend artistry.
- Codex shines here — boilerplate-heavy, and frontend isn't your interview target. Still: the ownership quiz on anything you couldn't rewrite (fetch flow, state for slots, token handling).
- Handle the three states everywhere: loading, error, empty. That alone puts your frontend above most junior take-homes.

### Fri — Drill + review day
- Planted-bug drill (one bug in React land this time — stale state or a missing await on fetch).
- SQL maintenance set continues.

### Sat — Rebuild + resume draft
- Rebuild: the outbox write + poller loop skeleton from memory.
- Draft the resume using [../03-phase3-interviews/07-resume-and-applications.md](../03-phase3-interviews/07-resume-and-applications.md) — Bookable bullet first.

### Sun — Mock #2 (rapid-fire technical Q&A format)

## Week 7

### Mon — Feature 11: CI
- GitHub Actions per [05-ci-cd-deploy-guide.md](05-ci-cd-deploy-guide.md): build + test on PR, Postgres service container for integration tests. Branch protection: green-to-merge on main.

### Tue — Feature 11: deploy
- Deploy per the guide (Railway/Fly.io/Azure). Seed demo data incl. a demo login. Smoke-test the public URL from your phone. Put the URL in the repo description.

### Wed — Feature 12: README + docs (treat as a feature, it is one)
- README: what/why (2 sentences) → live URL + demo creds → architecture diagram (mermaid) → local setup that actually works from scratch → **Decisions & tradeoffs** linking `docs/decisions/*` → "what I'd do next."
- Add `002-notifications-outbox.md` decision doc. Two decision docs = a candidate who documents thinking — rarer than it should be.

### Thu — Polish pass with fresh eyes
- Hiring-manager review prompt on the whole repo (same as Phase 1 boss fight, mid-level bar). Fix the top findings only — **do not start new features.**
- Clone the repo to a clean directory and follow your own README setup. Every hiccup, fix the README.

### Fri — Applications out 🚀
- ≥10 applications today per the applications guide. Drill day still happens (bugs + SQL) but applications are the priority.

### Sat — Rebuild + exit criteria
- Rebuild: booking-create service method incl. both concurrency layers, from memory.
- Run the Phase 2 exit checklist in [00-overview.md](00-overview.md). Then rehearse the 10-minute Bookable deep-dive once, out loud, teach-back prompt grading.

### Sun — Mock #3 (system-design-lite: they'll ask you to design... a booking system 🙂 — do a URL shortener instead so it's not memorized)
