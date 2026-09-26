# Chapter 2 — Objects, Identity, and Memory

*Part I — The Machine*

[← Chapter 1: The Execution Model](01-execution-model.md) · [Home](README.md) · [Chapter 3: Names, Scope, and Binding →](03-names-scope-binding.md)

---

## 2.1 The Question Behind the Chapter

In Chapter 1 we said everything in Python is an object. That sentence is repeated so often it has worn smooth, and a worn-smooth sentence stops carrying information. So let us make it sharp again by asking what it *costs*.

The integer `1` is an object. Not a machine word holding the value one — an object, a structure on the heap, with a header, a type pointer, a reference count, and somewhere in there the actual digit. The string `"hello"` is an object. The function you defined this morning is an object. The class it belongs to is an object. `None` is an object — a single, shared one. When you write `x = 1`, you are not putting the number one into a box called `x`. You are making the name `x` *refer to* an object that already exists.

This chapter is about the consequences of that. Three of them, specifically. First: because objects have headers and overhead, a million integers in a Python list cost far more memory than a million integers in a C array, and a staff engineer can estimate by how much. Second: because names refer to shared objects, the question "are these two things the same object or merely equal?" is a real question with a real answer, and getting it wrong produces bugs that pass every test until the input changes. Third: because objects must be freed when no longer needed, Python has a memory-management system — two systems, actually — and understanding them is the difference between a service that runs for months and one that has to be restarted every Tuesday.

## 2.2 The Anatomy of an Object

Every object in CPython, with no exceptions, begins with the same header. In the C source it is a structure that, stripped to its essentials, looks like this:

```c
typedef struct _object {
    Py_ssize_t ob_refcnt;     // the reference count
    PyTypeObject *ob_type;    // a pointer to the type
} PyObject;
```

Two fields. `ob_refcnt` is an integer counting how many references currently point at this object — we will spend most of the chapter on it. `ob_type` is a pointer to the object's *type* — itself an object — which is how `type(x)` is answered, how method lookup begins, how `isinstance` works. On a 64-bit system these two fields are eight bytes each: **sixteen bytes of header on every object before it holds any data of its own.**

Objects that hold a variable number of elements — lists, tuples, strings, bytes — carry one more header field, `ob_size`, recording how many elements they hold. That is the `PyVarObject` header, twenty-four bytes.

This header is not a curiosity. It is a number you can do arithmetic with, and doing that arithmetic is a genuinely useful skill. Consider the humble integer. A Python `int` on a 64-bit CPython is twenty-eight bytes: sixteen of header, and twelve more for the machinery that lets it be arbitrarily large (Python integers never overflow — they grow). Twenty-eight bytes to store a number that C stores in eight, or often in four.

Now scale it. A C array of one million 32-bit integers is four megabytes — one contiguous block, four bytes per element. A Python `list` of one million integers is something else entirely. The list itself stores not the integers but *pointers* to them: a contiguous array of one million eight-byte pointers, eight megabytes. And then each of those pointers leads to a twenty-eight-byte integer object, another twenty-eight megabytes. Thirty-six megabytes against four — roughly nine times the memory — and the Python version is also scattered across the heap rather than contiguous, which means worse cache behavior on top of the raw size.

You should be able to do that estimate in an interview, on a whiteboard, in under a minute. Not because the exact bytes matter — they shift between versions and platforms — but because the *shape* of the answer is the point. "Python objects have a fixed overhead in the tens of bytes; a collection of N small objects costs that overhead N times; if that is too much, the answer is a different representation" — `array.array`, a NumPy array, `bytes`, a single packed structure — "that stores the values without per-element object overhead." That sentence is the staff-level understanding. It is also, not coincidentally, the reason `__slots__` exists, the reason NumPy exists, and the reason a `dataclass(slots=True)` can be the right call. We will reach each of those.

## 2.3 Identity Versus Equality

Because a name refers to an object, and because two names can refer to *the same* object or to two *different but equal* objects, Python gives you two distinct questions and two distinct operators.

`==` asks: **are these equal?** Do they have the same value? It is answered by the objects themselves, via the `__eq__` method, and you can define what it means for your own types.

`is` asks: **are these the same object?** Is this literally the same thing in memory, the same `PyObject`? It is answered by comparing identities — in CPython, comparing memory addresses — and you cannot override it.

Most of the time you want `==`. You want to know whether two strings have the same characters, whether two lists have the same elements. You almost never genuinely care whether they are the same object — with exactly two important exceptions, which is why `is` exists at all.

