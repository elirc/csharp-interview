# SQL & Databases — 56 Questions (`S`)

⭐ The most common junior→mid rejection reason, and the cheapest to fix. [../drills/sql-30-day-plan.md](../drills/sql-30-day-plan.md) builds the typing muscle; this file builds the *talking* muscle — the "why did the planner do that" layer that separates someone who writes SQL from someone who owns a database.

Postgres-flavored (your stack) with SQL Server differences called out, because the interviewer's shop may run either.

Log misses as `S##`.

---

## Fundamentals

**S1. ⭐ Logical order of execution in a SELECT?**
`FROM` → `JOIN` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT`. This single fact answers three interview questions at once: why you **can't** use a SELECT alias in `WHERE` (the alias doesn't exist yet), why you **can** in `ORDER BY` (it does by then), and why `WHERE` filters rows but `HAVING` filters groups. Lead with this and the whole grouping topic collapses into one idea.

**S2. ⭐ INNER vs LEFT vs RIGHT vs FULL JOIN?**
Matches only · all left + matched right (NULLs where unmatched) · the mirror · both sides with NULLs on either. RIGHT JOIN exists mainly to confuse people — rewrite as LEFT. `CROSS JOIN` = every combination, used deliberately for calendars/matrices and accidentally by forgetting a join condition.

**S3. ⭐ "Customers with zero orders" — write it.**
```sql
SELECT c.id, c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name
HAVING COUNT(o.id) = 0;
```
The trap is `COUNT(*)`, which counts the NULL-padded row and returns **1** for the zero-order customers. `COUNT(o.id)` ignores NULLs. Also valid and often faster: `WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)`.

**S4. `COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)`?**
Rows · non-NULL values in col · distinct non-NULL values. `COUNT(*)` doesn't read any column — it's not slower than `COUNT(1)`, and that whole debate is folklore.

**S5. WHERE vs HAVING?**
Before grouping vs after aggregation (see S1). "Customers with >5 orders" → HAVING. "Orders from 2026" → WHERE. Filtering in WHERE is cheaper — fewer rows reach the grouping — so push predicates down whenever the semantics allow.

**S6. What must be in the GROUP BY?**
Every non-aggregated SELECT column. Postgres relaxes this when you group by the table's **primary key** (functional dependency) — MySQL historically let you do it *always* and returned arbitrary values, which is where the bad habit comes from.

**S7. `UNION` vs `UNION ALL`?**
`UNION` de-duplicates — which means a sort or hash over the whole result. `UNION ALL` just concatenates. Default to `UNION ALL` unless you actually need dedupe; using `UNION` out of habit is a silent, common performance tax.

**S8. `EXISTS` vs `IN` vs `JOIN` for "has at least one"?**
`EXISTS` expresses a semi-join (stop at the first match, no row multiplication). `IN` is usually planned identically — **except** for the NULL trap. `JOIN` can multiply rows if the right side has duplicates, silently inflating aggregates. Default to `EXISTS` for existence checks.

**S9. What's a correlated subquery, and why care?**
A subquery referencing the outer row, conceptually evaluated per row — the SQL version of N+1. Modern planners often rewrite it into a join, but not always. If it's slow, rewrite as a join or a lateral.

**S10. Self-join — when?**
Rows related to other rows in the same table: employee → manager, a booking overlapping another booking. `FROM bookings a JOIN bookings b ON a.provider_id = b.provider_id AND a.id <> b.id AND a.during && b.during` finds your overlaps — a good thing to be able to type on a whiteboard.

---

## NULL

**S11. ⭐ What is NULL, and why is `NULL = NULL` not true?**
NULL is "unknown," not a value. Comparing unknowns yields **unknown**, and `WHERE` keeps only rows that are *true* — so both `= NULL` and `<> NULL` return nothing. Use `IS NULL` / `IS NOT NULL`, or `IS DISTINCT FROM` when you want NULL-safe inequality.

**S12. ⭐ The `NOT IN` trap?**
`WHERE id NOT IN (SELECT customer_id FROM orders)` returns **zero rows** if any `customer_id` is NULL — because `id <> NULL` is unknown, so the whole condition can never be true. `NOT EXISTS` doesn't have this problem. This is the most-asked SQL gotcha in existence; know it cold.

**S13. NULLs in aggregates?**
Aggregates **skip** NULLs: `AVG(col)` averages over non-NULL rows only, so it isn't `SUM/COUNT(*)`. `SUM` of all-NULL is NULL, not 0 — wrap in `COALESCE(SUM(x), 0)` when a report needs a zero.

**S14. NULLs in ORDER BY?**
Postgres: NULLs sort **last** ascending, first descending (they're treated as largest). SQL Server: the opposite (smallest → first ascending). Portable answer: be explicit with `NULLS LAST`. That the two engines disagree is exactly the kind of thing that bites in a migration.

**S15. `COALESCE` vs `NULLIF` vs `ISNULL`?**
`COALESCE(a,b,c)` = first non-NULL (standard, n-ary). `NULLIF(a,b)` = NULL when a=b — the idiomatic divide-by-zero guard: `x / NULLIF(y,0)`. `ISNULL` is SQL Server's two-arg version; prefer `COALESCE`.

**S16. Unique constraint + NULLs — how many NULLs are allowed?**
Standard/Postgres: **many** (each NULL is unknown, so no two are "equal"). **SQL Server: exactly one** — a genuine, non-obvious engine difference and a great trivia question. Postgres 15+ added `NULLS NOT DISTINCT` to opt into SQL Server's behavior.

---

## Window functions & CTEs

**S17. ⭐ Top-N per group — type it.**
```sql
WITH ranked AS (
  SELECT p.category, p.name, SUM(oi.quantity) AS qty,
         ROW_NUMBER() OVER (PARTITION BY p.category ORDER BY SUM(oi.quantity) DESC) AS rn
  FROM products p JOIN order_items oi ON oi.product_id = p.id
  GROUP BY p.category, p.name
)
SELECT category, name, qty FROM ranked WHERE rn <= 3;
```
**The** mid-level SQL question. Note the two-layer structure: you can't filter on a window function in `WHERE` (S1 again — windows are computed at SELECT time, after WHERE), so it must be a CTE/subquery.

**S18. `ROW_NUMBER` vs `RANK` vs `DENSE_RANK`?**
Ties of 100,100,90: ROW_NUMBER → 1,2,3 (arbitrary tie order — nondeterministic unless you add a tiebreaker). RANK → 1,1,3 (gaps). DENSE_RANK → 1,1,2 (no gaps). "Top 3 with ties included" → RANK ≤ 3, not ROW_NUMBER.

**S19. What does `OVER (PARTITION BY x ORDER BY y)` mean?**
Compute per-row over a window: PARTITION = reset per group (like GROUP BY but **without collapsing rows**), ORDER = the ordering within the window. That's the whole superpower: you keep every row *and* get the aggregate next to it.

**S20. Running total — and the frame gotcha?**
`SUM(amount) OVER (ORDER BY ordered_at)`. The gotcha: with `ORDER BY` and no explicit frame, the default is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, and **RANGE includes all peer rows tied on the ordering key** — so two orders at the same timestamp both show the combined total. Want strict row-by-row? Say `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Beautiful trivia; almost nobody knows it.

