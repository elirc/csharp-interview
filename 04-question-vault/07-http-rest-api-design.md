# HTTP, REST & API Design — 38 Questions (`H`)

The warm-up round, and the one candidates lose points on by being vague. These have crisp answers; crisp is the whole grade. Then the design half, where the grade is whether you have *opinions with reasons*.

Log misses as `H##`.

---

## HTTP fundamentals

**H1. ⭐ What happens when you type a URL and hit enter?**
The 90-second rehearsed version: DNS resolution (cache → resolver → authoritative) → TCP handshake → **TLS handshake** (certificate validated against a trusted CA, keys negotiated) → HTTP request over the connection → maybe a CDN/load balancer/reverse proxy → Kestrel → middleware → your action → response → browser parses HTML, fetches subresources, renders. Breadth first, then let them pick a depth. Don't dive into TCP window scaling unasked — that's a "can't read the room" signal.

**H2. ⭐ Idempotency — define it, then classify the verbs.**
Same request repeated has the same *effect on server state* as once (not the same response — a repeated DELETE returns 404 the second time and is still idempotent). GET/PUT/DELETE/HEAD/OPTIONS: idempotent. **POST/PATCH: not.** GET is also *safe* (no state change at all) — which is why a browser prefetch, a crawler, or a proxy may replay it, and why `GET /orders/1/delete` is a real outage story.

