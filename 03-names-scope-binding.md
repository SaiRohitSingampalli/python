# Chapter 3 — Names, Scope, and Binding

*Part I — The Machine*

[← Chapter 2: Objects, Identity, and Memory](02-objects-identity-memory.md) · [Home](README.md) · [Chapter 4: Built-in Data Structures →](04-built-in-data-structures.md)

---

## 3.1 The Question Behind the Chapter

Ask an engineer what `x = 5` does and they will say "it assigns 5 to the variable x." That sentence carries a model — the *variable-as-box* model — imported, usually, from C or Java. In that model a variable is a named container, a box, and assignment puts a value into the box. The model is intuitive, it is what most of us were taught first, and in Python it is wrong in a way that produces real bugs.

This chapter replaces it. The replacement is not complicated — it is arguably *simpler* than the box — but it is different, and the difference is the whole point. Here is the claim, stated first so the rest of the chapter can earn it:

**In Python there are no variables, only names; and assignment does not put a value into a name, it makes a name refer to an object.**

A name is a label. An object is a thing. Assignment ties a label to a thing. Several labels can be tied to the same thing. A label can be retied to a different thing at any moment. The objects exist independently — they are the things from Chapter 2, with their headers and reference counts — and names are just the program's way of reaching them.

This sounds like philosophy. It is not; it is operational. Two of the most famous "gotchas" in Python — the mutable default argument, and the late-binding closure — are *not gotchas*. They are the correct, predictable behavior of the name model, and to an engineer who holds that model they are not surprising at all. The reason they are infamous is that they are surprising *to the box model*, and most engineers are still running the box model without knowing it. By the end of this chapter you will be running the name model, and a class of bugs will simply close.

## 3.2 Names Are Not Boxes

Start with the simplest possible demonstration.

```python
a = [1, 2, 3]
b = a
b.append(4)
print(a)        # [1, 2, 3, 4]
```

In the box model this is bewildering. `a` is a box, `b` is a box, we put something in `b` — why did `a` change? In the box model it should not have.

In the name model it is obvious and there is nothing to explain. `a = [1, 2, 3]` creates one list object and ties the name `a` to it. `b = a` does *not* create a second list — it ties the name `b` to *the same object* `a` is tied to. There is one list and two labels on it. `b.append(4)` reaches the list through the label `b` and mutates it. `a`, the other label on the same list, sees the change because there is only one list. Of course it does.

Now a second demonstration, the one that seems to contradict the first:

```python
a = 5
b = a
b = b + 1
print(a)        # 5
```

Here `a` did *not* change, and the box-modeler, having just been told names are shared, is now doubly confused. But the name model handles this too, with no extra rules. `a = 5` ties `a` to the integer object `5`. `b = a` ties `b` to the same `5`. Then `b = b + 1`: the right-hand side computes `b + 1`, which is `6` — a *new, different object*, because integers are immutable and arithmetic on them cannot modify them in place — and the assignment ties the name `b` to that new object `6`. The name `a` was never touched. It is still tied to `5`. There is no contradiction with the first example, and there is no second rule. The difference between the two examples is not about names at all — it is about *mutability*. The list could be mutated in place, so a change through one name was visible through the other. The integer could not be mutated, so "changing" `b` could only mean retying `b` to a new object, which leaves `a` alone.

This is the crux, and it is worth stating as a single sentence you can carry: **assignment always rebinds a name; whether other names "see" a change depends entirely on whether the object was mutated in place or replaced.** Mutable object, mutated in place — shared names see it. Any object, replaced by rebinding — only the rebound name moves. Hold that and the rest of the chapter is consequences.

A useful piece of vocabulary, because it is how Python's own documentation and the wider community talk about this: Python's argument passing is sometimes called "call by object reference" or "call by sharing." When you pass an object to a function, the function's parameter name becomes another label on the *same object*. If the function mutates that object in place, the caller sees it — same object. If the function rebinds the parameter name, the caller does not — rebinding a local name is local. This is not "pass by value" and it is not "pass by reference" in the C++ sense; it is its own thing, and "call by sharing" is the least misleading name for it.

## 3.3 Where Names Live: Namespaces and Scope

If a name is a label, it has to be written down somewhere — there has to be a registry mapping names to the objects they currently refer to. That registry is a *namespace*, and in CPython it is, very often, literally a dictionary.

A module has a namespace — that is what `globals()` hands you, and it is a real dict. A function call has a namespace for its locals — though, for speed, function locals are optimized into a fixed array of slots rather than a dict, which is exactly the `LOAD_FAST` instruction we saw being faster in Chapter 1. A class body has a namespace while it executes. Namespaces are not metaphor; they are objects, mostly dicts, that you can in many cases inspect directly.

