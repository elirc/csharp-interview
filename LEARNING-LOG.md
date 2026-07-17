# Learning Log

One entry per day, minimum one line. Honest > polished. By week 8 this file is the raw material for your behavioral story bank and take-home retros.

**Entry format:**

```
## YYYY-MM-DD (Week N, Day)
- Built: <what you shipped today>
- Broke: <what went wrong / what confused you>
- Learned: <the thing you'd explain differently now>
- Flag: <anything to revisit — becomes Saturday rebuild or quiz material>
```

**Example entries (delete these once you have real ones):**

## 2026-07-14 (Week 1, Mon)
- Built: Ticket CRUD endpoints, all 5 verbs returning correct status codes
- Broke: PUT returned 200 with no body; learned it should be 204 NoContent or 200 with the updated resource — picked 204 and documented why
- Learned: `[ApiController]` gives automatic 400s on model validation failure — I was writing manual ModelState checks for nothing
- Flag: still fuzzy on when EF tracks entities vs not — quiz me on change tracking Friday

## 2026-07-15 (Week 1, Tue)
- Built: DTOs for Ticket create/update/response; mapping by hand (no AutoMapper yet, on purpose)
- Broke: forgot the response DTO on the list endpoint, leaked entity with navigation props → JSON cycle error. Good lesson.
- Learned: the cycle error IS the argument for DTOs — wrote that down for interviews
- Flag: —

---

<!-- Real entries start below -->
