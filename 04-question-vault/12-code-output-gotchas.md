# Code Output Gotchas — 32 Snippets (`G`)

"What does this print?" is the highest-signal-per-second question in a screener, which is why it opens so many of them. There's no bluffing: you either know the mechanism or you don't.

**Answers are inline — cover them.** Better: run the Mode 3 prompt from [00-overview.md](00-overview.md) so you have to commit before seeing the answer. Reading these top to bottom feels productive and teaches you almost nothing; that's the retrieval-practice point from [../drills/rebuild-drills.md](../drills/rebuild-drills.md).

**The real drill:** for every one you miss, *run it*. Watching it print the wrong thing on your own machine is what makes it stick.

---

## Closures & capture

**G1.**
```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++) actions.Add(() => Console.Write(i));
foreach (var a in actions) a();
```
**Answer:** `333`
**Why:** the `for` loop has **one** variable `i`; all three lambdas capture the same variable, and by the time they run it's 3. Change `for` to `foreach (var i in Enumerable.Range(0,3))` and it prints `012` — since C# 5, `foreach` declares a *fresh* variable per iteration. Fix for `for`: copy into a local (`int copy = i;`) inside the loop.
**Where this bites:** event handler registration in a loop, and `Task.Run` in a loop (G2).

**G2.**
```csharp
var tasks = new List<Task>();
for (int i = 0; i < 3; i++) tasks.Add(Task.Run(() => Console.Write(i)));
await Task.WhenAll(tasks);
```
**Answer:** nondeterministic — commonly `333`, sometimes `033`, `123`, etc.
**Why:** same capture bug as G1, now racing the loop itself. The tasks may start before `i` reaches 3, so you get a *different wrong answer each run* — the worst kind of bug, because it "works on my machine" half the time.

**G3.**
```csharp
int threshold = 1;
var nums = new List<int> { 1, 2, 3 };
var query = nums.Where(n => n > threshold);
threshold = 2;
Console.WriteLine(string.Join(",", query));
```
**Answer:** `3`
**Why:** deferred execution — the lambda reads `threshold` **when enumerated**, not when defined, and by then it's 2. Add `.ToList()` at definition and you get `2,3`.
**Where this bites:** building a filter, changing state, then enumerating — the query silently uses the new state.

---

## Value vs reference

**G4.**
```csharp
struct P { public int X; }
var arr = new P[] { new P { X = 1 } };
arr[0].X = 5;
Console.WriteLine(arr[0].X);

var list = new List<P> { new P { X = 1 } };
// list[0].X = 5;   ← what happens here?
```
**Answer:** `5` for the array. The `List<T>` line is a **compile error**: "Cannot modify the return value of `List<P>.this[int]` because it is not a variable."
**Why:** an array indexer returns a *reference* to the element; a `List<T>` indexer is a property that returns a **copy**, so mutating it would silently do nothing — the compiler stops you. The array and the list behaving differently is the trap, and it's the best argument for `readonly struct`.

**G5.**
```csharp
struct Counter { public int Value; public void Inc() => Value++; }
object boxed = new Counter();
((Counter)boxed).Inc();
Console.WriteLine(((Counter)boxed).Value);
```
**Answer:** `0`
**Why:** unboxing **copies** the struct out of the box. You incremented a temporary that was thrown away. Mutable structs + boxing = changes that vanish with no error.

**G6.**
```csharp
object[] arr = new string[2];
arr[0] = 42;
```
**Answer:** compiles fine, throws `ArrayTypeMismatchException` **at runtime**.
**Why:** array covariance — a .NET 1.0 decision that predates generics. `string[]` is assignable to `object[]`, but the runtime checks every store. It's a type hole, it costs a check on every array write, and it's why generics are invariant by default.

---

## Equality & strings

