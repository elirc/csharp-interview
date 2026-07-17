# Junior → Mid-Level SWE Fast-Track Plan
**Stack:** C# / ASP.NET Core / EF Core / SQL / React (light) · **Tools:** VS Code, Claude Code, Codex · **Timeline:** 10 weeks aggressive (15–20 hrs/wk) · **Goal:** pass practical interviews for mid-level CRUD web dev roles

## TL;DR

- **The engine is a 4-step AI loop, not AI generation:** attempt each feature yourself first (45-min cap), use Claude Code as a strict reviewer/quizzer rather than an author, use Codex only for completions you could have typed yourself, and rebuild one feature from memory every Saturday with AI off.
- **Two projects instead of courses:** a ticket-tracker API (weeks 1–3) whose feature ladder forces out every classic screener topic, then a real product with one deliberately hard problem — e.g. concurrent booking slots — plus CI, deployment, and a decisions-focused README (weeks 4–7). The hard problem becomes the 10-minute interview deep-dive story.
- **Interview training targets the 4 practical formats:** timed take-home simulations (AI-generated and AI-graded against a mid-level rubric), pair-programming mocks with narration practice, rapid-fire technical Q&A, and system-design-lite with a repeatable script.
- **Daily SQL drills and a STAR story bank are non-negotiable** — raw SQL and vague behavioral answers are the two most common junior→mid rejection reasons.
- **Applications start at week 6–7,** before "ready" — real interviews are the best study guide.
- **Two highest-leverage drills:** the Friday "plant 3 bugs and let me find them" debugging exercise, and the Saturday from-memory rebuild.

---

---

## What "mid-level" actually means to an interviewer

Interviewers don't test juniors vs. mids on syntax. They test on these five signals — everything in this plan feeds one of them:

1. **Can ship a feature end-to-end unsupervised** — DB migration → API endpoint → validation → tests → PR.
2. **Debugs systematically** — reads stack traces, reproduces bugs, uses logs/breakpoints, doesn't guess.
3. **Knows why, not just how** — can explain why async/await, why an index, why DTOs instead of entities.
4. **Writes reviewable code** — small PRs, meaningful names, tests, handles the unhappy path.
5. **Talks about tradeoffs** — "I chose X over Y because…" in every design conversation.

---

## The AI learning loop (the whole trick)

Using Claude Code/Codex to *generate* code makes you faster but not better. Use this loop instead — it's the difference between a prompt operator and a mid-level engineer:

### The 4-step loop (apply to every feature you build)
1. **Attempt first (30–45 min cap).** Write the feature yourself. Struggle is the learning; the cap keeps you moving.
2. **AI as reviewer, not author.** `claude "review this code like a senior engineer in a strict PR review. Find bugs, missing edge cases, and naming problems. Don't rewrite it — ask me questions."`
3. **AI as explainer.** For anything you copied or didn't fully write: `"explain this line by line, then quiz me with 3 questions about what breaks if I change X."` If you can't answer the quiz, you don't own the code.
4. **Rebuild from memory once per week.** Pick one feature you built with AI help and rewrite it in a blank file with AI *off*. This is the retention step almost everyone skips — it's also exactly what a live interview simulates.

### Division of labor between the two tools
- **Claude Code (terminal, agentic):** code review, debugging sessions, mock interviews, explaining unfamiliar code, scaffolding tests, generating realistic bug scenarios for you to fix.
- **Codex (inline in VS Code):** autocomplete while *you* drive. Rule: never accept a multi-line completion you couldn't have typed yourself. If you could — accept it, speed is fine.

### Prompts that build skill (keep these on a sticky note)
- "Don't give me the answer. Give me a hint and ask what I'd try next."
- "Break this feature into subtasks a junior could do, then let me implement each one and review my work."
- "Introduce 3 realistic bugs into this working code and let me find them." *(best debugging drill that exists)*
- "I'm going to explain how this works. Correct me only where I'm wrong."
- "Interview me for 20 minutes about this project as a skeptical senior engineer."

