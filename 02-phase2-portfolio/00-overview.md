# Phase 2 — The Portfolio Project (Weeks 4–7)

**Output:** a deployed, CI-tested, README-documented product — **Bookable**, an appointment-booking system — containing one hard problem you can talk about for 10 minutes. This repo is what recruiters and interviewers will actually open.

## Files in this phase

| File | What |
|---|---|
| [01-project2-spec.md](01-project2-spec.md) | Full spec incl. the concurrency hard problem |
| [02-week4-5-guide.md](02-week4-5-guide.md) | Core domain, booking engine, the hard problem |
| [03-week6-7-guide.md](03-week6-7-guide.md) | Frontend slice, background jobs, CI/CD, deploy, README |
| [04-git-pr-workflow.md](04-git-pr-workflow.md) | Professional git habits + PR template |
| [05-ci-cd-deploy-guide.md](05-ci-cd-deploy-guide.md) | GitHub Actions + deployment walkthrough |

## What changes from Phase 1

- **Same 4-step loop, higher altitude.** You now also make *design* decisions (schema, API shape, where rules live) before implementation — write a 5-line design note in the PR description for anything non-obvious.
- **Workflow gets professional:** PR template, conventional-ish commits, CI must be green to merge. See [04-git-pr-workflow.md](04-git-pr-workflow.md).
- **Drills continue:** SQL (days 22–30, then the maintenance set), Friday planted bugs, Saturday rebuilds. **Sunday mocks start week 5** — see [../03-phase3-interviews/04-live-coding-drills.md](../03-phase3-interviews/04-live-coding-drills.md).
- **Applications go out at the end of week 6.** Not week 10. Resume guide: [../03-phase3-interviews/07-resume-and-applications.md](../03-phase3-interviews/07-resume-and-applications.md).

## Phase 2 concept checklist (the next 8)

Same teach-back bar as Phase 1's twelve:

13. **Transactions & isolation basics** — ACID; what a transaction protects against; read-committed as the default; when you'd reach for a transaction explicitly (multi-row invariants).
14. **Optimistic vs pessimistic concurrency ⭐** — your hard problem. Version column/`xmin` + retry vs locks; why optimistic fits web apps; the DB unique constraint as the last line of defense.
15. **Caching** — in-memory (`IMemoryCache`) for hot reads; invalidation as the real problem; when to say "Redis" (multiple instances need a shared cache) — and when it's overkill.
16. **Background jobs** — why not to do slow work in the request (email!); `BackgroundService`/hosted services + a queue (Channels); what happens to in-flight work on deploy (at-least-once thinking, idempotent handlers).
17. **Config & secrets** — appsettings layering, env vars, user-secrets; why secrets never enter git; what changes between dev and prod config.
18. **Browser to server** — DNS → TCP/TLS → HTTP request → reverse proxy → Kestrel → middleware → your code. The classic screener warm-up; rehearse a 90-second version.
19. **Indexes ⭐** — B-tree mental model; what to index (FKs, filter/sort columns); why not everything (write cost, storage); reading an EXPLAIN well enough to spot a seq scan. (SQL drills cover this hands-on.)
20. **REST design at scale** — resource naming, nesting depth, pagination as default, versioning strategies, partial updates. You'll defend these choices in the Bookable API review.

## Exit criteria

- [ ] Bookable deployed and reachable at a public URL; demo account seeded
- [ ] CI green on main: build + test on every PR
- [ ] The hard problem implemented WITH the concurrency test that proves it (spec §hard-problem)
- [ ] README with architecture diagram + "Decisions & tradeoffs" section
- [ ] 10-minute deep-dive story rehearsed twice (teach-back prompt), 90-second version too
- [ ] Concepts 13–20 ✅ · applications submitted to ≥10 roles