**G7.**
```csharp
object a = "hello";
object b = new string(new[] { 'h','e','l','l','o' });
Console.WriteLine(a == b);
Console.WriteLine(a.Equals(b));
Console.WriteLine((string)a == (string)b);
```
**Answer:** `False`, `True`, `True`
**Why:** `==` is a **static operator resolved at compile time**. With both sides typed `object`, the compiler picks `object`'s reference equality. `Equals` is virtual → dispatches to `String.Equals` → value comparison. Cast to `string` and the compiler picks string's `==` overload.
**Where this bites:** generic code (G8), and anything comparing values that arrived as `object`.

**G8.**
```csharp
static bool Eq<T>(T a, T b) where T : class => a == b;
Console.WriteLine(Eq("hello", new string(new[] { 'h','e','l','l','o' })));
```
**Answer:** `False`
**Why:** the `==` inside a generic method is compiled **once**, against the constraint (`class`), so it's reference equality — even when T turns out to be `string` at runtime. Fix: `EqualityComparer<T>.Default.Equals(a, b)`, which dispatches properly. This one catches very good developers.

**G9.**
```csharp
string a = "hello";
string b = "hel" + "lo";
string c = "hel";
string d = c + "lo";
Console.WriteLine(ReferenceEquals(a, b));
Console.WriteLine(ReferenceEquals(a, d));
```
**Answer:** `True`, `False`
**Why:** `"hel" + "lo"` is folded by the **compiler** into the literal `"hello"` and interned. `c + "lo"` is built at runtime → a new instance, not interned. Never reason about strings by reference — this is why.

**G10.**
```csharp
var p = new Person { Name = "A" };   // GetHashCode() derives from Name
var set = new HashSet<Person> { p };
p.Name = "B";
Console.WriteLine(set.Contains(p));
Console.WriteLine(set.Count);
```
**Answer:** `False`, `1`
**Why:** the object was bucketed by its **old** hash. Changing the field changes the hash, so the lookup probes the wrong bucket — the item is in the set, unfindable, forever. This is the "hash codes must be stable while in a hash collection" contract, and it's why keys should be immutable.

---

## Numbers

**G11.**
```csharp
Console.WriteLine(0.1 + 0.2);
Console.WriteLine(0.1 + 0.2 == 0.3);
Console.WriteLine(0.1m + 0.2m == 0.3m);
```
**Answer:** `0.30000000000000004`, `False`, `True`
**Why:** binary floating point can't represent 0.1 exactly, so the sum is a hair off. Bonus trivia: **.NET Core 3.0 changed `double.ToString()` to be shortest-round-trippable** — .NET Framework printed a friendly `0.3` and hid the problem. Decimal is base-10 and exact here.
**Where this bites:** money. Always.

**G12.**
```csharp
Console.WriteLine(5 / 2);
Console.WriteLine(5 / 2.0);
Console.WriteLine(-5 / 2);
Console.WriteLine(-5 % 2);
```
**Answer:** `2`, `2.5`, `-2`, `-1`
**Why:** integer division truncates **toward zero** (not floor — so `-5/2` is `-2`, not `-3`), and C#'s `%` takes the sign of the **dividend**. Python floors and gives `-3` and `1` — a real cross-language bug when porting.

**G13.**
```csharp
char a = 'a', b = 'b';
Console.WriteLine(a + b);
Console.WriteLine("" + a + b);
```
**Answer:** `195`, `ab`
**Why:** there's no `+` operator for `char`, so both promote to `int` (97 + 98). Prefix a string and it's string concatenation instead.

**G14.**
```csharp
int x = int.MaxValue;
Console.WriteLine(x + 1);
Console.WriteLine(checked(x + 1));
```
**Answer:** `-2147483648`, then an `OverflowException`.
**Why:** C# is **unchecked by default** — it silently wraps. `checked` (or the project-wide `<CheckForOverflowUnderflow>`) throws. `decimal` always throws on overflow regardless.