The first exception is the singletons. `None`, `True`, and `False` are each a single, unique object that exists once in the entire running program. There is only ever one `None`. So the idiomatic test for it is `if x is None`, not `if x == None` — you are asking the question you actually mean ("is this *the* None object"), it cannot be fooled by a class with a malicious `__eq__`, and it is a hair faster. This is not a style preference; `is None` is correct and `== None` is a small code smell.

The second exception is when you genuinely, by design, care about object identity — a cache keyed on "the same object," an `is`-based check in a framework's internals. Rare in application code, but real.

Outside those two cases, reaching for `is` is usually a mistake — and it is a *particular* mistake, one that produces a bug with a signature worth memorizing, because Python sets a trap here.

### The interning trap

Run this in a Python shell:

```python
>>> a = 256
>>> b = 256
>>> a is b
True
>>> a = 257
>>> b = 257
>>> a is b
False
```

Two integers with the same value, and `is` says the 256s are the same object but the 257s are not. This looks insane. It is not insane; it is an optimization, and once you see it the trap is disarmed.

Small integers are used constantly — loop counters, indices, small counts, the numbers zero and one above all. So CPython, at startup, pre-creates the integer objects from −5 to 256 and keeps them in a cache. Every time your code produces a small integer in that range, it does not allocate a new object — it hands back the cached one. So `a = 256` and `b = 256` both end up pointing at the *same* pre-made object, and `is` reports `True`. But 257 is outside the cached range, so `a = 257` and `b = 257` each allocate a *fresh* object, and `is` reports `False`.

Strings get a related treatment, called *interning*. Short strings that look like identifiers — `"hello"`, `"user_id"` — are often automatically interned: stored once, reused. So `"hello" is "hello"` is frequently `True`. But a string built at runtime — `"hel" + "lo"`, or a string read from a file or a network socket — is typically a fresh object, and `is` against an interned `"hello"` will be `False`.

Here is why this is a trap and not a trivium. Suppose an engineer, not fully understanding this, writes a status check as `if status is "active"`. They test it. The test data has the literal string `"active"` written in the test file, the compiler interns it, the two `"active"`s are the same object, `is` returns `True`, the test passes. The code ships. In production, `status` arrives from a JSON payload over the network — a freshly allocated string object. It *equals* `"active"`; it *is not* the interned `"active"`. The check silently returns `False`. Every request is mishandled, and every test still passes.

> ### War Story — The Status That Was Never Active
>
> The bug above is not hypothetical. A team shipped a feature-flag check written with `is` against a string literal. It passed code review — the reviewer's eye slid over `is` as a synonym for `==`. It passed every unit test, because the test fixtures used string literals and CPython interned them into identity. It reached production and the flag simply never registered as enabled, because the flag value came from a config service as a fresh string. The feature was "live" for a week, enabled for zero users, and nobody noticed until someone asked why the metrics were flat.
>
> The fix was one character: `is` to `==`. The lesson was larger. The bug existed because `is` and `==` *look* interchangeable and *usually behave* interchangeably for small literals — the interning cache makes the wrong code work just often enough to pass review and tests. The defense is a rule with no exceptions in application code: **`is` is for `None`, `True`, `False`, and deliberate identity checks. Value comparison is always `==`.** A linter rule (`ruff` flags `is` against literals) makes the rule enforceable rather than merely advised.

The deeper point, and the reason this lives in a chapter about the machine rather than a chapter of tips: the small-integer cache and string interning are *CPython implementation details*. They are not promised by the language. PyPy may intern differently. A future CPython may change the cached range. Code whose correctness depends on them is code built on sand. `is` for value comparison is wrong *even when it works*, because it works for a reason the language never guaranteed.

## 2.4 Reference Counting

An object is created; it is used; eventually it is needed no longer and its memory must be reclaimed. Languages answer the "when do we reclaim it" question in different ways. C makes it the programmer's job and punishes mistakes with crashes and leaks. Java and Go run a tracing garbage collector that periodically sweeps the whole heap. CPython's primary mechanism is the oldest and simplest idea in the field: *reference counting*.

Recall `ob_refcnt` in the object header. It counts how many references currently point at the object. The rule is mechanical. Every time a new reference to an object is created — you bind it to a name, append it to a list, pass it as an argument, store it in an attribute — the count goes up by one. Every time a reference goes away — a name is rebound or falls out of scope, an item is removed from a list, a function returns and its locals vanish — the count goes down by one. And the instant the count hits zero, the object is unreachable, and CPython frees it immediately.

You can watch the count:

```python
>>> import sys
>>> x = object()
>>> sys.getrefcount(x)
2
```

Two, not one — because at the moment `getrefcount` runs, there are two references: the name `x`, and the argument inside the `getrefcount` call itself. That `+1` artifact is a small thing but worth knowing; it confuses people who expect `1`.

Reference counting has two genuine virtues, and they are why CPython has never abandoned it. The first is *promptness*: an object dies the moment it becomes unreferenced, not at some later sweep. The memory comes back immediately, and any cleanup tied to the object's death — a file handle closing, a lock releasing — happens deterministically, at a predictable point in the code. The second is *smoothness*: the work of freeing memory is spread evenly through execution, one decrement at a time, rather than concentrated into stop-the-world pauses. For a latency-sensitive service that even-ness is worth a great deal.

But reference counting has two equally genuine flaws, and a staff engineer must be able to name both, because each one is the seed of a later chapter.

The first flaw is *cost under concurrency*. Every reference creation and destruction is a write to a shared counter. As long as one thread runs Python at a time, those writes are safe and cheap. The moment you have true multi-core parallelism, every one of those incessant counter updates must be made thread-safe — atomic — and atomic operations are markedly slower than plain ones. This is not a footnote. It is one of the central reasons the Global Interpreter Lock existed for thirty years, and one of the hardest problems the free-threaded Python effort (PEP 703) had to solve. We give it a full treatment in Chapter 13. For now, hold the link: *reference counting and the GIL are connected, and the connection is the cost of atomic counter updates.*

The second flaw is structural, and it gets the next section to itself.

## 2.5 The Cycle That Reference Counting Cannot Break

Reference counting frees an object when its count reaches zero. So consider two objects that refer to each other:

```python
a = {}
b = {}
a['partner'] = b      # a references b
b['partner'] = a      # b references a
```

Draw the references. The name `a` points at the first dict; the name `b` points at the second. The first dict points at the second; the second points at the first. The first dict's reference count is two — one from the name `a`, one from inside `b`. The second dict's count is likewise two.

Now delete the names:

```python
del a
del b
```

Each count drops by one. Both objects are now at a reference count of *one* — each still referenced by the other. Neither has hit zero. So reference counting frees neither. But the names `a` and `b` are gone; *no code anywhere can reach either dict.* They are unreachable garbage, and they are holding their memory, and reference counting will hold it forever, because reference counting can only see local counts and a cycle keeps every local count above zero.

This is the *reference cycle*, and it is reference counting's fatal structural gap. Cycles are not exotic — a doubly-linked list has them on every node, a tree with parent pointers has them, a graph is made of them, an object that holds a callback that closes over the object has one. Any non-trivial program creates reference cycles constantly.

So CPython has a *second* memory manager whose only job is to find and break cycles: the *cyclic garbage collector*, the thing you reach through the `gc` module. This is the point that surprises engineers who thought they knew this material. Python does not have "a garbage collector" in the singular. It has reference counting, which does the overwhelming majority of the work, promptly and smoothly — and it has a cycle collector that runs periodically to catch the specific thing reference counting structurally cannot: unreachable cycles.

The cycle collector works generationally, on the well-supported observation that most objects die young. Newly created objects go in *generation 0*. The collector examines generation 0 frequently; objects that survive a collection — because they are still reachable — are promoted to generation 1, examined less often; survivors of generation 1 reach generation 2, examined least often. The premise is that an object that has already survived a few collections is probably long-lived and not worth re-checking constantly. You can see and adjust the thresholds:

```python
>>> import gc
>>> gc.get_threshold()
(700, 10, 10)
```

By default, after 700 net allocations a generation-0 collection runs; after ten generation-0 collections, generation 1 is collected; and so on.

For most services the defaults are fine and you should not touch them. But you should know the two knobs exist, because at the extremes they matter. A latency-sensitive service can find that cycle collection introduces small pauses at inconvenient moments; some such services raise the thresholds, or call `gc.disable()` and run `gc.collect()` deliberately at a quiet point — between requests, say. The famous case is Instagram, which disabled cyclic GC in their web workers and reported real CPU and memory wins. That story gets cited a lot, and it is true, but the lesson engineers take from it is often wrong. Instagram disabled GC after *profiling their specific workload* and proving it helped. Disabling GC because you read that Instagram did is not optimization; it is cargo-culting, and on a workload that *does* generate cycles it is how you turn a healthy service into one with an unbounded memory leak. Measure first. Always.

## 2.6 Weak References: Pointing Without Owning

