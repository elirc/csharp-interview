# C# Language & Runtime — 72 Questions (`L`)

The trivia layer of the screener. These get asked fast, early, and are graded pass/fail on fluency — a hesitant right answer reads as a guess. Target: 3–10 seconds each.

⭐ = asked constantly. Log misses as `L##`.

---

## Types & memory

**L1. ⭐ Value types vs reference types — the real difference?**
Assignment copies the *value* vs copies the *reference*. Pass a struct to a method and mutate it → caller sees nothing; pass a class → caller sees the change. That's the whole thing; everything else (stack/heap, boxing) is a consequence.

**L2. ⭐ "Structs live on the stack" — true?**
No, and this is the trap. *Locals* of struct type usually live on the stack; a struct **field of a class lives on the heap inside that object**, and a struct in a `List<T>` lives in the heap array. Storage location is about *where the variable lives*, not what type it is. Say this and you've out-answered most candidates.

**L3. ⭐ What is boxing?**
Wrapping a value type in a heap object so it can be treated as `object`/an interface. Costs an allocation + a copy; unboxing copies back and can throw `InvalidCastException`. Triggers: `object o = 5`, struct → non-generic interface, old non-generic collections, `string.Format`/interpolation of value types in some paths. Generics exist largely to avoid it.

**L4. When would you actually write a struct?**
Small (≈≤16 bytes), immutable, value-semantic, allocated in bulk — a `Money`, a `Point`, a `SlotKey`. Default to `class`. If you can't state the size and the immutability, you want a class or a `record`.

**L5. Why are mutable structs a footgun?**
You mutate copies without noticing. `list[0].X = 5` won't even compile on `List<T>` (the indexer returns a copy); on an array it compiles and works, which is *worse* because the two behave differently. `readonly struct` makes the compiler enforce your intent.

**L6. `record` vs `class`?**
Record = class with compiler-generated value equality, `ToString`, deconstruction, and `with` expressions. DTOs and value objects → records. EF entities → classes: they have identity (Id), they're mutated by the change tracker, and value equality is actively wrong for them.

**L7. `record struct` vs `record class`?**
`record` = record class (heap, reference identity but value equality). `record struct` = value type with generated equality; **mutable by default** unless you write `readonly record struct` — which you almost always want.

**L8. `readonly struct` — what does it buy?**
Compiler enforces no field mutation, and it can stop making defensive copies when the struct is accessed through a `readonly` field or `in` parameter. Correctness + a perf win.

**L9. Stack vs heap — who cleans up?**
Stack frames pop automatically on return. Heap is the GC's job. You never free manually; you only release *unmanaged* handles, via `IDisposable`.

