# Mock Screener Scripts — 3 Full Rounds

The vault files drill questions in isolation. Real interviews are **45–60 minutes of sustained performance** with follow-up chains, interruptions, and a fatigue curve — a different skill, and one you can only train by simulating the whole thing.

Run one per week from week 7, in a single unbroken sitting, **out loud**, timed. Hand the script to Claude and let it play the interviewer.

> **Setup prompt:** You are running mock interview [N] from this file. Follow the script's structure and timings, but behave like a real interviewer: ask follow-ups when my answer is shallow, interrupt if I ramble past ~90 seconds, skip ahead if I'm clearly strong on something, and go off-script if I say something interesting or wrong. Do not tell me how I'm doing until the end. At the end, grade me with the rubric at the bottom of the script, with specifics and one behavior to change.

**Rules for you:** no notes, no IDE (except where the script says), no pausing to look things up. Say "I don't know" out loud when true — you're training that too, and it's a *passing* answer.

---

# Script 1 — The 30-Minute Phone Screener

**Who:** an engineer, not a recruiter. Calibrating whether you're worth a full loop. **Bar: no red flags, no bluffing.**
**Shape:** breadth, fast, few follow-ups. They're checking boxes, not exploring.

### 0:00 — "Tell me about yourself." *(2 min)*
**Tests:** can you be concise and relevant. **Not** your life story.
**Shape:** current situation → what you build → the one interesting thing → what you're looking for. **90 seconds, rehearsed.** Land on Bookable and the concurrency problem — you're planting the flag you want them to ask about.
**Red flag:** 5 minutes, chronological from school. **Green flag:** ends with a hook they want to pull.

### 0:02 — "Walk me through a project you're proud of." *(5 min)*
**Tests:** ownership, technical depth, communication.
**Shape:** problem → your design → **the hard part** → the tradeoff you made → what you'd do differently. Do the *90-second version* and stop. Let them ask for depth — a candidate who self-terminates at the right moment reads as senior; one who talks for 8 minutes reads as unaware.
**Guaranteed follow-up:** "Why optimistic concurrency and not a lock?" → have `S47`/`E33` ready.

### 0:07 — Rapid technical *(12 min, ~8 questions, ~90 sec each)*
Straight from the core bank and the vault. Expect roughly:
1. "DI lifetimes — and what breaks when you get them wrong?" *(`W7`, `W8`)*
2. "What does `async`/`await` actually do? Does it make a request faster?" *(`A1`, `A2`)*
3. "What's an N+1 query and how would you find one?" *(`E14`, `B7`)*
4. "Difference between `IEnumerable` and `IQueryable`?" *(`E18`)*
5. "When would you return 409?" *(`H7`)*
6. "How do you store passwords?" *(`X14`)*
7. "How would you get all customers with zero orders?" *(`S3` — spoken, not typed)*
8. "How do you test an API?" *(`T1`, `T2`)*
**Tests:** fluency. **These are pass/fail on hesitation** — a correct answer delivered slowly reads as reconstruction, not knowledge.
**Discipline:** answer in the four beats (claim → why → tradeoff → my project) and **stop**. The most common screener failure is a right answer that keeps going for three minutes and buries the signal.

### 0:19 — One curveball *(4 min)*
Something you probably haven't hit — "What's a thread pool starvation and how would you diagnose it?" *(`A19`)*, or "How would you do a zero-downtime schema change?" *(`E41`, `O28`)*.
**Tests:** how you behave at the edge of your knowledge. **This is the real question of the round.**
**The move:** say what you *do* know, be explicit about the boundary, say how you'd find out. *"I know the symptom is latency climbing while CPU stays low, and the cause is almost always blocking calls. I'd check thread pool queue length with dotnet-counters. I haven't debugged one in production — I'd want to look at a dump."* That's a **pass**. Bluffing is the only fail.

### 0:23 — "Any questions for me?" *(5 min)*
**Tests:** whether you're evaluating them (senior behavior) or auditioning (junior behavior).
**Have three, always, and make them specific:** "How do schema changes reach production here?" · "What does code review look like — what gets blocked?" · "What's the on-call/incident story for this team?" · "What would I own in the first six months?"
**Never:** "What's the culture like?" (nothing), or nothing at all (reads as uninterested or done).

### Screener pass bar
Zero bluffs · every rapid answer inside 90 seconds · the project story landed and provoked a follow-up · you asked something specific. **Technical perfection is not required. Not-being-a-risk is.**

---

# Script 2 — The 60-Minute Technical Deep Dive

**Who:** a senior/lead who will work with you. **Bar: can you go three questions deep and still be standing.**
**Shape:** they pick 3–4 areas and **drill until you hit your limit** — that's not hostility, it's calibration. *Everyone* bottoms out; where you bottom out, and how you behave there, is the grade.

