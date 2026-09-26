# Chapter 5 — Functions and Decorators

*Part II — The Language*

[← Chapter 4: Built-in Data Structures](04-built-in-data-structures.md) · [Home](README.md) · [Chapter 6: Objects and Classes →](06-objects-and-classes.md)

---

## 5.1 The Question Behind the Chapter

A function, to a beginner, is a way to avoid repeating yourself — a labeled block of code you can call more than once. That is true and it is the smallest part of the truth. In Python a function is something more powerful and more interesting: a *first-class object*, a value like any other, that can be stored in a variable, placed in a list, passed as an argument, and returned as a result. Functions in Python are not just *how you structure code* — they are *data your code can manipulate*.

That single fact — functions are objects — is the foundation of an enormous amount of Python's expressiveness. It is why you can pass a `key` function to `sorted`. It is why callbacks work. It is why the `functools` module exists. And it is why *decorators* exist — and decorators are the gateway to *metaprogramming*, code that operates on code, which is the most powerful and most abused capability in the language.

This chapter builds that capability from the ground up. We will treat the function as the unit of abstraction, learn its anatomy in full, and then construct the decorator from first principles — not as a piece of syntax to memorize but as something you could have invented yourself, given that functions are objects. By the end you will be able to write a parameterized decorator without looking anything up, you will know the one mistake that makes a decorator quietly break the programs that use it, and you will have your first real taste of metaprogramming.

## 5.2 Functions Are Objects

Begin with the demonstration, because the claim is abstract until you see it.

```python
def greet(name):
    return f"Hello, {name}"

# A function is a value. Bind another name to it.
say_hi = greet
print(say_hi("Ana"))            # Hello, Ana

# A function can go in a data structure.
operations = [greet, str.upper, str.strip]

# A function can be passed to another function.
def apply(func, value):
    return func(value)
print(apply(greet, "Ben"))      # Hello, Ben

# A function has attributes — it is an object.
print(greet.__name__)           # greet
print(greet.__doc__)            # None (no docstring)
greet.custom_tag = "important"  # you can even attach your own
```

Everything here follows from Chapter 3's name model: `def greet(...)` creates a function *object* and binds the name `greet` to it, exactly as `x = 5` creates an integer object and binds `x`. The name and the object are separate; `say_hi = greet` is just a second label on the one function object. The function carries attributes — `__name__`, `__doc__`, the `__code__` object from Chapter 1 — because it *is* an object, and you can read them and even add your own.

This is not a curiosity to file away. It is the mechanism behind a great deal of practical code. `sorted(data, key=len)` *passes the function `len`* as an argument. A plugin registry that maps strings to handler functions is just a dict whose values are functions. A retry helper that takes "the thing to retry" takes a function. Every time you treat behavior as a value — pass it, store it, return it — you are using functions-as-objects, and the rest of the chapter is the deep end of that pool.

## 5.3 The Anatomy of a Signature

Python's function signatures are unusually expressive — there are five distinct kinds of parameter, and a fluent engineer knows all five, because real APIs use all five and interview questions probe them.

**Positional-or-keyword parameters** are the ordinary kind: `def f(a, b)`. The caller may pass them by position (`f(1, 2)`) or by name (`f(a=1, b=2)`).

**Default values** make a parameter optional: `def f(a, b=10)`. Recall Chapter 3's hard-won lesson — the default expression is evaluated *once*, at definition time — so defaults must be immutable, and a mutable default is the bug the sentinel pattern (`b=None`, then build inside) exists to prevent. That lesson is load-bearing here; it is not repeated, it is *assumed*.

**`*args`** collects extra positional arguments into a tuple: `def f(*args)` lets the caller pass any number of positionals. Inside, `args` is a tuple.

**`**kwargs`** collects extra keyword arguments into a dict: `def f(**kwargs)` accepts any number of named arguments. Inside, `kwargs` is a dict. These two together — `def f(*args, **kwargs)` — are the signature that accepts *anything*, and you will see it constantly in decorators, for exactly the reason the next section makes clear.

