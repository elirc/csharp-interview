# System Design Lite — The Mid-Level Version

Nobody expects a junior→mid candidate to design Netflix. The round tests: can you turn a vague product into **a sane schema + API + one correctly-handled hard part**, out loud, with tradeoffs. That's ~35 minutes of structured talking, and it's fully scriptable.

## The script (same shape every time — practice until it's your reflex)

**1. Requirements (5 min).** 3–4 clarifying questions MAX, then move: Who are the users/roles? Rough scale (10 users or 10M — changes everything)? Which operations dominate — reads or writes? What's the consistency-critical part (money? inventory? uniqueness?)
*Saying "at this scale I'll start simple and name where it breaks" is itself a correct answer.*

**2. Data model (10 min).** Entities → fields that matter (not exhaustive) → relationships + cardinality → constraints, stated explicitly ⭐: "unique on X, and I'd enforce the no-overlap invariant with a DB constraint, not just app code — app checks race." (Your Bookable story generalizing = the strongest minutes of the round.)

**3. API (8 min).** Core endpoints, REST-shaped, verbs + codes; pagination on every list by default; auth in one line ("JWT, role + ownership checks — every fetch validates the caller can see that row").

**4. The hard part (8 min) — where the hire decision happens.** Every prompt has one; name it *before they ask*: "The interesting problem here is [double-spend / duplicate submission / hot reads / ordering]. Here's how I'd handle it…" Pattern-match from your toolkit: check-then-act race → constraint + optimistic retry · slow side effects → outbox + background worker · read-heavy hot path → cache + invalidation caveat · duplicate requests → idempotency keys.

**5. Scale & evolution (4 min).** One honest sentence each: first bottleneck (usually the DB) → first moves (indexes, read replicas, cache) → what you'd measure to know (latency percentiles, slow-query log). **Do not** volunteer microservices/Kafka/sharding at whiteboard-scale; reaching for distributed systems before measuring is a *negative* signal at this level.

## Practice problems (1–2/week from week 7; do them in a markdown file as the whiteboard)

Each: 35-min timebox, then the grading prompt below.

1. **URL shortener** — hard part: short-code uniqueness under concurrent creates (collision retry vs pre-generated pool); redirect read path is 100:1 reads → cache
2. **Parking garage reservations** — Bookable's cousin *(do this one first — feel the transfer)*; hard part: the overlap invariant
3. **Job board with applications** — hard part: application state machine + double-apply prevention (unique constraint again — notice the pattern repeating)
4. **Voting/polls service** — hard part: one-vote-per-user (unique) + hot read of tallies (cache w/ short TTL vs count-on-write)
5. **Expense approvals at company scale** *(sim-2 domain, zoomed out)* — hard part: approval routing rules + audit trail (append-only events)
6. **Waitlist/queue for a class signup** — hard part: capacity + ordered promotion when a spot frees (transactional pop; at-least-once notification)
7. **Notification preferences + delivery system** — hard part: fan-out writes vs read-time merge; retry/dead-letter thinking (your outbox, scaled up)

Notice what the practice teaches: **it's the same four hard parts wearing different costumes** — uniqueness under concurrency, state machines, hot reads, reliable side effects. You've *built* all four. Design rounds are recognition, not invention.

## Mock prompt

> Run a 35-minute system-design-lite interview for a mid-level backend role. Problem: [pick from list / invent similar]. I'll talk through it in stages (requirements → data model → API → hard part → scale), writing in a markdown whiteboard. Interrupt with realistic follow-ups: challenge one schema choice, ask "what if two users do X simultaneously", and probe one scale question. Then grade: structure, data modeling, correctness instincts on the hard part, communication, tradeoff honesty — 1–5 each with the one thing to fix next time.

## Phrases that score (because they're true for you)

- "App-level checks give good error messages; the constraint gives correctness. You need both — I've watched the race happen in a test."
- "I'd store money as integer cents — floats can't represent 0.1."
- "Every list endpoint paginates from day one; retrofitting is miserable."
- "Send the email after commit via an outbox row — I've seen why fire-and-forget drops work."
- "At this scale that's premature — I'd add the index, measure, and revisit."
