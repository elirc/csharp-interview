# Async & Concurrency — 46 Questions (`A`)

⭐ The highest-value depth topic in a .NET screener. Everyone can say "async frees the thread." The hire signal is the *second* and *third* question: what thread resumes it, why `.Result` deadlocks in one framework but starves in another, what `ValueTask` is for. This file is that chain.

Log misses as `A##`.

---

## The model

**A1. ⭐ What does `await` actually do?**
Splits the method at that point. The compiler builds a state machine: run to the await; if the awaited thing is already complete, continue synchronously (a real fast path); otherwise **return to the caller** and register the rest of the method as a continuation. When the operation completes, the continuation is scheduled and the method resumes at the split. Nothing is blocked, and no thread sits waiting.

**A2. ⭐ Does async make a request faster?**
No — it makes the *server* handle more requests. One request takes the same wall-clock time (arguably a hair longer from state-machine overhead). The win is that a thread isn't parked on a socket for 50ms; it goes back to the pool and serves someone else. Throughput and resilience under load, not latency.

**A3. ⭐ "Async releases the thread — so which thread does the I/O?"**
**None.** This is the question that separates people who understand it. Real async I/O has no thread waiting: the request goes to the OS/driver, and completion arrives via an I/O completion port, which then schedules your continuation onto a pool thread. There is no hidden background thread per request — that's the entire point. Candidates who say "a background thread does it" have described `Task.Run`, which is the opposite of the goal.

**A4. `Task` vs `Thread`?**
A `Thread` is an OS resource (~1MB stack, expensive to create, scheduled by the OS). A `Task` is a promise of a future result, usually run by the thread pool, and for I/O usually run by *no thread at all*. You essentially never `new Thread()` in a web app.

**A5. `Task.Run` vs `await` on an I/O call?**
`await SomeIoAsync()` releases the thread while the OS works. `Task.Run(() => SomeIoBlocking())` **takes a second thread from the pool** and blocks it — you've moved the block, not removed it. In ASP.NET Core, wrapping work in `Task.Run` typically makes throughput *worse*: same work, more thread-pool pressure, plus you've lost the request context.

**A6. So when is `Task.Run` right?**
CPU-bound work that would otherwise block a UI thread, or parallelizing genuine computation. In a web app: rarely — the request thread is already a pool thread, so offloading CPU work to another pool thread just adds hops. If it's long CPU work, it belongs in a background service or a queue, not in the request.

**A7. ⭐ Does async help CPU-bound work?**
No. There's nothing to wait on; you're occupying a core either way. `async` on a CPU-bound method is overhead and a lie in the signature.

**A8. `async` all the way down — why?**
Because the alternative is blocking somewhere, and one blocking call in a hot path pins a pool thread for the whole duration. Mixing sync and async gets you the worst of both: state-machine overhead *and* blocked threads. The rule is enforced socially — one `.Result` in a library ruins a whole call chain.

---

## Task internals

**A9. What is `Task<T>` really?**
A state machine + a completion signal + a result slot + a continuation list. It's a *promise*, not a thread. It can be already-complete when returned (`Task.FromResult`, cached completions), in which case awaiting it never yields.

**A10. ⭐ `ValueTask` vs `Task` — when and why?**
`Task` is a class → one allocation per call. `ValueTask` is a struct that either wraps a synchronous result (no allocation) or a `Task` when it must go async. Use it on hot paths that usually complete synchronously — the canonical example is a cache read that hits 95% of the time. **The rules:** await it exactly once, never `.Result` it before completion, don't store it, don't await it concurrently. Violating those is undefined behavior, not an exception. Default to `Task`; reach for `ValueTask` with a profile in hand.

**A11. What does `Task.CompletedTask` / `Task.FromResult` save?**
Returning a cached, already-complete task instead of allocating and running a state machine. Common in interface implementations with nothing async to do — but if the method has no async work, that's often a sign the interface shouldn't be async.

**A12. `Task.WhenAll` vs `Task.WhenAny` vs `Task.WaitAll`?**
`WhenAll` returns a task that completes when all complete — **awaitable, non-blocking**, and the way you fan out independent I/O. `WhenAny` completes at the first one (timeouts, races). `WaitAll`/`Wait` **block** — the sync trap.