**Keyword-only and positional-only parameters** are the precise controls. Anything after a bare `*` in the signature is *keyword-only* — it *must* be passed by name: `def f(a, *, b)` means `b` cannot be passed positionally, only as `b=...`. Anything before a `/` is *positional-only* — it *cannot* be passed by name: `def f(a, /, b)` means `a` must be positional. These exist for API design: keyword-only parameters force callers to be explicit at the call site (`connect(host, port, *, timeout=30, retries=3)` makes `connect("db", 5432, timeout=10)` readable and `connect("db", 5432, 10, 3)` impossible — no mystery positional arguments); positional-only parameters let you name a parameter freely in your own code without that name becoming part of your public contract.

The full ordering, when several kinds appear together: positional-only, then positional-or-keyword, then `*args`, then keyword-only, then `**kwargs`. You rarely use all five at once, but you should be able to read a signature that does.

## 5.4 Closures, Revisited as a Tool

Chapter 3 introduced the closure as the thing behind a famous bug — the late-binding loop. Here we meet it as what it actually is: a tool, and specifically the tool the rest of the chapter is built from.

A closure, recall, is a function defined inside another function that uses a name from the enclosing scope; it "closes over" that binding and carries it, alive, after the outer function has returned. Stripped of the loop-variable trap, that is a clean and powerful capability: it lets a function *carry state and configuration with it*.

```python
def make_validator(minimum, maximum):
    def validate(value):
        return minimum <= value <= maximum
    return validate

in_range = make_validator(0, 100)
print(in_range(50))     # True
print(in_range(150))    # False
```

`make_validator` is a *factory*: each call produces a fresh `validate` function carrying its own `minimum` and `maximum`. The returned function is a small bundle of behavior-plus-state — which is, if you think about it, most of what an object is, achieved with a function and an enclosing scope. Closures are how a function "remembers" the context it was created in. And a decorator, as we are about to see, is precisely a closure used for one specific purpose: remembering *the function it wraps*.

## 5.5 The Decorator, Built From First Principles

Decorators are where many engineers' understanding goes shallow — they learn the `@` syntax as an incantation and can apply existing decorators but freeze when asked to write a non-trivial one. We will not do that. We will *derive* the decorator, so that the `@` is revealed as convenient shorthand for something you fully understand and could have built yourself.

Start with a concrete need. You have a function, and you want to measure how long it takes to run, without editing the function's own body.

You know two things from this chapter. Functions are objects, so a function can take another function as an argument and return one. And closures exist, so a returned function can carry the original function with it. Put those together:

```python
import time

def timed(func):
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)          # call the original
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper
```

Read what this is. `timed` is a function that *takes a function* (`func`) and *returns a function* (`wrapper`). `wrapper` is a closure — it closes over `func`. When `wrapper` runs, it records a start time, calls the original `func` with whatever arguments it received, records the elapsed time, prints it, and returns the original's result. The `*args, **kwargs` signature on `wrapper` is what lets it stand in for *any* function regardless of that function's own signature — it accepts everything and passes everything through. That is why `*args, **kwargs` appeared in Section 5.3 with a promise; this is the promise paid.

Now use it, the manual way first, so the magic is visible:

```python
def compute(n):
    return sum(i * i for i in range(n))

compute = timed(compute)        # wrap it, rebind the name
compute(1000)                   # prints "compute took 0.0001s"
```

That line — `compute = timed(compute)` — is the whole idea. We passed `compute` to `timed`, got back a `wrapper` that times-then-calls, and rebound the name `compute` to that wrapper. Every later call to `compute` now goes through the timing logic. We changed what `compute` *does* without touching its source. That is a *decorator*, and we built it with nothing but functions-are-objects and closures.

The `@` syntax is *exactly* this and nothing more:

```python
@timed
def compute(n):
    return sum(i * i for i in range(n))
```

`@timed` placed above `def compute` means precisely `compute = timed(compute)`, run automatically right after the `def`. It is shorthand. It is not a separate mechanism, it has no extra power, and once you have seen `compute = timed(compute)` you have seen everything the `@` does. An engineer who understands the manual form is never confused by the syntactic form.

