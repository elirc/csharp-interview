# Security & Auth — 42 Questions (`X`)

Security questions at mid-level test *instincts*, not cryptography. Nobody expects you to implement AES. They expect you to never write your own hash, to reach for a parameterized query without thinking, and to know that the most common real-world API vulnerability is boring: **A01, broken access control** — user A fetching user B's booking by changing the id.

Log misses as `X##`.

---

## Authentication vs authorization

**X1. ⭐ Authentication vs authorization?**
Who you are vs what you may do. In ASP.NET Core they're two middlewares in that order, and the order is mandatory: authorization needs an identity to evaluate. AuthN → 401, AuthZ → 403.

**X2. ⭐ How does the server know who's calling on request #2 if HTTP is stateless?**
The client sends proof every time: a **cookie** (server-side session id, or an encrypted auth ticket) or a **bearer token** in the `Authorization` header. That's the entire mechanism — there's no memory of you between requests, only what you re-present.

**X3. ⭐ JWT vs session cookies — the real tradeoff?**
Session: state lives on the server, so it's **instantly revocable** and the token is opaque, but you need a shared store when you scale past one instance, and cookies bring CSRF. JWT: stateless — any instance validates it with a key, no lookup, great for services and mobile — but you **cannot revoke it before expiry**, so you mitigate with short-lived access tokens (5–15 min) + refresh tokens. Answer with the mitigation and you've answered the follow-up they were about to ask.

**X4. ⭐ JWT anatomy, and why can't a client forge one?**
`header.payload.signature`, base64url-encoded. The signature is HMAC (shared secret) or RSA/ECDSA (private key) over `header.payload`. Change one byte of the payload and the signature no longer verifies. **It is signed, not encrypted** — anyone can read the claims at jwt.io, so never put anything secret in there. "Base64 is encoding, not encryption" is the sentence.

**X5. What do you validate on an incoming JWT?**
Signature, **issuer**, **audience**, **lifetime** (with a small clock skew), and the algorithm. Nuance: audience and issuer matter because a validly-signed token from another app/tenant on the same key is still a valid signature — you'd accept someone else's token. This is a real breach class, not a formality.

**X6. What's the `alg: none` attack?**
Old JWT libraries honored a header claiming "no signature" and accepted unsigned tokens — attacker rewrites the payload to `role: admin` and walks in. Lesson generalizes: **never let untrusted input choose the verification algorithm.** Modern libraries pin allowed algorithms; the `kid` header selecting a key must be validated against a known key set, not used to fetch arbitrary keys.

**X7. Where do you store a JWT in a browser?**
Real tradeoff, no free lunch: `localStorage` is readable by **any XSS** on your origin — one bad npm package and every token is exfiltrated. `httpOnly` cookie is invisible to JS but rides along automatically → **CSRF** to handle (mitigated by `SameSite=Lax/Strict` + antiforgery tokens). Current consensus for first-party web apps: httpOnly + Secure + SameSite cookie, and treat XSS as game-over regardless.

**X8. What's a refresh token and how do you handle rotation?**
A long-lived credential exchanged for new access tokens, stored server-side (hashed) so it *is* revocable. **Rotation**: each use issues a new refresh token and invalidates the old — and if an old one is reused, that means it was stolen and replayed, so you revoke the whole family. That detection story is the good answer.

**X9. Claims — what are they and where do authorization decisions live?**
Key/value assertions inside the identity (`sub`, `role`, `tenant_id`). They authorize **coarse** decisions (is this an admin). What they cannot do is answer "does this user own booking 42" — that's a **database check**, because the token was issued before that booking existed. Candidates who put ownership in claims have a broken mental model, and this distinction is worth stating unprompted.

**X10. Role-based vs policy-based vs resource-based authorization?**
Roles: coarse (`[Authorize(Roles="Admin")]`) — simple and rigid. Policies: named requirements + handlers (`[Authorize(Policy="Over18")]`) — testable, composable, my default. Resource-based: `IAuthorizationService.AuthorizeAsync(User, booking, "CanEdit")` — the only one that answers **ownership**, since it needs the actual row. Naming all three, and that ownership needs the third, is a strong answer.

**X11. OAuth2 vs OIDC?**
OAuth2 is **authorization delegation** — "let this app act on my behalf at that API," it says nothing about who you are. OIDC is a thin identity layer on top that adds the **id_token** and `/userinfo`. "Sign in with Google" is OIDC; "let this app read my Drive" is OAuth2. Using OAuth2 alone for login is the classic misuse.

