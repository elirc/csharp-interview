# EF Core — 52 Questions (`E`)

The ORM questions go one layer deeper than the core bank: not "what's N+1" but "why did EF pick that SQL, and what would you do when it picks wrong." Every answer here should end in SQL — EF questions are really database questions wearing a C# costume.

Log misses as `E##`.

---

## DbContext & lifetime

**E1. ⭐ What *is* a `DbContext`?**
Unit of work + identity map + query builder over a connection. It tracks the entities it loads, computes a diff on `SaveChanges`, and issues the minimal SQL. "Repository + Unit of Work, already implemented" — which is the crux of the do-I-need-a-repository argument.

**E2. ⭐ Why is `DbContext` registered as scoped?**
One unit of work per request: everything an action touches commits together, and the context dies with the request so the change tracker never grows unbounded. It's also **not thread-safe** — a scope-per-request keeps one context to one logical thread of execution.

**E3. What's `AddDbContextPool` and its catch?**
Pools context *instances* to skip re-initialization (a real win at high throughput). Catch: instances are **reused**, so any custom state you put in your context (a tenant id, a user id field) leaks between requests unless you reset it. Pooling constrains your constructor too. Default to `AddDbContext`; pool when a profile says so.

**E4. When do you use `IDbContextFactory`?**
When the lifetime isn't a request: background services, Blazor Server components, or when you genuinely need two contexts concurrently (parallel queries). It hands you a fresh context you own and dispose.

**E5. What does the identity map do?**
Within one context, a given key maps to exactly **one** instance. Query the same ticket twice → same object, and the second query's data is **discarded by default**, not re-applied over your local changes. That surprises people: "I refreshed and my edit is still there." Use `AsNoTracking` or `Reload()` if you want fresh data.

---

## Change tracking

**E6. ⭐ Entity states?**
Added, Modified, Deleted, Unchanged, Detached. `SaveChanges` turns them into INSERT/UPDATE/DELETE/nothing. `context.Entry(x).State` inspects it — a good debugging tool to name.

**E7. How does EF know what changed?**
Snapshot change tracking: it stores original values on load and compares field-by-field on `SaveChanges` (unless the entity implements change-tracking proxies). That snapshot is why tracking costs memory and why `AsNoTracking` is faster.

**E8. ⭐ `AsNoTracking` — when and what does it cost you?**
Read-only paths. Skips the snapshot and identity map → less memory and CPU. Cost: entities are detached, so `SaveChanges` won't persist edits, and duplicate rows in a join produce **duplicate object instances** (no identity resolution) — `AsNoTrackingWithIdentityResolution` if that matters. Note the nuance most miss: **projections to a DTO aren't tracked anyway**, so `AsNoTracking` on a `Select`-into-DTO query is a no-op.

**E9. What does `Update(entity)` actually do?**
Attaches it and marks **every property Modified** — so the UPDATE writes every column, including ones you never touched, clobbering concurrent changes to other fields. Better: load, mutate the fields, `SaveChanges` (diff-based), or `Entry(x).Property(p).IsModified = true` for a surgical update.

**E10. How do you update a row without loading it?**
`Attach` + mark modified, or `ExecuteUpdate` (EF Core 7+). Attach is the classic; `ExecuteUpdate` is a direct SQL UPDATE with no round-trip and no tracking.

**E11. What's `ExecuteUpdate`/`ExecuteDelete` and what do you give up?**
Set-based SQL directly from LINQ: `db.Tickets.Where(t => t.Old).ExecuteDeleteAsync()` — one statement, no loading, no tracking. You give up: change-tracker awareness (already-loaded entities go **stale**), `SaveChanges` interceptors/events, and it executes **immediately outside your SaveChanges transaction** unless you opened one explicitly. That last point is the interview nuance.

**E12. Relationship fixup?**
When you load related entities into a context, EF wires up navigation properties automatically from the tracked graph — even if you never `Include`d them, another query may have already populated them. It's why a tracked query can look like it lazy-loaded when it didn't.

**E13. What are shadow properties?**
Properties in the model with no CLR property — e.g. a FK EF infers, or `LastModified` set in `SaveChanges` overrides. Access via `Entry(x).Property("Name")`. Handy for audit columns you don't want on the entity.

---

## Querying

**E14. ⭐ Eager vs lazy vs explicit loading?**
Eager: `Include` in the query (one round-trip, you decide up front). Lazy: navigation loads on first access — needs `virtual` navs + the proxies package, and it is the **N+1 factory** (and it silently breaks when the context is gone or you serialize the entity). Explicit: `Entry(x).Collection(...).LoadAsync()` when you conditionally need it. Real answer: "eager or projection; lazy loading is off in my projects."