## 5.6 The Bug That Decorators Introduce — and `functools.wraps`

There is a flaw in the `timed` decorator above, and it is not a stylistic nitpick — it is a real bug that breaks real programs, and it is the single most important thing in this chapter to retain.

After `@timed`, ask the decorated function its name:

```python
@timed
def compute(n):
    """Compute the sum of squares up to n."""
    return sum(i * i for i in range(n))

print(compute.__name__)     # 'wrapper'   — not 'compute'
print(compute.__doc__)      # None        — the docstring is gone
```

The name is `compute` is now bound to the `wrapper` object — and `wrapper` is a different function, with its *own* `__name__` (`'wrapper'`), its own (absent) `__doc__`, its own signature metadata. The decorator has *replaced* the function, and in doing so has thrown away the original's *identity*. The docstring is lost. The name is wrong. The signature, as far as introspection can see, is now `(*args, **kwargs)`.

Why does this matter beyond cosmetics? Because real systems *read* that metadata. Documentation generators read `__doc__` and `__name__`. Web frameworks inspect function signatures to do request routing and dependency injection — FastAPI does exactly this, and we will see it in Chapter 17. Debuggers and tracebacks report `__name__`. A decorator that destroys this metadata can silently break a framework's introspection, and the failure shows up far from the decorator, as a baffling routing error or empty documentation, with nothing pointing back at the cause.

The fix is one line, and `functools` provides it:

```python
from functools import wraps

def timed(func):
    @wraps(func)                    # <-- the fix
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.4f}s")
        return result
    return wrapper
```

`@wraps(func)` is itself a decorator — applied to `wrapper` — that copies the identity metadata (`__name__`, `__doc__`, `__module__`, the signature, and more) *from the original `func` onto `wrapper`*. With it, the decorated `compute` reports its name as `'compute'`, keeps its docstring, and presents its real signature to anything that introspects it.

The rule is absolute: **every decorator you write wraps its `wrapper` with `@wraps(func)`.** There is no situation in normal code where you want a decorator to discard the wrapped function's identity. Omitting `wraps` is not a style violation; it is a latent bug, and the War Story at the end of this chapter is what it costs.

## 5.7 Parameterized Decorators: The Three-Layer Function

The decorators so far take no configuration. But often you want one that does — `@retry(times=3)`, `@cache(ttl=60)`, `@route("/users")`. A decorator that takes arguments has one more layer, and the layer trips people up until they see *why* it must be there.

The reasoning is mechanical, and worth walking slowly. We established that `@timed` means `compute = timed(compute)` — the thing after `@` is called with the function as its argument. So what must `@retry(times=3)` mean? The thing after `@` is `retry(times=3)` — and *that whole expression* must, by the same rule, be something that can be called with the function. So `retry(times=3)` must itself *return a decorator*. Which means `retry` is a function that takes the configuration (`times=3`) and returns a decorator, and that decorator takes the function and returns the wrapper. Three nested layers, and each layer exists for a precise reason:

```python
from functools import wraps

def retry(times):                          # LAYER 1: takes the config
    def decorator(func):                    # LAYER 2: takes the function
        @wraps(func)
        def wrapper(*args, **kwargs):       # LAYER 3: takes the call's arguments
            last_exc = None
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as exc:
                    last_exc = exc
            raise last_exc                  # all attempts failed
        return wrapper
    return decorator

@retry(times=3)
def fetch(url):
    ...
```

Trace it once and it is permanent. `@retry(times=3)` runs `retry(times=3)` — Layer 1 — which captures `times` in a closure and returns `decorator` — Layer 2. Then the `@` applies that returned `decorator` to `fetch`, so `decorator(fetch)` runs, capturing `func` in a closure and returning `wrapper` — Layer 3. The name `fetch` is bound to `wrapper`. When `fetch` is finally called, `wrapper` runs, and it has — through its nested closures — access to both `times` (from Layer 1) and `func` (from Layer 2). Three layers: one for the config, one for the function, one for the call. Each is a closure over the layer above it. Once you have seen *why* there are three, you can write a parameterized decorator from scratch, and that ability is a genuine staff-level marker — it proves you understand decorators as constructed things, not as memorized syntax.