---

## Phase 1 — Foundations under a real project (Weeks 1–3)

Don't do a "C# fundamentals" course. Build **Project 1** and let the gaps surface; fill each gap the day you hit it.

**Project 1: Task/Ticket Tracker API** (the interview-classic domain)
ASP.NET Core Web API + EF Core + SQL Server or Postgres + xUnit.

Feature ladder — each one is a git branch and a self-reviewed PR:
1. CRUD endpoints for `Ticket` (proper REST verbs, status codes, route design)
2. DTOs + validation (FluentValidation or data annotations) — never expose entities
3. EF Core: migrations, relationships (Ticket→User, Ticket→Comments), eager vs lazy loading
4. Filtering, sorting, pagination on the list endpoint (this is asked in ~every take-home)
5. Global error handling middleware + structured logging (Serilog)
6. Auth: JWT bearer, register/login, `[Authorize]`, roles
7. Unit tests for services, integration tests with `WebApplicationFactory`

**Concepts to be able to *explain* by end of Phase 1** (drill each with the AI quiz prompt):
async/await and why it matters in a web server · dependency injection lifetimes (transient/scoped/singleton — a favorite interview question) · what a migration actually does · N+1 queries and how to spot them · why DTOs · HTTP semantics (idempotency, status codes, PUT vs PATCH).

