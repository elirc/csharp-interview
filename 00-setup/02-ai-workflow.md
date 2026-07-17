# The AI Workflow & Prompt Library

This is the operating system for the whole 10 weeks. The difference between "junior who prompts" and "mid who ships" is entirely in *how* you use the tools.

## The 4-step loop (every feature, no exceptions)

### 1. Attempt first — 30–45 min cap
Write it yourself. Codex inline completions are allowed **only** for lines you could have typed anyway (property lists, obvious boilerplate). The moment Codex suggests a block you don't fully understand: reject it, write your own version.

*Why the cap:* below 30 min you're not struggling enough to learn; past 45 you're burning time on something a hint would unblock.

### 2. AI as reviewer, not author
When your attempt works (or you're stuck at the cap):

> Review this code like a senior engineer doing a strict PR review. Find bugs, missing edge cases, security issues, and naming problems. Cite line numbers. Do NOT rewrite it — for each finding, ask me a question that leads me to the fix.

Fix everything yourself. Re-request review until clean.

### 3. AI as explainer + quizmaster
For any code you didn't fully author (accepted completion, pattern copied from docs, fix you applied without full understanding):

> Explain this block line by line. Then quiz me with 3 questions: one "what does this do", one "what breaks if I change X", one "why this approach over the alternative". Grade my answers harshly.

**Rule: if you fail the quiz, the code doesn't get committed until you pass.**

### 4. Weekly rebuild (Saturdays — see drills/rebuild-drills.md)
One feature from this week, blank file, AI completely off, from memory. This is the retention step and the live-interview simulator.

## Division of labor

| Task | Tool | Mode |
|---|---|---|
| Inline completions while you drive | Codex | Only accept what you could've typed |
| Code review | Claude Code | Strict reviewer prompt |
| Concept explanations + quizzing | Claude Code | Explainer prompt |
| Debugging when truly stuck (past cap) | Claude Code | Hint mode — never "fix it" |
| Test scaffolding | Claude Code | It writes the skeleton, YOU write the assertions |
| Planted-bug drills | Claude Code | See drills/debugging-drills.md |
| Mock interviews | Claude Code | Scripts in 03-phase3-interviews/ |
| Second-opinion review of your PRs | Codex (chat) | Cross-check Claude's review — models catch different things |

## Prompt library

Copy-paste these. They're tuned to teach, not to do.

### Hint mode (when stuck)
> I'm stuck on: [problem]. Here's what I've tried: [attempts]. Do NOT give me the solution. Give me one hint and ask me what I'd try next. If I'm still stuck after two hints, show me the smallest possible nudge (a method name, a doc link), not code.

### Task breakdown (start of each feature)
> I'm implementing [feature]. Break it into subtasks sized for a junior to do one at a time. For each: what done looks like and one gotcha to watch for. Don't write any code. I'll implement each and you review.

### Strict review
> Review this diff like a senior engineer who has to maintain it. Bugs, edge cases, security, naming, missing tests, needless complexity. Line numbers. Questions, not rewrites.

### Ownership quiz
> Quiz me on the code in [file]: 3 questions escalating from "what" to "what breaks if" to "why this over alternatives". Harsh grading. If I score under 2/3, tell me exactly what to reread.

### Concept deep-dive (for the concept checklist)
> Explain [concept] in three passes: (1) two sentences, (2) how it shows up concretely in an ASP.NET Core app, (3) the interview-question version with the answer a mid-level candidate gives vs the answer a junior gives. Then ask me to explain it back and correct me only where I'm wrong.

### Planted bugs (Friday drill)
> Take the code on this branch and introduce 3 realistic bugs: one logic error, one async/concurrency issue, one data/EF issue. Make them subtle — the kind that pass a lazy review. Commit them with an innocent-looking message. Do not tell me what they are. I'll find them with the debugger and logs, then you'll confirm.

### Teach-back (best retention tool in this file)
> I'm going to explain how [thing I built] works, as if you're a junior teammate. Interrupt only when I'm wrong or hand-waving. At the end, rate the explanation: would it pass an interview deep-dive?

### Same-day interview retro (phase 3)
> I just had a real interview. They asked: [questions]. I answered: [summary]. Grade each answer at a mid-level bar, give the answer I should have given, and build me a 30-minute drill on the weakest area.

## Anti-patterns — catch yourself

- **The paste-and-pray:** error message straight into AI without reading it yourself first. Read the stack trace, form a hypothesis, THEN ask — and ask for a hint.
- **The ghost feature:** a feature works but you can't explain it. It will be found in an interview. Quiz prompt, immediately.
- **The velocity trap:** shipping 5 AI-written features feels better than 2 hand-written ones and teaches you less than 1.
- **Review theater:** requesting review and accepting all fixes without understanding each one. Every finding gets the "why did this matter" treatment.

## The weekly self-check
Saturday, after the rebuild drill, one honest question: **"Could I have built this week's features on a laptop with no internet?"** If no — shift next week's ratio toward attempt-first.
