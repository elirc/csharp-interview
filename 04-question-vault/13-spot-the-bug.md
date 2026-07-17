# Spot the Bug — 24 Review Snippets (`B`)

The closest thing in this vault to a real interview round. Many practical shops replace whiteboarding entirely with **"review this PR out loud"** — and they're not grading whether you find every bug. They're grading:

1. Do you find the **worst** one first? (Security > correctness > performance > style)
2. Can you explain the **mechanism**, not just "that looks wrong"?
3. Do you say what you'd **do about it** — fix, test, or ask?

**Narrate every one of these aloud.** Say the sentence you'd say to the interviewer. Silent reading trains the wrong muscle — see [../03-phase3-interviews/04-live-coding-drills.md](../03-phase3-interviews/04-live-coding-drills.md).

Each snippet has **exactly one primary bug** (a couple have a bonus). Answers are inline — cover them.

---

**B1.**
```csharp
public class NotificationService : INotificationService
{
    private readonly AppDbContext _db;
    public NotificationService(AppDbContext db) => _db = db;
    public async Task NotifyAsync(int userId) { /* ... */ }
}
builder.Services.AddSingleton<INotificationService, NotificationService>();
```
**Bug:** captive dependency — a **singleton** capturing a **scoped** `DbContext`.
**Mechanism:** the context is resolved once and held for the app's lifetime: shared across all requests (and it isn't thread-safe), the change tracker grows forever, and one broken connection breaks it permanently.
**Fix:** make the service scoped, or inject `IServiceScopeFactory` and create a scope per unit of work.
**Say this:** *"In Development the container's scope validation would throw at startup — so this either isn't being run locally, or validation is off. That's worth knowing either way."*

---

**B2.**
```csharp
[HttpPost]
public async void Create(CreateBookingDto dto)
{
    await _service.CreateAsync(dto);
}
```
**Bug:** `async void` on an action.
**Mechanism:** the framework can't await it, so the response may complete before the work does; and an exception inside it can't be caught by the pipeline — it's raised on the sync context and can **kill the process**. Your exception middleware never sees it.
**Fix:** `async Task<IActionResult>`, return `CreatedAtAction(...)`.

---

**B3.**
```csharp
[HttpPost]
public async Task<IActionResult> Create(CreateOrderDto dto)
{
    var order = await _service.CreateAsync(dto);
    _emailService.SendConfirmationAsync(order.Id);   // ← fire and forget
    return Ok(order);
}
```
**Bug:** un-awaited task.
**Mechanism:** three failures at once — (1) the exception is swallowed silently (unobserved), so you never learn emails are failing; (2) the work outlives the request, and the request scope disposes the `DbContext` **underneath it** → `ObjectDisposedException`; (3) a compiler warning was ignored to write this.
**Fix:** await it — or better, don't send email in the request at all: write an **outbox row in the same transaction** and let the background worker send it.
**Say this:** *"This is the exact case for the outbox — even if we await it, an exception after the commit still loses the email."*

---

**B4.**
```csharp
public async Task<Booking> BookAsync(int slotId, int userId)
{
    var existing = await _db.Bookings.AnyAsync(b => b.SlotId == slotId);
    if (existing) throw new SlotTakenException();
    var booking = new Booking { SlotId = slotId, UserId = userId };
    _db.Bookings.Add(booking);
    await _db.SaveChangesAsync();
    return booking;
}
```
**Bug:** check-then-act race — the centerpiece.
**Mechanism:** between `AnyAsync` and `SaveChangesAsync` there's a window where another request runs the same check. Under concurrency both pass, both insert, both succeed. **No amount of application code closes this**, because the check and the write aren't atomic.
**Fix:** a **unique constraint** on `SlotId` (or an exclusion constraint for time-range overlap), catch the violation, map to 409. Keep the pre-check for the friendly error message — it's UX, not correctness.
**Say this:** *"I'd also want a test that fires N parallel requests and asserts exactly one 201 — I've watched this exact code pass a single-threaded test and fail that one."*

---