### 0:00 — Project deep dive *(15 min)*
Start with the 90-second Bookable story, then expect the chain:
- *"Walk me through what happens when two people book the same slot at the same millisecond."* → `E35`, `B4`
- *"Why not just lock the row?"* → `S49` — held locks across app logic, contention, deadlock risk; optimistic fits web because conflicts are rare
- *"Your version column — does that actually stop two inserts?"* → **`E35`. This is the question.** "No — two INSERTs have no existing row to mismatch. That's why the guarantee is a database exclusion constraint, not the token."
- *"So why keep the app-level check at all?"* → UX and error quality; the constraint is correctness. Defense in depth *(`X38`)*
- *"How do you know it works?"* → the parallel test: N concurrent requests, exactly one 201, and **you've watched it fail with the constraint removed** *(`T26`)*
- *"What would you do differently?"* → have a real answer. "I'd have written the concurrency test first — I built the feature, then proved it, and the test found a boundary bug in the overlap predicate that I'd have caught a day earlier."
**Tests:** is the story yours. Fabricated depth collapses at question three; this chain is *designed* to find that.
**Green flag:** you name the limitation before they find it.

### 0:15 — Language & runtime *(10 min)*
Expect 3–4 from `L`/`A` with follow-ups, e.g.:
- "Value vs reference types" → *"and do structs live on the stack?"* → **`L2`** (no — a struct field of a class is on the heap). This follow-up exists to separate memorizers from understanders.
- "What's boxing" → "where does it happen in real code" → `L3`
- "`.Result` on a task — what happens?" → *"why doesn't it deadlock in ASP.NET Core?"* → **`A18`** (no SynchronizationContext → starvation instead) → *"how would you notice?"* → `A19` (latency up, CPU flat)
- "What's `IDisposable` for — and would you write a finalizer?" → `L69`, `L70`

### 0:25 — SQL, live *(12 min — you will type)*
Shared editor. Typically:
1. "Customers with zero orders." → `S3` — type it, then **explain why `COUNT(o.id)` and not `COUNT(*)`** before they ask
2. "Now the top-selling product per category." → `S17` — the CTE + `ROW_NUMBER`. Say out loud *why* it needs two layers: you can't filter a window function in `WHERE`
3. "This is slow on 10M rows. What do you do?" → `S42` — EXPLAIN ANALYZE, estimated vs actual, index the predicate/sort, check SARGability
4. "What index would you add, and what does it cost?" → `S25`, `S26` — leftmost prefix, write amplification
**Tests:** whether you can produce SQL under observation. **Narrate while typing** — silence reads as struggle even when you're fine.
**The differentiator:** volunteer `S38` (keyset vs OFFSET) when pagination comes up. It's the single highest-yield unprompted thing in this round.

### 0:37 — Code review *(10 min)*
They paste ~30 lines — realistically B4, B5, B7, or B15 from [13-spot-the-bug.md](13-spot-the-bug.md), often with two bugs stacked.
**Tests:** priority and mechanism.
**The script:** ask what it's meant to do → security first → correctness → performance → **"the one I'd block on is X."** Explain the *mechanism*, not the vibe. Say what test you'd add.

### 0:47 — Design lite *(8 min)*
"Sketch how you'd add [waitlist / recurring bookings / notifications] to your system."
**Tests:** can you extend a design you own. Follow the script from [../03-phase3-interviews/05-system-design-lite.md](../03-phase3-interviews/05-system-design-lite.md): data model → API → **name the hard part before they ask** (capacity + ordered promotion when a spot frees; at-least-once notification → idempotent handler) → one scale sentence.
**Red flag:** microservices, Kafka, event sourcing for a booking app *(`D37`)*.

### 0:55 — Your questions *(5 min)*
Go deeper than the screener: "What's the most painful part of this codebase?" · "What's the last incident and what came out of it?" · "How do you decide when to pay down debt?" Senior interviewers **enjoy** these — you're talking shop, and the round ends on a peer conversation instead of an exam.

### Deep-dive pass bar
The project chain held to question 4 · you typed both SQL queries · you ranked the review findings · you named at least two limitations of your own work unprompted · **zero bluffs**. Note: "I don't know" three times in an hour is completely fine. One bluff can end it.

---

# Script 3 — The 45-Minute Practical / Pairing Round

**Who:** one or two engineers, shared editor, a real (or realistic) codebase.
**Bar:** would we hand you a ticket on Monday?
**Shape:** they watch you work. **The code is maybe 40% of the grade; narration and process are the rest.**

