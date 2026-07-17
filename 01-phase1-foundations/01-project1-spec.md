# Project 1 Spec — Ticket Tracker API

A help-desk ticket system. Chosen because the domain is instantly understandable, it forces every CRUD interview topic, and "build a ticket/task API" is the single most common take-home archetype.

**Repo:** `ticket-tracker-api` (separate from this curriculum repo). Stack: ASP.NET Core Web API, EF Core, PostgreSQL (or LocalDB), xUnit.

## Domain model

```
User      (Id, Email [unique], DisplayName, Role [Agent|Admin], CreatedAt)
Ticket    (Id, Title, Description, Status [Open|InProgress|Resolved|Closed],
           Priority [Low|Medium|High|Critical], CreatedAt, UpdatedAt,
           ReporterId → User, AssigneeId → User?, nullable)
Comment   (Id, TicketId → Ticket, AuthorId → User, Body, CreatedAt)
```

Business rules (these create the interesting code — don't skip them):
- A ticket starts `Open`, unassigned. Assigning it moves it to `InProgress`.
- Status can only move forward (`Open→InProgress→Resolved→Closed`), except `Resolved→InProgress` (reopen). Invalid transitions → 422 with a clear message.
- Only the assignee or an Admin can resolve a ticket.
- Closed tickets reject new comments.

## The feature ladder

Each rung = one branch + PR. Acceptance criteria are your definition of done — test them via `.http` file before opening the PR.

### Rung 1 — Ticket CRUD (week 1)
`POST /api/tickets` · `GET /api/tickets` · `GET /api/tickets/{id}` · `PUT /api/tickets/{id}` · `DELETE /api/tickets/{id}`
- [ ] Correct codes: 201 + Location header on create, 200, 404 (not 500!) for missing id, 204 on update/delete
- [ ] In-memory list is fine for THIS rung only; EF lands in rung 3

### Rung 2 — DTOs + validation (week 1)
- [ ] Separate `CreateTicketRequest`, `UpdateTicketRequest`, `TicketResponse` — entities never cross the API boundary
- [ ] Validation: Title required/≤200 chars, Description ≤4000, Priority a valid enum value. Invalid → 400 with field-level errors (automatic via `[ApiController]` + annotations, or FluentValidation — pick one, be able to defend it)

### Rung 3 — EF Core + relationships (week 2)
- [ ] DbContext, entity configs, initial migration, seed data (3 users, 10 tickets, comments)
- [ ] `GET /api/tickets/{id}` includes reporter/assignee names and comments — **watch for N+1 and JSON cycles here; both will happen to you, and both are checklist concepts**
- [ ] `POST /api/tickets/{id}/comments`, `GET /api/tickets/{id}/comments`
- [ ] Status-transition + closed-ticket rules enforced in a service layer (not the controller), 422 on violation

### Rung 4 — List power features (week 2) ← the take-home classic
`GET /api/tickets?status=Open&assigneeId=…&search=login&sortBy=priority&sortDir=desc&page=1&pageSize=20`
- [ ] All filters combinable; search hits Title + Description (case-insensitive)
- [ ] Response envelope: `{ items, page, pageSize, totalCount, totalPages }`
- [ ] pageSize capped at 100; invalid sort field → 400, not exception
- [ ] Built as composable `IQueryable` — one DB query, verified by logging the SQL

### Rung 5 — Error handling + logging (week 2)
- [ ] Global exception middleware → RFC 7807 ProblemDetails; stack traces never reach the client outside Development
- [ ] Serilog: request logging + structured logs in services (`log.Information("Ticket {TicketId} assigned to {UserId}", …)`)
- [ ] Domain rule violations are 4xx via a custom exception type (e.g. `DomainRuleException`), not 500s

### Rung 6 — Auth (week 3)
- [ ] `POST /api/auth/register` (hash with BCrypt/Identity's hasher — never roll your own), `POST /api/auth/login` → JWT with role claim
- [ ] All ticket/comment endpoints `[Authorize]`; "only assignee or Admin resolves" enforced from claims
- [ ] Secrets (signing key, connection string) via user-secrets/env vars — nothing sensitive in git. Ever.

### Rung 7 — Tests (week 3, though you write some at every rung)
- [ ] Unit tests on the service layer: status transitions (valid + invalid), authorization rules, comment-on-closed rejection
- [ ] Integration tests via `WebApplicationFactory`: full CRUD cycle, filter/paging behavior, 401/403 paths
- [ ] A failing test fails the build; ~everything meaningful covered (don't chase %; chase behaviors)

## Explicitly out of scope (resist)
Frontend · Docker for the app itself · CQRS/MediatR/clean-architecture layering · caching · rate limiting. Phase 2 earns some of these.