**B5.**
```csharp
[Authorize]
[HttpGet("{id}")]
public async Task<IActionResult> Get(int id)
{
    var booking = await _db.Bookings.FindAsync(id);
    if (booking is null) return NotFound();
    return Ok(booking);
}
```
**Bug:** IDOR — authenticated but **not authorized for this row**. Any logged-in user reads anyone's booking by changing the id.
**Mechanism:** `[Authorize]` only proves *who you are*. Ownership is data, not a claim.
**Fix:** scope the query, don't add an `if`: `FirstOrDefaultAsync(b => b.Id == id && b.UserId == currentUserId)` → a missing check returns 404 instead of leaking. Admins get an explicit policy branch.
**Say this:** *"This is OWASP A01, the most common real API vulnerability — and I'd grep for the same shape across every other endpoint, because it's never in just one place."*
*(Bonus: returning the entity directly — see B6.)*

---

**B6.**
```csharp
[HttpPost]
public async Task<IActionResult> Register([FromBody] User user)
{
    _db.Users.Add(user);
    await _db.SaveChangesAsync();
    return Ok(user);
}
```
**Bug:** mass assignment (over-posting) — binding straight to the entity.
**Mechanism:** the client posts `{"email":"x","isAdmin":true,"passwordHash":"..."}` and the binder sets **every** public property. Privilege escalation via JSON.
**Fix:** a `RegisterDto` with only the settable fields; server-controlled values (role, id, timestamps) set server-side. Also don't return the entity — that's how the password hash ends up in the response body.

---

**B7.**
```csharp
var orders = await _db.Orders.ToListAsync();
foreach (var order in orders)
{
    var customer = await _db.Customers.FindAsync(order.CustomerId);
    result.Add(new OrderDto { Id = order.Id, CustomerName = customer.Name });
}
```
**Bug:** N+1 — 1 query + one per order.
**Mechanism:** fast on 3 seed rows, catastrophic on 10,000 — and the cost is *round-trips*, not CPU, so a bigger server doesn't help (see the latency ladder in [11-rapid-fire-trivia.md](11-rapid-fire-trivia.md)).
**Fix:** one projection: `_db.Orders.Select(o => new OrderDto { Id = o.Id, CustomerName = o.Customer.Name }).ToListAsync()` — one query, only the two columns needed.
**Say this:** *"I'd catch this by logging the SQL — the query count in the log is the evidence, and 'fast locally, slow with data' is the symptom."*

---

**B8.**
```csharp
try
{
    await _paymentGateway.ChargeAsync(amount);
}
catch (Exception)
{
    // TODO
}
return Ok();
```
**Bug:** swallowed exception → **returns 200 on a failed payment**.
**Mechanism:** the caller believes it worked. The user believes it worked. There's no log, so nobody finds out until reconciliation.
**Fix:** catch only what you can handle (a specific gateway exception → 402/503 + retry policy), let the rest reach the global handler. If you must swallow, log at Warning with context and an explicit comment explaining *why* it's safe.

---

**B9.**
```csharp
public async Task<Rate> GetRateAsync(string currency)
{
    using var client = new HttpClient();
    return await client.GetFromJsonAsync<Rate>($"https://api.rates.com/{currency}");
}
```
**Bug:** `new HttpClient()` per call (and disposing it).
**Mechanism:** each instance opens its own connection pool; disposal leaves sockets in `TIME_WAIT` for minutes → **socket exhaustion** under load ("address already in use"). The naive fix — a `static HttpClient` — swaps it for permanently **stale DNS**.
**Fix:** `IHttpClientFactory` (typed client), which pools and rotates handlers. Add a timeout — the default is 100 seconds, which is an outage.

---

**B10.**
```csharp
var sql = $"SELECT * FROM Tickets WHERE Title LIKE '%{search}%'";
var tickets = await _db.Tickets.FromSqlRaw(sql).ToListAsync();
```
**Bug:** SQL injection.
**Mechanism:** `search = "'; DROP TABLE Tickets--"` — the data became code. The interpolation happened *before* `FromSqlRaw` ever saw it, so nothing can parameterize it.
**Fix:** `FromSql($"SELECT * FROM Tickets WHERE Title LIKE {'%' + search + '%'}")` — the **interpolated** `FromSql` builds parameters. Or just use LINQ. Note the trap: `FromSqlRaw($"...{x}")` **looks identical** and is fully vulnerable — the method name is the entire difference.