Sometimes you need to refer to an object *without* keeping it alive — to point at it without that pointer counting toward its reference count. The motivating case is a cache. You want a cache that maps keys to expensive-to-build objects; but you do not want the cache itself to be the reason an object never dies. If every other reference to an object has gone away, the cache holding the last one is just a memory leak wearing a useful-sounding name.

The `weakref` module is the answer. A weak reference points at an object but does not increment its count. If every *strong* reference to the object goes away, the object is collected as normal — and the weak reference, asked for its target afterward, returns `None`. It knew the object could die, and it did not stop it.

```python
import weakref

class Resource:
    pass

r = Resource()
weak = weakref.ref(r)

print(weak())     # <Resource object> — still alive, r holds it
del r
print(weak())     # None — the last strong reference is gone
```

The standard library builds the useful pieces on top of this: `WeakValueDictionary`, where the values are held weakly so a cache entry vanishes once the rest of the program is done with the object; `WeakKeyDictionary`, for attaching data to objects without preventing those objects from being collected; `WeakSet`. These are the correct tools for caches, for observer registries, and for any "I want to know about this object but I do not want to be the reason it lives" relationship. Weak references are also one of the clean ways to *break a cycle deliberately* — make the back-pointer in a parent/child structure a weak reference, and the cycle never forms, and the cycle collector never has to get involved.

## 2.7 The `__del__` Trap

An object can define a `__del__` method — a *finalizer* — that runs when the object is about to be destroyed. It sounds like a destructor from C++, and engineers coming from that world reach for it to do cleanup: close the file, release the resource.

Resist. `__del__` is one of the genuinely sharp edges in the language, and the safe practice is to almost never write one.

The problems are several. The *timing* of `__del__` is not as predictable as it looks: it runs promptly when an object dies by reference counting, but an object caught in a cycle dies on the cycle collector's schedule, whenever that next runs — which could be much later, or, at interpreter shutdown, in an order that is not guaranteed. An exception raised inside `__del__` cannot propagate anywhere sensible — there is no call site to propagate to — so it is caught and printed and otherwise *swallowed*, meaning a `__del__` that fails fails silently. And at interpreter shutdown, module globals may already have been torn down when your `__del__` runs, so the names it needs may be gone.

The correct tool for "clean up this resource reliably" is not the finalizer. It is the *context manager* — the `with` statement — which makes cleanup explicit, scoped, and deterministic, and runs it even when an exception is in flight. We build context managers properly in Chapter 7. For now the rule is simply stated: **do not put cleanup logic you actually depend on inside `__del__`.** If you need a safety-net finalizer, `weakref.finalize` is a better-behaved mechanism than `__del__`. But the resource itself should be managed with `with`.

> ### The Level Line — Objects and Memory
>
> **A senior engineer** knows the difference between `is` and `==` and uses `is` only for `None`; understands that Python manages memory automatically and that they should not normally think about it; can use a context manager correctly.
>
> **A staff engineer** can explain reference counting *and* the cyclic collector as two distinct mechanisms and say what each is for; can estimate the memory cost of a data-structure choice and knows the alternatives (`__slots__`, `array`, NumPy, `bytes`) when the cost is too high; understands the interning behaviors well enough to know they are implementation details and not to depend on them; knows why `__del__` is unreliable and reaches for `with`.
>
> **A principal engineer** carries the connection between reference counting and the GIL into design conversations; can decide, with profiling evidence, whether GC tuning is warranted for a given workload and is not fooled by cargo-culted advice; treats the object model's costs as an input to architecture — choosing representations, not just algorithms — and can teach all of the above to the engineers around them.

## 2.8 At the Interview Table

Memory and the object model are favorite interview territory because the questions are short to ask, the shallow answer and the deep answer are clearly distinguishable, and the topic resists bluffing.

**The question: "What is the difference between `is` and `==`?"**

This sounds like a beginner question and is not — it is a *setup*. The bare answer ("`is` checks identity, `==` checks equality") is necessary but it is not what is being probed. The interviewer is waiting for the follow-up: *"When would `is` give you a surprising result?"*

A strong answer walks straight into the interning trap: "`is` compares object identity. The surprise is that small integers, −5 to 256, are cached and reused by CPython, and short identifier-like strings are often interned — so `a is b` can be `True` for two separate-looking small integers and `False` for two equal larger ones. That makes `is` *dangerous* for value comparison, because it can appear to work — your tests use literals, which get interned, so the test passes — and then fail in production when the value arrives fresh from a network payload. So the rule is `is` for `None`, `True`, `False`, and deliberate identity checks only; everything else is `==`." That answer shows you know not just the definitions but the *failure mode*, and that you have a rule that prevents it. If you can tell the War Story version in two sentences, better still — it shows the knowledge is lived, not memorized.