**A13. `WhenAll` and exceptions — what do you catch?**
The returned task holds an `AggregateException` with *all* failures, but `await` rethrows only the **first**. If you need every error (validation fan-out, batch jobs), inspect `task.Exception.InnerExceptions` after catching, or await each individually. Silently losing the 2nd–Nth exception is a real bug.

**A14. How do you add a timeout to an async call?**
Modern: `task.WaitAsync(TimeSpan)` (.NET 6+), or a `CancellationTokenSource(timeout)` passed into the call. Old-school `WhenAny(task, Task.Delay(t))` works but **leaks the original task**, which keeps running unobserved. Naming the leak is the mid-level part.

**A15. What happens to an unobserved task exception?**
Historically it crashed the process on finalization; since .NET 4.5 it's swallowed and raised on `TaskScheduler.UnobservedTaskException`. Practically: **fire-and-forget failures disappear silently.** That's why `_ = DoWorkAsync();` in a controller is a bug — the work outlives the request scope, the exception vanishes, and the DbContext is disposed underneath it.

**A16. ⭐ `async void` — when is it acceptable?**
Only event handlers. Otherwise: the caller can't await it, can't know when it finishes, and **an exception in it is thrown on the synchronization context and takes down the process** rather than propagating. `async Task` everywhere else. If you see `async void` on a controller action, that's the bug.

**A17. `ConfigureAwait(false)` — do you need it in ASP.NET Core?**
ASP.NET Core has **no SynchronizationContext**, so in app code it changes nothing. It still matters in **libraries** (they may be consumed by WinForms/WPF/legacy ASP.NET where a context exists, and capturing it there causes the deadlock) and it saves a tiny bit of context-capture work. Answer: "not in my app code; yes in any library I publish." That nuance is the whole question.

**A18. ⭐ Why does `.Result` deadlock in classic ASP.NET but "only" starve in ASP.NET Core?**
Classic ASP.NET/WinForms have a SynchronizationContext that allows one thread in. The continuation is queued *back to that context*, which is blocked by your own `.Result` — mutual deadlock, guaranteed, on a single request. ASP.NET Core has no context, so there's no deadlock — instead each `.Result` **blocks a pool thread**, and under load the pool exhausts: latency spikes, then timeouts. Worse in a way, because it passes every test and dies only in production. Nailing this distinction is a strong signal.

**A19. What's thread-pool starvation, and how would you spot it?**
All pool threads blocked, work queued, nothing progressing. The pool injects new threads only slowly (roughly one per second after the minimum, by design — hill-climbing). Symptoms: latency climbing under load with **low CPU** — the tell. Diagnose: `dotnet-counters` on `ThreadPool Queue Length` / thread count, or a dump showing threads parked in `Monitor.Wait`/`.Result`. Cause is almost always sync-over-async.

**A20. `Task.Delay` vs `Thread.Sleep`?**
`Delay` is a timer + continuation — no thread held. `Sleep` blocks the thread for the duration. `Thread.Sleep` in a request path or a `BackgroundService` loop is a code-review finding.

**A21. What's the fast path — does `await` always yield?**
No. If the awaited task is already complete, the state machine continues **synchronously** on the same thread, no scheduling at all. This is why `ValueTask` and cached tasks matter, and why "await always costs a context switch" is wrong.

---

## Cancellation

**A22. ⭐ How does cancellation work in .NET?**
Cooperative. A `CancellationTokenSource` owns the token; you pass the token down; callees check `IsCancellationRequested` or call `ThrowIfCancellationRequested()`, or hand it to an API that does. **Nothing is forcibly aborted** — code that ignores the token runs to completion.

**A23. Where does the token come from in an ASP.NET Core action?**
Bind a `CancellationToken` parameter and the framework hands you `HttpContext.RequestAborted` — signalled when the client disconnects. Pass it to EF (`ToListAsync(ct)`) and `HttpClient`. Payoff: a user who closes the tab stops costing you a database query. Free win, rarely done, great thing to have in your repo.