**E15. ⭐ `Include` vs projection — which and why?**
Projection (`Select` into a DTO) for reads: only the columns you need, no tracking, no over-fetch. `Include` when you need to **modify** the graph or genuinely need whole entities. The line to say: *"project for queries, include for commands."*

**E16. What is cartesian explosion?**
`Include` two sibling collections and the SQL JOIN multiplies rows: a ticket with 10 comments and 10 attachments returns **100 duplicated rows**, EF de-dupes in memory, and you paid to transfer the ticket's columns 100 times. It's why a query with plenty of Includes gets mysteriously huge.

**E17. `AsSplitQuery` — the fix and its cost?**
Splits one query into one per collection, avoiding duplication. Cost: multiple round-trips, and — the part to volunteer — **no longer a single atomic snapshot**, so concurrent writes between the queries can give you an inconsistent graph unless you're in a transaction with sufficient isolation. Configurable globally or per query.

**E18. ⭐ `IEnumerable` vs `IQueryable` in EF terms?**
`IQueryable` builds an expression tree the provider translates to SQL; the moment you hit `AsEnumerable()`/`ToList()`, everything downstream is LINQ-to-Objects **in memory**. So `db.Tickets.ToList().Where(t => t.Open)` selects the entire table and filters in C# — the single most common EF performance bug after N+1.

**E19. Client-side evaluation — what's the modern behavior?**
EF Core 3.0 made untranslatable `Where`/predicate code **throw** instead of silently pulling the table into memory (2.x did the latter and quietly destroyed performance). The **final projection** can still evaluate client-side — `Select(t => new { Label = Format(t.Name) })` works, and that asymmetry is worth knowing.

**E20. Give me something EF can't translate.**
Your own C# method (`Where(t => IsBusinessDay(t.Date))`), most culture-aware string ops, some `DateTime` arithmetic. Fixes: express it in translatable terms, filter server-side then finish in memory *deliberately* (`AsEnumerable` after narrowing), or push it into SQL as a computed column/function mapped via `HasDbFunction`.

**E21. Filtered includes?**
`Include(t => t.Comments.Where(c => !c.Deleted))` (EF Core 5+). Before that you needed a projection. Restriction: the same navigation can't have two different filters in one query.

**E22. Global query filters?**
`modelBuilder.Entity<Ticket>().HasQueryFilter(t => !t.IsDeleted)` — applied to every query for that entity automatically. The soft-delete and multi-tenant workhorse. Traps to name: it applies to `Include`d navigations too; `IgnoreQueryFilters()` is the escape hatch; and a filter capturing a **scoped** service (the current tenant) plus a **pooled** context is a genuine bug factory.

**E23. `Find` vs `FirstOrDefault`?**
`Find(id)` checks the change tracker **first** and returns the tracked instance without a query if it's there. `FirstOrDefault` always queries. `Find` is a nice micro-optimization in a loop, but it can't be composed or projected.

**E24. Does `Skip/Take` without `OrderBy` work?**
It runs, and it's **nondeterministic** — SQL has no inherent row order, so page 2 may repeat or skip rows from page 1. Always order by something unique (or a tiebreaker: `OrderBy(x => x.Date).ThenBy(x => x.Id)`). EF Core warns about this; a lot of pagination bugs are exactly this.

**E25. What SQL does `Contains` on a list generate?**
`WHERE Id IN (@p1, @p2, ...)`. Trap: a huge list means a huge parameter list — SQL Server caps at 2,100 parameters and the plan cache pollutes with a distinct plan per list size. EF Core 8 mitigates this on SQL Server by sending a JSON array/`OPENJSON` instead. Batch big lists or join to a temp table.

**E26. How do you page a large result set efficiently?**
Keyset over OFFSET: `WHERE (CreatedAt, Id) < (@last, @lastId) ORDER BY CreatedAt DESC, Id DESC LIMIT 20`. OFFSET 100000 makes the database *walk and discard* 100,000 rows every time. Knowing keyset cold is a real differentiator — see [05-sql-databases.md](05-sql-databases.md) `S38`.

---

## Writes & transactions

**E27. ⭐ Is `SaveChanges` atomic?**
Yes — EF wraps all its statements in a transaction (unless one is already ambient) and rolls back entirely on failure. So multiple entity changes in one `SaveChanges` are all-or-nothing. You only open an explicit transaction when **multiple `SaveChanges` calls** or non-EF work must commit together.