**G15.**
```csharp
int? a = null;
Console.WriteLine(a + 1);
Console.WriteLine(a > 5);
Console.WriteLine(a <= 5);
Console.WriteLine(!(a > 5));
```
**Answer:** *(empty line)*, `False`, `False`, `True`
**Why:** lifted operators — arithmetic with null yields null (which prints as an empty string), and **both** `>` and `<=` are false because the comparison is *unknown*. So `if (a > 5) ... else ...` sends null down the else branch, which is usually not what the author meant. Same three-valued logic as SQL's NULL.

---

## Control flow

**G16.**
```csharp
static int F() { int x = 1; try { return x; } finally { x = 2; } }
static List<int> G() { var l = new List<int>(); try { return l; } finally { l.Add(1); } }
Console.WriteLine(F());
Console.WriteLine(G().Count);
```
**Answer:** `1`, `1`
**Why:** the return **value** is evaluated and stashed before `finally` runs, so reassigning the local `x` is too late. But `G` returns a *reference* — the object it points to is mutated in `finally`, and the caller sees it. Value vs reference, one more time.

**G17.**
```csharp
class Base { public Base() { Print(); } protected virtual void Print() => Console.WriteLine("Base"); }
class Derived : Base
{
    private string _fromInitializer = "init";
    private string _fromCtor;
    public Derived() { _fromCtor = "ctor"; }
    protected override void Print() => Console.WriteLine($"{_fromInitializer ?? "null"} / {_fromCtor ?? "null"}");
}
new Derived();
```
**Answer:** `init / null`
**Why:** two things at once. (1) The virtual call in the base constructor dispatches to `Derived.Print` **before `Derived`'s constructor body runs** — so `_fromCtor` is still null. (2) But `_fromInitializer` **is** set, because in C# the derived class's *field initializers* run **before** the base constructor — the opposite of Java, where the superclass constructor runs first and both would be null. Never call a virtual method from a constructor; the object isn't finished being born.

**G18.**
```csharp
var list = new List<int> { 1, 2, 3 };
foreach (var x in list) if (x == 2) list.Remove(x);
```
**Answer:** `InvalidOperationException: Collection was modified; enumeration operation may not execute.`
**Why:** `List<T>`'s enumerator tracks a version stamp and refuses to continue after a structural change. Fix: iterate a copy (`foreach (var x in list.ToList())`), loop backwards by index, or `list.RemoveAll(x => x == 2)`.

**G19.**
```csharp
static IEnumerable<int> Get(List<int> src)
{
    if (src is null) throw new ArgumentNullException(nameof(src));
    foreach (var x in src) yield return x;
}
var e = Get(null);
Console.WriteLine("still here");
foreach (var x in e) { }
```
**Answer:** prints `still here`, **then** throws `ArgumentNullException`.
**Why:** the body of an iterator doesn't run until `MoveNext` — so your guard clause fires at enumeration time, far from the call site, with a useless stack trace. Fix: a public non-iterator method that validates and returns a private iterator.

---

## Async

**G20.**
```csharp
try { Fire(); Console.WriteLine("after"); }
catch { Console.WriteLine("caught"); }
await Task.Delay(100);

static async void Fire() { await Task.Yield(); throw new InvalidOperationException(); }
```
**Answer:** prints `after`, then the process **crashes** with an unhandled exception.
**Why:** `async void` has no Task to carry the exception, so it's rethrown on the synchronization context — nothing can catch it. Change to `async Task` + `await Fire()` and the catch works. This is why `async void` is banned outside event handlers.

**G21.**
```csharp
var t1 = Task.Run(() => throw new Exception("A"));
var t2 = Task.Run(() => throw new Exception("B"));
try { await Task.WhenAll(t1, t2); }
catch (Exception ex) { Console.WriteLine(ex.Message); }
```
**Answer:** `A` (only)
**Why:** `WhenAll`'s task holds an `AggregateException` with both, but `await` unwraps and rethrows only the **first**. Exception B is silently lost. To see them all: catch, then read `Task.Exception.InnerExceptions` on the WhenAll task.