---

**B11.**
```csharp
public async Task<List<Ticket>> Search(string sortBy, bool desc)
{
    var direction = desc ? "DESC" : "ASC";
    return await _db.Tickets.FromSqlRaw($"SELECT * FROM Tickets ORDER BY {sortBy} {direction}").ToListAsync();
}
```
**Bug:** injection via `ORDER BY` — and the reason it's a separate question from B10.
**Mechanism:** **identifiers cannot be parameterized.** There is no parameterized form of `ORDER BY <column>`, so the usual advice doesn't apply and people conclude it's unavoidable.
**Fix:** an **allowlist**: map an incoming string to a known column via a dictionary/switch, reject anything else. Or `IQueryable` + a switch of `OrderBy` expressions.
**Say this:** *"Any place a user picks a column, table, or direction is an allowlist problem — parameterization can't help there."*

---

**B12.**
```csharp
public string HashPassword(string password)
{
    using var sha = SHA256.Create();
    return Convert.ToBase64String(sha.ComputeHash(Encoding.UTF8.GetBytes(password)));
}
```
**Bug:** fast, unsalted hash.
**Mechanism:** SHA-256 is *designed* to be fast — a GPU does billions/sec, so a leaked table is cracked over a weekend. No salt means identical passwords produce identical hashes and rainbow tables apply directly.
**Fix:** Argon2id / bcrypt / PBKDF2 with a per-password salt and a tuned work factor (~100ms). In ASP.NET Core: `IPasswordHasher<T>`. Never hand-roll this.
**Say this:** *"Slow is the feature — I verify one password per login, the attacker tries billions."*

---

**B13.**
```csharp
private static readonly SemaphoreSlim _lock = new(1, 1);

public async Task<Report> BuildAsync()
{
    await _lock.WaitAsync();
    var report = await _generator.RunAsync();   // throws sometimes
    _lock.Release();
    return report;
}
```
**Bug:** no `try/finally` — an exception leaks the permit.
**Mechanism:** one throw and the semaphore is **never released**. Every subsequent request waits forever. The app doesn't crash — it just silently stops serving this path until a restart, which makes it a nightmare to diagnose.
**Fix:**
```csharp
await _lock.WaitAsync(ct);
try { return await _generator.RunAsync(); }
finally { _lock.Release(); }
```

---

**B14.**
```csharp
public class RateLimitService
{
    private readonly PricingOptions _options;
    public RateLimitService(IOptionsSnapshot<PricingOptions> options) => _options = options.Value;
}
builder.Services.AddSingleton<RateLimitService>();
```
**Bug:** `IOptionsSnapshot` (scoped) injected into a singleton — a captive dependency, and the container will throw on it.
**Mechanism:** snapshot is per-request by design; a singleton would freeze one request's snapshot forever, defeating the reload it exists for.
**Fix:** `IOptionsMonitor<T>` (singleton + change notifications) and read `.CurrentValue` per use, or plain `IOptions<T>` if reload isn't needed.

---

**B15.**
```csharp
public IActionResult GetDashboard()
{
    var data = _service.LoadDashboardAsync().Result;
    return Ok(data);
}
```
**Bug:** sync-over-async.
**Mechanism:** in ASP.NET Core there's no SynchronizationContext, so it doesn't deadlock — it **blocks a thread-pool thread** for the whole call. Under load the pool exhausts and the *whole app* slows down: latency climbs while **CPU stays low** (the tell). It passes every test and dies in production. On legacy ASP.NET the same line is a guaranteed deadlock.
**Fix:** `public async Task<IActionResult> GetDashboard() => Ok(await _service.LoadDashboardAsync());`

---