**SQL in parallel (30 min/day, non-negotiable):** raw SQL, not just EF. JOINs, GROUP BY/HAVING, indexes, EXPLAIN. Practice on [pgexercises.com](https://pgexercises.com) or SQLBolt. Mid-level interviews *always* have a SQL question, and it's the most common junior fail.

---

## Phase 2 — The portfolio project + professional workflow (Weeks 4–7)

**Project 2: A real product, not a toy.** Pick one with inherent complexity — booking system (time slots, conflicts), inventory/order system (stock, transactions), or expense/invoice app (money, reporting). It needs:

- ASP.NET Core API + small React or Razor frontend (enough to demo, not a frontend showcase)
- One **hard problem** you can talk about for 10 minutes in interviews: concurrency on bookings (optimistic concurrency / unique constraint), money handling, a reporting query that needed optimization, background job (email on order placed) — pick one and go deep
- CI on GitHub Actions (build + test on PR), deployed somewhere live (Azure App Service free tier, Railway, or Fly.io)
- README with an architecture diagram and a "decisions and tradeoffs" section — interviewers read this

**Workflow rules (this is where "works like a mid" is built):**
- Everything through branches + PRs, even solo. Write real PR descriptions.
- Before merging, run `claude "/code-review"` style review, fix findings, *then* have Claude quiz you on why each finding mattered.
- Weekly debugging drill: have Claude plant 3 bugs in a branch (`"introduce 3 subtle bugs: one logic, one async/concurrency, one data/EF"`), find them using only debugger + logs, no AI help.
- Weekly from-memory rebuild (loop step 4).

**Phase 2 concepts:** transactions and isolation basics · optimistic vs pessimistic concurrency · caching (in-memory, and when to say "Redis" in an interview) · background jobs (hosted services) · secrets/config management · what happens between browser and server (DNS→TLS→request pipeline — a classic screener question).

---

## Phase 3 — Interview conversion (Weeks 8–10, overlapping job applications)

Start applying at **week 6–7**. Interviews are the best study guide; don't wait until "ready."

### The four practical interview formats and how to train each

**1. Take-home assignment** (most common for CRUD roles)
- Do 2 full timed practice runs: `claude "give me a realistic 4-hour take-home for a mid-level .NET role: an API with auth, filtering, and tests, with intentionally ambiguous requirements"`. Build it in 4 hours, then have Claude grade it against a mid-level rubric.
- What graders actually look for: tests exist, errors handled, README explains setup + decisions, commits tell a story, no giant single commit, unhappy paths covered. Not cleverness.

**2. Live coding / pair programming** (practical, not LeetCode — "add an endpoint to this codebase," "fix this bug," "implement this small feature while talking")
- Train **thinking aloud**: 3×/week, 45-min mock: `claude "act as a pair-programming interviewer. Give me a small feature to add to an existing ASP.NET API. Interrupt me with questions. Evaluate my communication at the end."`
- Drill the muscle memory that dies under pressure: write a controller + service + test from a blank file, no autocomplete, until it's boring. (This is what the weekly rebuild trains.)
- The evaluated skill is narration: "First I'll check how existing endpoints handle validation so I'm consistent…" — say the plan before typing.

**3. Technical Q&A / screener call**
Build a question bank and drill it conversationally: `claude "rapid-fire interview me on ASP.NET Core, EF Core, SQL, HTTP, and DI. Push back on shallow answers with 'why?'"`.
Must-have topics: DI lifetimes · async/await internals (roughly) · middleware pipeline · EF change tracking + N+1 · indexes and when they hurt · JWT vs session auth · REST design · transactions · unit vs integration tests · "how would you find a slow endpoint in production?"

**4. System design lite** (mid-level version: "design a URL shortener / booking API," expecting sane schema + API + one scaling idea, not distributed-systems theater)
- Learn one repeatable script: requirements → data model → API endpoints → the one hard part (concurrency, uniqueness, scale read path) → "at higher load I'd add caching / a queue."
- 4–5 mock sessions with Claude playing interviewer, whiteboarding in a markdown file.

### Behavioral + story bank (don't skip — this fails more candidates than tech)
Write 6 STAR stories from your projects: hardest bug, a tradeoff decision, receiving critical feedback, a deadline call, a disagreement, something you'd do differently. Have Claude interview you on them and attack vague answers. Your Project 2 "hard problem" is the anchor story — rehearse the 10-minute deep-dive version and the 90-second version.

### Resume/positioning
- Frame projects with outcome + tech + hard problem: "Built a booking API (ASP.NET Core, Postgres) handling concurrent slot reservations via optimistic concurrency; 85% test coverage, CI/CD to Azure."
- Being effective *with* AI tools is now a real selling point — mention your workflow (AI-assisted review, test generation) when asked about productivity, while making clear you own the code.

---

## Weekly operating cadence (all phases)

| Day | Focus |
|---|---|
| Mon–Thu | Feature work on current project via the 4-step loop (60–90 min blocks) |
| Daily | 30 min SQL drills |
| Fri | Debugging drill (planted bugs) + PR self-review with AI |
| Sat | From-memory rebuild of one feature, AI off |
| Sun (from wk 5) | One mock interview (rotate the 4 formats) + review notes |

**Keep a `LEARNING-LOG.md`**: one line per day — what you built, what broke, what you'd explain differently now. By interview time it's your story bank raw material.

---

## Milestones / exit criteria

- **Week 3:** Project 1 API complete with tests; can explain every line; passes a Claude-run "junior→mid" code review with no major findings.
- **Week 5:** Comfortable writing a CRUD endpoint + test from blank file with no AI, while narrating.
- **Week 7:** Project 2 deployed with CI, README, and one rehearsed deep-dive story. Applications out.
- **Week 9:** Passed 2 full timed take-home simulations at mid-level grade; 5+ mock live-coding sessions done.
- **Week 10+:** Interviewing for real. Every real interview → same-day retro with Claude ("here's what they asked, here's what I said — drill my weak spots").

## Failure modes to guard against
- **AI dependency:** if the Saturday rebuild feels hard, you're generating too much. Shift ratio toward attempt-first.
- **Tutorial drift:** no courses longer than a day. The project is the curriculum.
- **Polishing instead of applying:** Project 2 at 80% + interviews beats Project 3 at 100% and zero applications.
- **Skipping SQL and behavioral prep:** the two highest-frequency failure points for junior→mid candidates, and the cheapest to fix.