**G22.**
```csharp
int count = 0;
Parallel.For(0, 100_000, _ => count++);
Console.WriteLine(count);
```
**Answer:** some number **less than** 100,000, different every run.
**Why:** `count++` is read-modify-write, not atomic — threads read the same value and both write the same +1, losing increments. `Interlocked.Increment(ref count)` fixes it. Note that `volatile int` would **not**: volatile controls visibility/ordering, not atomicity.

**G23.**
```csharp
var sw = Stopwatch.StartNew();
await Task.WhenAll(Task.Delay(1000), Task.Delay(1000), Task.Delay(1000));
Console.WriteLine(sw.ElapsedMilliseconds);

sw.Restart();
await Task.Delay(1000); await Task.Delay(1000); await Task.Delay(1000);
Console.WriteLine(sw.ElapsedMilliseconds);
```
**Answer:** ~`1000`, then ~`3000`.
**Why:** the concurrency question in four lines. `WhenAll` starts all three then waits once; sequential awaits serialize them. The application: three independent HTTP calls in a controller should be `WhenAll` — but **not** three EF queries on the same `DbContext`, which throws (`A45`). Knowing both halves is the mid-level answer.

---

## LINQ

**G24.**
```csharp
var q = Enumerable.Range(1, 3).Select(x => { Console.Write("*"); return x; });
var count = q.Count();
var list = q.ToList();
Console.WriteLine();
Console.WriteLine(count + list.Count);
```
**Answer:** `******` then `6`
**Why:** six stars — the sequence is enumerated **twice**, so the projection runs twice. Harmless with `Select`; catastrophic when the source is a database query (two round-trips) or has side effects. This is why you materialize once with `ToList()` when you'll use a result more than once.

**G25.**
```csharp
var empty = new List<int>();
Console.WriteLine(empty.Sum());
Console.WriteLine(empty.FirstOrDefault());
Console.WriteLine(empty.Average());
```
**Answer:** `0`, `0`, then `InvalidOperationException: Sequence contains no elements`.
**Why:** `Sum` of nothing is defensibly 0; `Average` of nothing is genuinely undefined so it throws (as does `Max`/`Min`/`First`). And `FirstOrDefault` on `int` gives **0, not null** — so `if (result == null)` doesn't even compile, and `if (result == 0)` can't tell "empty" from "found a 0". Use `int?` or `Any()` first.

**G26.**
```csharp
var people = new[] { new { Name = "A", Age = 30 }, new { Name = "B", Age = 30 } };
var byAge = people.GroupBy(p => p.Age).Select(g => new { g.Key, First = g.First().Name });
// vs in EF Core:
// db.People.GroupBy(p => p.Age).Select(g => new { g.Key, First = g.First().Name })
```
**Answer:** in memory: works, gives `{30, "A"}`. In EF Core: **throws** a translation failure (or silently degrades depending on version/shape).
**Why:** SQL `GROUP BY` can only return aggregates and the grouping keys — there's no "give me a whole row from the group" in a plain GROUP BY. EF translates aggregate-shaped projections and rejects the rest. The fix in SQL is a window function (`ROW_NUMBER`, `S17`). **The same LINQ means different things over different providers** — the single most important idea about IQueryable.

---

## Dates & time

**G27.**
```csharp
// Machine in Europe/Berlin. DST starts Sunday 29 March 2026: 02:00 → 03:00.
var t = new DateTime(2026, 3, 28, 23, 0, 0, DateTimeKind.Local);
Console.WriteLine(t.AddHours(4));
```
**Answer:** `29/03/2026 03:00:00`
**Why:** looks right, is wrong. `DateTime.AddHours` is **calendar arithmetic** — it adds 4 to the clock face and knows nothing about DST. Only **3 real hours** elapse between 23:00 and 03:00 that night, because 02:00–03:00 doesn't exist. Add 4 hours to a `DateTimeOffset`/UTC instant and you get the timeline answer. This is the exact class of bug your slot-computation tests exist for — and "add an hour to a local DateTime" is how booking systems double-book on one Sunday per year.