**The question: "How does Python manage memory?" or "Does Python have garbage collection?"**

The trap here is the engineer who answers "yes, Python has a garbage collector" and stops. It is not *wrong*, but it reveals a single-mechanism mental model.

The staff-level answer names *both* mechanisms and the relationship between them: "Python's primary mechanism is reference counting — every object has a count of references to it, and when the count hits zero the object is freed immediately. That's prompt and smooth. But reference counting cannot reclaim *cycles* — objects that reference each other keep each other's counts above zero even when nothing else can reach them. So CPython has a *second* mechanism, the cyclic garbage collector, which runs periodically and generationally to find and break exactly those unreachable cycles. Reference counting does the bulk of the work; the cycle collector is the backstop for the case reference counting structurally can't handle."

The principal-level answer adds a consequence: "...and that has design implications. Reference counting means atomic counter updates under true parallelism, which is a real cost and part of the GIL story. And the cycle collector can be tuned — thresholds raised, or disabled with manual collection — for latency-sensitive workloads, but only with profiling evidence. Disabling GC because you read that someone else did is how you ship a memory leak."

**The question: "Why might a long-running service slowly consume more and more memory?"**

This is a diagnostic question, and the strong answer is a *differential*, not a single guess: "Several candidates, and I'd profile to distinguish them. An unbounded cache — a dict or an `lru_cache` that only ever grows — is the most common. A reference cycle involving objects with `__del__`, which historically could defeat collection. Module-level or class-level state that accumulates across requests. A C extension leaking memory the Python-level tools can't see. I'd reach for `tracemalloc` to get allocation snapshots over time and diff them, and `memray` if I needed to see native allocations too. The fix depends on which it is — but the investigation always starts with measurement, not a guess." That structure — enumerate causes, name the tools, insist on measurement — is what a staff-level diagnostic answer sounds like.

**The red flags:** using `is` for value comparison and not knowing why it sometimes works; the single-mechanism "Python just has a garbage collector" with no awareness of reference counting; thinking `__del__` is a reliable destructor; and, on the memory-leak question, jumping to a single confident cause without mentioning measurement. The last one is the most telling — guessing instead of profiling is the habit that most reliably marks an engineer as not yet staff.

## 2.9 The Forge

**Drill 2.1.** Predict the output of `sys.getrefcount` for: a freshly created `object()`; an integer literal `1`; a string literal `"hello"`. Run them. The integer and string results will look strange — explain why, using the cache and interning.

**Drill 2.2.** Without running it, predict `True` or `False` for each: `256 is 256`; `257 is 257`; `[] is []`; `a = b = []; a is b`; `"abc" is "abc"`; `"a" * 3 is "aaa"`. Then run them and reconcile every surprise against the chapter.

**Build 2.1.** Implement a small `WeakCache` class: it maps keys to objects, holds the objects weakly, and so automatically forgets an entry once the rest of the program no longer references that object. Use `weakref.WeakValueDictionary` (or build the behavior directly with `weakref.ref` for the harder version). Demonstrate with a test that an entry disappears after its object's last strong reference is deleted.

**Build 2.2.** Write a function `find_cycles()` that creates a reference cycle, deletes the external names, calls `gc.collect()`, and uses the collector's hooks (`gc.get_objects`, `gc.garbage`, or `gc.get_referrers`) to demonstrate that the cycle was unreachable and was reclaimed by the cycle collector rather than by reference counting.

**Investigate 2.1.** You are given a small service (or write one) that leaks memory. Use `tracemalloc` to take snapshots at intervals, diff them with `compare_to`, and identify the leaking allocation site. Write up what you found and how the tooling led you there. The point is the *method*, not the specific bug.

**Design 2.1.** A service holds, in memory, a large collection of small records — tens of millions of them — and is running close to its memory limit. You are asked to cut its memory footprint. Write a one-page analysis: estimate the current per-record overhead, lay out the candidate representations (a `dataclass` with `__slots__`, `NamedTuple`, a columnar layout with `array` or NumPy, packed `bytes`, an out-of-process store), and give the trade-offs of each — memory saved, access cost, code complexity, what you give up. End with what you would measure before committing. Do not pick a single answer; build the decision.

---

[← Chapter 1: The Execution Model](01-execution-model.md) · [Home](README.md) · [Chapter 3: Names, Scope, and Binding →](03-names-scope-binding.md)
