# ASP.NET Core — 62 Questions (`W`)

This is the job. The core bank covers the pipeline and DI lifetimes; this file is everything the interviewer asks *after* you answer those well — plus the framework surface you're assumed to have touched.

Log misses as `W##`.

---

## Hosting & startup

**W1. What does `WebApplication.CreateBuilder(args)` set up for you?**
Configuration (appsettings → appsettings.{Env} → user-secrets in Dev → env vars → command line), logging providers, the DI container, Kestrel, and the host lifetime. The .NET 6+ minimal hosting model collapsed the old `Program`+`Startup` split into one file — know that `Startup.cs` with `ConfigureServices`/`Configure` is the old shape and still works, because you'll meet it in real codebases.

**W2. ⭐ `builder.Services.Add...` vs `app.Use...` — what's the difference?**
Registration vs pipeline. `Add*` puts things in the DI container (available for injection). `Use*` inserts middleware into the request pipeline (executed per request, **in order**). Confusing the two is the tell of someone who has only copy-pasted a `Program.cs`.

**W3. How does the app know it's in Development?**
`ASPNETCORE_ENVIRONMENT` (or `DOTNET_ENVIRONMENT`). Drives `appsettings.{Environment}.json` layering, `app.Environment.IsDevelopment()`, the developer exception page, and user-secrets. It's a plain env var — nothing magic, and it defaults to Production when unset, which is the safe default *and* the reason "it works locally" fails in a container.

**W4. What's `IHostedService` vs `BackgroundService`?**
`IHostedService` is the interface (`StartAsync`/`StopAsync`); `BackgroundService` is the abstract base with a long-running `ExecuteAsync(stoppingToken)`. Trap: **blocking work in `StartAsync` delays app startup** — the host awaits it before serving traffic. Long work belongs in `ExecuteAsync`, and you should return quickly from Start.

**W5. What happens on graceful shutdown?**
The host signals the stopping token, stops accepting new requests, waits for in-flight ones and hosted services up to the shutdown timeout (default 30s), then disposes the container. Practical consequence: in-memory queues and un-awaited work are **lost on every deploy** — the argument for a durable outbox in one sentence.

**W6. Kestrel vs IIS vs a reverse proxy?**
Kestrel is the in-process HTTP server. A proxy (nginx/IIS/cloud LB) fronts it for TLS termination, static assets, multiple apps per host, and buffering slow clients. If you're behind a proxy you need `UseForwardedHeaders`, or every client IP in your logs is the proxy and `Request.Scheme` says http.

---

## Dependency injection

**W7. ⭐ The three lifetimes, precisely?**
Transient: a new instance per resolve — even twice within one request. Scoped: one per scope, and in ASP.NET Core the scope is the **request**. Singleton: one per application, created on first resolve (or at registration if you pass an instance), shared across all requests and threads → **it must be thread-safe**.

**W8. ⭐ Captive dependency — what is it and what does .NET do about it?**
A longer-lived service holding a shorter-lived one: a singleton captures a scoped `DbContext` at construction and keeps it **forever**, shared across all requests — an ever-growing change tracker, cross-request data bleeding, and a non-thread-safe object used concurrently. .NET's container validates scopes **in Development** and throws at startup. In Production that validation is off by default, which is exactly why you don't skip it locally.

**W9. How do you use a scoped service from a singleton?**
Inject `IServiceScopeFactory`, create a scope per unit of work, resolve inside it, dispose it. Never one scope for the singleton's lifetime — that's the same bug wearing a hat.

**W10. Is middleware a singleton?**
Yes — convention-based middleware is constructed **once** at startup. So constructor-injected dependencies are effectively singletons (captive!). Inject scoped services as **parameters of `InvokeAsync`** instead; the framework resolves them per request. Genuinely good question, genuinely often missed.

**W11. `AddTransient` vs `AddScoped` for a stateless service?**
Functionally identical in behavior; scoped is one fewer allocation per request and is the common default for services touching a DbContext (so they share the same unit of work). Transient for cheap, stateless, short-lived helpers. Honest answer: "for most of my services it doesn't matter, and I default to scoped so they share the request's DbContext."

**W12. What is `TryAdd*` for?**
Registers only if the service type isn't already registered — the pattern library authors use in `AddXyz()` extension methods so consumers can override defaults. Plain `Add` appends: registering `IFoo` twice means `GetService<IFoo>()` returns the **last** one, while `GetServices<IFoo>()` returns both.