(A professional-grade `retry` would add exponential backoff, jitter, and a filter for *which* exceptions are retryable — retrying a "bad request" forever is pointless. That is exactly Build 5.2, and Chapter 19 returns to retry logic as a distributed-systems concern. The skeleton above is the *structure*; the production version is the structure plus judgment.)

## 5.8 The `functools` Toolkit

`functools` is the standard library's collection of tools for working with functions. Five members earn their place in a fluent engineer's working memory.

**`wraps`** — covered above; the non-negotiable companion to every decorator.

**`partial`** — *partial application*: it takes a function and some of its arguments, and returns a new function with those arguments pre-filled.

```python
from functools import partial

def connect(host, port, timeout):
    ...

connect_local = partial(connect, "localhost", 5432)
connect_local(timeout=30)       # calls connect("localhost", 5432, timeout=30)
```

`partial` is the clean way to *specialize* a general function — to turn a function with many parameters into one with fewer by fixing some. It is more honest and more introspectable than wrapping the call in a lambda, and it is the right tool whenever you find yourself passing the same leading arguments to a function over and over.

**`lru_cache`** and **`cache`** — *memoization* as a decorator. `@lru_cache(maxsize=N)` makes a function remember its results: call it twice with the same arguments and the second call returns the stored result without recomputing. `@cache` (Python 3.9+) is `lru_cache(maxsize=None)` — an unbounded cache. For a pure, expensive function called repeatedly with a limited set of inputs, this is a one-line, often dramatic speedup.

But it carries two sharp edges a staff engineer must know. First, the cached arguments must be *hashable* — the cache is a dict keyed on the arguments — so a function taking a list or dict argument cannot be cached this way without adaptation. Second, and more dangerous: `@lru_cache` *holds a reference to every cached argument and result for as long as the cache lives.* Apply `@cache` (unbounded) to a method of a class, and the cache — which lives as long as the function object, effectively forever — holds a reference to every instance it was ever called on. Those instances can never be garbage-collected. It is a memory leak, and a slow, silent one, and it is the most common way `lru_cache` causes a production incident. The lesson connects straight back to Chapter 2: a cache that holds strong references is a cache that pins memory; if that is wrong for your case, you need bounds (`maxsize`), or a weak-reference-based cache, or to not cache on `self` at all.

**`reduce`** — folds an iterable into a single value by repeatedly applying a two-argument function. It is genuinely useful for a true fold (`reduce(operator.mul, numbers)` for a product), but it is often *less* readable than an explicit loop or a comprehension, and Python's designers deliberately moved it out of the built-ins into `functools` to discourage reflexive use. Reach for it for real folds; do not reach for it to look clever.

**`singledispatch`** — turns a function into one that dispatches on the *type of its first argument*, letting you register different implementations for different types. It is Python's clean answer to the kind of problem the Visitor pattern solves in other languages — and we will see in Chapter 12 that this is precisely why Python rarely needs the Visitor pattern at all.

## 5.9 Decorators in the Wild, and the Limits of the Tool

Decorators are powerful, and like every powerful tool they can be overused. A short orientation on good taste.

Decorators are *excellent* for *cross-cutting concerns* — behavior that many functions need but that is not the core logic of any of them. Timing, logging, retrying, caching, access control, input validation, registering a function in a plugin table, rate-limiting: each of these is a concern that cuts *across* many functions, and lifting it into a decorator removes the repetition and keeps each function focused on its actual job. A web framework's `@app.route("/path")` is a registration decorator; `@cache` is a performance decorator; `@require_auth` is a policy decorator. Used this way, decorators make code *clearer* — the decorator names the concern, and the function body is left clean.