**L10. What is `default(T)`?**
Zero-bits: `0` for numerics, `false`, `null` for reference types, all-fields-default for structs. `default` alone (C# 7.1+) infers the type. Trap: `default(SomeEnum)` is 0 **even if no member has value 0** — a real source of "impossible" enum values.

**L11. Nullable value types (`int?`) — what are they really?**
`Nullable<T>`, a struct with `HasValue` + `Value`. Lifted operators: `null + 1` is `null`, and `null > 5` is **false** — so `if (x > 5)` and `if (!(x > 5))` can both be false. Boxing an `int?` boxes the `int` or produces plain `null`.

**L12. ⭐ Nullable *reference* types — what do they actually do?**
Compile-time flow analysis only. `string?` and `string` are the same type at runtime; NRT adds warnings, not runtime checks. A `null` can still arrive from JSON, EF materialization, or a non-NRT library. Mid answer: "enabled with warnings-as-errors, but I still guard at trust boundaries — the compiler doesn't validate my API's callers."

**L13. What does `!` (null-forgiving) do?**
Silences the compiler. Zero runtime effect. Every `!` is a claim you're making that the compiler can't verify — fine after a `TryGetValue`-style check, a smell when sprinkled to clear warnings.

---

## Object model

**L14. ⭐ Abstract class vs interface — when each?**
Interface = capability contract, multiple inheritance, no state. Abstract class = shared implementation + state + a common base identity, single inheritance. Rule of thumb: "can-do" → interface; "is-a with shared guts" → abstract class. Since C# 8, interfaces can have default implementations — that changes the tradeoff at the margin but not the intent.

**L15. `virtual`/`override` vs `new`?**
`override` = polymorphic dispatch on the *runtime* type. `new` = hiding: dispatch on the *compile-time* type, so `Base b = new Derived(); b.M();` calls **Base.M**. `new` is almost always a mistake or an accident. Classic whiteboard trick — see [12-code-output-gotchas.md](12-code-output-gotchas.md).

**L16. Why is calling a virtual method in a constructor dangerous?**
The derived override runs *before* the derived constructor body — so it sees uninitialized fields (`null`, `0`) and possibly `readonly` fields not yet assigned. The base ctor runs first, but virtual dispatch already points at the derived type.

**L17. `sealed` — why?**
Prevents inheritance. Signals intent ("this isn't an extension point"), enables devirtualization by the JIT, and closes the "someone overrode this and broke my invariant" hole. Free perf on hot classes.

**L18. Default interface methods — what problem do they solve?**
Adding a member to a published interface without breaking every implementer. It's a versioning tool, not a mixin system. Overuse gets you diamond-problem confusion; use it to evolve contracts, not to share logic.

**L19. `is` vs `as` vs cast?**
`is` tests (+ patterns, + declares a variable). `as` converts or yields `null` — reference/nullable types only. `(T)x` converts or throws. Prefer `is T t` for the check-then-use case: one operation, no double-check.

**L20. What are extension methods, and can you call one on `null`?**
Static methods that the compiler rewrites into instance-call syntax, resolved at compile time by the *static* type. And yes — `((string)null).IsNullOrEmpty()` style works, because there's no instance dispatch, so no `NullReferenceException`. Surprises people.

**L21. `object`'s virtual members?**
`ToString`, `Equals`, `GetHashCode`, `Finalize`. (`GetType` is non-virtual.) Knowing `Finalize` is there sets up the GC question.

---

## Members & modifiers

**L22. ⭐ `const` vs `readonly`?**
`const` = compile-time constant, implicitly static, and **baked into the calling assembly's IL**. `readonly` = set at runtime, in the declaration or the constructor, per instance (or `static readonly`). The interview point: change a public `const` in a library, don't recompile the consumer, and it keeps the *old* value. Cross-assembly public constants should be `static readonly`.

**L23. `static readonly` vs `const` for a `TimeSpan`?**
`const` only supports compile-time constants (numerics, strings, bools, enums, null) — a `TimeSpan` can't be `const` at all. `static readonly TimeSpan Timeout = TimeSpan.FromSeconds(30);` is the idiom.

**L24. When does a static constructor run?**
Lazily, before the first access to any static member or instance creation, exactly once, thread-safely (the runtime guarantees it). Trap: an exception inside it becomes `TypeInitializationException` and the type is **permanently unusable** for the process lifetime — a genuinely nasty production failure.

**L25. `static` classes — what's the cost in a web app?**
Untestable seams (can't substitute), hidden global state, and thread-safety bugs when they cache mutable data across requests. Fine for pure helpers. `DateTime.Now` in domain logic is the canonical example of the testability cost — inject `TimeProvider` instead.

**L26. Auto-properties — what's generated?**
A private backing field + get/set. `{ get; init; }` allows assignment only during object initialization. `{ get; }` (getter-only) can still be assigned in the constructor. C# 13+ lets you write `field` in an accessor to touch the backing field without declaring it.

**L27. `ref` vs `out` vs `in`?**
`ref` = pass by reference, must be initialized by caller. `out` = pass by reference, must be assigned by callee, caller need not initialize. `in` = pass by reference, read-only — a perf tool for big structs. All three are aliases to the caller's storage.

**L28. What's `params` for, and what does it cost?**
Variable arg count; the compiler allocates an array at each call site — including `Log("x", args: nothing)` which allocates an empty array (mitigated by cached empties in some APIs). C# 13 added `params` over any collection type, including `Span<T>`, which lets it be allocation-free.

**L29. Access modifiers — name them all.**
`private` (type), `protected` (type + derived), `internal` (assembly), `protected internal` (assembly **OR** derived), `private protected` (assembly **AND** derived), `public`. The last two are the ones people fumble; the OR/AND distinction is the answer.

---

## Equality & hashing

**L30. ⭐ `==` vs `.Equals()`?**
`==` is a *static, compile-time-resolved operator*; `Equals` is virtual and dispatches at runtime. So `object a = "hi"; object b = "h"+"i"; a == b` uses `object`'s reference `==` while `a.Equals(b)` uses string's value equality. For `string` typed as `string`, `==` is overloaded to compare values.

**L31. ⭐ The `Equals`/`GetHashCode` contract?**
Equal objects **must** return equal hash codes (the reverse isn't required — collisions are legal). Override both or neither. Hash codes must be stable while the object is in a hash-based collection — mutating a key field after inserting into a `Dictionary`/`HashSet` makes the entry unfindable, effectively a leak.

**L32. What does the default `Equals` do for a struct?**
`ValueType.Equals` compares fields — and unless the struct is blittable it does it **via reflection**, which is startlingly slow. That's why you implement `IEquatable<T>` on structs used in hot paths or as dictionary keys, or use a `record struct`, which generates it for you.

**L33. `ReferenceEquals` vs `==`?**
`ReferenceEquals` is identity, always, unoverloadable — the escape hatch when a type has overloaded `==` and you truly want "same object."

**L34. Is `GetHashCode` stable across runs?**
No. String hashing is randomized per process by default in .NET Core. Never persist a hash code, never send it over the wire, never use it as a DB key or cache key across processes. Real bug, seen in the wild.

---

## Generics & variance

**L35. Why generics instead of `object`?**
Compile-time type safety and no boxing — `List<int>` stores ints inline, `ArrayList` boxes every one. Also: the JIT specializes value-type generics into distinct native code; reference types share one instantiation.

**L36. Generic constraints — which exist?**
`where T : class`, `struct`, `new()`, `BaseType`, `IInterface`, `notnull`, `unmanaged`, `T : U`. `new()` means "has a public parameterless ctor" and must come last.

**L37. ⭐ Covariance and contravariance — plain English?**
`out T` (covariant): `IEnumerable<Dog>` is usable as `IEnumerable<Animal>` — safe because you can only *take* Ts out. `in T` (contravariant): `IComparer<Animal>` is usable as `IComparer<Dog>` — safe because you only *put* Ts in. `List<Dog>` is neither (it does both), which is why `List<Dog>` is not an `IList<Animal>`.

**L38. Why are arrays covariant, and why is that bad?**
`Animal[] a = new Dog[1];` compiles — a .NET 1.0 decision predating generics. Then `a[0] = new Cat();` throws `ArrayTypeMismatchException` **at runtime**. A type hole that generics deliberately closed.

**L39. `typeof(List<>)` vs `typeof(List<int>)`?**
Open vs closed generic type. Open ones show up in DI registration: `services.AddScoped(typeof(IRepository<>), typeof(Repository<>))` registers the whole family.

---

## Collections

**L40. ⭐ `List<T>` — how does it grow, and what does that cost?**
Backed by an array; on overflow it allocates a new array of double the capacity and copies. Amortized O(1) append, but the copies churn the heap and big ones land on the LOH. If you know the count, `new List<T>(capacity)`.

**L41. `Dictionary<K,V>` lookup — O(1) always?**
Average O(1); worst case O(n) when every key collides into one bucket. Requires a good `GetHashCode`. Not thread-safe: concurrent writes can corrupt the buckets and **spin forever on read** — a classic 100%-CPU production hang.

**L42. `IEnumerable<T>` vs `ICollection<T>` vs `IList<T>` vs `IReadOnlyList<T>`?**
Iterate → count/add/remove → indexer → indexer without mutation. Accept the weakest interface that works (usually `IEnumerable<T>`); return the strongest thing that's honest (often `IReadOnlyList<T>`) so callers don't have to guess whether enumeration is cheap.

**L43. Why is returning `IEnumerable<T>` from a repository sometimes a bug?**
It's lazy: the caller may enumerate after the `DbContext`/connection is gone, enumerate twice (two DB round-trips), or never — while holding a lock or a reader. Materialize at the boundary (`ToListAsync`) and return `IReadOnlyList<T>`.

**L44. `Array` vs `List<T>` vs `Span<T>`?**
Fixed-size contiguous storage · growable wrapper over an array · a *view* over contiguous memory with no allocation. `Span<T>` is a `ref struct`: stack-only, so it can't be a field of a class, can't be captured in a lambda, and **can't cross an `await`**. Use `Memory<T>` in async code.

**L45. What's a `ref struct` and why does it exist?**
A struct guaranteed to live on the stack (`Span<T>`, `ReadOnlySpan<T>`). The restrictions exist because a heap reference to stack memory would be a dangling pointer. C# 13 loosened the rules (allows them in generics via `allows ref struct`).

**L46. Frozen/immutable collections?**
`ImmutableList/Dictionary` — every mutation returns a new instance, structurally shared; good for shared state, slower writes. `FrozenDictionary`/`FrozenSet` (.NET 8) — expensive to build, fastest possible reads, for lookup tables built once at startup and read forever. Naming these shows currency.

---

## LINQ

**L47. ⭐ `First` vs `FirstOrDefault` vs `Single` vs `SingleOrDefault`?**
First: first match, throws if none. FirstOrDefault: `default` if none. Single: exactly one match, **throws if more than one** — it must scan for a second element. Use `Single` when "more than one" means the data is corrupt and you *want* the exception; `First` needs an `OrderBy` to be deterministic.

**L48. `Any()` vs `Count() > 0`?**
`Any()` stops at the first element; `Count()` may enumerate everything (or issue `SELECT COUNT(*)` in EF, which is fine). On an in-memory `IEnumerable` from an expensive generator, `Count() > 0` is a real waste. On `ICollection` both are O(1). `Any()` also reads better.

**L49. ⭐ Deferred execution — where does it bite?**
The query runs at enumeration, not definition. Bites: enumerate twice → query twice; the source mutates between definition and enumeration → different results; the `DbContext` is disposed before enumeration → `ObjectDisposedException`; a captured loop variable changes before the query runs.

**L50. Which LINQ operators execute immediately?**
`ToList`/`ToArray`/`ToDictionary`/`ToHashSet`, `Count`, `Sum`/`Min`/`Max`/`Average`, `Any`/`All`, `First*`/`Single*`/`Last*`, `Contains`, `ElementAt`. Everything else (`Where`, `Select`, `OrderBy`, `GroupBy`, `Join`, `Skip`, `Take`) is deferred.

**L51. Is `OrderBy` stable?**
Yes in LINQ-to-Objects — equal keys keep their original relative order. `List<T>.Sort` and `Array.Sort` are **not** stable (introsort). And in SQL, order among ties is undefined regardless. Fun trivia that occasionally matters.

**L52. `Select` vs `SelectMany`?**
Map one→one vs flatten one→many. `orders.SelectMany(o => o.Items)` gives you all items across all orders as a flat sequence. In query syntax it's two `from` clauses.

**L53. `GroupBy` in LINQ-to-Objects vs in EF?**
In memory: buffers everything, returns groups with full element lists, preserves first-appearance order. In EF: translates to SQL `GROUP BY` **only if the projection is aggregate-shaped**. Selecting the group's elements forces a client-side grouping or a translation failure — a common "why did this get slow" moment.

**L54. `yield return` — what does the compiler build?**
A state machine class implementing `IEnumerator<T>`; each `yield return` becomes a labeled resume point. The method body doesn't run at all until `MoveNext` — so argument validation in an iterator method fires *late*, which is why the guard-clause + private-iterator split exists.

**L55. `Aggregate` — when have you actually used it?**
Honest mid answer: rarely — `Sum`/`Count`/`Max` cover 95% and read better. It's for custom folds (running state, non-standard combination). Knowing *when not to use it* is the answer.

**L56. `Zip`, `Chunk`, `DistinctBy`, `MinBy` — know them?**
`Zip` pairs two sequences; `Chunk(n)` batches (.NET 6) — genuinely useful for batching DB writes; `DistinctBy`/`MaxBy`/`MinBy` (.NET 6) replace the old `GroupBy(...).Select(g => g.First())` dance. Naming these signals you write current C#.

---

## Strings, numbers, dates

**L57. ⭐ Why `StringBuilder` in a loop?**
Strings are immutable, so each `+=` allocates a new string and copies both operands — O(n²) work and O(n²) garbage over a loop. `StringBuilder` mutates one buffer. Note the nuance: `a + b + c` in a *single expression* is one `String.Concat` call and is fine — it's the loop that kills you.

**L58. String interning?**
Identical string *literals* share one instance from the intern pool, so `"hi" == "hi"` is reference-equal. Runtime-built strings are not interned unless you call `string.Intern`. Never rely on reference equality for strings.

**L59. ⭐ `StringComparison` — why does it matter?**
`"str".ToLower() == input.ToLower()` allocates twice and is *culture-dependent* — in Turkish, `I`.ToLower() isn't `i`, so your case-insensitive comparison silently changes behavior by locale (the famous "Turkish I" bug). Correct: `string.Equals(a, b, StringComparison.OrdinalIgnoreCase)`. Ordinal for identifiers/keys/protocol; culture-aware only for text shown to humans.

**L60. ⭐ `decimal` vs `double` for money?**
`double` is binary floating point — 0.1 has no exact representation, so `0.1 + 0.2 != 0.3` and errors compound over sums. `decimal` is base-10, 28–29 significant digits, exact for the values money actually takes. Best answer: store money as **integer minor units** (cents) in the DB, use `decimal` in the domain, and never `double` anywhere near a currency.

**L61. `DateTime` vs `DateTimeOffset` vs `DateOnly`/`TimeOnly`?**
`DateTime` carries a `Kind` (Utc/Local/**Unspecified**) that's easy to lose across serialization and DB round-trips. `DateTimeOffset` carries an actual UTC offset — unambiguous instant. `DateOnly`/`TimeOnly` (.NET 6) for birthdays and opening hours, where a time zone is meaningless. Rule: store instants as UTC (`DateTimeOffset`/`timestamptz`), convert at the edge for display.

**L62. Why isn't an offset the same as a time zone?**
`+02:00` is an offset; `Europe/Berlin` is a *rule set* that maps instants to offsets and changes with DST and legislation. Store the zone ID for future recurring events ("every Monday 9am Berlin"); store the instant for past events. Getting this distinction out unprompted is a strong signal — it's exactly the DST bug your Bookable slot tests cover.

**L63. `DateTime.Now` vs `UtcNow` — and why does `Now` hurt in a web app?**
`Now` is the *server's* local time — meaningless in a container, wrong after a deploy to another region, and it makes your logic untestable. Use `UtcNow`, or better, inject `TimeProvider` (.NET 8) so tests can control the clock deterministically.

**L64. Integer overflow — checked or unchecked?**
Unchecked by default: `int.MaxValue + 1` silently wraps to `int.MinValue`. `checked { }` or `<CheckForOverflowUnderflow>` throws instead. `decimal` always throws on overflow. Relevant anywhere you sum quantities or money.

---

## Exceptions

**L65. ⭐ `throw` vs `throw ex`?**
`throw;` rethrows and **preserves the original stack trace**; `throw ex;` resets it to the current line, destroying the evidence of where the bug actually was. If you need to add context, wrap: `throw new DomainException("...", ex)`. This is asked constantly and it's a free point.

**L66. `when` filters — what do they buy over a catch + rethrow?**
`catch (SqlException ex) when (ex.Number == 2601)` — the filter is evaluated **before the stack unwinds**, so if it doesn't match, the stack stays intact for the debugger and the original handler. Catch-check-rethrow unwinds first, losing that.

**L67. Is `try/catch` expensive?**
Entering a `try` is essentially free; *throwing* is expensive (stack walk, first-chance machinery) — microseconds, but thousands per second on a hot path shows up in a profile. Cost is why exceptions are for exceptional flow, not for validating every request; that's what result types are for.

**L68. `finally` — does it always run?**
Effectively always for normal flow and exceptions — but not on `StackOverflowException`, `Environment.FailFast`, or a process kill. `using` is compiled to try/finally.

---

## Disposal & GC

**L69. ⭐ `IDisposable` — what's it for and who calls it?**
Deterministic release of *unmanaged* or expensive resources: file handles, sockets, DB connections. `using` calls it on scope exit including on exception. The GC handles memory; `Dispose` handles everything else. In ASP.NET Core, DI disposes scoped/transient `IDisposable` services it created when the request scope ends.

**L70. ⭐ Finalizers — why almost never write one?**
Non-deterministic (whenever the GC feels like it), promotes the object to a later generation (so it lives longer and costs more), runs on a single finalizer thread that can back up, and can resurrect objects. Only for directly-held unmanaged handles — and `SafeHandle` already does that better. If you have one, call `GC.SuppressFinalize(this)` in `Dispose`.

**L71. GC generations — the mental model?**
Gen0 (new, tiny, collected constantly and cheaply), Gen1 (survivors, a buffer), Gen2 (long-lived, expensive collections). Generational hypothesis: most objects die young. **LOH**: allocations ≥ 85,000 bytes go straight to the Large Object Heap, collected only with Gen2 and not compacted by default → fragmentation. Practical takeaway: big arrays/buffers are what you pool, not small objects.

**L72. Can you leak memory in .NET?**
Yes — "leak" here means unreachable-in-spirit but still rooted. The usual suspects: static collections that only grow, **event handler subscriptions never unsubscribed** (the publisher holds the subscriber alive), captured contexts in long-lived closures/timers, and `HttpClient` sockets held by never-disposed handlers. GC frees unreachable objects; it can't know you *meant* to drop a reference.

---

## Coverage tracker

`L1–L13` types/memory · `L14–L21` object model · `L22–L29` members · `L30–L34` equality · `L35–L39` generics · `L40–L46` collections · `L47–L56` LINQ · `L57–L64` strings/numbers/dates · `L65–L68` exceptions · `L69–L72` disposal/GC

**Fluency bar:** every ⭐ in under 10 seconds, spoken, including the tradeoff beat. If you're reconstructing the answer, it isn't fluent — that hesitation is exactly what a screener is listening for.
