# The Question Vault — Depth Layer

[../03-phase3-interviews/01-question-bank.md](../03-phase3-interviews/01-question-bank.md) is the **core 56** — the questions you will actually be asked, drilled until fluent. This folder is the **depth layer underneath it**: ~490 topic Q&A, 270 rapid-fire flashcards, and 56 code-snippet drills (~800 items) — the trivia, the gotchas, and the follow-ups that come after "why?"

**Do not drill this folder until the core 56 are fluent.** Breadth without the core is how candidates fail — they can define `ValueTask` but can't walk a request through the pipeline.

## What this folder is for

Three real situations:

1. **The follow-up chain.** You answer well; the interviewer probes twice more. The core bank gets you through answer #1. This folder is answers #2 and #3.
2. **The trivia opener.** Many .NET screeners open with 10 minutes of rapid factual questions — `const` vs `readonly`, what's boxing, DI lifetimes — to calibrate before the real conversation. Miss too many and the rest of the interview is graded downward.
3. **The gap you don't know you have.** Your projects teach you what your projects use. These files cover the standard surface area a mid-level .NET dev is *assumed* to have touched.

## Files

| File | Prefix | Count | Priority |
|---|---|---|---|
| [01-csharp-language.md](01-csharp-language.md) | `L` | 72 | High — screener favorite |
| [02-async-concurrency.md](02-async-concurrency.md) | `A` | 46 | ⭐ Highest — most-failed depth topic |
| [03-aspnetcore.md](03-aspnetcore.md) | `W` | 62 | ⭐ Highest — it's the job |
| [04-efcore.md](04-efcore.md) | `E` | 52 | ⭐ High |
| [05-sql-databases.md](05-sql-databases.md) | `S` | 56 | ⭐ Highest — most common fail |
| [06-testing.md](06-testing.md) | `T` | 36 | Medium-high |
| [07-http-rest-api-design.md](07-http-rest-api-design.md) | `H` | 38 | Medium |
| [08-security-auth.md](08-security-auth.md) | `X` | 42 | High |
| [09-architecture-patterns.md](09-architecture-patterns.md) | `D` | 44 | Medium — mid-level differentiator |
| [10-devops-git-docker-cicd.md](10-devops-git-docker-cicd.md) | `O` | 38 | Medium |
| [11-rapid-fire-trivia.md](11-rapid-fire-trivia.md) | `R` | 270 | Warm-up fuel |
| [12-code-output-gotchas.md](12-code-output-gotchas.md) | `G` | 32 | High — pure signal |
| [13-spot-the-bug.md](13-spot-the-bug.md) | `B` | 24 | ⭐ High — mirrors real rounds |
| [14-mock-screener-scripts.md](14-mock-screener-scripts.md) | — | 3 | Weekly from week 7 |

**The prefix is the point.** When you miss one, log `E23` in LEARNING-LOG — not "something about EF tracking." Precise miss-logs make re-drilling cheap; vague ones make it impossible.

## The answer shape (every answer in this folder follows it)

> **Claim → why → tradeoff → my project.**

"AsNoTracking skips the change tracker *(claim)*, so reads don't pay for snapshot state you'll never diff *(why)* — the cost is you can't SaveChanges on those entities, so it's read paths only *(tradeoff)*. I default my Bookable list queries to projections, which don't track at all *(project)*."

Four beats, ~15 seconds. A junior gives you beat one. A senior gives you all four without being asked. **The tradeoff beat is what's being graded** — it's the hardest to fake and the one candidates skip.

## Three drill modes

**Mode 1 — Rapid-fire (10 min, daily).** [11-rapid-fire-trivia.md](11-rapid-fire-trivia.md), out loud, no thinking time. Target: <3 seconds per answer. This is recall, not reasoning; if you have to reason, it's not fluent yet.

> Rapid-fire me 20 questions from [file/section]. One at a time. Do not explain — just tell me right or wrong and move on. At the end, list what I missed.

**Mode 2 — Conversational (20 min, 4×/week).** Two categories per session. This is the real drill.

> Quiz me from [file] categories [X, Y]. Ask one at a time, wait for my spoken answer, grade at a mid-level bar. After every answer, push back once with a "why" or a "what would break if" — I need the follow-up chain, not just the first answer. Track which numbers I miss and list them at the end.

**Mode 3 — Gotcha gauntlet (15 min, weekly).** [12-code-output-gotchas.md](12-code-output-gotchas.md) + [13-spot-the-bug.md](13-spot-the-bug.md).

> Give me one snippet at a time from [file]. I answer what it prints / where the bug is and why. Don't confirm until I commit to an answer. If I'm wrong, make me explain the actual mechanism before moving on.

## Weekly rotation (weeks 6–10)

| Day | Vault work (on top of existing cadence) |
|---|---|
| Mon | Mode 2: `L` + `A` |
| Tue | Mode 2: `W` + `E` |
| Wed | Mode 2: `S` + `X` |
| Thu | Mode 2: `T` + `D` / `H` / `O` (rotate) |
| Fri | Mode 3: gotcha gauntlet (pairs with the debugging drill) |
| Sat | Mode 1 on everything missed this week, then rebuild drill |
| Sun | One [mock screener script](14-mock-screener-scripts.md) inside the full mock loop |

Daily 10-minute Mode 1 warm-up regardless of the day. It's the cheapest habit here and it's what keeps the trivia layer from decaying.

## The rule that makes this work

**A question missed twice becomes a 15-minute deep-dive, not a re-read.** Deep-dive means: open the docs, write the code that proves the answer, run it, then teach it back aloud. You are allowed to be wrong once. Being wrong twice on the same item means you filed the answer instead of learning the mechanism.

## Honest calibration

You will not know all 600 of these, and no mid-level hire does. The bar isn't coverage — it's this:

- **Fluent** on everything ⭐-marked (roughly 90 items across the folder). Non-negotiable.
- **Confident** on the rest of `L`/`A`/`W`/`E`/`S`. That's the job's surface area.
- **Aware** on `D`/`O` — enough to say something true and name the tradeoff, not enough to be an architect. That's honest and it's expected.
- **"I don't know, here's how I'd find out"** on anything else, said without flinching. This is a *passing* answer at mid-level and candidates forget that constantly. Bluffing is the only failing answer.
