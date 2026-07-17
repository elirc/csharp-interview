# Git, Docker, CI/CD & Ops — 38 Questions (`O`)

Nobody hires a mid-level backend dev for their Kubernetes expertise. They hire you to not be scared of the pipeline, to recover from a bad merge without panicking, and to know what happens to your app between `git push` and a user's request. That's the bar this file targets.

Log misses as `O##`.

---

## Git

**O1. ⭐ Merge vs rebase?**
Merge preserves history exactly and creates a merge commit (true, sometimes noisy). Rebase replays your commits on top of the target — linear, readable history, but it **rewrites commit hashes**, so the golden rule is: never rebase a branch someone else has pulled. My habit: rebase my own feature branch onto main to stay current, merge (or squash) into main.

**O2. What does squash-merge do, and when is it wrong?**
Collapses a PR into one commit on main — clean history, one revert reverts the whole feature. Wrong when the PR contains genuinely separable commits worth keeping (a refactor + a fix, so a `git bisect` can tell them apart). The real answer: "match the team's convention" — this is a house-style question, not a correctness one.

**O3. `git reset --soft/--mixed/--hard`?**
Moves the branch pointer, then: `--soft` keeps changes staged, `--mixed` (default) keeps them in the working tree unstaged, `--hard` **discards everything**. `--hard` is the one that loses work — and even then, a committed state is usually recoverable via reflog.

**O4. ⭐ Revert vs reset — which for a bad commit on main?**
**Revert.** It creates a new commit undoing the change, so shared history is preserved and nobody's clone breaks. `reset` rewrites history and requires a force-push to a shared branch, which corrupts everyone else's view and is how a team loses work. Reset is for your own local mess; revert is for anything published.

**O5. ⭐ You committed a secret and pushed. Now what?**
**Rotate the secret first** — it's compromised the moment it's pushed; bots scrape public pushes within seconds and it's already in forks, clones, and CI logs. Then purge history (filter-repo/BFG + force-push + everyone re-clones), then add prevention (pre-commit scanning, push protection). Everyone reaches for the history rewrite first; leading with rotation is the answer that shows you understand the threat model. (Same as `X36` — it gets asked in both rounds.)

**O6. What's the reflog and when did it save you?**
A local log of everywhere HEAD has been — so a "lost" commit after a bad reset/rebase is usually right there: `git reflog`, find the hash, `git checkout`/`reset` to it. It's the reason `git` almost never truly loses committed work, and knowing it converts a panic into two minutes.

**O7. ⭐ `git bisect` — what is it and why is it a superpower?**
Binary search over history to find the commit that introduced a bug: mark a known-good and known-bad commit, git checks out the midpoint, you test, repeat — 1,000 commits in ~10 steps. `git bisect run ./test.sh` automates it entirely. It's the answer to "how would you find when this broke" and it's a great flex in a debugging interview, because it's *systematic* rather than intuitive — the exact quality being graded.

**O8. Cherry-pick — when?**
Apply one commit onto another branch: a hotfix that needs to go to both main and a release branch. Overused, it duplicates commits and confuses history. Fine occasionally, a smell as a workflow.

**O9. What's in a good commit message?**
Imperative subject under ~50 chars ("Add overlap constraint to bookings"), a blank line, then **why** — the diff already shows what. Conventional Commits (`feat:`/`fix:`) if the team automates changelogs. The test: will this help someone doing a `git blame` in two years understand the *reason*? "fixes" and "wip" fail it.

