# Rapid-Fire Trivia — 270 Flashcards (`R`)

Recall, not reasoning. **Target: under 3 seconds each, spoken.** If you're reasoning it out, it isn't fluent — and the hesitation is exactly what a screener hears.

Format: `Question → Answer`. Cover the right column with your hand, or run the Mode 1 prompt from [00-overview.md](00-overview.md).

Use as a **10-minute daily warm-up**, and in the car on the way to the interview. This is the cheapest habit in the whole curriculum.

---

## C# keywords & syntax

1. `const` vs `readonly` → compile-time, baked into callers vs runtime, set in ctor
2. `readonly` field vs `readonly struct` → can't reassign field vs can't mutate any field
3. `static` constructor runs when → before first use, once, thread-safe, lazily
4. `sealed` → can't be inherited; enables JIT devirtualization
5. `partial` → one type split across files (source generators, designer files)
6. `virtual`/`override` → runtime dispatch on actual type
7. `new` on a method → hides; dispatch by compile-time type
8. `abstract` method → no body, must override
9. `internal` → visible within the assembly
10. `protected internal` → assembly **OR** derived
11. `private protected` → assembly **AND** derived
12. `ref` → pass by reference, caller must initialize
13. `out` → pass by reference, callee must assign
14. `in` → pass by readonly reference (perf for big structs)
15. `params` → variable args; allocates an array per call
16. `yield return` → lazy iterator; compiler builds a state machine
17. `using` statement → try/finally calling Dispose
18. `using` declaration (C# 8) → `using var x = ...;` disposes at scope end
19. `nameof` → compile-time string of the symbol; survives renames
20. `is` vs `as` → test (+pattern) vs convert-or-null
21. `??` → null-coalescing; `??=` → assign if null
22. `?.` → null-conditional, short-circuits the whole chain
23. `!` (postfix) → null-forgiving; compile-time only, zero runtime effect
24. `@` prefix on identifier → escape a keyword (`@class`)
25. `$""` → interpolation; `@""` → verbatim; `"""` → raw string literal (C# 11)
26. `=>` on a member → expression-bodied
27. `_` → discard
28. `record` → value equality + `with` + deconstruct
29. `with` expression → non-destructive copy
30. `init` → settable only during object initialization
31. `required` (C# 11) → the compiler forces the initializer to set it
32. `global using` (C# 10) → one using for the whole project
33. `file` modifier (C# 11) → type visible only in that file
34. `checked`/`unchecked` → overflow throws vs wraps (default: unchecked)
35. `nint`/`nuint` → native-sized integer
36. `stackalloc` → allocate on the stack; use with `Span<T>`
37. `unsafe`/`fixed` → pointers; pin memory so the GC can't move it
38. `volatile` → prevents reordering; does **not** make anything atomic
39. `lock(x)` → syntactic sugar for `Monitor.Enter/Exit` in try/finally
40. `default` literal → `default(T)` with the type inferred

## Types & runtime

41. Value type assignment → copies the value
42. Reference type assignment → copies the reference
43. Boxing → wrapping a value type in a heap object
44. Unboxing → copy back out; can throw `InvalidCastException`
45. Do structs live on the stack? → not necessarily; a struct field of a class lives on the heap
46. `Nullable<T>` → a struct with `HasValue`/`Value`
47. `null > 5` → **false**; and `null <= 5` is also false
48. NRT (`string?`) at runtime → identical type; compile-time analysis only
49. Default of an enum → 0, even if no member has that value
50. Default `Equals` on a struct → field comparison, via reflection if non-blittable (slow)
51. Is `GetHashCode` stable across processes → **no** (randomized string hashing)
52. Equal objects must have → equal hash codes
53. Immutable + interned → string literals
54. String concat in a loop → O(n²); use StringBuilder
55. `string.Empty` vs `""` → identical
56. `decimal` → base-10, 28–29 digits, exact for money
57. `double` → binary float; `0.1 + 0.2 != 0.3`
58. Money in a database → integer minor units (cents)
59. `DateTime.Kind` values → Utc, Local, **Unspecified**
60. `DateTimeOffset` → an instant with an offset (unambiguous)
61. `DateOnly`/`TimeOnly` → .NET 6; no time zone involved
62. Offset vs time zone → `+02:00` vs `Europe/Berlin` (a rule set that changes)
63. `int.MaxValue + 1` → wraps to `int.MinValue` (unchecked)
64. LOH threshold → **85,000 bytes**
65. GC generations → 0, 1, 2 (+ LOH); most objects die young
66. Finalizer cost → non-deterministic, promotes a generation, one finalizer thread
67. `GC.SuppressFinalize` → call in Dispose so the finalizer is skipped
68. Can .NET leak memory? → yes: statics, **unremoved event handlers**, timers, undisposed handlers
69. `IDisposable` is for → unmanaged/expensive resources, not memory
70. `SafeHandle` → the correct wrapper for an unmanaged handle
71. JIT tiered compilation → tier 0 compiles fast, tier 1 re-optimizes hot code
72. ReadyToRun → precompiled to native; faster startup, bigger binary
73. Native AOT → no JIT at all; fastest startup, no runtime codegen/limited reflection
74. Server GC vs Workstation → per-core heaps, throughput (ASP.NET default) vs latency
75. `Span<T>` → stack-only view; can't cross an `await`, can't be a class field
76. `Memory<T>` → the heap-friendly, async-safe counterpart

## LINQ & collections

77. `IEnumerable` vs `IQueryable` → delegates in memory vs an expression tree translated by a provider
78. Deferred execution → runs at enumeration, not definition
79. Operators that execute immediately → `To*`, `Count`, `Any`, `First*`, `Single*`, `Sum`, `Contains`
80. `First` vs `Single` → first match vs exactly one (throws on 2+)
81. `FirstOrDefault` on an empty int sequence → `0`, not null
82. `Any()` vs `Count() > 0` → stops at the first element vs may enumerate all
83. `Select` vs `SelectMany` → map vs flatten
84. `OrderBy` in LINQ-to-Objects → **stable**; `List.Sort` → not stable
85. `List<T>` growth → doubles the array and copies
86. `Dictionary` lookup → O(1) average, O(n) worst
87. Is `Dictionary` thread-safe? → no; concurrent writes can corrupt it into an infinite read loop
88. `ConcurrentDictionary.GetOrAdd` catch → the value factory can run more than once
89. Fix for that → store `Lazy<T>` as the value
90. `ImmutableList` → copy-on-write, structurally shared
91. `FrozenDictionary` (.NET 8) → slow to build, fastest reads; build-once lookup tables
92. `Chunk(n)` → batch a sequence (.NET 6)
93. `DistinctBy`/`MaxBy`/`MinBy` → .NET 6
94. Mutating a dictionary key's hash after insert → the entry becomes unfindable
95. Removing from a `List` while foreach-ing → `InvalidOperationException`

## Async

96. `await` → returns to the caller and registers a continuation
97. Which thread does async I/O → **none**; completion arrives via an I/O completion port
98. Does async make a request faster → no; it increases throughput
99. Does async help CPU-bound work → no
100. `Task` vs `Thread` → a promise vs an OS resource (~1MB stack)
101. `Task.Run` for I/O → wrong; it blocks a pool thread instead of freeing one
102. `async void` → only for event handlers; exceptions crash the process
103. `.Result` in classic ASP.NET → deadlock (SynchronizationContext)
104. `.Result` in ASP.NET Core → no deadlock; **thread-pool starvation** under load
105. Starvation's tell → latency climbs while **CPU stays low**
106. `ConfigureAwait(false)` in ASP.NET Core app code → no effect (no SyncContext)
107. Where it does matter → libraries
108. `ValueTask` → struct; for hot paths that usually complete synchronously
109. `ValueTask` rules → await exactly once, never store it
110. `Task.WhenAll` exceptions → `await` rethrows only the first
111. `WhenAny` + `Task.Delay` for timeout → leaks the original task; use `WaitAsync`
112. Unobserved task exception → silently swallowed
113. `Task.Delay` vs `Thread.Sleep` → timer + continuation vs a blocked thread
114. Cancellation model → **cooperative**; nothing is forcibly aborted
115. Token in a controller → bind `CancellationToken` → `HttpContext.RequestAborted`
116. `OperationCanceledException` → expected, not an error; don't log at Error
117. Why can't you `lock` across an `await` → `Monitor` is thread-affine
118. Async lock → `SemaphoreSlim.WaitAsync()` + release in `finally`
119. `lock(this)` → bad; the lock object is publicly reachable
120. Is `i++` atomic → no; use `Interlocked.Increment`
121. `Interlocked.CompareExchange` → atomic "set if equal" — optimistic concurrency in memory
122. `volatile` makes ops atomic? → **no**
123. Deadlock prevention → acquire locks in a consistent order; keep them short
124. Concurrency vs parallelism → interleaved progress vs simultaneous execution
125. `Parallel.ForEachAsync` → .NET 6; I/O fan-out with `MaxDegreeOfParallelism`
126. `Channel<T>` → async producer/consumer with backpressure; **lost on restart**
127. Is `DbContext` thread-safe → no: "a second operation was started on this context"
128. Scoped service in a singleton → inject `IServiceScopeFactory`, scope per unit of work

## ASP.NET Core

129. Pipeline order → exceptions → HSTS/HTTPS → static files → routing → CORS → authN → authZ → endpoint
130. authN before authZ because → authorization needs an identity **and** the matched endpoint
131. Transient / Scoped / Singleton → per resolve / per request / per app
132. Captive dependency → a longer-lived service capturing a shorter-lived one
133. Scope validation → on by default in Development only
134. Is middleware a singleton → yes; inject scoped deps via `InvokeAsync` parameters
135. `Use` vs `Run` vs `Map` → may call next / terminal / branch
136. Forgot to `await next()` → every request short-circuits with an empty 200
137. `IOptions` → singleton, never reloads
138. `IOptionsSnapshot` → scoped, per-request, **can't go in a singleton**
139. `IOptionsMonitor` → singleton + change notifications; the one for background services
140. `ValidateOnStart` → config errors fail at boot, not at 3am
141. Config precedence → appsettings → appsettings.{Env} → user-secrets (Dev) → env vars → CLI
142. Env var nesting → double underscore: `ConnectionStrings__Default`
143. `[ApiController]` gives you → auto-400, binding inference, attribute routing required, ProblemDetails
144. Only one `[FromBody]` because → the body is a forward-only stream, read once
145. ProblemDetails → RFC 7807/9457 standard error body
146. `new HttpClient()` per request → socket exhaustion (TIME_WAIT)
147. One static HttpClient → **stale DNS** forever
148. `IHttpClientFactory` → pools + rotates handlers (default 2 min) → both problems solved
149. Default `HttpClient.Timeout` → **100 seconds**
150. Retry only → idempotent operations; with backoff **and jitter**
151. Liveness vs readiness → restart me vs take me out of the load balancer
152. DB check in a liveness probe → restart-loops the whole fleet when the DB blips
153. Output caching vs response caching → server stores it vs client/proxy is asked to
154. Cache stampede → hot key expires, everything misses at once
155. Rate limit response → **429** + `Retry-After`
156. `AllowAnyOrigin` + credentials → illegal per spec; name the origins
157. CORS protects → users in browsers, not your API (curl ignores it)
158. `IHttpContextAccessor` in a background service → null (no request)
159. Data Protection keys on multiple instances → cookies rejected across nodes unless shared
160. .NET 9 OpenAPI → built-in `AddOpenApi()`; Swashbuckle dropped from the template

## EF Core

161. `DbContext` lifetime → scoped (one unit of work per request)
162. `SaveChanges` atomicity → implicit transaction; all or nothing
163. When to open an explicit transaction → multiple SaveChanges must commit together
164. Change tracking mechanism → snapshot of original values, diffed on save
165. `AsNoTracking` → no snapshot, no identity map, no SaveChanges
166. `AsNoTracking` on a projection to a DTO → **no-op** (projections aren't tracked)
167. `Update(entity)` → marks **every** property modified
168. N+1 → 1 list query + N per-row queries
169. Detect N+1 → log the SQL and count the queries
170. Cartesian explosion → two collection Includes multiply rows
171. `AsSplitQuery` cost → multiple round-trips; **no single atomic snapshot**
172. Lazy loading needs → `virtual` navigations + the proxies package
173. `.ToList()` before `.Where()` → downloads the table and filters in C#
174. Client evaluation since EF Core 3 → throws (except in the final projection)
175. `Skip/Take` without `OrderBy` → nondeterministic paging
176. Concurrency token SQL → `UPDATE ... WHERE Id=@id AND Version=@v` → 0 rows → throw
177. Postgres concurrency token → `xmin`; SQL Server → `rowversion`
178. Why a version token doesn't stop double-booking → **two INSERTs don't conflict**
179. What does → a unique index, or a Postgres **exclusion constraint** for overlaps
180. `__EFMigrationsHistory` → the table recording applied migrations
181. `EnsureCreated` → no migration history; not upgradable
182. EF renames a column as → DROP + ADD (data loss) unless you edit the migration
183. `ExecuteUpdate`/`ExecuteDelete` → EF 7; bypasses the change tracker, runs immediately
184. InMemory provider for tests → no constraints, no transactions; use Testcontainers
185. `EnableSensitiveDataLogging` → dev only (it logs parameter values)

## SQL

186. Logical order → FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
187. Why no SELECT alias in WHERE → WHERE runs before SELECT
188. WHERE vs HAVING → rows before grouping vs groups after
189. Customers with zero orders → LEFT JOIN + `COUNT(o.id) = 0` (not `COUNT(*)`)
190. `COUNT(*)` vs `COUNT(col)` → rows vs non-NULL values
191. `NULL = NULL` → unknown, so not true; use `IS NULL`
192. `NOT IN (subquery with a NULL)` → returns **zero rows**; use `NOT EXISTS`
193. Aggregates and NULL → skipped; `AVG` ignores them
194. Unique index + NULLs → many allowed in Postgres, **exactly one** in SQL Server
195. `UNION` vs `UNION ALL` → dedupes (costs a sort/hash) vs concatenates
196. Top-N per group → `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` in a CTE
197. `ROW_NUMBER`/`RANK`/`DENSE_RANK` on 100,100,90 → 1,2,3 / 1,1,3 / 1,1,2
198. Running total default frame gotcha → `RANGE` includes tied peers; use `ROWS`
199. Composite index `(a,b)` serves → `a`, and `a AND b`; **not `b` alone**
200. Non-SARGable → a function on the column: `WHERE YEAR(d) = 2026`
201. `LIKE 'foo%'` vs `'%foo'` → index seek vs no index
202. Covering index → contains every column the query needs → index-only scan
203. Clustered index → SQL Server: the table itself; Postgres: doesn't have them
204. Are FKs indexed automatically → **no** (neither engine) — index them
205. Partial/filtered unique index → uniqueness among live rows only (soft delete)
206. `EXPLAIN` vs `EXPLAIN ANALYZE` → estimate vs actually runs it
207. Best signal in a plan → estimated vs actual rows diverging → stale statistics
208. OFFSET 20000 → the DB produces and discards 20,000 rows
209. Keyset pagination → `WHERE (sort_key, id) < (@last, @lastId) ORDER BY ... LIMIT n`
210. Default isolation level → Read Committed (both engines)
211. Dirty / non-repeatable / phantom → uncommitted data / row changed / new rows appear
212. Postgres serializable failure code → `40001` → **your app must retry**
213. Deadlock → cycle of lock waits; the engine kills a victim; retry it
214. Deadlock prevention → consistent lock ordering
215. `FOR UPDATE SKIP LOCKED` → a work queue in the DB; N workers, disjoint batches
216. Does a rollback reclaim a sequence value → **no**; gaps are permanent
217. Postgres UPSERT → `INSERT ... ON CONFLICT DO UPDATE`
218. Why app code can't guarantee uniqueness → check-then-act has a race window
219. Read replica risk → replication lag → read-your-own-writes fails

## HTTP & security

220. Idempotent verbs → GET, PUT, DELETE, HEAD, OPTIONS (not POST/PATCH)
221. Safe verb → GET (no state change — proxies may replay it)
222. 201 must include → a `Location` header
223. 202 → accepted, work is queued (the background-job answer)
224. 401 vs 403 → who are you vs you can't
225. 400 vs 422 → malformed vs semantically impossible
226. 409 → state conflict (duplicate, concurrent edit, slot taken)
227. 429 → rate limited; send `Retry-After`
228. 502 vs 504 → upstream sent garbage vs upstream timed out
229. `no-cache` vs `no-store` → revalidate every time vs never store at all
230. ETag + `If-None-Match` → 304 Not Modified
231. ETag + `If-Match` on PUT → optimistic concurrency over HTTP → **412**
232. Idempotency key → client-supplied id + a unique constraint → safe retries
233. Exactly-once delivery → doesn't exist; at-least-once + idempotent handlers
234. JWT structure → `header.payload.signature`, base64url
235. Is a JWT encrypted → **no**, only signed — never put secrets in claims
236. Validate on a JWT → signature, issuer, audience, lifetime, algorithm
237. `alg: none` → the classic forgery attack
238. JWT revocation → not possible before expiry → short expiry + refresh tokens
239. localStorage vs httpOnly cookie → XSS-readable vs CSRF-manageable
240. CSRF applies to → ambient credentials (cookies), not `Authorization` headers
241. `SameSite` default in modern browsers → Lax
242. Password storage → Argon2id/bcrypt/PBKDF2 + per-password salt; slow is the feature
243. Salt vs pepper → per-password, stored with the hash vs app-wide, stored elsewhere
244. Compare secrets with → `CryptographicOperations.FixedTimeEquals`
245. Random for tokens → `RandomNumberGenerator`, never `System.Random`
246. Mass assignment fix → input DTOs; server-controlled fields never from the body
247. IDOR fix → scope the query by the caller (`&& b.CustomerId == userId`)
248. #1 real-world API vuln → **A01 broken access control**
249. SSRF → your server fetches an attacker-chosen URL (cloud metadata!)
250. Dynamic ORDER BY → can't be parameterized → **allowlist**
251. Leaked secret → **rotate**, don't just delete

## Tooling & CLI

252. Create/build/run/test → `dotnet new webapi` / `build` / `run` / `test`
253. Add a migration → `dotnet ef migrations add Name`
254. Apply migrations → `dotnet ef database update`
255. Migration SQL script → `dotnet ef migrations script --idempotent`
256. Dev secret → `dotnet user-secrets set "Key" "value"`
257. Vulnerable packages → `dotnet list package --vulnerable --include-transitive`
258. Hot reload on save → `dotnet watch`
259. Pin the SDK version → `global.json`
260. Shared MSBuild settings for a tree → `Directory.Build.props`
261. Central package versions → `Directory.Packages.props` + `ManagePackageVersionsCentrally`
262. Reproducible restores → `packages.lock.json`
263. Diagnostics tools → `dotnet-counters`, `dotnet-trace`, `dotnet-dump`, `dotnet-gcdump`
264. Docker: image vs container → the class vs the instance
265. `EXPOSE` → documentation only; `-p` publishes
266. Multi-stage build → SDK to build, ASP.NET runtime to ship
267. Compose networking → connect by **service name**, not localhost
268. Revert vs reset on a shared branch → revert
269. Recover a lost commit → `git reflog`
270. Find the commit that broke it → `git bisect` (+ `bisect run`)

---

## Version cheat sheet

Being able to say "that's C# 12" or "that shipped in .NET 8" is a cheap currency signal. Don't over-claim — knowing the *shape* is enough.

| C# | .NET | Headline features |
|---|---|---|
| 8 | Core 3.0 | **Nullable reference types**, async streams (`IAsyncEnumerable`), switch expressions, ranges/indices, `using` declarations, default interface methods |
| 9 | 5 | **Records**, `init`, top-level statements, target-typed `new`, source generators |
| 10 | 6 | Global usings, file-scoped namespaces, `record struct`; **.NET 6: minimal APIs, `DateOnly`/`TimeOnly`, LTS** |
| 11 | 7 | Raw string literals, `required` members, generic math, list patterns; **.NET 7: rate limiting, output caching, EF `ExecuteUpdate`** |
| 12 | 8 | **Primary constructors**, collection expressions `[1,2,3]`, alias any type; **.NET 8: keyed DI, `TimeProvider`, frozen collections, `IExceptionHandler`, LTS** |
| 13 | 9 | `params` collections, `System.Threading.Lock`, `field` (preview); **.NET 9: built-in OpenAPI, `HybridCache`** |
| 14 | 10 | **Extension members**, `field` keyword, null-conditional assignment; **.NET 10: LTS** |

**Support policy:** even-numbered releases are **LTS** (3 years); odd are **STS** (18 months). .NET 8 (Nov 2023) and .NET 10 (Nov 2025) are the LTS ones you'll meet in the wild. If asked what you'd target for a new app: "the current LTS, unless there's a reason."

---

## Numbers worth knowing

Quoting one of these accurately in a design conversation is disproportionately convincing.

| Thing | Number |
|---|---|
| Large Object Heap threshold | **85,000 bytes** |
| Default `HttpClient.Timeout` | **100 seconds** |
| `IHttpClientFactory` handler lifetime | 2 minutes |
| Default DB command timeout | 30 seconds |
| Default host shutdown timeout | 30 seconds |
| Kestrel default max request body | ~30 MB |
| ADO.NET / Npgsql default max pool size | 100 connections |
| JWT default clock skew | 5 minutes |
| ASP.NET Core Identity password hash | PBKDF2-HMAC-SHA256, 100k iterations |
| SQL Server page size | 8 KB |
| SQL Server max parameters per query | 2,100 |
| SQL Server lock escalation threshold | ~5,000 locks |
| Thread pool minimum threads | = processor count |
| Default ports | Postgres 5432 · SQL Server 1433 · Redis 6379 |

**Latency ladder** (order of magnitude — the *ratios* are the point):

| Operation | ~Time | Relative |
|---|---|---|
| L1 cache reference | 1 ns | 1× |
| Main memory reference | 100 ns | 100× |
| SSD random read | ~100 µs | 1,000× memory |
| Indexed DB query (warm, local) | ~1 ms | 10× SSD |
| Network round trip in a datacenter | ~0.5 ms | — |
| Round trip US ↔ Europe | ~150 ms | **1.5M× memory** |

The takeaway to *say*: an in-datacenter round trip costs about as much as thousands of memory reads — **which is why N+1 is catastrophic and why the fix is one query, not a faster server.**

---

## How to drill this file

- **Daily, 10 minutes**, out loud, in order. Rotate the starting section so the back half isn't always the tired half.
- Mark misses. A miss goes into the [12-code-output-gotchas.md](12-code-output-gotchas.md) mindset: don't re-read the answer — go find out *why*.
- **Speed is the metric.** These are the questions where thinking looks like not knowing. Save your thinking for the questions that deserve it.