**S21. `LAG`/`LEAD` — a real use?**
Previous/next row's value: gap since a customer's last order, `value - LAG(value) OVER (ORDER BY d)` for deltas, detecting state transitions in an audit table. Common in reporting asks.

**S22. What's a CTE, and is it a performance tool?**
A named subquery (`WITH x AS (...)`) — a **readability** tool. Postgres ≤11 always materialized CTEs (an optimization fence); **PG 12+ inlines** them when referenced once, with `MATERIALIZED`/`NOT MATERIALIZED` to force it. SQL Server always inlines. So "I used a CTE to make it faster" is wrong on both — the right claim is "to make it readable, and I checked the plan."

**S23. Recursive CTE — when?**
Hierarchies and graphs: org charts, category trees, dependency chains. Structure: anchor member `UNION ALL` recursive member referencing the CTE. Say "with a depth guard, because a cycle in the data is an infinite loop." SQL Server needs `MAXRECURSION` awareness (default 100).

---

## Indexes

**S24. ⭐ What is an index, physically?**
A B-tree: a sorted, balanced structure whose leaves point at rows (Postgres: the heap tuple's ctid; SQL Server nonclustered: the clustered key). Turns a full scan O(n) into a seek O(log n) plus the row fetch. That's it — everything else is a corollary.

**S25. ⭐ Why not index everything?**
Every INSERT/UPDATE/DELETE must maintain **every** index on the touched columns — write amplification. Plus storage, plus memory pressure in the buffer cache, plus more plans for the planner to weigh. And an index the planner won't use (low selectivity) is pure cost. Index for the queries you actually run, verify with EXPLAIN, and delete unused ones.

**S26. ⭐ Composite index `(a, b)` — which queries use it?**
Leftmost prefix: `WHERE a=?` ✅ · `WHERE a=? AND b=?` ✅ · `WHERE b=?` ❌ (it can't seek without `a`) · `WHERE a=? ORDER BY b` ✅ (already sorted — free ordering). Column order = your query patterns. Rule of thumb: equality columns first, then the range/sort column.

**S27. Why does a range column belong last?**
`(status, created_at)` with `WHERE status='open' AND created_at > x` seeks straight to the right slice. `(created_at, status)` makes the engine scan a whole date range and filter status per row. Once a column is used for a range, columns after it can't be used for seeking.

**S28. What's a covering index?**
One that contains **every column the query needs**, so the engine answers from the index and never touches the table — an "index-only scan" (PG) / "covered query" (SQL Server). SQL Server has `INCLUDE (cols)` for non-key payload columns. Postgres has `INCLUDE` too (11+), and its index-only scans also need the **visibility map** to be fresh, i.e. recently vacuumed — otherwise it still visits the heap.

**S29. Clustered vs nonclustered index?**
SQL Server: the **clustered** index *is* the table (rows stored in key order) — one per table; nonclustered indexes store the clustered key as a pointer, so a wide clustered key bloats every other index. Postgres: no clustered index at all — the heap is unordered and every index is secondary (`CLUSTER` is a one-time physical reorder that isn't maintained). This difference is a favorite of interviewers who've worked both.

**S30. Why are random GUID primary keys a problem?**
On SQL Server with a clustered PK, random values insert into the *middle* of the physical order → page splits, fragmentation, and a hot-page write pattern. Fixes: sequential GUIDs (`NEWSEQUENTIALID`), ULIDs/UUIDv7 (time-ordered), or an int identity PK with the GUID as a unique nonclustered column. Postgres feels this less (heap, appends), but wide random keys still hurt index locality and cache hit rates.

**S31. Are foreign keys indexed automatically?**
The **referenced** side needs a unique/PK index (required). The **referencing** column is **not** indexed automatically in either Postgres or SQL Server — you must do it. Consequences: slow joins, and slow/locking cascade deletes that scan the child table. "Index your FKs" is one of the highest-value cheap wins and a great thing to mention unprompted.

**S32. Partial / filtered index?**
`CREATE INDEX ... WHERE is_deleted = false` — index only the rows you query. Smaller, cheaper to maintain. Killer use with soft delete: a **partial unique index** `(email) WHERE deleted_at IS NULL` gives you "unique among live rows, reusable after deletion" — which a plain unique constraint cannot express. SQL Server calls it a filtered index.

**S33. When is a sequential scan the *right* plan?**
Small tables (an index adds a hop for nothing), low selectivity (you're returning 40% of rows — random I/O per row is slower than reading it all sequentially), or no useful index. A seq scan in a plan isn't automatically a bug; a seq scan over 10M rows to return 3 is.

**S34. ⭐ What makes a predicate non-SARGable?**
Wrapping the column in a function or arithmetic: `WHERE YEAR(created_at) = 2026`, `WHERE lower(email) = ?`, `WHERE price * 1.2 > 100`, `WHERE col LIKE '%foo'`. The index is on `col`, not `f(col)`, so it can't seek. Fixes: rewrite as a range (`created_at >= '2026-01-01' AND < '2027-01-01'`), or build an **expression index** (`CREATE INDEX ON users (lower(email))`). `LIKE 'foo%'` **is** SARGable; leading-wildcard is not (that's a trigram/full-text job).

**S35. Index types beyond B-tree?**
Postgres: **GIN** (JSONB, arrays, full-text — inverted), **GiST** (ranges, geometry — the one your exclusion constraint uses), **BRIN** (huge naturally-ordered tables, tiny index), Hash (equality only, rarely worth it). Naming GiST as the mechanism behind your no-overlap constraint ties your project story to real database internals.

---

## Plans & performance

**S36. ⭐ `EXPLAIN` vs `EXPLAIN ANALYZE`?**
EXPLAIN = the planner's estimate, doesn't run the query. EXPLAIN ANALYZE = **actually executes it** and reports real times and row counts. Two things to say: (1) on an UPDATE/DELETE, ANALYZE really does modify data — wrap it in `BEGIN; ... ROLLBACK;`; (2) the gold is comparing **estimated vs actual rows** — a big divergence means stale statistics, and the planner made a bad choice from bad data.

**S37. Plan operators you should recognize?**
`Seq Scan` (read it all), `Index Scan` (seek + heap fetch per row), `Index Only Scan` (covered — no heap), `Bitmap Heap Scan` (many matches: collect ctids, then read the heap in physical order). Joins: `Nested Loop` (good when the outer side is tiny), `Hash Join` (big unsorted sets — builds a hash of the smaller side), `Merge Join` (both inputs already sorted). SQL Server's equivalents read the same. Rule: a nested loop over a large outer side is the classic disaster.

**S38. ⭐ OFFSET pagination at page 1000 — what's wrong?**
`OFFSET 20000 LIMIT 20` makes the database produce and **discard 20,000 rows** every single time — cost grows linearly with page number. It's also unstable: an insert on page 1 shifts everything, so users see duplicates while paging. Keyset/cursor: `WHERE (created_at, id) < (@lastDate, @lastId) ORDER BY created_at DESC, id DESC LIMIT 20` — constant time, stable, index-friendly. Tradeoff to volunteer: **no random jump to page 500**, which is why "load more" UIs use it and "page 1 2 3" UIs don't.

**S39. What are statistics and why do plans go bad overnight?**
The planner estimates row counts from sampled distribution data. Stale stats after a bulk load → wrong estimates → wrong join strategy → a query that ran in 10ms takes 40s with **no code change**. Fix: `ANALYZE` / update statistics (autovacuum usually does it). This is the answer to "the query was fine yesterday."

**S40. Parameter sniffing (SQL Server)?**
The plan is compiled for the **first** parameter values it saw and cached — great for that value, catastrophic for a skewed one (a plan built for `status='rare'` reused for `status='common'`). Mitigations: `OPTIMIZE FOR UNKNOWN`, `RECOMPILE`, or restructure. Postgres has a milder version via generic vs custom plans for prepared statements. Naming this is senior-flavored.

**S41. Same query, fast in the tool and slow from the app — why?**
Classic causes: different **parameters** (sniffing), different session settings (SET options change plan cache entries in SQL Server), the app sends `nvarchar` against a `varchar` column so the predicate becomes non-SARGable (an implicit-conversion scan!), or it isn't the query at all — it's connection pool exhaustion, N+1, or serialization time. A structured list here reads as experience.

**S42. Your query got slow with production data — checklist?**
Reproduce with real volume → EXPLAIN ANALYZE → estimates vs actuals → is there an index for this predicate/sort → is the predicate SARGable → how many rows are we shipping (pagination? projection?) → is it N+1 from the app → check locks/waits if it's intermittent. Then measure again. **Sequence, not vibes** — same as your debugging drill.

---

## Transactions, isolation, locking

**S43. ⭐ ACID in one breath?**
Atomicity (all or nothing), Consistency (constraints hold at commit), Isolation (concurrent transactions don't corrupt each other's view), Durability (committed = survives a crash, via the write-ahead log). Then the practical line: *"the one I actually reason about weekly is isolation — the other three the engine just gives me."*

**S44. ⭐ The anomalies, cheapest to strictest?**
**Dirty read**: seeing another transaction's uncommitted data. **Non-repeatable read**: the same row read twice differs (someone updated it). **Phantom read**: the same *query* returns new rows (someone inserted). **Lost update / write skew**: two transactions each read a consistent state and write results that are impossible together — your double-booking, exactly. Levels: Read Uncommitted → Read Committed → Repeatable Read → Serializable, each preventing more, each conceding concurrency.

**S45. What's the default isolation level, really?**
**Read Committed** in both Postgres and SQL Server — you never see uncommitted data, but the same query re-run in the same transaction can change. Engine difference worth knowing: Postgres implements it with **MVCC snapshots** (readers never block writers); SQL Server's default uses **shared locks** (readers *do* block writers) unless RCSI (`READ_COMMITTED_SNAPSHOT`) is on, which makes it snapshot-based like Postgres. "Why does my read block?" is a SQL Server question and a non-question in PG.

**S46. Repeatable Read in Postgres — a surprise?**
It's snapshot isolation, so it prevents **phantoms too**, which the standard doesn't require. Instead of blocking, a conflicting write fails at commit with a serialization error — **so your app must be prepared to retry**. Anything above Read Committed in Postgres means writing retry logic; that's the real cost.

**S47. ⭐ Serializable — why not just always use it?**
It's correct by definition (results equal *some* serial order), and it's implemented in Postgres via SSI, which **aborts** transactions with `40001` on conflict rather than locking. So you trade lock contention for retry storms, plus real throughput loss under contention. Your answer: *"I solved my booking overlap with an exclusion constraint and a 409, because a targeted constraint gives me the invariant at read-committed with no retry loop — serializable would've been a much bigger hammer for one invariant."* That's a hire-level tradeoff sentence.

**S48. ⭐ What's a deadlock, and how do you avoid one?**
Two transactions each hold a lock the other needs; the engine detects the cycle and kills a victim (SQL Server error 1205; PG 40P01). Avoid: **acquire resources in a consistent order** (the #1 cause is two code paths locking A→B and B→A), keep transactions short, don't hold locks across app logic or network calls, index enough that you lock rows instead of ranges/tables. Then: retry the victim — deadlocks are normal at load, not exotic.

**S49. Pessimistic locking in SQL?**
`SELECT ... FOR UPDATE` (PG) / `WITH (UPDLOCK, HOLDLOCK)` (SQL Server) — take the row lock during the read so nobody else can. Right when conflicts are common and retries expensive; wrong when you'd hold it across a long request. Its virtue over optimistic: no lost work. Its vice: contention, and deadlock risk.

**S50. `FOR UPDATE SKIP LOCKED` — what's it for?**
A work queue in the database: each worker grabs unlocked rows and skips ones other workers hold, so N workers pull disjoint batches with no coordination. SQL Server: `READPAST`. This is exactly how you'd scale your outbox poller past one instance — a great, concrete answer to "how would this scale?"

**S51. Lock escalation?**
SQL Server escalates many row locks into a table lock (around 5,000 locks on one object) — so a big UPDATE can lock out the whole table and turn a slow query into an outage. Batch large modifications. **Postgres does not escalate**, it just keeps row locks. Another engine difference worth carrying.

**S52. Does a transaction roll back a sequence/identity value?**
No. Sequences are **non-transactional** by design (otherwise every insert would serialize on the counter), so a rollback leaves a permanent gap. Consequence: never expose an auto-increment id as a "count," and never use it for gapless invoice numbers — that needs a separate, deliberately serialized allocation.

---

## Schema & operations

**S53. ⭐ Why can't application code guarantee uniqueness?**
`SELECT ... IF NOT EXISTS THEN INSERT` is check-then-act with a race window between the two statements: under two concurrent requests, both checks pass, both insert. Only a **constraint is atomic with the write**, because the engine evaluates it inside the same operation. App check = good error message; constraint = correctness. You cannot say this too often — it's the load-bearing idea of your whole portfolio project.

**S54. Normalization — 1NF/2NF/3NF, and when do you denormalize?**
1NF: atomic values, no repeating groups. 2NF: no partial dependency on part of a composite key. 3NF: no transitive dependency (non-key → non-key). Practical framing: *"normalize by default so there's one place to change a fact; denormalize deliberately for read performance — a cached counter, a stored total — and then own the invalidation. I'd rather add an index than a denormalized column."*

**S55. UPSERT — how, and the catch?**
Postgres: `INSERT ... ON CONFLICT (col) DO UPDATE SET ...` — atomic, needs a unique constraint to conflict on, and it's the clean way to make an insert idempotent (retry-safe). SQL Server: `MERGE` exists but has a long, well-documented history of concurrency bugs and gotchas — many practitioners avoid it in favor of an explicit `UPDATE`-then-`INSERT` under a lock or a `try INSERT / catch dup` pattern. Knowing MERGE is contested is nice trivia.

**S56. Read replicas — what breaks?**
Replication lag. You write, immediately redirect the read to a replica, and the row **isn't there yet** — "I saved it and it vanished." Fixes: read-your-own-writes routed to the primary for a window, sticky sessions, or accepting staleness where it's harmless (reports, dashboards). The mid-level point is that a replica is an *eventual consistency* decision, not a free performance knob.

---

## Coverage tracker

`S1–S10` fundamentals ⭐ · `S11–S16` NULL ⭐ · `S17–S23` windows/CTEs ⭐ · `S24–S35` indexes ⭐ · `S36–S42` plans/perf ⭐ · `S43–S52` transactions/locking ⭐ · `S53–S56` schema/ops

**The six that decide the round:** S3 (zero-orders LEFT JOIN), S12 (`NOT IN` + NULL), S17 (top-N per group, typed cold), S26 (leftmost prefix), S38 (keyset pagination), S53 (why app code can't guarantee uniqueness).

**Weekly maintenance:** type S3, S17, and one EXPLAIN read from memory. Speaking SQL fluently and *typing* it under observation are different skills — the interview tests the second one.