**E28. Show me the explicit transaction shape.**
```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
db.Bookings.Add(booking);
db.OutboxMessages.Add(new OutboxMessage(...)); // intent, same commit
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
```
And the point: booking + outbox row commit together, so a crash can't leave a booking with no email — the atomicity argument for the outbox in six lines.

**E29. Does EF batch statements?**
Yes — it groups multiple INSERT/UPDATE/DELETE into batched round-trips (configurable `MaxBatchSize`), so `SaveChanges` on 50 entities isn't 50 round-trips. But it's still row-by-row SQL, not a bulk copy: inserting 100k rows wants `SqlBulkCopy`/`COPY`/EFCore.BulkExtensions, not `AddRange`.

**E30. What's `DbUpdateException` vs `DbUpdateConcurrencyException`?**
The first wraps a database error on save — inspect `InnerException` for the provider error and map a unique-violation to a **409**. The second means the concurrency token didn't match: 0 rows affected where 1 was expected.

**E31. ⭐ How do you catch a unique-constraint violation cleanly?**
There's no provider-agnostic way — you check the provider error number (`SqlException.Number` 2601/2627, `PostgresException.SqlState` 23505). Wrap it once in your persistence layer and translate to a domain error → 409. The honest addendum: *"and I do catch it, because the pre-check has a race — the constraint is the guarantee, the catch is the UX."*

**E32. Connection resiliency?**
`EnableRetryOnFailure()` retries transient failures (Azure SQL loves those). The gotcha you must name: with a retrying **execution strategy**, a user-initiated transaction throws — because a retry would replay only part of it. Fix: `strategy.ExecuteAsync(async () => { ...transaction... })`. Nasty in production, easy point in an interview.

---

## Concurrency

**E33. ⭐ How does optimistic concurrency work in EF Core?**
Mark a property as a concurrency token (`[Timestamp]`/`rowversion` on SQL Server, `xmin` on Postgres). EF then appends it to the WHERE clause: `UPDATE ... WHERE Id=@id AND Version=@version`. If someone else changed the row, 0 rows match → EF throws `DbUpdateConcurrencyException`. No locks are held; the loser finds out at write time.

**E34. What do you do when you catch it?**
Depends on the domain, and saying that is the answer: **client wins** (reload originals, reapply), **store wins** (discard and reload), or **merge/ask the user**. For a booking: reload, re-check availability, retry once, then 409 with a clear message. `ex.Entries` gives you the entity + `GetDatabaseValuesAsync()` for the current row.

**E35. ⭐ Why doesn't a concurrency token prevent double-booking?**
Because **two INSERTs don't conflict on a version column** — there's no existing row to mismatch. The token protects updates to a row that exists; it does nothing about two people creating overlapping bookings. That's why you need the database-level guarantee: a unique index for exact duplicates, or a Postgres **exclusion constraint** (`EXCLUDE USING gist (provider_id WITH =, during WITH &&)`) for overlaps. This is your centerpiece — the insight that separates you from a candidate who read a tutorial.

**E36. Pessimistic locking in EF?**
No first-class API — you drop to raw SQL: `FromSql($"SELECT * FROM bookings WHERE id = {id} FOR UPDATE")` (Postgres) or `WITH (UPDLOCK, HOLDLOCK)` (SQL Server). Right when conflicts are frequent and retries are expensive; wrong for a web app's normal path because you're holding a lock across your own application logic.

---

## Migrations

**E37. ⭐ What is a migration, mechanically?**
A generated C# diff between the model snapshot and the current model, with `Up`/`Down` methods. Applied in order; applied names are recorded in `__EFMigrationsHistory` so the tool knows what's pending. It's a **diff of your model**, not of the database — which is why manual DB changes cause chaos.

**E38. `EnsureCreated` vs `Migrate`?**
`EnsureCreated` builds the schema from the model with **no migration history** — you can't evolve it later, and mixing the two corrupts the story. Tests/prototypes only. `Migrate()` applies pending migrations.

**E39. ⭐ How do migrations reach production?**
Options with tradeoffs — have a preference: (a) `Database.Migrate()` at startup — simple, but **racy with multiple instances** and it gives the app DDL rights forever; (b) a CI/CD step running `dotnet ef database update` or a **migration bundle** (a self-contained executable) — my default; (c) generate an idempotent SQL script (`--idempotent`) for a DBA to review — required in regulated shops. Always review the generated SQL for destructive operations.

**E40. Why review the generated SQL?**
Because EF can't read your mind: a rename looks like DROP + ADD, and it will **silently discard the column's data**. Also: adding a NOT NULL column with no default fails on a non-empty table, and an index creation can lock a big table. Those are the three that bite.