**X12. Which OAuth flow for a SPA, and why not implicit?**
**Authorization Code + PKCE.** Implicit is deprecated: it returned tokens in the URL fragment — landing in browser history, referrers, and logs. PKCE binds the code to the client with a one-time verifier, so an intercepted code is useless. Machine-to-machine → **client credentials**. Knowing implicit is dead is a currency check.

**X13. Would you build your own identity provider?**
No. ASP.NET Core Identity for local accounts, or an external IdP (Entra ID, Auth0, Keycloak, Duende) for anything real. Reason: password reset, MFA, lockout, token rotation, and breach response are a product, not a feature. "I'd rather integrate than maintain a login page that gets me in the news."

---

## Passwords & crypto

**X14. ⭐ How do you store passwords?**
A **slow, adaptive, salted** hash: Argon2id (current preference), bcrypt, or PBKDF2 (what ASP.NET Core Identity ships). Per-password random salt (defeats rainbow tables), and a work factor tuned so it costs ~100ms. Never encryption — that's reversible and implies a key someone can steal. Never a fast hash (SHA-256, MD5) — a GPU does billions/sec, so your whole leaked table cracks over a weekend.

**X15. Why is slowness a *feature*?**
Because the attacker's advantage is throughput. You verify one password per login; they try billions. Make each attempt cost 100ms and offline cracking becomes economically pointless, while your user never notices. Increase the work factor as hardware improves — that's what "adaptive" means.

**X16. Salt vs pepper?**
Salt: unique per password, stored **alongside** the hash — stops precomputation and stops two users with the same password having the same hash. Pepper: a secret added to every hash, stored **outside** the database (env var/HSM) — so a pure DB dump is uncrackable without the app's secret. Salt is mandatory; pepper is a nice defense-in-depth.

**X17. Hashing vs encryption vs encoding?**
One-way (passwords, integrity) · two-way with a key (secrets you must read back) · **not security at all** (base64, URL encoding — a transport format). "Base64 encrypted" in a code review is a red flag and a great thing to be able to name instantly.

**X18. What's a timing attack, and where would you meet one?**
Comparison time leaks information: `==` on a byte array/string returns early at the first difference, so an attacker measures response times to recover a token byte by byte. Anywhere you compare a **secret** (API key, HMAC signature, reset token) use `CryptographicOperations.FixedTimeEquals`. Niche but a very good signal.

**X19. Generating a password reset token — how?**
`RandomNumberGenerator` (CSPRNG) — never `System.Random`, which is predictable and seedable, so an attacker who sees a few tokens can generate the rest. Then: store it **hashed**, short expiry, single-use, invalidated on use and on password change. Same rule as passwords: the DB should never hold something that grants access as-is.

**X20. `Guid.NewGuid()` as a security token?**
V4 GUIDs are random and usually from a CSPRNG in .NET, so it's *often* fine in practice — but the type doesn't promise it and it's only 122 bits with no guarantee across platforms. Use a CSPRNG and say why: "I don't want my security property to depend on an implementation detail of a type designed for uniqueness, not secrecy."

**X21. What is the ASP.NET Core Data Protection API?**
The framework's built-in key management for encrypting/signing things it owns — auth cookies, antiforgery tokens, `[TempData]`. Ops trap that bites everyone: keys default to the local filesystem/registry, so in a **multi-instance or containerized** deployment every instance has different keys → cookies issued by one are rejected by another → users randomly logged out. Fix: a shared key ring (Redis/blob) + persistence across deploys. Genuinely practical knowledge.

---

## Web attack surface

**X22. ⭐ XSS — the types and the fix?**
Injecting script into a page: **stored** (persisted, served to everyone — the worst), **reflected** (bounced off a query param), **DOM-based** (client JS writes untrusted data into the DOM). Fix: **context-aware output encoding** — Razor encodes by default, React escapes by default; you defeat both with `@Html.Raw` and `dangerouslySetInnerHTML`. Defense in depth: a CSP, and httpOnly cookies so a successful XSS at least can't read the session.

**X23. Why does XSS matter so much for an API dev?**
Because the *consequence* lands on you: an XSS on the frontend can read `localStorage`, take the token, and call **your** API as that user — perfectly authenticated, indistinguishable from the real thing. That's why token storage is an API concern too, and it's the answer to "why not localStorage."

