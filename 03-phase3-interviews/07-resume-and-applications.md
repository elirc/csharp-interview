# Resume & Applications Guide

Goal: get into interview loops, at volume, starting **end of week 6** — not when you feel ready. Feeling ready is a trailing indicator.

## The resume — one page, projects up top

For a junior→mid jump without a long employment history, **projects section leads** (after a 2-line summary), then experience, then skills. Recruiters give it ~30 seconds; the Bookable bullet must land in the first 10.

### Project bullets — formula: what + stack + hard problem + proof

> **Bookable — appointment booking platform** · *ASP.NET Core, EF Core, PostgreSQL, React, GitHub Actions* · [live demo](URL) · [github](URL)
> - Designed and deployed a full-stack booking system preventing double-booking under concurrent load via optimistic concurrency + a Postgres exclusion constraint, verified by a parallel-request integration test (20 simultaneous requests → exactly 1 success)
> - Implemented transactional-outbox notifications with a background worker and retry, ensuring no lost sends on crash or deploy
> - CI pipeline running unit + integration tests against real Postgres on every PR; auto-deploy to Railway; ~40 tests

> **Ticket Tracker API** · *ASP.NET Core, EF Core, xUnit* · [github](URL)
> - REST API with JWT auth and role-based rules; eliminated an N+1 query found via SQL logging (21 queries → 1); integration-tested with WebApplicationFactory

Rules: every bullet has a *thing you did* + *how it's verifiable*. Numbers where honest (test counts, query counts, the 20→1). No "passionate," no "familiar with." Skills section: only what you can be interviewed on — it's a menu of questions you're ordering.

### The AI line
In skills or summary: `AI-augmented workflow: Claude Code / Codex for review, test scaffolding, and pairing`. It's 2026 — teams want this, and hiding it reads worse than owning it. Your rebuild-drill answer covers the skeptics (see story bank).

## Targeting — where a 10-week runway converts best

**Sweet spot:** product companies and agencies, ~10–200 eng, .NET shops, hiring "Software Engineer" / "Backend Engineer" / "Full-stack (C#)" with 2–4 yrs listed. Job-listing keywords that predict *practical* interviews (your strength): take-home, pair programming, "real-world problems," "no whiteboarding/LeetCode."
**Apply anyway when:** you meet ~60% of listed requirements (lists are wishlists) · they ask 3–5 yrs and you're at 1–2 with this portfolio.
**Lower priority:** big-tech LeetCode pipelines (wrong prep for this plan — a deliberate tradeoff, revisit after landing the mid role) · pure-frontend roles · staffing-agency reposts.

**Volume model:** applications are a funnel, not verdicts. Plan on double-digit applications per week; a handful of screens from every ~20 sends is a normal healthy rate, not failure. Track in a simple sheet: company, date, source, stage, notes. **Cadence: 10+/week, every week, from week 7 — no pausing to "fix the resume" more than once.**

**Warm channels beat cold (use all):** referrals from anyone you know at .NET shops (one referral ≈ 20 cold apps) · local .NET / user-group meetups (walking portfolio-talk material) · recruiters who specialize in .NET (they're paid to place you; the portfolio makes their pitch easy) · a short LinkedIn post walking through the double-booking fix with the repo linked — hiring managers do read those.

## Cover letter (when asked): 4 sentences

Who you are → the one Bookable sentence with the live link → one sentence mirroring THEIR stack/domain from the listing → available to talk. Done. Claude can draft variants; keep your voice, keep it under 120 words.

## Handling the experience question

"You're at N years — this role says 3." Prepared answer, not apology:
> "Right — and I'd point at the work rather than the years. I've shipped a deployed booking system that handles concurrent-write correctness with constraint-backed guarantees and a tested CI pipeline — the day-to-day of this role per your listing. Happy to pair on real code as part of the process; that's where I'll make the case."
Inviting the practical evaluation is the whole strategy of this plan — you've trained *exactly* that round.

## Weekly application ritual (Fri, from week 7)

1. 10 applications minimum (tracked)
2. Follow up on anything ≥1 week stale (one polite nudge, once)
3. Every screen/interview scheduled → same-day: research the company's product + stack, prep 3 questions for them
4. Every rejection with feedback → into LEARNING-LOG, feeds next week's drills. Rejections without feedback → data point, not verdict; the funnel math already priced them in.