**W13. Multiple implementations of one interface — how do you get the right one?**
Inject `IEnumerable<IHandler>` and pick/loop (the strategy/chain pattern), or use **keyed services** (.NET 8): `AddKeyedScoped<IFoo, A>("a")` + `[FromKeyedServices("a")] IFoo foo`. Before .NET 8 you needed a factory delegate — mentioning that shows you know the version story.

**W14. Constructor injection vs `IServiceProvider` injection?**
Constructor = explicit dependencies, verifiable at startup, testable. Injecting `IServiceProvider` and resolving by hand is **service location** — an anti-pattern: it hides dependencies, defers failures to runtime, and makes tests guess. The legit exception is the scope-factory pattern in singletons/background services.

**W15. Does the DI container dispose what it creates?**
Yes, for `IDisposable`/`IAsyncDisposable` services **it constructed**: scoped/transient at scope end, singletons at app shutdown. Two traps: (1) transients registered as `IDisposable` are held by the scope until it ends — a "transient disposable" registered on the *root* scope leaks for the app's lifetime; (2) instances you pass in yourself (`AddSingleton(new Foo())`) are **not** disposed by the container.

**W16. How do you register something that needs config to construct?**
Factory overload: `services.AddSingleton<IFoo>(sp => new Foo(sp.GetRequiredService<IOptions<Cfg>>().Value.Url))`. Or the options pattern + a typed client. The factory overload is also the answer to "how do I decorate a service."

---

## Middleware

