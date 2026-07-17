# Day 1: Environment Setup

Target: by end of day you can create a .NET web API, run it, hit it with a request, run a test, and commit — all from VS Code.

## 1. Core installs

```powershell
# .NET SDK (LTS)
winget install Microsoft.DotNet.SDK.8

# Git (if not present)
winget install Git.Git

# Verify
dotnet --version
git --version
```

## 2. Database — pick ONE and stick with it

**Recommended: PostgreSQL via Docker** (matches more job postings than LocalDB, and Docker itself is a resume line):

```powershell
winget install Docker.DockerDesktop
# after Docker starts:
docker run -d --name pg-dev -e POSTGRES_PASSWORD=devpass -p 5432:5432 postgres:16
```

Alternative if Docker is a fight: SQL Server LocalDB (`winget install Microsoft.SQLServer.2022.LocalDB`). Everything in the guides notes the EF provider difference where it matters.

**GUI client:** `winget install DBeaver.DBeaver.Community` — you will use this daily for SQL drills.

## 3. VS Code extensions

```powershell
code --install-extension ms-dotnettools.csdevkit        # C# Dev Kit (compiler, debugger, test explorer)
code --install-extension ms-azuretools.vscode-docker
code --install-extension humao.rest-client               # .http files for testing endpoints
code --install-extension eamodio.gitlens
```

Plus **Codex** (OpenAI extension) — sign in, confirm inline completions work in a `.cs` file.

## 4. Claude Code

Already installed if you're reading this via Claude Code. Confirm it runs from the projects folder. You'll use it from the *project repo* directory, not this curriculum repo.

## 5. Smoke test — prove the whole toolchain (30–45 min)

```powershell
mkdir ..\ticket-tracker-api; cd ..\ticket-tracker-api
dotnet new webapi -n TicketTracker.Api
dotnet new xunit -n TicketTracker.Tests
dotnet new sln; dotnet sln add TicketTracker.Api TicketTracker.Tests
cd TicketTracker.Api; dotnet run
```

- Hit the sample endpoint with a `.http` file (REST Client extension) — not the browser. Get comfortable with `.http` files now; they're your fastest feedback loop.
- Set a breakpoint in the sample controller, F5, hit the endpoint, watch it pause. **If you skip learning the debugger today, you'll pay for it in week 2.**
- `dotnet test` from the solution root — green.
- `git init`, first commit, create a private GitHub repo, push.

## 6. Project-level Claude config

In each project repo, create `CLAUDE.md`:

```markdown
# Working agreement
- I am training for mid-level interviews. Do NOT write feature code for me unless I explicitly say "write it".
- Default modes: review my code, explain concepts, quiz me, generate test scaffolds, plant bugs for drills.
- When I ask for help, give a hint and a question first, not the answer.
- When reviewing, be strict: bugs, edge cases, naming, missing tests. Cite line numbers.
```

This makes the AI workflow (next doc) the default instead of something you have to remember to ask for.

## Done when
- [ ] `dotnet run` serves an endpoint you can hit from a `.http` file
- [ ] Breakpoint hit and inspected in VS Code
- [ ] `dotnet test` passes
- [ ] Postgres container running, connected in DBeaver
- [ ] Repo pushed to GitHub (private)
- [ ] `CLAUDE.md` working agreement in the project repo