Because the same name can appear in several namespaces at once — a `count` in a function, a `count` at module level — Python needs a rule for *which one wins* when a name is used. That rule is **LEGB**, four scopes searched in a fixed order:

**L — Local.** Names bound inside the current function. Searched first.

**E — Enclosing.** If this function is defined inside another function, the locals of those enclosing functions, from innermost outward. This is the scope that makes closures possible, and we are about to spend the rest of the chapter on it.

**G — Global.** The enclosing module's namespace — its top-level names.

**B — Built-in.** The names that are always available without import: `len`, `print`, `range`, `dict`, `Exception`, and the rest. Searched last.

When Python evaluates a name, it walks L, then E, then G, then B, and uses the first binding it finds. If it reaches the end of B without a match, that is a `NameError`.

LEGB explains a class of small puzzles. It explains why a function can *read* a module-level name without any ceremony — the name is not Local, not Enclosing, but it is found at Global. It explains why shadowing a built-in is possible and occasionally catastrophic: if you write `list = [1, 2, 3]` at module level, you have bound the name `list` in the Global namespace, and every subsequent use of `list` in that module finds your list *before* it reaches the Built-in `list` type — and the next person who writes `list(some_iterable)` gets a baffling `TypeError` because `list` is no longer the type, it is your variable. (This, incidentally, is why `ruff` and other linters flag the shadowing of built-ins. It is a real bug source.)

## 3.4 Assignment Makes a Name Local — and the `global`/`nonlocal` Keywords

Here is a subtlety that trips up engineers who think they have understood scope. Whether a name is Local inside a function is not decided at the moment the name is used. It is decided, for the whole function, at *compile time*, by a simple rule: **if a function body assigns to a name anywhere, that name is Local for the entire function** — unless the function explicitly declares otherwise.

This produces a genuinely confusing error:

```python
count = 0

def increment():
    count = count + 1     # UnboundLocalError

increment()
```

The intuition says: read the global `count`, which is `0`, add one, store it back. What actually happens: because the function body contains an assignment to `count` (`count = ...`), the compiler has marked `count` as a *Local* name for the entire function. So when the line runs, `count + 1` tries to *read* the Local `count` — which has not been assigned yet, because we are in the middle of assigning it — and Python raises `UnboundLocalError: local variable 'count' referenced before assignment`. The global `count` is right there and is never consulted, because the name was decided to be Local before the function ever ran.

To actually modify a name in an *outer* scope, you must say so explicitly:

`global name` tells the function: this name refers to the *module-global* binding; reads and writes go there, not to a Local.

`nonlocal name` tells the function: this name refers to a binding in an *enclosing function's* scope; reads and writes go there.

```python
count = 0

def increment():
    global count
    count = count + 1     # now works — count is the module global
```

But — and this is the important part, more important than the mechanics — a staff engineer's instinct on seeing `global` should be *suspicion*, not relief at having fixed the error. A function that modifies module-global state through `global` is a function with a hidden input and a hidden output. It cannot be understood from its signature. It cannot be tested in isolation without setting up and tearing down global state. It cannot be safely run concurrently. `global` is occasionally the right tool — module-level lazy initialization, a few legitimate caching patterns — but each use of it is a small piece of shared mutable state, and shared mutable state is the substance most bugs are made of. The better instinct, nearly always, is to pass the value in as an argument and return the new value out. Make the input and output visible. `nonlocal` is somewhat less alarming — it is the honest mechanism behind a closure that maintains state, which is a legitimate and useful pattern — but the same principle holds: if you find yourself reaching for it, pause and check whether a class with an explicit attribute, or a returned value, would not be clearer.

## 3.5 Closures, and the Trap That Is Not a Trap

We now have everything we need for the chapter's main event: the closure, and the single most infamous "gotcha" attached to it.

A *closure* is what you get when a function is defined inside another function and uses a name from that enclosing scope. The inner function "closes over" the name — it carries a live link to the enclosing binding, and that link stays valid even after the outer function has returned.

```python
def make_multiplier(factor):
    def multiply(n):
        return n * factor
    return multiply

double = make_multiplier(2)
triple = make_multiplier(3)
print(double(10))   # 20
print(triple(10))   # 30
```

`make_multiplier(2)` runs, `factor` is bound to `2` in its local scope, and it returns the inner `multiply`. Even though `make_multiplier` has now returned and its frame is gone, the returned `multiply` still works — it closed over `factor`, and the binding survives. `double` and `triple` are two separate closures over two separate `factor` bindings. This is a clean, powerful tool: it is how decorators carry their configuration (Chapter 5), how callbacks carry context, how a great deal of Python's expressiveness is built.