**A24. `OperationCanceledException` — error or not?**
Not an error: it's the expected outcome of a cancelled operation. Don't log it at Error level, don't map it to a 500 — a disconnected client can't receive a status anyway (499 by nginx convention). Catch it separately from real failures.

**A25. Linked cancellation tokens?**
`CancellationTokenSource.CreateLinkedTokenSource(a, b)` — cancels when *either* fires. The idiom for "cancel when the client disconnects **or** my 5-second timeout hits." **Dispose it** — linked sources register callbacks on the parents and leak if the parent is long-lived.

**A26. How does a `BackgroundService` learn about shutdown?**
The `stoppingToken` passed to `ExecuteAsync`, signalled when the host stops. Honor it in every loop and every await, or the host waits out its shutdown timeout (default 30s) and then kills you mid-work — which is exactly how half-finished writes happen during a deploy.

---

## Synchronization

**A27. ⭐ Why can't you `lock` around an `await`?**
It's a compile error, and the reason is the mechanism: `lock` is `Monitor`, which is **thread-affine** — the thread that enters must be the thread that exits. After an await, you may resume on a different pool thread, so the release would happen on the wrong thread. Use `SemaphoreSlim.WaitAsync()` in try/finally.

**A28. `lock(this)` or `lock(typeof(X))` — why bad?**
The lock object is publicly reachable, so *anyone else* can lock on it and deadlock you from outside your class. Lock on a `private static readonly object` (or the .NET 9 `System.Threading.Lock` type). Also never lock on a string — interning means unrelated code may share the literal.

**A29. `lock` vs `Interlocked` vs `volatile`?**
`lock` = mutual exclusion for a region + full memory barrier. `Interlocked` = single lock-free atomic op (`Increment`, `CompareExchange`) — much cheaper for counters. `volatile` = prevents certain compiler/CPU reorderings and ensures reads see the latest write; it makes **no** operation atomic — `volatile int i; i++;` is still a race. That last clause is the trivia.

**A30. Why is `i++` not atomic?**
Read → increment → write. Two threads can read the same value and both write back the same +1: one increment vanishes. `Interlocked.Increment(ref i)` does it as one indivisible operation.

**A31. What's `Interlocked.CompareExchange` for?**
The primitive under lock-free algorithms: "set to X only if it's currently Y," atomically. Read-modify-write in a loop = optimistic concurrency at the memory level — the *same idea* as your version-column retry in Bookable, one abstraction layer down. Making that connection out loud is a good moment in an interview.

**A32. `SemaphoreSlim` — two uses?**
(1) An async-compatible mutex: `new SemaphoreSlim(1,1)` + `await WaitAsync()`. (2) A throttle: `new SemaphoreSlim(10)` caps concurrent outbound calls. **Always release in `finally`** — a leaked permit deadlocks the app permanently, and it's the classic review finding.

**A33. `Monitor.Wait`/`Pulse`, `ManualResetEventSlim`, `AutoResetEvent` — care?**
Know they're the "wait until signalled" family and that in async code you use `Channel<T>`, `TaskCompletionSource`, or `SemaphoreSlim` instead. You will not be asked to implement a producer/consumer with `Pulse` at mid-level. Knowing the modern replacement is the better answer.

**A34. What's a `TaskCompletionSource`?**
A manually-controlled `Task` — you hand out `.Task` and later call `SetResult`/`SetException`. The adapter between callback-based APIs and `await`. Gotcha: use `TaskCreationOptions.RunContinuationsAsynchronously`, or the continuation runs *on the thread that calls SetResult*, which can deadlock the setter's own lock.

**A35. What's a race condition, in one sentence?**
Correctness that depends on timing between concurrent operations. Your canonical example is check-then-act: two requests both read "slot free," both write, both succeed. Say it with your booking story.

**A36. Deadlock — the four conditions, or just the practical answer?**
Practical: two parties each hold something the other needs, and neither releases. Prevention that actually applies: **acquire locks in a consistent order**, keep critical sections short, don't call out to unknown code (or `await`) while holding a lock, prefer a single lock over a hierarchy.

---

## Concurrent collections

