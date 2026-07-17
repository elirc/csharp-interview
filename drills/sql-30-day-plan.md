# SQL 30-Day Drill Plan (30 min/day)

Raw SQL, typed by hand in DBeaver — not LINQ, not copy-paste. SQL is the most common junior→mid interview fail and the cheapest to fix. After day 30, switch to the maintenance set (bottom).

**Setup (day 1, 10 min):** create a practice database with this schema — it mirrors interview questions almost exactly:

```sql
CREATE TABLE customers (id serial PRIMARY KEY, name text, city text, signup_date date);
CREATE TABLE products  (id serial PRIMARY KEY, name text, category text, price_cents int);
CREATE TABLE orders    (id serial PRIMARY KEY, customer_id int REFERENCES customers,
                        ordered_at timestamptz, status text); -- placed|shipped|cancelled
CREATE TABLE order_items (order_id int REFERENCES orders, product_id int REFERENCES products,
                          quantity int, unit_price_cents int, PRIMARY KEY (order_id, product_id));
```

Ask Claude: *"Generate INSERT statements: 50 customers, 30 products, 500 orders, 1500 order items, realistic distributions, some customers with zero orders, some cancelled orders."* Save as `seed.sql`.

## Days 1–7: SELECT fundamentals
- D1: setup + basic SELECT/WHERE/ORDER BY/LIMIT (customers by city, recent orders, price ranges)
- D2: aggregate functions — COUNT/SUM/AVG/MIN/MAX; orders per status; revenue total (watch: cancelled orders in or out? *always ask* — that ambiguity is planted in interviews)
- D3: GROUP BY — revenue by category, orders per customer, signups per month (`date_trunc`)
- D4: HAVING vs WHERE — customers with >5 orders; categories averaging >$50. Be able to *say* the difference (filter rows before grouping vs filter groups after)
- D5: INNER JOIN — order lines with product names; customer names on orders
- D6: LEFT JOIN ⭐ — all customers with order counts **including zeros** (the #1 interview SQL question; the trick: `COUNT(o.id)` not `COUNT(*)`, and why)
- D7: review — redo D6 cold + explain INNER vs LEFT vs RIGHT vs FULL aloud

## Days 8–14: the interview classics
- D8: multi-join chains — customer → order → items → product in one query (revenue per customer by category)
- D9: subqueries — customers whose spend > average spend; products never ordered (`NOT EXISTS` vs `NOT IN` — and the NULL trap in `NOT IN` ⭐ classic gotcha)
- D10: CTEs — rewrite D9 with `WITH`; monthly revenue CTE then month-over-month listing
- D11: **top-N per group ⭐** — best-selling product per category (window: `ROW_NUMBER() OVER (PARTITION BY category ORDER BY qty DESC)`). THE mid-level interview question. Do it twice.
- D12: window functions continued — running revenue total; each order's gap since customer's previous order (`LAG`)
- D13: CASE + conditional aggregation — pivot-style: one row per month, columns for placed/shipped/cancelled counts
- D14: review — redo D11 cold; quiz prompt: *"rapid-fire me on JOINs, GROUP BY vs HAVING, NOT IN + NULL, ROW_NUMBER vs RANK"*

## Days 15–21: writes, transactions, constraints
- D15: INSERT/UPDATE/DELETE with RETURNING; UPDATE with a JOIN/`FROM` (raise prices in one category)
- D16: transactions — BEGIN/COMMIT/ROLLBACK; two DBeaver sessions, watch one block/not-see the other; concept #13 gets hands-on here
- D17: constraints — add CHECK (quantity > 0), UNIQUE, FK violations; force each error on purpose and read the messages
- D18: UPSERT — `INSERT ... ON CONFLICT DO UPDATE`; when you'd use it (idempotent seeding, counters)
- D19: NULL semantics ⭐ — `NULL = NULL` is not true; COALESCE; NULLs in aggregates, ORDER BY, and that `NOT IN` trap again
- D20: DDL fluency — add columns with defaults, create indexes, drop safely
- D21: review — build a small "bank transfer" script that's transaction-safe

## Days 22–30: performance (Phase 2 timing — pairs with feature 9)
- D22: `EXPLAIN` basics — read Seq Scan vs Index Scan on your orders table
- D23: create an index on `orders(customer_id)`; EXPLAIN before/after on a customer lookup; then EXPLAIN a query the index *doesn't* help — know why
- D24: composite indexes — `(customer_id, ordered_at)`: which queries it serves, which orderings it can't (leftmost-prefix rule)
- D25: EXPLAIN ANALYZE — actual vs estimated rows; find the slowest of your day-8–13 queries and speed it up
- D26: index costs — why not index everything (write amplification, storage); when a Seq Scan is *correct* (small tables, low selectivity)
- D27: pagination at depth — OFFSET 10000 pain, keyset pagination (`WHERE (ordered_at, id) < (?, ?) ORDER BY ... LIMIT`) — great interview differentiator
- D28: your real project — EXPLAIN Bookable's dashboard query, index it, document before/after (this IS feature 9)
- D29: schema design mini-drill — design tables for a given prompt (Claude supplies: e.g. course enrollment) in 15 min, then get critiqued
- D30: final review — the full gauntlet: Claude gives 6 questions covering D6, D11, D16, D19, D23, D27 under 30 minutes, graded

## Maintenance set (day 31 → interviews)
3×/week, 20 min: one top-N-per-group, one LEFT-JOIN-with-aggregate, one EXPLAIN read — plus: *"give me one mid-level SQL interview question, wait for my answer, grade it."* Rotate.

## The prompt that runs every session
> Give me exercise(s) for day N of my SQL plan: [paste the day's line]. Give me the problem only. When I submit SQL, tell me if it's right; if wrong, give a hint, not the answer. End with one harder variant.