**G28.**
```csharp
var utc = DateTime.UtcNow;
var roundTripped = DateTime.Parse(utc.ToString());
Console.WriteLine(roundTripped.Kind);
Console.WriteLine(utc == roundTripped);
```
**Answer:** `Unspecified`, `False` (and the equality is wrong for two reasons at once).
**Why:** `ToString()` drops the Kind **and** the sub-second precision. The parsed value is `Unspecified` — a `DateTime` that no longer knows what it means, which the next `ToLocalTime()` will happily mangle. Round-trip with `"O"` (`ToString("O")`) or use `DateTimeOffset`. Losing `Kind` across a serialization boundary is one of the most common time bugs in .NET.

---

## Enums, generics, statics

**G29.**
```csharp
enum Status { Active = 1, Inactive = 2 }
Status s = default;
Console.WriteLine(s);
Console.WriteLine((Status)99);
Console.WriteLine(Enum.IsDefined(typeof(Status), (Status)99));
```
**Answer:** `0`, `99`, `False`
**Why:** an enum is just an int with names. `default` is 0 even though **no member has value 0** — so an uninitialized enum field is an invalid state that your switch won't catch. Casting any int works, and `ToString()` prints the number when there's no name. Lessons: always define a `None = 0` member, and never trust an enum arriving from JSON or a database without validating it.

**G30.**
```csharp
Console.WriteLine(Child.Value);

class Parent { public static string Value = Init(); static string Init() { Console.WriteLine("Parent init"); return "P"; } }
class Child : Parent { static Child() { Console.WriteLine("Child ctor"); } }
```
**Answer:** `Parent init`, then `P` — `Child ctor` never runs.
**Why:** `Child.Value` is really `Parent.Value` (statics aren't inherited, they're *found*), so accessing it triggers **Parent's** initialization, not Child's. Static initialization is lazy and per-type. The related horror worth knowing: if a static initializer **throws**, you get `TypeInitializationException` and the type is **permanently unusable for the process lifetime** — every later access throws the same cached error.

**G31.**
```csharp
class Base { public virtual string Who() => "Base"; }
class Over : Base { public override string Who() => "Over"; }
class Hide : Base { public new string Who() => "Hide"; }

Base a = new Over(), b = new Hide();
Console.WriteLine($"{a.Who()} {b.Who()} {((Hide)b).Who()}");
```
**Answer:** `Over Base Hide`
**Why:** `override` dispatches on the **runtime** type; `new` hides, so dispatch follows the **compile-time** type. The same object answers differently depending on the variable you hold it in — which is exactly why `new` is almost always an accident. The compiler warns; people add `new` to silence it, which is the wrong fix.

**G32.**
```csharp
var dict = new Dictionary<string, int> { ["b"] = 2, ["a"] = 1, ["c"] = 3 };
dict.Remove("a");
dict["d"] = 4;
foreach (var kv in dict) Console.Write(kv.Key);
```
**Answer:** `bdc` (typically) — and **the order is not part of the contract**.
**Why:** `Dictionary<K,V>` enumerates in internal bucket/entry order, which *looks* like insertion order until a removal frees a slot that a later insert reuses. It has never been guaranteed and has changed between runtimes. If you need order, use `SortedDictionary`, an ordered list, or `OrderBy` at the point of use — code that depends on dictionary order works right up until production data changes.

---

## Scoring & what this trains

| Score /32 | Read |
|---|---|
| 28+ | Genuinely strong. These are the questions people fail; you won't. |
| 22–27 | Solid mid-level. Drill the misses, run each one. |
| 15–21 | Expected before the drills. Two passes a week closes this fast. |
| <15 | Start here before the depth files — this is where the cheap points are. |

**Why this file works:** every snippet is a *mechanism*, not a fact. You can't memorize your way to 28/32 — you get there by understanding value vs reference, deferred execution, compile-time vs runtime dispatch, and three-valued logic. Those four ideas generate about 25 of these 32, and they also generate the answers to half of [01-csharp-language.md](01-csharp-language.md). If you can explain **why** to a rubber duck, the trivia comes free.