**B16.**
```csharp
[HttpGet]
public async Task<IActionResult> GetAll()
{
    return Ok(await _db.Tickets.ToListAsync());
}
```
**Bug:** unbounded list endpoint.
**Mechanism:** works on seed data; at 500k rows it OOMs the app, saturates the network, and takes the database with it. Bonus: no `AsNoTracking`, and it returns entities (serialization cycles + over-exposure).
**Fix:** pagination **from day one** with a server-enforced max page size, a projection, and `AsNoTracking`.
**Say this:** *"Adding pagination later is a breaking change for every client — that's why it's day one, not a v2."*

---

**B17.**
```csharp
public class CacheService
{
    private static readonly List<Product> _cache = new();
    public void Add(Product p) => _cache.Add(p);
    public List<Product> GetAll() => _cache;
}
```
**Bug:** static mutable collection shared across all requests — three bugs stacked.
**Mechanism:** (1) `List<T>` isn't thread-safe: concurrent `Add` during a resize can lose items or throw from a *read*; (2) `GetAll` hands out the **live internal list**, so a caller can mutate your cache — or enumerate it while another thread adds and get `InvalidOperationException`; (3) it grows forever — no eviction, no TTL. That's a memory leak with a race condition inside it.
**Fix:** `IMemoryCache` (with size limits + expiry), or `ConcurrentDictionary` + return a copy/`IReadOnlyList`. And on a second instance, an in-memory cache diverges per node anyway.

---

**B18.**
```csharp
public class Invoice
{
    public double Total { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.Now;
}
```
**Bug:** two classics in two lines — `double` for money, `DateTime.Now` for a stored timestamp.
**Mechanism:** `double` is binary floating point: 0.1 + 0.2 ≠ 0.3, and errors compound across a sum — invoices that don't reconcile by a cent. `DateTime.Now` is the *server's local time*, so it's meaningless in a container, changes when you deploy to another region, breaks across DST, and can't be tested.
**Fix:** money as `decimal` in the domain and integer minor units in the DB; `DateTimeOffset` from an injected `TimeProvider`, stored as UTC.

---