**H3. ⭐ PUT vs PATCH vs POST?**
PUT replaces the resource wholesale at a known URI (idempotent — send it twice, same final state). PATCH applies a partial modification (not idempotent in general: "increment by 1" isn't). POST creates/acts, server picks the URI, not idempotent. Practical trap: implementing PUT with a partial DTO means the omitted fields get **nulled** — that's PUT semantics behaving correctly, and it's a real bug when people wanted PATCH.

**H4. Status codes — the ones you actually use?**
200 OK · 201 Created **+ `Location` header** · 202 Accepted (queued, not done — the async/background answer) · 204 No Content (DELETE/PUT with nothing to say) · 301 vs 302 (permanent, cached forever by the browser — a wrong 301 is very hard to take back — vs temporary) · 304 Not Modified · 400 · 401 · 403 · 404 · 405 · 409 · 415 · 422 · 429 · 500 · 502/503/504.

**H5. ⭐ 401 vs 403?**
401 Unauthenticated: I don't know who you are — re-authenticate (and it should carry `WWW-Authenticate`). 403 Unauthorized: I know who you are and you still can't. Naming quirk to note: the spec labels 401 "Unauthorized," which is why everyone mixes them up. Security nuance worth a bonus point: for a resource you don't own, some APIs return **404 instead of 403** to avoid confirming the resource exists.

**H6. ⭐ 400 vs 422?**
400: malformed/unparseable or failed shape validation. 422: syntactically fine, semantically impossible — "cancel a booking that's already cancelled," "end before start." Say the meta-answer: *"either is defensible; what matters is that the API is consistent and the client can distinguish 'fix your JSON' from 'fix your intent.'"* The awareness is what's tested, not the choice.

**H7. When is 409 the right code?**
State conflict: duplicate unique value, concurrent edit lost the version check, booking the slot someone just took. It says "your request was fine, the world changed." That's your `DbUpdateConcurrencyException` and your unique-violation catch, both mapping to 409.

**H8. 500 vs 502 vs 503 vs 504?**
500: my code threw. 502: I'm a proxy and got garbage from upstream. 503: I'm up but can't serve right now (overloaded/shutting down — pair with `Retry-After`). 504: I'm a proxy and upstream timed out. Practical value: when you're paged, the code tells you *which box* to look at.

**H9. What's in an HTTP request?**
Method, path, HTTP version, headers, optional body. Response: status line, headers, body. Headers you should know cold: `Content-Type`, `Accept`, `Authorization`, `Cache-Control`, `ETag`/`If-None-Match`, `Location`, `Retry-After`, `Set-Cookie`.

**H10. HTTP/1.1 vs HTTP/2 vs HTTP/3?**
1.1: one request at a time per connection → head-of-line blocking → browsers open ~6 connections and everyone invents workarounds (sprites, domain sharding). 2: multiplexed streams over one TCP connection + header compression → those workarounds became counterproductive. 3: same idea over **QUIC/UDP**, which removes TCP-level head-of-line blocking and survives network switches. Enough to be conversational; you're not being hired for this.

**H11. What's a preflight request?**
For non-simple cross-origin requests, the browser sends `OPTIONS` first with `Access-Control-Request-Method/Headers`; the real request only goes if the server allows it. Costs a round-trip → hence `Access-Control-Max-Age`. The key insight: **the server still receives and could act on the real request** if a client chooses to send it — CORS is browser-enforced, not server-enforced.

**H12. Content negotiation?**
Client sends `Accept: application/json`; the server picks a formatter and answers with `Content-Type`. Mismatch → **406** (can't produce) or **415** (can't consume what you sent). Mostly invisible in a JSON-only API, and that's a fine thing to say.

**H13. `Content-Type: application/json` vs `application/problem+json`?**
The second is ProblemDetails (RFC 7807/9457) — a standard machine-readable error shape. Using it means clients get one error contract across your API instead of a bespoke `{ "error": "..." }` per endpoint.

---

## Caching & conditional requests

**H14. `Cache-Control` — the directives that matter?**
`no-store` (never persist — auth responses, PII), `no-cache` (store but revalidate every time — the confusing one; it does *not* mean "don't cache"), `private` (browser only, not shared proxies), `public`, `max-age=seconds`. Getting `no-cache` vs `no-store` right is a small, high-signal detail.

**H15. ⭐ What's an ETag and how is it used?**
A version identifier for a representation. Client re-requests with `If-None-Match: "abc"`; unchanged → **304 Not Modified** with no body (saves bandwidth, not server work — you still computed the ETag). The better use for you: `If-Match` on a **PUT** gives you optimistic concurrency **over HTTP** — 412 Precondition Failed instead of a lost update. That's your EF concurrency token, one layer up, and connecting the two is a great interview moment.

**H16. 304 vs 200 from cache?**
304: the client asked, the server said "yours is still good" — a round-trip happened. 200 (from cache): no request at all, `max-age` hadn't expired. Freshness avoids the network; validation shortens it.

**H17. Where can caching happen?**
Browser → CDN/proxy → your app (output cache/`IMemoryCache`) → the database's own buffers. Each layer has its own invalidation problem, and each stale layer is a support ticket. It's why the honest answer to "should we cache this" starts with "how wrong can it be, and for how long?"

---

## REST design

**H18. ⭐ What does RESTful actually mean?**
Resources identified by URIs, manipulated with uniform verbs, stateless requests, representations (JSON), and standard status codes. Then the honest sentence: *"almost nobody implements HATEOAS, so most 'REST' APIs are really pragmatic HTTP APIs — I aim for consistent, predictable resource design rather than Fielding purity."* That candor plays well; reciting the constraints like a textbook does not.

**H19. Nouns or verbs in URIs?**
Nouns, plural, hierarchical: `/bookings/42/cancellations`, not `/getBooking?id=42`. The verb is the HTTP method. Then the pragmatic exception: genuine actions that aren't resources (`POST /bookings/42/cancel`) are fine and common — **shipping a consistent API beats winning a URI argument.**

**H20. How deep do you nest?**
One level, usually: `/providers/7/slots` when the child is meaningless without the parent. Past that (`/a/1/b/2/c/3`) it's unreadable and brittle — flatten to `/bookings?providerId=7`. Nesting encodes a relationship you may want to change.

**H21. ⭐ How do you version an API?**
URL segment (`/v1/`): visible, cache-friendly, trivially routable — the pragmatic default. Header/media-type: cleaner URIs, invisible to a browser, harder to debug. Query param: easy, ugly. The mid-level point beneath all three: **most changes shouldn't need a version at all** — adding a field is backwards-compatible; removing or renaming one is what forces v2. Version on breaking change, and have a deprecation path.

**H22. What counts as a breaking change?**
Removing/renaming a field, tightening validation, changing a type, changing a status code's meaning, making an optional field required, changing default behavior. Not breaking: adding an optional field, adding an endpoint, adding an enum value **if** clients tolerate unknowns — and that "if" is exactly why enums in public contracts are a trap.

**H23. ⭐ How do you design a list endpoint?**
Pagination **by default from day one** (never an unbounded list — retrofitting it is a breaking change and a mid-level lesson), a max page size the server enforces, filtering via query params, sorting with an **allowlist** of sortable fields (a raw `?sort=` into SQL is an injection hole), and a consistent envelope with the items plus a next-cursor or total. Say "keyset for deep pages, offset for a page-number UI, and I know why."

**H24. Filtering conventions?**
Simple equality as `?status=open&providerId=7`. Ranges as `?from=&to=`. Resist inventing a query language (`?filter=status eq 'open'`) unless you're implementing OData for real — you'll spend a year on a parser. Cap the combinations you support, and index the ones you promise.

**H25. Envelope or bare array?**
Bare array is clean but has nowhere to put paging metadata, and it can't grow without breaking. An envelope (`{ items, nextCursor, total }`) is future-proof. Either is defensible; **consistency across the API is not optional.** Note `total` is often an expensive `COUNT(*)` on a big filtered table — I'd make it opt-in.

**H26. Should DELETE be idempotent, and what do you return?**
Yes: 204 on success. The second call — 404 or 204? Both defensible: 404 is honest, 204 is friendlier to retries. Pick one, document it. Soft delete changes nothing about the contract; it changes your query filters.

**H27. Error contract for a validation failure?**
ProblemDetails with per-field errors — `ValidationProblemDetails` gives you that free from `[ApiController]`. Client needs to bind errors to form fields; a flat string can't. And never leak stack traces or SQL — that's an information-disclosure finding, not just untidiness.

**H28. How would you design a bulk endpoint?**
The interesting part is failure: all-or-nothing (one transaction, one 400 for the whole batch, easy to reason about) vs per-item results (`207`-style body with per-row status — better UX, more client complexity). Ask which the client needs. Also: cap the batch size, or someone posts 100,000 rows on a Friday.

**H29. Sync or async for a slow operation?**
If it's beyond a couple of seconds: **202 Accepted** + a status URL or a job id, do the work in the background, let the client poll or get a webhook/SignalR push. Holding an HTTP connection for 60 seconds fails to timeouts and proxies you don't control, and it burns a thread of yours for the duration.

---

## Reliability & real-world

**H30. ⭐ How do you make a POST safe to retry?**
**Idempotency keys**: the client sends `Idempotency-Key: <uuid>`; you store it with the result in a table with a **unique constraint**; a repeat with the same key returns the stored response instead of doing the work again. This is how Stripe does payments and it's the correct answer to "the client's network dropped after we charged them — did it go through?" Notice it's the same primitive as everything else in your portfolio: **a unique constraint doing the real work.**

**H31. At-least-once vs at-most-once vs exactly-once?**
At-least-once: retry until acked → **duplicates possible** → consumers must be idempotent. At-most-once: fire and forget → loss possible. Exactly-once **doesn't exist end-to-end** across a network — what people mean is at-least-once delivery plus idempotent processing, which gives you exactly-once *effects*. Saying that sentence correctly is a genuine differentiator at mid-level.

**H32. Webhooks — what do you have to get right?**
Verify the signature (HMAC of the body with a shared secret) so anyone can't POST you fake events; return **fast** (queue the work, 200 immediately, or the sender times out and retries); be **idempotent** on the event id, because every webhook provider retries; and handle out-of-order arrival. Also: your endpoint is public and unauthenticated by design, which makes it the most-attacked surface you own.

**H33. Rate limiting — what would you return?**
**429** with `Retry-After`, and ideally `X-RateLimit-Remaining`/`Reset`. Algorithms: fixed window (simple, allows a 2× burst at the boundary — the classic flaw), sliding window (fairer, more state), **token bucket** (allows bursts up to a cap, my default for APIs), leaky bucket (smooths output). Partition by user/API key, not just IP — NAT means one IP is an office.

**H34. Timeouts — where do they belong?**
Everywhere: HttpClient (default is 100 seconds, which is effectively "forever" in a request path), DB command timeout, the whole request. The reason to voice: without a timeout, a slow dependency **converts into an outage of yours** — your threads pile up waiting, and now you're down too. Timeout + retry + circuit breaker is one thought, not three.

**H35. What's a circuit breaker, in one sentence?**
After N consecutive failures, stop calling the dependency for a cooldown and fail fast — it protects *you* from wasting resources on a dead service and protects *them* from your retry storm while they recover. Half-open state probes with one request before fully closing.

**H36. A client complains your API is slow. First questions?**
Which endpoint, since when, all clients or one, what's your p95 vs my p95 (network vs server), what payload sizes, are you paginating, are you calling it in a loop? Half the time the answer is **the client is doing N+1 over your API** — which is a design conversation about a missing bulk/expand endpoint, not a performance bug.

**H37. GraphQL vs REST — do you have a view?**
Yes, briefly: GraphQL solves over/under-fetching and the "mobile needs a different shape" problem, at the cost of caching (everything is a POST to one URL), query-complexity attacks (a client can request a cartesian explosion — you need depth/cost limits), and N+1 in resolvers (needs DataLoader). For a CRUD product with known clients, REST is simpler and I'd default to it. **Having a view and its cost is the answer**; enthusiasm without the costs is a junior tell.

**H38. When would you use a queue instead of an HTTP call?**
When the caller doesn't need the result now (email, report generation), when the receiver can be down without failing the caller (decoupling), when you need to absorb bursts (backpressure), or when the work must survive a crash (durability). The cost you must name: eventual consistency, duplicate delivery, ordering, and a whole new failure mode to monitor — a queue is a distributed system you just adopted.

---

## Coverage tracker

`H1–H13` HTTP ⭐ · `H14–H17` caching · `H18–H29` REST design ⭐ · `H30–H38` reliability ⭐

**The five that decide the round:** H2 (idempotency, defined precisely), H5/H6 (the status code distinctions), H15 (ETag → 412, the HTTP-level version of your EF concurrency story), H23 (list endpoint design, unprompted pagination), H30 (idempotency keys — and noticing it's a unique constraint again).
