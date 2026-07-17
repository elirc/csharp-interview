# Git & PR Workflow — Working Like You Already Have the Job

Interviewers *will* read your commit history and PRs on Bookable. A repo with disciplined history says "mid-level" louder than any resume bullet. These habits are also directly probed: "walk me through how you ship a change."

## Branching

- `main` is always green and deployable. Never commit to it directly (add branch protection once CI exists).
- One branch per feature-ladder item: `feature/slot-computation`, `fix/dst-slot-boundary`.
- Branches live days, not weeks. If a feature is too big to merge in ~2–3 days, split it.

## Commits

- Small and coherent: one logical change each. "WIP" and "fix" (×5) tell interviewers you don't curate.
- Message format (lightweight conventional): `feat: compute open slots from availability windows` · `fix: use half-open intervals in overlap check` · `test: parallel booking race reproduction` · `refactor:`, `docs:`, `chore:`.
- Body when the *why* isn't obvious — the DST fix commit deserves three lines about what was wrong.
- **The story test:** `git log --oneline` should read like a build log a stranger could follow. Check it before opening each PR.

## Pull requests (yes, solo — the discipline is the product)

Save as `.github/PULL_REQUEST_TEMPLATE.md` in the project repo:

```markdown
## What
One or two sentences: the change from the user's perspective.

## Why / Design notes
Decisions made and alternatives rejected. 5 lines max.
(e.g., "Slots computed on demand rather than stored: stored slots
require sync on every availability/booking change; computed slots
are always consistent. Cost: recompute per request — fine at this scale.")

## How tested
- [ ] Unit: <what>
- [ ] Integration: <what>
- [ ] Manual: <the .http requests / UI path you ran>

## Review notes
Anything you're unsure about — ask your AI reviewer to focus here.
```

## The merge ritual (every PR)

1. Self-review the diff in the GitHub UI first — you'll catch embarrassments before the AI does.
2. Claude strict-review prompt on the branch → fix findings → ownership quiz on anything you didn't fully author.
3. Codex second opinion on the diff (models catch different things; disagreements between them are the most instructive part).
4. CI green (once it exists).
5. Squash-merge or merge — pick one, stay consistent. Delete the branch.

## The payoff script (memorize the shape)

"Walk me through how you ship a change" — your answer, now true:

> "Branch off main, small commits as I go. Before the PR I self-review the diff, then run an AI-assisted review pass and fix findings. PR description covers the design decision and how it's tested. CI runs build and tests including integration tests against Postgres; green is required to merge. Squash-merge, delete the branch, and main stays deployable — it's what's running on the live demo."

That paragraph, backed by a public repo that proves it, is a mid-level signal few candidates have.