**E41. How do you do a zero-downtime schema change?**
Expand/contract: (1) add the new nullable column, deploy; (2) backfill + dual-write, deploy code reading both; (3) make it required, drop the old, deploy. Every intermediate state must work with **both** the old and new app version, because during a rolling deploy both are running. Knowing this is a strong mid+ signal.

**E42. Someone edited the DB by hand — how do you recover?**
Reconcile: make the model match reality, generate a migration, then delete the body of the `Up` if the DB is already there (or insert the row into `__EFMigrationsHistory` manually) so the history is honest. Then say the real fix: nobody hand-edits; the migration is the only path.

---

## Modeling

**E43. Fluent API vs data annotations?**
Annotations: quick, on the entity, limited. Fluent (`IEntityTypeConfiguration<T>` classes): full model surface (composite keys, indexes with filters, precision, delete behavior, converters), and it keeps persistence concerns **out of the domain type**. Preference + reason = the answer.

**E44. Owned types vs complex types vs a related entity?**
Owned (`OwnsOne`): a value object with no identity of its own, stored in the same table by default (an `Address` on a `Customer`). Complex types (EF 8): true value semantics, never tracked as an entity, no identity at all. Related entity: has its own key and lifecycle. Mapping `Money`/`Address` as owned/complex is the thing that makes a rich domain model survive an ORM.

**E45. Value converters?**
`HasConversion` to store a type differently from its CLR shape: an enum as a string, a strongly-typed `BookingId` as a Guid, a `DateOnly` on a provider that lacks it. Cost to name: EF can't always **translate** operations over a converted column, so an ill-chosen converter kills SQL-side filtering (a converted enum-as-string still compares fine; a JSON-serialized blob does not).

**E46. Cascade delete behaviors?**
`Cascade` (delete children), `Restrict` (throw), `SetNull` (orphan with null FK), `NoAction` (leave it to the DB), `ClientSetNull` (default for optional relationships — EF nulls **loaded** children only). The trap: EF's behavior applies to entities it has **tracked**; the database's FK rule applies to everything else. Two rule sets, and they can disagree.

**E47. Required vs optional relationships?**
A non-nullable FK = required = cascade delete by default. Nullable FK = optional = `ClientSetNull`. So making a FK nullable quietly changes your delete semantics — a fun one to be bitten by once.

**E48. `HasData` seeding — what's the catch?**
Model-level seed data becomes part of migrations, so it's applied by `Update-Database` and **diffed on every model change**. Requires explicit PKs, no dynamic values (`DateTime.Now` in a seed regenerates a migration every build). For anything real, seed with a startup routine or a SQL script instead.

---

## Performance & diagnostics

**E49. ⭐ How do you see the SQL EF generates?**
Log it: `LogTo(Console.WriteLine, LogLevel.Information)`, or the `Microsoft.EntityFrameworkCore.Database.Command` category at Information, plus `EnableSensitiveDataLogging()` **in development only** (it logs parameter values — that's PII/credentials in your logs otherwise). Then read the SQL. Cite your week-2 N+1 hunt: the query count in the log *is* the evidence.

**E50. Compiled queries?**
`EF.CompileAsyncQuery(...)` caches the expression-tree→SQL translation for a hot query. EF already caches plans by shape; compiled queries skip the tree-walk on top. A micro-optimization worth ~10–20% on very hot paths — reach for it with a profile, not a hunch.

**E51. Your endpoint is slow. What's your EF checklist?**
In order: (1) log the SQL — how many queries? (N+1) (2) how many columns/rows are coming back? (missing projection, no pagination) (3) `EXPLAIN` the actual query (missing index, seq scan) (4) is it tracked when it doesn't need to be? (5) is anything materializing early (`ToList` before filtering)? (6) cartesian explosion from Includes. Then measure again, and **add a regression test or an assertion on query count**.

**E52. Would you use the InMemory provider for tests?**
No, and this is a good opinion to hold. It's not a relational database: no constraints, no transactions, no raw SQL, no concurrency tokens — so it silently passes tests for the exact bugs you care about (your unique constraint, your 409). Use the real engine via Testcontainers, or SQLite for pure-logic cases while knowing it still differs. Microsoft themselves now recommend against it.

---

## Coverage tracker

`E1–E5` context · `E6–E13` change tracking · `E14–E26` querying ⭐ · `E27–E32` writes/transactions · `E33–E36` concurrency ⭐ · `E37–E42` migrations ⭐ · `E43–E48` modeling · `E49–E52` performance

**The four that decide the round:** E35 (why the version token doesn't stop double-booking — your story), E18 (`IQueryable` boundary), E39 (migrations to prod, with the multi-instance race named), E51 (the slow-endpoint checklist, in order, out loud).