And now the famous bug. Here it is, and you have likely either hit it or will:

```python
functions = []
for i in range(3):
    def show():
        return i
    functions.append(show)

print([f() for f in functions])    # expected [0, 1, 2] — actually [2, 2, 2]
```

Three functions are built, one per loop iteration, and the intuition is that each captured "its" value of `i` — `0`, then `1`, then `2`. Instead all three return `2`. Every closure produces the same number, the *last* one.

To the box model this is a maddening gotcha with no apparent logic. To the name model — the model you have now built over this whole chapter — it is not a gotcha at all. It is exactly, precisely correct, and here is why, in one paragraph.

A closure does not capture a *value*. It captures a *name* — a live link to a binding, not a snapshot of what the binding held at the moment the inner function was defined. There is exactly one name `i` in this code. The `for` loop binds and rebinds that single name `i`: to `0`, then to `1`, then to `2`. All three `show` functions close over *the same name* `i` — the same binding — because there is only one. The closures are not evaluated during the loop; `show` does not run until `f()` is called in the final list comprehension. By the time any of them runs, the loop is long over, and the one name `i` they all share holds its final value, `2`. So all three return `2`. There is no other answer the name model could give. The behavior is not a bug in Python; the bug was in the expectation, and the expectation came from the box model — from imagining that `def show(): return i` somehow froze a copy of `i`. It never froze anything. It captured a name, and names are live links.

Once you see *why*, the fix is not a magic incantation to memorize — it follows directly. The problem is that all three closures share one binding; the fix is to give each closure its own. The cleanest way is a default argument, because — and this is the connecting thread to the next section — *default arguments are evaluated once, at definition time*, so each iteration's default captures that iteration's current value:

```python
functions = []
for i in range(3):
    def show(captured=i):     # default evaluated now, this iteration
        return captured
    functions.append(show)

print([f() for f in functions])    # [0, 1, 2]
```

Each `show` now has its own parameter `captured`, and each one's default was computed during its own iteration, freezing that iteration's value of `i`. A second clean fix is a factory function — `make_shower(i)` returning the inner function — which gives each closure a fresh enclosing scope, a fresh binding, by the mechanism of Section 3.5's first example. Both fixes work because both *break the sharing*. Knowing that — knowing the fixes work because they stop the closures from sharing one binding — is the difference between an engineer who memorized a workaround and an engineer who understands the language.

## 3.6 The Mutable Default Argument

The other infamous gotcha is a cousin of the first, and it falls to the same model.

```python
def append_to(item, target=[]):
    target.append(item)
    return target

print(append_to(1))    # [1]
print(append_to(2))    # [2]  — surely?
```

The second call returns `[1, 2]`, not `[2]`. The list "remembered" the `1` from a previous, entirely separate call. To most engineers the first time, this looks like a deep and sinister bug.

It is, again, the name model behaving correctly, plus one fact: **a function's default argument values are evaluated exactly once — when the `def` statement runs, at definition time — not on each call.** When Python executes `def append_to(item, target=[]):`, it evaluates the expression `[]`, producing *one* list object, and stores that one object as the default. That object is created once and reused as the default for every call that does not supply its own `target`. Every defaulting call shares the one list. The first call appends `1` to it; the second call, defaulting again, gets *the same list*, which already contains `1`, and appends `2`. The list is doing exactly what a single shared mutable object does when several callers mutate it — which is the lesson of Section 3.2, arriving again.

The "fix" is the sentinel pattern, and it is worth writing out because it is genuinely the correct idiom and you will use it constantly:

```python
def append_to(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

The default is now `None` — an immutable singleton, sharing-proof, nothing to mutate. Inside the function, if the caller did not supply `target`, you build a *fresh* list *on that call*. Each defaulting call gets its own new list. The bug cannot occur. Note the `is None` — Chapter 2's rule, used correctly, for exactly its intended purpose.

And note the symmetry with the closure fix. The closure bug was "many things sharing one binding"; the default-argument bug is "many calls sharing one default object." They are the same shape of bug — unintended sharing of mutable state — wearing two different costumes. An engineer who has internalized that *defaults are evaluated once* and *names are shared links* sees both of these coming and is never caught by either. That is what it means to have replaced the box model. The gotchas did not get patched. You did.

> ### War Story — The Cache That Leaked Across Customers
>
> A team had a function that processed a customer's records and, for convenience, accumulated some intermediate results into a collection. The collection was a default argument — `def process(records, accumulator=[])` — written by someone moving fast, reviewed by someone who did not pause on it.
>
> In testing, each test called `process` once, got a clean accumulator, and passed. In production the process was long-lived: it handled customer after customer, and each call that relied on the default got *the same accumulator*, the one created when the `def` ran at import time. Customer B's processing saw fragments of customer A's data sitting in the "fresh" accumulator. It was not just a bug; for a multi-tenant system it was a data-isolation incident, the kind that involves the security team and, depending on the data, a disclosure obligation.
>
> The fix was four lines — the sentinel pattern. The cost of the bug was a week of incident response and a hard conversation about how it had passed review. The lesson the team wrote into their linter configuration that day: mutable default arguments are flagged as an error, not a warning. `ruff` and the older `pylint` both catch this. There is no good reason to leave the rule off, and this is what it costs to learn that the expensive way.

## 3.7 When the Sharing Is the Point

It would be a mistake to leave this chapter thinking shared bindings and once-evaluated defaults are purely hazards to be defended against. They are tools. The hazard is *unintended* sharing; *intended* sharing is often exactly what you want.

The once-only evaluation of default arguments, the same mechanism behind the bug, is a legitimate technique. The default-argument fix for the closure loop in Section 3.5 *used* it deliberately. It is also, occasionally, a way to bind a fast local reference or a constant at definition time. The point is never "defaults evaluated once is bad" — it is "defaults evaluated once is a *fact*, and a fact you must know, because using it on purpose is fine and being surprised by it is the bug."

Likewise closures-as-shared-state. A closure that closes over a `nonlocal` counter and increments it is a small, legitimate stateful object — it is, in fact, one of the things a class is, with less ceremony. Decorators, which we build properly in Chapter 5, are closures used exactly this way: the inner wrapper closes over the configuration and the wrapped function. None of that is a hazard. It is the language working as designed.

The single discipline that separates the tool from the trap is this: **know whether you are sharing, and mean it.** Unintended sharing of mutable state is the most common bug in this chapter and one of the most common bugs anywhere. Intended sharing is a clean technique. The model you have built — names as links, assignment as rebinding, mutation as the thing that propagates through shared links, defaults as evaluated-once — is precisely the model that lets you tell which one you are doing. That is the whole reason the chapter replaced the box. Not to make you afraid of names. To make you fluent in them.

> ### The Level Line — Names, Scope, and Binding
>
> **A senior engineer** knows the LEGB rule, knows not to use mutable default arguments, and can apply the sentinel fix; can write and use a closure; knows that `global` is generally discouraged.
>
> **A staff engineer** holds the name model explicitly enough to *derive* the late-binding and mutable-default behaviors rather than memorizing them as gotchas; can explain to a junior *why* each happens, in terms of bindings and once-only evaluation; recognizes shared-mutable-state bugs by their shape across different disguises; and treats `global` as a design smell to be questioned each time.
>
> **A principal engineer** uses the model to make architectural calls — minimizing shared mutable state at the design level, knowing where intended sharing is clean and where it is a latent hazard — and can teach the name model itself, replacing other engineers' box model, which is one of the highest-leverage pieces of mentoring available on the language.

## 3.8 At the Interview Table

These topics are interview staples *because* they cleanly separate the box model from the name model. An interviewer asking about closures or mutable defaults is rarely interested in whether you have memorized the fix. They want to hear *why*, because the "why" is the proof that you understand how Python actually works.

**The question: "What does this print?" — the late-binding closure.**

You will be shown the loop-of-closures code, or something close to it, and asked for the output. Saying "`[2, 2, 2]`" is correct and is *worth almost nothing on its own* — it is a fact that could be memorized. The answer that lands is the explanation: "All three. The closures don't capture the *value* of `i` — they capture the *name* `i`, a live link to the one binding the loop keeps rebinding. The functions aren't called until after the loop finishes, and by then the single shared `i` holds its final value. So all three return `2`." Then, the fix *with its reason*: "To fix it, each closure needs its own binding — a default argument `captured=i`, which is evaluated once per iteration and freezes that iteration's value, or a factory function that gives each closure a fresh enclosing scope. Both work because both stop the closures from sharing one name."

If you can deliver that, you have demonstrated the entire chapter. The interviewer now knows you have the name model, and they will likely not need to probe scope much further.

**The question: "What's wrong with `def f(x, items=[]):`?" — the mutable default.**

The strong answer, again, leads with the mechanism, not the rule: "The default value is evaluated once, when the `def` statement runs — not per call. So that single list is shared by every call that uses the default, and because a list is mutable, mutations from one call leak into the next. The fix is the sentinel pattern: default to `None`, and build a fresh list inside the function when the argument wasn't supplied." A strong candidate then adds the judgment: "And I'd have the linter enforce it — `ruff` flags mutable defaults — because it's a bug that passes tests, since each test typically calls the function once."

The interviewer's likely follow-up is the discriminating one: *"Is the once-only evaluation always bad? Are there legitimate uses?"* The answer that marks a staff-level understanding: "No, it's not bad — it's just a fact you have to know. It's legitimately useful: it's actually how you fix the late-binding closure, by using a default argument to capture a per-iteration value. The once-only evaluation is only a *bug* when the default is *mutable* and gets mutated. An immutable default — `None`, a number, a string, a tuple — is completely safe, because there's nothing to mutate." A candidate who can say that has shown the two famous gotchas are, to them, one understood mechanism rather than two memorized warnings.

**The question: "Explain `global` and `nonlocal`. When would you use them?"**

Cover the mechanics briefly — `global` rebinds at module scope, `nonlocal` at an enclosing-function scope, and both are needed because assignment otherwise makes a name local for the whole function. But the part the interviewer is weighing is the *judgment*: "`global` I treat as a smell — it's hidden shared mutable state, it makes a function untestable in isolation and unsafe under concurrency. Occasionally it's right, for module-level lazy initialization, but my default is to pass state in and return it out. `nonlocal` is more defensible — it's the honest mechanism behind a stateful closure — but if I'm reaching for it, I check whether a small class with an explicit attribute wouldn't be clearer." Mechanics plus that judgment is a complete staff-level answer.

**The red flags.** Getting the closure output right but being unable to explain it — the clearest possible sign of memorization over understanding. Describing assignment as "copying a value into a variable" — the box model, audible in one sentence. Not knowing *why* the sentinel fix works (if you cannot say "because `None` is immutable so there is nothing to share-and-mutate," you have memorized a recipe). And treating `global` as an unremarkable tool rather than as state that needs justifying — which signals an engineer who has not yet felt the cost of shared mutable state in a real system.

## 3.9 The Forge

**Drill 3.1.** For each snippet, predict the output, then run it: (a) `a = [1]; b = a; b += [2]; print(a)` — note that `+=` on a list mutates in place, and reason about what that means for `a`; (b) `a = (1,); b = a; b += (2,); print(a)` — note that `+=` on a *tuple* cannot mutate in place; explain the difference from (a); (c) the standard late-binding loop, but using a list comprehension instead of a `for` loop to build the closures — does the comprehension change anything, and why or why not?

**Drill 3.2.** Predict whether each raises `UnboundLocalError`, `NameError`, or runs cleanly, and explain each using LEGB and the assignment-makes-local rule: a function that reads a global without assigning it; a function that reads a global on one line and assigns it on a later line; the same with `global` declared; a function that references a name that exists in no scope at all.

**Build 3.1.** Implement a `make_counter()` function that returns a closure: each call to the returned function yields the next integer, starting at 0. Use `nonlocal`. Then implement the *same* behavior as a small class with an explicit `count` attribute. Write a paragraph comparing the two — which is clearer, when would you prefer each, what does the class make visible that the closure hides?

**Build 3.2.** Write a decorator `call_count` that wraps a function and tracks how many times it has been called, exposing the count as an attribute on the wrapper. You will need a closure over mutable state. (This is a deliberate bridge to Chapter 5 — keep your solution; you will refine it there.)

**Investigate 3.1.** Find a real late-binding-closure or mutable-default bug in the public history of an open-source Python project (search issue trackers and commit messages for "mutable default" or for closure-in-loop fixes). Read the bug, the discussion, and the fix. Write up: what the symptom was, why the name model predicts it, and whether the fix that was merged is the one this chapter would recommend.

**Design 3.1.** You are reviewing a module written by a junior engineer. It maintains a dozen pieces of state as module-level globals, mutated by various functions via `global` declarations. The code works and is tested. The engineer asks why you have concerns. Write the review: explain — concretely, in terms of testability, concurrency, and reasoning-about-the-code — what the costs of this design are; describe what you would suggest instead; and, importantly, decide and justify *which* of the dozen globals, if any, are legitimate to keep as module state. Do not simply say "globals are bad" — make the real, nuanced case, the way you would actually want it made to you.

---

[← Chapter 2: Objects, Identity, and Memory](02-objects-identity-memory.md) · [Home](README.md) · [Chapter 4: Built-in Data Structures →](04-built-in-data-structures.md)