Decorators are a *poor* tool when they hide behavior that the reader genuinely needs to see, or when the "concern" is really just the function's core logic in disguise, or when stacking many of them makes the actual control flow impossible to follow. A function under five decorators, each transforming arguments and results, is a function whose behavior cannot be read from its source — and unreadable is a high price. The taste, briefly: a decorator should encapsulate a concern that is genuinely *orthogonal* to the function's purpose, and it should be nameable in a word or two. If you cannot say in two words what the decorator does, or if it is doing something central rather than cross-cutting, it should probably be a plain function call inside the body where the reader can see it.

> ### War Story — The Decorator Without `wraps`
>
> A team built an internal web service on a framework that did request routing by *introspecting handler functions* — it read each handler's signature to work out what to pull from the request and bind to which parameter. Standard modern-framework behavior; FastAPI works much like this.
>
> An engineer wrote a small, useful decorator — it added structured logging around any handler it was applied to. The decorator worked: handlers it decorated did get the logging. But the engineer, newish to writing decorators, did not know about `functools.wraps`, and omitted it.
>
> The handlers that carried the logging decorator began behaving strangely — parameters arriving as `None`, request data not binding. The errors were at *request time*, deep in the framework, with stack traces that pointed into framework internals and never once mentioned the logging decorator. It took two engineers most of a day. The cause: the decorator's `wrapper` had replaced each handler, and without `wraps`, every decorated handler now presented the signature `(*args, **kwargs)` to the framework's introspection. The framework, seeing no real parameters, bound nothing. The decorator had silently lopped the *identity* off every function it touched, and the framework — entirely reasonably — believed what introspection told it.
>
> The fix was one line: `@wraps(func)` on the wrapper. The lesson was larger than one line. A decorator does not *modify* a function; it *replaces* it with a different object, and unless you explicitly copy the original's identity across, you have handed every introspecting tool in your stack a function wearing the wrong name and the wrong signature. `functools.wraps` is not optional polish. It is part of the definition of a correct decorator, and the cost of learning that in production is a wasted day and a class of bug that hides perfectly.

> ### The Level Line — Functions and Abstraction
>
> **A senior engineer** uses functions fluently, knows the parameter kinds, can apply existing decorators, and can write a simple decorator — and knows to use `functools.wraps`.
>
> **A staff engineer** can derive the decorator from first principles — functions-are-objects plus closures — and write a parameterized (three-layer) decorator without reference; understands *why* `wraps` is mandatory and what breaks without it; knows the `functools` toolkit and, critically, the `lru_cache`-on-a-method memory leak; has the taste to know when a decorator clarifies and when it obscures.
>
> **A principal engineer** treats decorators as one instrument of metaprogramming among several, choosing them deliberately against the alternatives; sets the team's conventions for where decorators are appropriate; and can teach the first-principles derivation, so that the engineers around them understand decorators as constructed things rather than memorized syntax.

## 5.10 At the Interview Table

Functions and decorators are heavily favored interview ground, because writing a decorator live is a compact, revealing test: it exercises closures, scope, `*args`/`**kwargs`, and — if the candidate truly understands it — the ability to *construct* rather than recall.

**The question (a live-coding staple): "Write a decorator that retries a function on failure."**

This is asked constantly. The interviewer is watching the *process*. A strong candidate states the structure before typing — "a decorator is a function taking a function and returning a wrapper; since this one needs a retry count it's parameterized, so it's three layers: config, function, call" — then writes it. The complete answer has `@wraps(func)` on the wrapper *without being reminded* (its absence is a noted deduction), catches exceptions in a loop, and re-raises the last exception when all attempts fail rather than silently returning `None`. The follow-ups are predictable and you should pre-empt them: "In production I'd add exponential backoff and jitter so retries don't synchronize into a thundering herd, and I'd filter which exceptions are retryable — retrying a 400-level error forever is pointless." Mentioning those before being asked signals real-world experience.

**The question: "What does `functools.wraps` do, and what happens without it?"**

