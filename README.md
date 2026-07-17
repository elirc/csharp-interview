# Junior → Mid-Level Fast-Track: Curriculum Repo

This repo is the complete working material for the 10-week plan in [UPSKILL-PLAN.md](UPSKILL-PLAN.md). Everything you need is here — no external courses.

## How to use this repo

1. **Read [UPSKILL-PLAN.md](UPSKILL-PLAN.md) once, fully.** It's the map; everything else is terrain.
2. **Do [00-setup](00-setup/) on day 1.** Environment + the AI workflow rules you'll use every day.
3. **Work the phases in order.** Each phase folder has an overview that tells you exactly what to open next.
4. **Run the drills on schedule** ([drills/](drills/)) — daily SQL, Friday debugging, Saturday rebuild. The drills are where retention happens; the projects alone are not enough.
5. **Log every day** in [LEARNING-LOG.md](LEARNING-LOG.md). One line minimum. This becomes your interview story bank.

## Directory map

| Folder | Contents | When |
|---|---|---|
| [00-setup/](00-setup/) | Environment setup, AI workflow + full prompt library | Day 1 |
| [01-phase1-foundations/](01-phase1-foundations/) | Project 1 (Ticket Tracker API) spec, week 1–3 guides, concept checklist | Weeks 1–3 |
| [02-phase2-portfolio/](02-phase2-portfolio/) | Project 2 (Booking System) spec, week 4–7 guides, git/PR workflow, CI/CD + deploy | Weeks 4–7 |
| [03-phase3-interviews/](03-phase3-interviews/) | Question bank w/ answers, 2 take-home simulations w/ rubrics, live-coding scripts, system design, behavioral | Weeks 8–10 (start reading week 6) |
| [04-question-vault/](04-question-vault/) | The depth layer: ~490 Q&A across 10 topics, 270 rapid-fire flashcards, 32 code-output gotchas, 24 spot-the-bug snippets, 3 full mock screener scripts | Daily warm-up from week 6 |
| [drills/](drills/) | SQL 30-day plan, debugging drills, rebuild drills | Ongoing from week 1 |

## Where the actual code goes

Create the projects as **separate repos** (interviewers will look at them — keep this curriculum repo private):

- `ticket-tracker-api/` — Project 1 (weeks 1–3)
- `bookable/` (or your name) — Project 2 (weeks 4–7), this is the portfolio piece

## The one rule

**Attempt first, AI second.** Every feature: 30–45 minutes of your own effort before asking Claude Code or accepting a multi-line Codex completion. The full workflow is in [00-setup/02-ai-workflow.md](00-setup/02-ai-workflow.md).

## Weekly cadence (pin this)

| Day | Focus |
|---|---|
| Mon–Thu | Feature work via the 4-step loop (60–90 min blocks) |
| Daily | 30 min SQL ([drills/sql-30-day-plan.md](drills/sql-30-day-plan.md)) |
| Daily (wk 6+) | 10 min rapid-fire trivia ([04-question-vault/11-rapid-fire-trivia.md](04-question-vault/11-rapid-fire-trivia.md)) — out loud, in the car, anywhere |
| Fri | Debugging drill + AI-assisted PR self-review + gotcha gauntlet ([04-question-vault/12](04-question-vault/12-code-output-gotchas.md) + [13](04-question-vault/13-spot-the-bug.md)) |
| Sat | From-memory rebuild, AI off |
| Sun (wk 5+) | One mock interview, rotating format |