### 0:00 — Orientation *(5 min)*
They hand you a repo and a task: *"Add `PATCH /bookings/{id}/reschedule`. It should validate the new slot is free and that the caller owns the booking."*
**What a strong candidate does first — and this IS the graded part:**
1. **Restate the task** in your own words + one clarifying question ("Does rescheduling into a taken slot return 409, or should I 422 it? Should it be allowed after the 24h cutoff?")
2. **Read before writing** — say it: *"Before I type anything, I want to see how the existing create endpoint handles validation and auth so I'm consistent."*
3. **Say the plan aloud** — *"DTO → controller action → service method → the overlap check → the ownership check → a test. I'll start with the failing test."*
**Red flag:** typing within 60 seconds. Every time.

### 0:05 — Build it *(25 min)*
**The narration contract (PACE — the framework's in [../03-phase3-interviews/04-live-coding-drills.md](../03-phase3-interviews/04-live-coding-drills.md)):**
- *"I'll scope the query by the caller rather than fetching then checking — that way a missing ownership check returns 404 instead of leaking."* ← **This one sentence is worth more than the rest of the endpoint.** It shows a security instinct in the flow of ordinary work.
- *"I'm returning 409 here — you could argue 422, happy to go either way; this is a state conflict so 409 reads right to me."*
- *"The check-then-act here has a race. The constraint is what actually enforces it — I'm catching the violation and mapping to 409."*
- *"I don't remember the exact `ExecuteUpdate` overload — I'd check IntelliSense; the shape I want is X."* ← honest recovery, **not** a spiral

**Expected interruptions (they're steering, not attacking — follow them):**
- *"Why did you put that there?"* → answer in one sentence, then continue
- *"What if the slot is taken between your check and your save?"* → **the whole reason this task was chosen.** `B4`.
- *"Do you need a transaction here?"* → `E27` — SaveChanges is already atomic for one call; I'd open one explicitly if the outbox row goes in too
- *"We're running low on time — what would you skip?"* → `D39` — happy path + the invariant + one test; polish and the extra edge cases go in the PR description as known gaps

### 0:30 — Tests *(8 min)*
Write at least one, even if time is short. Say the plan out loud regardless: *"The one I'd write first is the 409 on an overlapping reschedule, because that's the invariant. Then 403/404 for a non-owner, then the happy path."*
**Never** finish with "and I'd add tests later" and nothing else — say **which** tests, in priority order. That converts a gap into a demonstrated judgment call.

### 0:38 — Wrap-up *(7 min)*
*"What would you do with another hour? What are you unhappy with?"*
**The best answer in the whole round:** a specific, honest self-critique. *"The service method is doing three things — I'd extract the overlap check into the domain so it's testable without EF. And I hardcoded the 24h rule; that belongs in config. What I'd want reviewed is the concurrency handling — that's where I'd want a second pair of eyes."*
Self-awareness at the end retroactively upgrades everything before it. Defensiveness here undoes a good hour.

### Practical pass bar
Didn't type for 3 minutes · asked ≥1 real clarifying question · narrated continuously (no >30s silences) · caught the race unprompted · wrote or specified tests · a genuine self-critique · **zero bluffing**. **Unfinished but well-narrated beats finished in silence, every time** — they're hiring a colleague, and they just spent 45 minutes finding out what working with you is like.

---

## Grading rubric (all three scripts)

Score 1–5. **Ready-for-real bar: two consecutive mocks with everything ≥4 and zero bluffs.**

| Dimension | 1–2 | 3 | 4–5 |
|---|---|---|---|
| **Communication** | Rambling or silent; buries the answer | Answers, but no structure | Four beats, stops on time, invites redirection |
| **Depth** | Collapses at follow-up 2 | Holds to follow-up 2 | Still standing at 4; names their own limits |
| **Correctness** | Wrong mechanisms | Right, missing tradeoffs | Right, with the cost named unprompted |
| **Process** | Types first, guesses | Reasonable, needs prompting | Restate → read → plan → build → test, aloud |
| **Honesty** | Bluffed at least once | Hedged vaguely | "I don't know — here's how I'd find out" |
| **Ownership** | Story is thin under probing | Holds up | Volunteers limitations before being asked |

Log every score in [../LEARNING-LOG.md](../LEARNING-LOG.md) with the one behavior to change. **Track the trend, not the score** — mock 1 is supposed to be bad. That's the *point* of running it in week 7 instead of the night before.

## The four things that actually get people rejected

Not knowledge gaps. These:

1. **Bluffing.** One fabricated answer poisons every true one — now they have to verify everything you said. "I don't know" is free; bluffing is fatal.
2. **Not stopping.** A 4-minute answer to a 60-second question reads as unable to prioritize — the same signal as a 900-line PR.
3. **Silence under observation.** They cannot grade what they cannot hear. Narrate the *question* you're stuck on: *"I'm deciding between X and Y."*
4. **No tradeoffs.** Every "I used X" without a "because Y, and the cost was Z" is a junior answer to a mid-level question.

**All four are behaviors, not knowledge.** That's the good news: they're trainable in the ten weeks you have, and they're what the reps are for.