"A decorator replaces a function with the wrapper — a different object, with its own `__name__`, `__doc__`, and signature metadata. `@wraps(func)` copies the original function's identity onto the wrapper. Without it, the decorated function reports the wrapper's name, loses its docstring, and presents a `(*args, **kwargs)` signature to anything that introspects it. That breaks documentation tools, debuggers, and — most damagingly — frameworks that do signature introspection for routing or dependency injection. It's not cosmetic; it's a real bug, and it hides well because the failure surfaces far from the decorator." Telling the War Story in two sentences makes it land as experience rather than book knowledge.

**The question: "Explain why a parameterized decorator needs three levels."**

Walk the derivation, do not assert it: "`@deco` means `f = deco(f)` — the thing after `@` is called with the function. So `@deco(arg)` means the thing after `@` is `deco(arg)`, and *that* must be callable with the function. So `deco(arg)` has to *return a decorator*. That gives three layers: the outer function takes the configuration, the layer it returns takes the function, and the innermost wrapper takes the call's arguments — each a closure over the one above." A candidate who derives it understands decorators; one who just says "you need three `def`s" has memorized a shape.

**The question: "When is `lru_cache` dangerous?"**

"Two cases. The arguments must be hashable, so it can't directly cache a function taking a list or dict. And more seriously, the cache holds strong references to every cached argument and result for as long as the cache lives — and the cache lives as long as the function. Put `@cache` on a method and it pins every instance it's ever been called on, which can never be garbage-collected: a slow, silent memory leak. The fixes are a bounded `maxsize`, a weakref-based cache, or simply not caching on `self`."

**The red flags.** Writing a decorator without `@wraps` and not noticing. Being unable to explain *why* a parameterized decorator has three layers — the clearest sign of memorized syntax over understanding. A retry decorator that swallows the exception and returns `None` on total failure. And no awareness that `lru_cache` has memory implications — which signals an engineer who has used it but never operated it at scale.

## 5.11 The Forge

**Drill 5.1.** Write each of these decorators, each with `@wraps`: (a) `@count_calls` — prints how many times the function has been called; (b) `@debug` — prints the function's name and arguments before each call and its return value after; (c) `@suppress(*exception_types)` — a *parameterized* decorator that catches the listed exception types and returns `None` instead of raising.

**Drill 5.2.** Take the un-parameterized `timed` decorator from Section 5.5 (without `wraps`) and the parameterized `retry` from Section 5.7. For each, write down — before running anything — what `decorated_func.__name__` will be with and without `@wraps`, then verify.

**Build 5.1.** Implement `@memoize`, your own version of `lru_cache` with a `maxsize`. It should cache results keyed on the call arguments, evict the least-recently-used entry when full (reuse the LRU work from Chapter 4), and expose `cache_info()` reporting hits, misses, and current size. Then write a short note on the memory-leak hazard of applying your decorator to a method, and how a user of your decorator could avoid it.

**Build 5.2.** Write a production-grade `@retry` decorator: parameterized with the number of attempts, an exponential backoff with jitter between attempts, and a tuple of *retryable* exception types (non-matching exceptions propagate immediately rather than being retried). Re-raise the final exception on exhaustion. Write tests that prove the backoff timing, the exception filtering, and the final re-raise.

**Investigate 5.1.** Read the actual CPython source for `functools.lru_cache` (it is in `Lib/functools.py`). Identify how it stores the cache, how it makes a hashable key from the call arguments — including how it handles a mix of positional and keyword arguments — and how the LRU eviction is implemented. Write a one-page summary of what you found and one thing it does that your Build 5.1 implementation did not.

**Design 5.1.** Your team is building a service and several engineers have started writing decorators — for logging, caching, auth, validation, rate-limiting, metrics. There is no shared convention, and some handlers are now under five or six stacked decorators. You are asked to write the team's guidance. Write a one-to-two-page document: when a decorator is the right tool and when it is not; rules every team decorator must follow (`wraps`, naming, what may and may not be a decorator); how to handle the ordering and stacking of multiple decorators; and how to keep heavily-decorated functions readable. Make real recommendations, not platitudes — this is a judgment exercise.

---

[← Chapter 4: Built-in Data Structures](04-built-in-data-structures.md) · [Home](README.md) · [Chapter 6: Objects and Classes →](06-objects-and-classes.md)