**W17. ⭐ Walk the default pipeline, in order, and say why each is where.**
`UseExceptionHandler` (outermost — must wrap everything below to catch it) → `UseHsts`/`UseHttpsRedirection` → `UseStaticFiles` (early — no need to route/auth a logo) → `UseRouting` (decides *which* endpoint) → `UseCors` → `UseAuthentication` (who are you) → `UseAuthorization` (may you — needs both the identity and the matched endpoint's policy) → `UseEndpoints`/endpoint execution. The *why* is the answer; the list alone is a junior answer.

**W18. What happens if `UseAuthorization` comes before `UseRouting`?**
Authorization can't know which endpoint was matched, so it can't see `[Authorize]` metadata — and your endpoints are silently **unprotected**. A security bug caused purely by line order, which is why the order question gets asked.

**W19. `Use` vs `Run` vs `Map`?**
`Use` = a middleware that may call `next` (or short-circuit). `Run` = terminal, never calls next. `Map`/`MapWhen` = branch the pipeline on a path/predicate. If you forget to `await next(context)` in a `Use`, every request short-circuits to a 200 with an empty body — a fun bug to plant on a Friday.

**W20. Write a middleware from memory.**
```csharp
public class TimingMiddleware(RequestDelegate next, ILogger<TimingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext ctx, IMetrics metrics) // scoped deps go HERE
    {
        var sw = Stopwatch.GetTimestamp();
        try { await next(ctx); }
        finally { logger.LogInformation("{Path} took {Ms}ms",
                    ctx.Request.Path, Stopwatch.GetElapsedTime(sw).TotalMilliseconds); }
    }
}
```
Points scored: `finally` (so it logs on exceptions too), scoped dep in `InvokeAsync` not the ctor, structured log properties, and `Stopwatch.GetTimestamp` over allocating a Stopwatch.

**W21. ⭐ Middleware vs filter vs endpoint filter?**
Middleware: whole pipeline, before routing, no MVC context — cross-cutting infrastructure (logging, exceptions, CORS). Filters: run inside MVC after model binding, with `ActionContext`/ModelState/action args — validation, per-action authorization, result shaping. Endpoint filters (.NET 7): the minimal-API equivalent of action filters. Rule: need to know *which action and its arguments*? Filter. Applies to every request including static files? Middleware.

**W22. Can middleware modify the response after `next`?**
Headers only until the response starts. Once the first byte is flushed, headers are sent and any change throws ("Headers are read-only, response has already started"). To rewrite the body you have to swap `Response.Body` for a buffer before calling next — expensive and the reason exception handlers that run *after* a partial write can't produce clean error JSON.

---

## Routing & endpoints

**W23. Attribute routing vs conventional routing?**
Attribute (`[HttpGet("api/tickets/{id:int}")]`) is the norm for APIs — explicit, per-action, and required by `[ApiController]`. Conventional (`MapControllerRoute` with `{controller}/{action}`) is an MVC-views thing. Route constraints (`:int`, `:guid`, `:min(1)`) participate in *matching*, so `/tickets/abc` yields a clean 404 rather than a binding error.

**W24. How does the framework pick between two matching routes?**
Precedence: literal segments beat parameters, parameters beat catch-alls, more constrained beats less. A genuine tie throws `AmbiguousMatchException` **at request time**, not startup — which is why you find it in production.

**W25. Minimal APIs vs controllers — your take?**
Have an opinion: minimal APIs are lighter, faster to start, AOT-friendly, great for small/focused services; controllers give you filters, model binding conventions, and organization that scales past a few dozen endpoints. Mid answer: "minimal for a small service or an internal endpoint; controllers when there's a real domain and cross-cutting conventions. They share the same pipeline underneath — it's not a performance argument at my scale."

**W26. What does `[ApiController]` actually turn on?**
Four things: automatic 400 on invalid ModelState (a `ValidationProblemDetails`), binding-source inference (complex types → body, simple → route/query), attribute routing **required**, and `ProblemDetails` for error status codes. Interview-worthy nuance: the auto-400 means your manual `if (!ModelState.IsValid)` check in every action is dead code.

**W27. `IActionResult` vs `ActionResult<T>` vs returning `T`?**
`IActionResult` = any result, but the type is invisible to OpenAPI. `ActionResult<T>` = both worlds: return `NotFound()` or the `T` directly, with the type documented. Returning bare `T` means you can only ever express 200. Prefer `ActionResult<T>` + `[ProducesResponseType]`.

**W28. `Results.Ok()` vs `TypedResults.Ok()` (minimal APIs)?**
`TypedResults` returns a concrete type — testable without casting and self-documenting for OpenAPI. `Results` returns `IResult`. Use `TypedResults`.

---

## Model binding & validation

**W29. ⭐ Where does the framework look for a parameter's value?**
Route values → query string → form → body (only one `[FromBody]` per action, since the body is a forward-only stream that can be read once) → header, plus DI via `[FromServices]`. `[ApiController]` infers most of it. Order matters: a route value and a query value with the same name → route wins.

**W30. Why can there be only one `[FromBody]`?**
The request body is a non-seekable stream consumed by the formatter. Two would require buffering and splitting a JSON document by parameter — so: one DTO, one body.

**W31. Model binding fails to parse `"abc"` into an `int` — what do you get?**
A ModelState error → `[ApiController]` returns a 400 `ValidationProblemDetails` with the field name. Your action never runs. Know this, because "who returned this 400 that isn't in my code?" is a genuinely confusing first encounter.

**W32. Data annotations vs FluentValidation?**
Annotations: built-in, declarative, fine for shape (`[Required]`, `[MaxLength]`, `[Range]`). FluentValidation: testable, composable, conditional rules, no attribute soup, injectable dependencies. Real answer: **shape rules either way; business rules in neither** — "cancellation must be ≥24h before start" needs the domain, not the DTO.

**W33. How do you validate something that needs the database?**
Not in the DTO validator (chatty, and it's a check-then-act race anyway). In the service/domain layer, returning a domain error mapped to 409/422 — and for uniqueness, backed by a **DB constraint** because the validator's check-then-act has a race window. That's your Bookable answer generalized.

**W34. What is ProblemDetails?**
RFC 7807/9457: a standard error body (`type`, `title`, `status`, `detail`, `instance`, + extensions). `AddProblemDetails()` makes the framework produce it consistently for exceptions and status codes. Using a standard error contract instead of `{ "error": "..." }` is a small thing that reads as professional.

---

## Filters

**W35. Filter types and their order?**
Authorization → Resource → Action (→ action executes →) → Exception → Result. Each has before/after halves; they nest like an onion around the action. Authorization first is deliberate: reject before you spend anything.

**W36. `[ServiceFilter]` vs `[TypeFilter]` vs a plain attribute?**
Attributes can't have DI-injected constructor args. `[ServiceFilter(typeof(X))]` resolves X from DI (must be registered). `[TypeFilter(typeof(X))]` constructs it with DI for the args (registration not required). Plain attribute = no dependencies. This is the "how do I inject into an attribute" question, and it comes up.

**W37. When would you write a global filter vs middleware?**
Global filter when you need MVC context on every action (auditing action args, wrapping results). Middleware when it should apply to everything including static files and 404s. Register global filters via `AddControllers(o => o.Filters.Add<X>())`.

---

## Configuration & options

**W38. ⭐ Configuration sources and precedence?**
appsettings.json → appsettings.{Environment}.json → User Secrets (Dev only) → environment variables → command-line args. **Later wins.** Env var nesting uses `__`: `ConnectionStrings__Default`. That last detail is what every container deployment needs and what candidates fumble.

**W39. ⭐ `IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor`?**
`IOptions<T>`: singleton, computed once, **never reloads** — safe to inject anywhere. `IOptionsSnapshot<T>`: **scoped**, recomputed per request, picks up file changes — and therefore **cannot be injected into a singleton** (captive dependency, and the container will tell you). `IOptionsMonitor<T>`: singleton, current value + `OnChange` callbacks — the only one usable for live-reloading config inside a singleton or a `BackgroundService`. This is the highest-yield options question.

**W40. How do you validate configuration?**
`services.AddOptions<X>().Bind(config.GetSection("X")).ValidateDataAnnotations().ValidateOnStart()`. `ValidateOnStart` is the point: **fail at boot, not at 3am on the first request** that touches the missing setting. Fast-fail config is a mid-level instinct worth voicing.

**W41. Where do connection strings live in production?**
Environment variables or a secret manager (Key Vault / AWS Secrets Manager / platform config). Never in appsettings.json in git. If one leaks, the response is **rotate**, not delete — history is forever, and a deleted-but-pushed secret is still compromised.

---

## Logging

**W42. ⭐ Why structured logging?**
`_logger.LogInformation("Booking {BookingId} failed for {UserId}", id, userId)` keeps the message template and the named values separate, so a sink can index them — you can query `BookingId = 42` across millions of lines. String interpolation into the message (`$"Booking {id}"`) destroys that *and* pays the formatting cost even when the level is disabled. Use the template. Always.

**W43. Log levels — when is each right?**
Trace/Debug: dev diagnostics, off in prod. Information: business events worth keeping (booking created). Warning: unexpected but handled (validation failure, retry). Error: this request failed and someone may care. Critical: the app is going down. The common sin is Error for a user typo — noise that trains people to ignore alerts.

**W44. What are logging scopes for?**
`using (_logger.BeginScope("{CorrelationId}", id))` attaches properties to every log inside the block, so you can pull one request's whole story out of a shared log. Correlation IDs + scopes are how you debug distributed anything.

**W45. What must never be logged?**
Passwords, tokens/Authorization headers, connection strings, PII beyond an ID, full request bodies on auth endpoints, card data. Say the operational half too: logs get shipped to third parties and read by people who shouldn't see that data — a log is a data-export surface.

**W46. Serilog vs `Microsoft.Extensions.Logging`?**
MEL is the abstraction (`ILogger<T>`) you code against; Serilog is a provider/implementation with rich sinks and enrichers. You log through `ILogger<T>` regardless, which is why swapping providers costs nothing.

---

## HttpClient

**W47. ⭐ What's wrong with `new HttpClient()` per request?**
Each one holds its own connection pool; disposing it leaves sockets in `TIME_WAIT` for ~4 minutes → **socket exhaustion** under load ("address in use"). The naive fix — one static `HttpClient` — leaks the opposite way: it **caches DNS forever**, so it keeps calling an IP that has moved (a real outage cause during failover).

**W48. So what does `IHttpClientFactory` fix?**
It pools and recycles `HttpMessageHandler`s on a rotation (default 2 minutes), giving you connection reuse *and* eventual DNS refresh. Plus: named/typed clients, a central place for base address + default headers, and a `DelegatingHandler` pipeline for auth headers, logging, and Polly retry/circuit-breaker policies.

**W49. Typed clients — what are they?**
`services.AddHttpClient<IPaymentApi, PaymentApi>(c => c.BaseAddress = ...)` — the class gets an `HttpClient` injected, configured centrally, and is itself injectable/mockable. The idiomatic shape for calling another service.

**W50. Retry policy — what would you actually configure?**
Retry only **idempotent** operations (GET/PUT/DELETE, or POST with an idempotency key), exponential backoff **with jitter** (synchronized retries are a self-inflicted DDoS), a cap on attempts, a per-attempt timeout, and a circuit breaker so a dead dependency fails fast instead of consuming your threads. Retrying a non-idempotent POST is how you double-charge a customer.

---

## Errors, health, performance

**W51. ⭐ How do you handle exceptions globally?**
`UseExceptionHandler` as the outermost middleware, or `IExceptionHandler` (.NET 8) implementations registered with `AddExceptionHandler` — map domain exceptions to status codes (not-found → 404, concurrency → 409, rule violation → 422), log with context, return `ProblemDetails`, and **never leak stack traces to clients**. The developer exception page is Development-only, deliberately.

**W52. Why not try/catch in every controller?**
Repetition, inconsistency, and it hides bugs. Catch *locally* only where you add value: translate a specific exception, retry, or enrich. Everything else belongs to the global handler.

**W53. Health checks — liveness vs readiness?**
Liveness: "is the process alive" — if it fails, restart me. Readiness: "can I serve traffic" — checks dependencies (DB reachable); if it fails, take me out of the load balancer but **don't restart** (the DB being down isn't fixed by killing the app). Two endpoints, filtered by tag. Getting this distinction right is a real ops signal.

**W54. Response caching vs output caching vs `IMemoryCache`?**
Response caching (.NET Core) emits `Cache-Control` headers so *clients/proxies* cache — fragile, requires cooperating clients. Output caching (.NET 7): the **server** stores the response, you control invalidation via tags — much more useful. `IMemoryCache`: you cache arbitrary objects in-process. Distributed: `IDistributedCache`/Redis when instances multiply, or `HybridCache` (.NET 9) which combines L1+L2 and fixes stampedes for you.

**W55. What's a cache stampede?**
A hot key expires and 500 concurrent requests all miss and all hit the DB at once — the DB falls over at exactly the moment traffic is highest. Mitigate: per-key locking (`SemaphoreSlim`/`Lazy`), stale-while-revalidate, TTL **jitter** so keys don't expire in lockstep. `HybridCache` does this out of the box.

**W56. Rate limiting?**
Built-in middleware since .NET 7: fixed window, sliding window, token bucket, concurrency. Know why: protect the DB and the downstream from abuse and stampedes; return **429 with `Retry-After`**. Per-user partitioning (`RateLimitPartition.GetFixedWindowLimiter(userId, ...)`) is the shape you actually want.

**W57. When would response compression bite you?**
It's mostly a win, but: don't compress already-compressed content (images), and enabling compression over HTTPS on attacker-influenced content has a historical vulnerability class (BREACH). Also, most real deployments do it at the proxy. Fine to say "the proxy does it."

---

## The rest of the surface

**W58. CORS — what actually happens on a preflight?**
For non-simple requests the browser sends `OPTIONS` with `Access-Control-Request-Method/Headers`; the server must answer with matching `Access-Control-Allow-*` headers before the real request is sent. Key nuance: **`AllowAnyOrigin` + `AllowCredentials` is illegal** by spec (the browser rejects `*` with credentials) — you must name origins. And CORS protects *users in browsers*, not your API: curl doesn't care.

**W59. What is `IHttpContextAccessor` and when is it a trap?**
Access to the current `HttpContext` outside a controller, backed by `AsyncLocal`. Traps: it's `null` in a background service (no request), and **capturing `HttpContext` in a captured closure or a singleton field** gets you a disposed context later. Prefer passing what you need (the user id) down rather than the whole context.

**W60. SignalR / gRPC — one line each?**
SignalR: real-time bidirectional messaging over WebSockets with fallbacks; needs a backplane (Redis) once you scale past one instance, since connections are stateful and sticky. gRPC: HTTP/2 + protobuf, contract-first, fast, great service-to-service, poor browser story without grpc-web. Knowing *when* each applies is enough at mid-level.

**W61. How do you document an API now?**
OpenAPI. Note the version story: Swashbuckle was the default template dependency for years; **.NET 9 replaced it with the built-in `Microsoft.AspNetCore.OpenApi`** (`AddOpenApi()`/`MapOpenApi()`), and the template ships Scalar/no UI rather than Swagger UI. Saying "Swagger" is fine and universally understood; knowing the template changed is a currency signal.

**W62. ⭐ Walk me through what happens from `POST /bookings` hitting Kestrel to the 201 going out.**
The one to rehearse end to end: Kestrel parses the request onto a connection from the socket → middleware chain in order (exception handler wraps → HTTPS → routing matches the endpoint → CORS → authentication builds `ClaimsPrincipal` from the JWT → authorization checks the endpoint's policy) → **a DI scope is created for the request** → model binding deserializes JSON into the DTO → validation → filters → the action runs → the service enforces domain rules → EF translates LINQ to SQL over a pooled connection → `SaveChanges` in a transaction → result object → JSON serialization → status + Location header → response flushes back out the chain → **the scope disposes, disposing the DbContext**. Ninety seconds, no notes. This is the single most useful narration in your kit — it is the map for every other question in this file.

---

## Coverage tracker

`W1–W6` hosting · `W7–W16` DI ⭐ · `W17–W22` middleware ⭐ · `W23–W28` routing · `W29–W34` binding/validation · `W35–W37` filters · `W38–W41` config ⭐ · `W42–W46` logging · `W47–W50` HttpClient · `W51–W57` errors/health/perf · `W58–W62` the rest

**The five that decide the round:** W17 (pipeline order *with reasons*), W8 (captive dependency), W39 (`IOptions` trio), W47/W48 (HttpClient — asked far more than its importance warrants, so it's free marks), W62 (the full narration).