**O10. What's in a good PR?**
Small (a reviewer's attention falls off a cliff past ~400 lines), one concern, a description with context + the decision + how to test it, green CI, and self-reviewed first. The thing to say: **I review my own diff before requesting review** — it catches the debug logging and the accidental file every time, and it respects the reviewer's time.

**O11. Trunk-based vs GitFlow?**
Trunk-based: short-lived branches, merge to main daily, release from main, feature flags for unfinished work — fewer, smaller conflicts, faster feedback; the modern default for a web app that deploys continuously. GitFlow: long-lived develop/release/hotfix branches — heavy, made for versioned/shipped software, and it produces merge pain. Opinion + the reason it exists.

**O12. How do you resolve a nasty merge conflict?**
Understand **both** intentions first (`git log`/`blame` on the region — what were they trying to do?), don't just pick a side; run the tests after; and if it's genuinely tangled, talk to the other author for two minutes instead of guessing for thirty. Prevention is the real answer: small PRs, merge often. `git rerere` if you're re-resolving the same conflict repeatedly.

**O13. `.gitignore` — what always goes in?**
`bin/`, `obj/`, `.vs/`, `*.user`, `appsettings.Development.json` if it holds secrets, `.env`, local DB files. And the trap: gitignore doesn't untrack an **already-tracked** file — you need `git rm --cached`. That's why the secret is still there after "I added it to gitignore."

---

## Docker

**O14. ⭐ Image vs container?**
Image = the immutable, layered filesystem + metadata (the class). Container = a running instance with a writable layer on top (the object). Same image, N containers, and anything written inside dies with the container — hence volumes.

**O15. Why are Docker layers important?**
Each instruction is a cached layer, and a change invalidates that layer **and everything after it**. Practical consequence you should state: copy the `.csproj` and `dotnet restore` **before** copying the rest of the source, so a code change doesn't re-download every NuGet package. That single ordering trick is the whole Dockerfile-caching question.

**O16. ⭐ What's a multi-stage build and why does it matter for .NET?**
Build in the **SDK** image (compilers, NuGet, ~1GB+), then copy only the published output into the **runtime** (ASP.NET) image. You get a much smaller image, faster deploys, and — the security point — **no compilers, source, or build secrets in production**. Every real .NET Dockerfile does this; not knowing it is a tell.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj ./            # restore layer caches independently of source
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

**O17. `CMD` vs `ENTRYPOINT`?**
ENTRYPOINT = the executable (hard to override), CMD = default arguments (easily overridden at `docker run`). Common pattern: ENTRYPOINT the binary, CMD the default args. If you only set CMD, `docker run img bash` replaces the whole thing.

**O18. Does `EXPOSE` publish a port?**
**No** — it's documentation/metadata. `-p 8080:80` (or compose `ports:`) actually publishes it. A classic "why can't I reach my container" and a nice quick trivia hit.

**O19. `.dockerignore` — why care?**
Everything not ignored is sent to the build daemon as context: `bin/`, `obj/`, `.git/` bloat the build, slow it down, and can **leak secrets or local artifacts into the image**. Also, a stale local `obj/` copied in has caused genuinely baffling build failures.

**O20. Volumes vs bind mounts?**
Volumes: Docker-managed storage, portable, the right answer for a database's data. Bind mounts: a host path mapped in — great for local dev (live source), platform-dependent otherwise. The principle: **containers are ephemeral; state lives in a volume or a managed service.**

**O21. How do containers talk to each other in Compose?**
A user-defined network with DNS by **service name**: your API connects to `Host=db`, not `localhost`. This is the #1 local-dev confusion — inside a container, `localhost` is *that container*, and `localhost:5432` on your host is not the DB container's port.

**O22. Why does my app work locally but not in the container?**
Run the list: config (env vars not passed, `ASPNETCORE_ENVIRONMENT` defaulting to Production), networking (`localhost` vs service name), the app binding to `localhost` instead of `0.0.0.0` (`ASPNETCORE_URLS=http://+:8080`), file paths/case sensitivity (Linux is case-sensitive, Windows isn't — this one bites constantly), missing files excluded by `.dockerignore`, and time zone/culture differences. A structured list here reads as lived experience.

**O23. Should the container run as root?**
No — a container escape becomes root on the host, and it's a trivial hardening win. `USER app` (the official .NET images ship a non-root `app` user, and .NET 8+ images default to it). Also worth naming: **chiseled/distroless** images strip the shell and package manager, so there's nothing for an attacker to use — smaller *and* safer.

**O24. Do you need Kubernetes?**
Honest, and this is the right answer at mid-level: no, not for one app. App Service/Railway/Fly/ECS handles it, and K8s is a platform with a full-time cost. I'd want it when there are many services, real orchestration needs, and a platform team. **Willingness to say "that's overkill for us" is the signal being tested.**

---

## CI/CD

**O25. ⭐ What's in your pipeline?**
On PR: restore → build **with warnings as errors** → unit tests → integration tests (containers spun up in the job) → a vulnerability scan → required to be green to merge. On main: the same, then publish + deploy (+ migrations). The properties that matter: **fast** (people ignore a 30-minute pipeline), **deterministic** (no flakes — a red build must mean something), and **no manual steps**.

**O26. Why is a green CI a *requirement* to merge, not a suggestion?**
Because a broken main blocks everyone and gets normalized ("oh that test is always red") — and once it's normalized, CI signals nothing and you've paid for it for free. Branch protection makes the rule structural rather than social.

**O27. Where do migrations run in the pipeline?**
Have a preference with the tradeoff: a **deploy step** running an EF migration bundle or an idempotent script, gated before the new app version starts. Not `Database.Migrate()` at startup — that races across multiple instances and forces the app's own credentials to hold DDL rights permanently. And for zero downtime, migrations must be **backwards compatible** with the currently-running version, because both versions run at once during a rolling deploy (`E41`, expand/contract).

**O28. How do you deploy with zero downtime?**
Rolling/blue-green/canary + backwards-compatible schema changes + a **readiness probe** so traffic only arrives once the instance is actually ready + graceful shutdown so in-flight requests finish. The insight to voice: **the hard part isn't the deploy mechanism, it's that two versions of your code run simultaneously against one database.** That's the sentence that shows you've thought about it.

**O29. Blue/green vs canary vs rolling?**
Blue/green: two full environments, flip the router, instant rollback (costs 2× infra). Canary: 1% → 10% → 100%, catches problems with real traffic at small blast radius (needs good metrics to be meaningful). Rolling: replace instances gradually — the cheap default, slower to roll back. All three assume you can roll *back*, which is why schema changes are the constraint.

**O30. How do you roll back?**
Redeploy the previous artifact — which requires that the DB schema still supports it. That's the constraint that governs everything: **never ship a migration and the code that requires it in one release** if you want rollback to be real. Otherwise your "rollback" is a restore from backup, and that's an incident, not a rollback.

**O31. Where do secrets live in CI?**
The platform's secret store (GitHub Actions secrets / OIDC federation to the cloud — no long-lived keys at all, which is the modern answer), injected as env vars, never echoed. The trap worth naming: secrets are masked in logs but **a base64 or a JSON-dumped secret is not masked**, and PRs from forks must not receive them.

**O32. What do you cache in CI?**
The NuGet package cache keyed on `packages.lock.json`/csproj hashes. Point to make: cache **inputs, not outputs** — caching build output produces bugs that only reproduce in CI, which is the worst kind.

**O33. Semantic versioning?**
MAJOR.MINOR.PATCH — breaking / additive / fix. The practical part: it's a *promise to consumers*, so it matters for a library and is mostly theater for a deployed app (which uses build numbers/commit SHAs). Knowing which one you're in is the answer.

---

## Runtime & ops

**O34. ⭐ Your app is down. What do you do first?**
Sequence, and the first step is the answer: **check if it's actually down** (health endpoint, monitoring, or is it one user's DNS?) → scope it (all endpoints or one? all users? when did it start? what changed — a deploy, a migration, a config change, a dependency?) → **mitigate before diagnosing** (roll back, take the bad node out — restoring service beats understanding it) → *then* find the root cause from logs/traces → write the postmortem. The instinct being tested: mitigate first, diagnose second, blame nobody.

**O35. What do you actually monitor?**
The **RED** method for a request-driven service: **R**ate, **E**rrors, **D**uration (p95/p99) per endpoint. Plus: dependency health, DB connection pool saturation, thread pool queue length (`A19`!), and business metrics — a booking rate of zero at noon means something is broken that no technical dashboard is red about. Alerts should be **symptom-based** (users are getting 500s) not cause-based (CPU is 80%), or people learn to ignore them.

**O36. Logs vs metrics vs traces?**
Logs: discrete events with context — *what happened* (expensive at volume). Metrics: cheap aggregated numbers over time — *is something wrong* (no per-request detail). Traces: one request's path across components with timings — ***where* the time went** (the one that solves "the API is slow" in a distributed system). You need all three; OpenTelemetry is now the vendor-neutral way to emit them, and .NET has it built in (`Activity`, `Meter`).

**O37. How would you find out why the app crashed at 3am?**
Logs first — where do they go when the container dies? (If the answer is "the container," that's the finding.) Look at the last requests before death, then classify: unhandled exception, OOM (memory limit → check for a leak or an unbounded query loading 1M rows), thread pool starvation, a host/orchestrator restart, or a liveness probe killing a healthy-but-slow app. Then the honest addendum: *"my demo logs to console + platform capture; for production I'd want centralized logs, an error tracker, and an alert — plus a dump on OOM."* Naming the gap honestly beats pretending.

**O38. What's a health check, and what shouldn't it do?**
Liveness (restart me if this fails) vs readiness (take me out of the LB if this fails) — see `W53`. What it shouldn't do: check dependencies in the **liveness** probe. If the DB is down, every instance fails liveness, the orchestrator restarts them all in a loop, and you've converted a database outage into a total outage with a stampede of cold starts on recovery. That's a real, famous failure mode and a very good thing to know.

---

## Coverage tracker

`O1–O13` git ⭐ · `O14–O24` docker ⭐ · `O25–O33` CI/CD · `O34–O38` ops ⭐

**The five that decide the round:** O4 (revert vs reset on main), O7 (bisect — the systematic-debugging flex), O16 (multi-stage builds), O27/O28 (migrations + zero downtime: *two versions run at once*), O34 (down at 3am — mitigate before diagnose).

**The honest frame for this whole file:** you're a backend developer, not an SRE. The bar is "I can read the pipeline, I don't fear the deploy, and I know what I don't know." Confidently saying "I've never run Kubernetes in anger, here's what I'd want to learn" is a *pass* — inventing experience is the only fail.