**B19.**
```csharp
var ticket = await _db.Tickets.AsNoTracking().FirstOrDefaultAsync(t => t.Id == id);
ticket.Status = TicketStatus.Closed;
await _db.SaveChangesAsync();
return NoContent();
```
**Bug:** `AsNoTracking` + mutation → **`SaveChanges` does nothing** and returns 204.
**Mechanism:** the entity is detached, so the change tracker has no record of it — no UPDATE is generated. The silent kind: no exception, correct-looking response, data unchanged. Users report "it doesn't save sometimes."
**Fix:** drop `AsNoTracking` on a write path (it's for reads), or `Attach` + mark modified, or `ExecuteUpdate`.

---

**B20.**
```csharp
var userTask = _db.Users.FirstOrDefaultAsync(u => u.Id == userId);
var ordersTask = _db.Orders.Where(o => o.UserId == userId).ToListAsync();
await Task.WhenAll(userTask, ordersTask);
```
**Bug:** two concurrent operations on **one** `DbContext`.
**Mechanism:** `DbContext` is not thread-safe — one operation at a time. This throws *"A second operation was started on this context instance before a previous operation completed."* Worse, it's timing-dependent: it can pass locally and fail under load.
**Fix:** await them sequentially (the round-trips are usually cheap), or use `IDbContextFactory` for a context per query if the parallelism genuinely matters.
**Say this:** *"This is someone applying the WhenAll fan-out idea, which is right for HTTP calls and wrong for one DbContext — and the exception message tells you exactly that."*

---

**B21.**
```csharp
app.UseExceptionHandler("/error");
app.UseAuthorization();
app.UseRouting();
app.UseAuthentication();
app.MapControllers();
```
**Bug:** pipeline order — `UseAuthorization` before `UseRouting`, and `UseAuthentication` after `UseAuthorization`.
**Mechanism:** authorization runs before an endpoint has been matched, so it can't see the endpoint's `[Authorize]` metadata — and it runs before authentication has built the `ClaimsPrincipal`, so there's no identity to evaluate. Net effect: **your endpoints aren't protected.** A security hole caused entirely by line order, with no error message.
**Fix:** `UseExceptionHandler` → `UseRouting` → `UseAuthentication` → `UseAuthorization` → `MapControllers`.

---

**B22.**
```csharp
_logger.LogInformation("Login attempt: {@Request}", loginRequest);
_logger.LogInformation($"Issued token {token} for user {user.Email}");
```
**Bug:** logging secrets + PII — and destroying structure while doing it.
**Mechanism:** `{@Request}` serializes the **whole object including the password**. The second line logs a **live bearer token** — anyone with log access can replay it — and uses string interpolation, so the log sink gets one flat string with no queryable properties (and the formatting cost is paid even when the level is disabled).
**Fix:** log identifiers, never credentials: `_logger.LogInformation("Login attempt for {UserId}", user.Id)`. Treat logs as a data-export surface — they get shipped to third parties and read by people who shouldn't see that data.

---

**B23.**
```csharp
public async Task<IActionResult> Export(CancellationToken ct)
{
    var rows = await _db.Events.Where(e => e.Date > from).ToListAsync();   // no ct
    var csv = await _renderer.RenderAsync(rows);                           // no ct
    return File(csv, "text/csv");
}
```
**Bug:** the `CancellationToken` is accepted and never used.
**Mechanism:** the framework binds it to `HttpContext.RequestAborted`, so the plumbing is there — but the user closes the tab and your server keeps grinding through a 60-second export nobody will ever receive. Cancellation is **cooperative**: code that ignores the token runs to completion.
**Fix:** pass `ct` to every async call. Free throughput, and a small detail that reads as care.
**Bonus:** catch `OperationCanceledException` separately — it's expected, not an error, and shouldn't page anyone.

---

**B24.**
```csharp
public async Task<Booking> CreateAsync(CreateBookingDto dto)
{
    var booking = new Booking { /* ... */ };
    _db.Bookings.Add(booking);
    await _emailService.SendAsync(dto.Email, "Booked!");
    await _db.SaveChangesAsync();
    return booking;
}
```
**Bug:** the **dual-write problem**, in its worst ordering — the email is sent *before* the commit.
**Mechanism:** `SaveChangesAsync` can fail (constraint violation, concurrency conflict, connection drop) **after** the customer already has a confirmation email for a booking that doesn't exist. Flip the two lines and you get the opposite bug: a crash between commit and send silently loses the email. There is no ordering of these two statements that is correct, because two systems can't share one transaction.
**Fix:** write an **outbox row in the same transaction as the booking**; a background worker polls, sends, marks sent. Delivery becomes at-least-once, so the send must be idempotent.
**Say this:** *"There's no line order that fixes this — that's the tell that it needs an outbox rather than a reshuffle. It's my favorite example of a bug that's invisible in code review until you ask 'what if it crashes here?'"*

---

## The review script (use this shape every time)

> **"Before I go line by line — is there context on what this is meant to do, and is this the whole change?"**
> Then, in priority order:
> 1. **Security** — authz on every fetch, injection, secrets, over-posting
> 2. **Correctness** — races, transactions, error paths, nulls
> 3. **Resilience** — timeouts, retries, what happens if this throws *here*
> 4. **Performance** — N+1, unbounded queries, sync-over-async
> 5. **Clarity** — naming, dead code, tests
>
> Then: **"The one I'd block on is X. The rest are comments."**

That last sentence is what a senior says and a junior doesn't. Reviewers who flag 14 things equally are exhausting; reviewers who **rank** are trusted. If you say nothing else in a review round, say which one you'd block on and why.

## Scoring

| Found | Read |
|---|---|
| 20+ primary bugs | Ready. You'd pass this round today. |
| 14–19 | Solid. Re-drill the misses, narrated. |
| 8–13 | Expected mid-curriculum. Pair this with Friday's planted-bug drill. |
| <8 | Read the mechanism for each, then redo cold in 3 days. |

**Notice the pattern across all 24:** roughly half are one of four ideas wearing different clothes — *check-then-act races* (B4, B17), *ownership/binding trust* (B5, B6, B10, B11), *lifetime mismatches* (B1, B3, B14, B19, B20), and *dual-writes/failure ordering* (B3, B8, B24). You built the real versions of all four in Bookable. **These aren't trivia for you — they're your project's changelog.**