**X24. ⭐ CSRF — what is it and when does it apply?**
The attacker's page makes the victim's browser POST to your site; the browser **helpfully attaches the cookie**, so the request is authenticated even though the user never intended it. It only applies to **ambient credentials** (cookies, basic auth) — a bearer token in an `Authorization` header is not attached automatically, so a pure token API isn't CSRF-vulnerable. Fixes: antiforgery tokens + `SameSite=Lax` (the browser default now), and never mutate state on GET.

**X25. `SameSite` — the values?**
`Strict` (never cross-site — logging in from a link breaks), `Lax` (sent on top-level GET navigations; the modern browser default and a good default), `None` (always — **requires `Secure`**, and it's what breaks third-party embeds). Knowing Lax is the default explains why CSRF got dramatically less common without anyone doing anything.

**X26. Cookie flags you always set?**
`HttpOnly` (no JS access → XSS can't read it), `Secure` (HTTPS only → not sent in cleartext), `SameSite` (CSRF), plus a sensible `Path`/`Domain` scope and an expiry. Four flags, four distinct attacks — a tidy answer.

**X27. ⭐ SQL injection — what is it, and is EF safe?**
Untrusted input concatenated into SQL so the data becomes code (`'; DROP TABLE--`). Parameterization fixes it because parameters are never parsed as SQL. EF parameterizes LINQ automatically, and `FromSql($"...{id}")` (interpolated) is **safe** — it builds parameters. `FromSqlRaw("... " + id)` is **not**. And the one everybody forgets: **dynamic ORDER BY / column names cannot be parameterized** — you need an allowlist. Cite your sort-field allowlist; it's the exact case.

**X28. Other injections?**
Command injection (shelling out with user input), LDAP, XPath, NoSQL, **log injection** (a newline in a username forges log lines), and template injection. Same shape, same rule: never let data become code — use the API's parameterized path, don't build the string.

**X29. What's SSRF, and why is it on the Top 10 now?**
Your server fetches a URL the user supplied → an attacker points it at `http://169.254.169.254/` (cloud metadata) or your internal network, and now they're reading credentials **from behind your firewall** using your server as a proxy. It's how Capital One happened. Mitigate: allowlist destinations, block private IP ranges *after* DNS resolution (or the redirect will get you), disable redirects, and use a separate egress path. Any "import from URL" or webhook-registration feature is an SSRF question.

**X30. ⭐ Mass assignment / over-posting?**
Binding the request body straight to an entity lets a client set fields you never exposed: `{"email":"...","isAdmin":true}`. Fix: an input DTO containing **only** what's settable, and take server-controlled values (user id, timestamps, status) from the token or the server, never the body. This is the whole reason "don't bind to entities" is a rule and not a style preference.

**X31. IDOR — and how do you actually prevent it?**
Insecure Direct Object Reference: `GET /bookings/42` returns someone else's booking because you checked authentication but not **ownership**. It's the most common real API vulnerability. Prevent structurally, not by remembering: **scope every query by the caller** — `Where(b => b.Id == id && b.CustomerId == currentUserId)` — so a missing check returns 404 instead of leaking. Guessable ids make it trivial; GUIDs are obscurity, not a fix.

**X32. ⭐ OWASP Top 10 — which do you actually handle?**
Four with concrete mitigations beats ten names: **A01 Broken Access Control** (ownership checks on every fetch — the #1 by real incident count), **A03 Injection** (parameterized everything + a sort allowlist), **A02 Cryptographic Failures** (Argon2/PBKDF2 for passwords, TLS everywhere, no secrets in git), **A07 Auth Failures** (short token expiry, lockout, no username enumeration). Bonus points: **A04 Insecure Design** exists — meaning some vulns are architecture, not bugs — and **A06 Vulnerable Components**, which is `dotnet list package --vulnerable` in CI.

**X33. Username enumeration?**
"No such user" vs "wrong password" tells an attacker which emails are registered — that's a user list for credential stuffing. Same for registration ("email taken") and password reset. Fix: identical messages and similar **timing** on both paths, and always send the reset email ("if that address exists, we sent a link"). Small thing; shows you think like an attacker.

**X34. How do you defend against credential stuffing / brute force?**
Rate limit + progressive lockout per account **and** per IP, MFA, breached-password checks (HaveIBeenPwned's k-anonymity API), and CAPTCHA as a last resort. Understand the threat: attackers use passwords **already leaked elsewhere**, so complexity rules don't help — MFA and breach checks do.

---

## Practice & operations

**X35. ⭐ Where do secrets live?**
Dev: user-secrets (`dotnet user-secrets`) or env vars — never appsettings.json. Prod: platform env vars or a secret manager (Key Vault/Secrets Manager) with managed identity so nothing is typed anywhere. Never in git. And the ritual: a leaked secret must be **rotated**, not deleted — a force-push doesn't erase what's already cloned, forked, and scraped by bots within minutes.

**X36. A secret got committed. Walk me through it.**
Rotate first (assume compromise from the moment of push), then audit for use, then purge history (BFG/filter-repo, force-push, notify everyone to re-clone), then prevent (pre-commit scanning, GitHub secret scanning + push protection). The order is the answer: **containment before cleanup.** Candidates who start with "rewrite history" are optimizing the wrong thing.

**X37. Principle of least privilege — a concrete example?**
The app's DB user doesn't need DDL rights if migrations run as a separate deploy identity. An API key scoped to read-only. A container that doesn't run as root. Least privilege is what turns a compromise into an incident instead of a catastrophe — and it's the argument against `Database.Migrate()` at startup, which requires DDL rights in the app's own credentials, forever.

**X38. Defense in depth — say it with your project.**
The booking overlap: the UI hides taken slots, the service checks availability, the database enforces an exclusion constraint. Three layers; the last one is the only one that's actually a guarantee, and the first two exist for UX and error quality. Security has the same shape — assume every layer above will eventually fail.

**X39. What's in a dependency vulnerability workflow?**
`dotnet list package --vulnerable --include-transitive` in CI, Dependabot/Renovate PRs, and a policy for how fast criticals get patched. Point to make: **transitive** dependencies are the risk — you didn't choose them and you probably don't know they're there. Reference the supply-chain reality (a compromised package publishes to thousands of apps at once); that's A08.

**X40. HTTPS — what does the handshake actually give you?**
Server authentication (the cert chains to a CA you trust — this is what stops a man in the middle), key exchange to derive session keys, then symmetric encryption for the data. So: confidentiality, integrity, and authenticity of the *server*. HSTS tells the browser "never speak http to me again" so the first request can't be downgraded. And TLS terminating at your load balancer means the hop behind it is plaintext — know where your encryption actually stops.

**X41. How do you protect PII specifically?**
Collect less (the cheapest control by far), encrypt at rest and in transit, mask in logs and error messages, restrict by role, and have a deletion path — GDPR's "right to erasure" is an architectural requirement, and it's genuinely hard when you have backups, replicas, and an append-only event log. Naming that tension shows real-world thinking.

**X42. ⭐ You're asked to add a "user uploads a profile picture" feature. What are the security concerns?**
A great open-ended one — think in surfaces: validate **content**, not just the extension (magic bytes; a .jpg can be a script); cap size (DoS via a 4GB upload, or a decompression bomb); store **outside the webroot** or in blob storage so nothing you accept is ever executed; generate your own filename (path traversal via `../../`, and Windows reserved names); serve from a different origin/domain with `Content-Disposition` and a strict `Content-Type` so a stored HTML file can't XSS your main origin; strip EXIF (that photo carries GPS coordinates); and scan for malware if it's ever re-downloaded by others. If a URL fetch is involved: SSRF. Reeling off five of those calmly is a *very* strong signal — it shows you threat-model a feature rather than a checklist.

---

## Coverage tracker

`X1–X13` authN/authZ ⭐ · `X14–X21` crypto ⭐ · `X22–X34` attack surface ⭐ · `X35–X42` practice

**The five that decide the round:** X31 (IDOR + scoping queries structurally), X30 (mass assignment — the DTO rule's real reason), X27 (SQLi + the ORDER BY allowlist), X14 (password storage + *why* slow), X3 (JWT vs session, answered with the revocation mitigation).

**The mid-level bar, honestly stated:** you are not expected to be a security engineer. You are expected to (1) never hand-roll crypto, (2) check ownership on every resource fetch, (3) never concatenate SQL, (4) never bind to entities, (5) keep secrets out of git, and (6) say "I'd get a review on this" when something is genuinely over your head. That last one is a *strength* — the failure mode interviewers fear is the confident junior who ships a hand-rolled auth scheme.