**A37. ⭐ Is `List<T>` thread-safe? What actually happens?**
No. Concurrent `Add` can interleave during array growth and lose items, throw `IndexOutOfRange` from a *read*, or leave the internal state inconsistent. `Dictionary<K,V>` is worse: a concurrent write during resize can corrupt the bucket chain into a **cycle**, and a later read spins forever — a real, famous production hang at 100% CPU on one core.

**A38. `ConcurrentDictionary` — what's the catch with `GetOrAdd`?**
The dictionary operation is atomic, but **the value factory is not** — under contention two threads can both run it; one value wins and the other is discarded. If the factory is expensive or has side effects (opens a connection, sends an email), that's a bug. Fix: store `Lazy<T>` as the value, so only the winner's lazy is ever evaluated.

**A39. `ConcurrentBag`/`ConcurrentQueue`/`BlockingCollection` — when?**
Queue for FIFO producer/consumer; Bag when order doesn't matter and the same thread often produces and consumes (thread-local optimized); `BlockingCollection` for a bounded blocking pipeline — but in async code, `System.Threading.Channels` is the modern answer (`Channel<T>` is async-native and backpressure-aware).

**A40. `Channel<T>` — why is it the right in-process queue?**
Async producer/consumer with optional bounded capacity → **backpressure** (`WriteAsync` waits instead of ballooning memory). It's what you'd use behind a `BackgroundService` for in-process work. Caveat you must volunteer: in-memory = **lost on restart** — which is precisely why the outbox row goes in the DB first.

---

## Parallelism

**A41. Concurrency vs parallelism?**
Concurrency = multiple things in progress (interleaved, one core is enough) — a web server. Parallelism = multiple things executing *simultaneously* on multiple cores — image processing. Async gives you concurrency without parallelism; `Parallel.For` gives you parallelism.

**A42. `Parallel.ForEach` vs `Task.WhenAll`?**
`Parallel.*` is for CPU-bound work over a collection, using the pool, with a sync body — using it for I/O blocks threads. `Task.WhenAll` (or `Parallel.ForEachAsync`, .NET 6) is for I/O-bound fan-out. `Parallel.ForEachAsync` also gives you `MaxDegreeOfParallelism`, which `WhenAll` doesn't — with `WhenAll` over 10,000 items you just launched 10,000 concurrent calls at your database.

**A43. PLINQ — would you use it?**
`.AsParallel()` on genuine CPU-bound in-memory work over big sequences. Honest answer: almost never in a web app — the server is already saturated with concurrent requests, so stealing cores from other requests to speed one up is usually the wrong trade.

**A44. How do you throttle 1,000 outbound HTTP calls?**
`Parallel.ForEachAsync` with `MaxDegreeOfParallelism`, or a `SemaphoreSlim(n)` inside the task body, or `Chunk(n)` + `WhenAll` per chunk. Then name the real-world concerns: the remote's rate limit, retry with **jitter**, and a circuit breaker so you don't hammer a service that's already down.

---

## In the web app

**A45. ⭐ Is `DbContext` thread-safe? What's the actual error?**
No — one context, one operation at a time. Firing two queries in parallel on the same context throws *"A second operation was started on this context instance before a previous operation completed"*. It happens when someone "optimizes" a controller with `Task.WhenAll` over two repository calls sharing the scoped context. Fix: sequential awaits, or a separate context per operation via `IDbContextFactory`.

**A46. A singleton service needs a scoped `DbContext` — what do you do?**
Inject `IServiceScopeFactory`, and create a scope **per unit of work**: `using var scope = _scopeFactory.CreateScope(); var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();`. This is exactly what a `BackgroundService` must do — it's a singleton, so it cannot constructor-inject a DbContext, and creating one scope for the whole service lifetime is the subtle version of the same bug (an ever-growing change tracker + a connection held for days).

---

## Coverage tracker

`A1–A8` model · `A9–A21` task internals ⭐ · `A22–A26` cancellation · `A27–A36` synchronization · `A37–A40` concurrent collections · `A41–A44` parallelism · `A45–A46` web app

**The four that decide the round:** A3 (which thread does the I/O — *none*), A18 (`.Result`: deadlock vs starvation), A19 (starvation's low-CPU tell), A45 (DbContext concurrency). Get those four cold and you're above the mid-level bar on async, which is where most candidates are weakest.
