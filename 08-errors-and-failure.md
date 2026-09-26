# Chapter 8 — Errors and the Discipline of Failure

*Part II — The Language*

[← Chapter 7: Iteration, Generators, and the Lazy Mindset](07-iteration-and-generators.md) · [Home](README.md) · [Chapter 9: Modules, Packages, and Imports →](09-modules-packages-imports.md)

---

## 8.1 The Question Behind the Chapter

A junior engineer writes the happy path. The code that handles the expected input, produces the expected output, and works in the demo. A senior engineer writes the happy path *and the failure paths* — and a staff engineer treats the failure paths as a *design problem*, given the same care as the features, because in a real system the failure paths are what runs when things go wrong, and things going wrong is not the exception, it is Tuesday.

This is the shortest chapter in Part II, and it is not short because the topic is small. It is short because the topic is *focused*: error handling is not a large body of mechanism, it is a small body of mechanism plus a large body of *judgment*, and judgment is taught by principle and example, not by page count. The mechanism — the exception hierarchy, the `try` clauses, chaining, exception groups — fits in a few pages. The judgment — when to catch and when to let it propagate, EAFP versus LBYL, how to design an exception hierarchy, what an anti-pattern actually costs — is the chapter's real subject.

The thesis, stated up front: **error handling is a design discipline, and the quality of a system's failure behavior is one of the clearest signals of the seniority of the people who built it.** Code that crashes with a useful message and a clean traceback was built by someone thinking about failure. Code that crashes with a bare `KeyError` three layers from the cause, or — worse — does not crash but silently produces wrong answers, was built by someone who wrote only the happy path.

## 8.2 The Exception Hierarchy

Exceptions in Python are objects, and the exception *classes* form an inheritance tree. Knowing the shape of that tree matters, because `except SomeClass` catches `SomeClass` *and every subclass of it* — so where a class sits in the hierarchy determines what catches it.

At the root is `BaseException`. Directly under it sit a few special exceptions — `SystemExit` (raised by `sys.exit()`), `KeyboardInterrupt` (raised when the user presses Ctrl-C), and `GeneratorExit` — and, separately, `Exception`, which is the base of essentially every *ordinary* error: `ValueError`, `TypeError`, `KeyError`, `IndexError`, `FileNotFoundError`, `RuntimeError`, and the rest.

That split — `SystemExit` and `KeyboardInterrupt` under `BaseException` but *not* under `Exception` — is deliberate and important. It means `except Exception:` catches ordinary errors but does **not** catch `KeyboardInterrupt` or `SystemExit`. And that is exactly what you want: when a user presses Ctrl-C to kill a hung program, that `KeyboardInterrupt` should *not* be swallowed by some broad `except Exception` and ignored — the program should actually stop. The hierarchy is arranged so that `except Exception` — the broadest catch you should ordinarily write — leaves the "the program is being deliberately terminated" signals alone.

This is the first concrete rule of the chapter: **catch `Exception`, never `BaseException`, and never use a bare `except:`**. A bare `except:` (or `except BaseException:`) catches *everything*, including Ctrl-C and `SystemExit`, which means it can make a program impossible to interrupt and can swallow a deliberate shutdown. `except Exception:` is the broad catch that still behaves; bare `except:` is the broad catch that misbehaves. There is no situation in ordinary code that calls for the bare form.

## 8.3 The Four Clauses

The `try` statement has four clauses, and most engineers use two of them and have only a hazy sense of the other two. All four:

```python
try:
    result = risky_operation()
except SomeError as e:
    handle(e)                       # runs if risky_operation raised SomeError
else:
    use(result)                     # runs ONLY if try succeeded with no exception
finally:
    cleanup()                       # runs ALWAYS, exception or not
```

`try` and `except` are the familiar pair — attempt something, handle a failure. The two less-used clauses earn their place:

**`else`** runs only if the `try` block completed *without* an exception. Its value is *scoping precision*. Consider the common mistake of putting too much in the `try` block: if both `risky_operation()` and `use(result)` are inside `try`, then an exception raised by `use(result)` — which has nothing to do with the risk you were guarding against — gets caught by your `except SomeError`, and now you are handling an unrelated error as if it were the expected one. Putting `use(result)` in the `else` clause means it runs only on success, and any exception *it* raises propagates normally rather than being wrongly caught. `else` keeps the `try` block tight around exactly the operation that might fail — which is precisely what good error handling requires.

**`finally`** runs *no matter what* — whether the `try` succeeded, raised an exception that was caught, or raised one that was not. It even runs if the `try` block executes a `return`. It is for cleanup that must happen on every path, and Chapter 7 already showed its importance: the file-handle War Story was, at bottom, about cleanup that was not in a `finally`. (And recall: the `with` statement is `finally`-based cleanup made automatic — `with` is usually the better tool, but `finally` is the mechanism underneath.)

## 8.4 Exception Chaining: `__cause__` and `__context__`

When an exception is raised while you are already handling another exception, Python does not throw the first one away — it *chains* them, and the chaining is what produces those tracebacks that say "During handling of the above exception, another exception occurred." Understanding the chain is understanding how to make failures debuggable.

There are two kinds of chaining, and the distinction is worth knowing precisely.

**Implicit chaining** happens automatically. If an exception is raised inside an `except` block, Python attaches the *original* exception to the new one as its `__context__`. Nothing is lost — the traceback shows both, the original first. This is the "during handling of the above exception" message.

**Explicit chaining** is something you do deliberately, with `raise ... from ...`:

```python
try:
    config = json.loads(raw_config)
except json.JSONDecodeError as e:
    raise ConfigError("the configuration file is malformed") from e
```

`raise NewError(...) from original` raises your new, higher-level exception while attaching the original as its `__cause__` — and the traceback then reads "The above exception was the direct cause of the following exception." The difference between `__context__` and `__cause__` is the difference between "this happened to occur while we were handling that" and "*that* is the reason *this* was raised" — and `raise from` lets you state the causal relationship explicitly.

Why this matters is a real piece of design judgment. When a low-level exception crosses a layer boundary, you often want to *translate* it — to re-raise it as something meaningful in your layer's vocabulary. A `json.JSONDecodeError` is meaningful to the JSON parser; to the caller of your `load_config()` function, a `ConfigError` is far more meaningful. The translation makes your API's failures comprehensible. But translation done carelessly *destroys information* — if you catch the `JSONDecodeError` and raise a bare `ConfigError` without `from e`, the original error and its traceback are lost, and the engineer debugging at 3 a.m. sees "the configuration file is malformed" with no indication of *what* was malformed or *where*. `raise from` gives you both: the meaningful high-level error *and* the full original cause, chained, in the traceback. The rule: **when you translate an exception across a layer boundary, use `raise ... from ...` to preserve the cause.** Translating without chaining is throwing away the evidence.

## 8.5 Exception Groups and `except*`

Python 3.11 added `ExceptionGroup` and the `except*` syntax, and they solve a problem that older Python genuinely could not express: *multiple exceptions happening at once.*

Ordinary exception handling assumes one exception at a time — one thing went wrong, you catch it, you handle it. But concurrency breaks that assumption. If you run ten tasks concurrently and three of them fail, you have *three* exceptions, simultaneously, and the old model has no way to represent that — you could surface one and lose the others.

An `ExceptionGroup` is an exception that *contains* multiple exceptions. And `except*` (note the star) is a `except` variant that can reach *into* a group and handle the exceptions of a particular type that it contains, while letting the rest of the group propagate:

```python
try:
    await run_many_tasks_concurrently()
except* ValueError as eg:
    # eg is a group of the ValueErrors that occurred
    handle_value_errors(eg)
except* ConnectionError as eg:
    # eg is a group of the ConnectionErrors that occurred
    handle_connection_errors(eg)
```

You will most often *encounter* `ExceptionGroup` rather than raise it yourself, because the concurrency tool that produces it — `asyncio.TaskGroup` — is exactly the modern structured-concurrency primitive Chapter 14 covers in depth. When a `TaskGroup` has several failing tasks, it collects their exceptions into an `ExceptionGroup` and raises that. For now the point is conceptual: `ExceptionGroup` exists because *concurrent failure is plural*, and `except*` is how you handle plural failure. The connection to Chapter 14 is deliberate — this is the same compounding the book has done throughout.

## 8.6 EAFP Versus LBYL: A Real Design Choice

Two philosophies of dealing with things that might go wrong, and Python has a strong cultural preference between them — but the preference is not absolute, and knowing *when each is right* is the judgment a staff engineer is expected to have.

**LBYL** — "Look Before You Leap" — means checking that an operation will succeed *before* attempting it:

```python
# LBYL
if "key" in some_dict:
    value = some_dict["key"]
else:
    value = default
```

**EAFP** — "Easier to Ask Forgiveness than Permission" — means just attempting the operation and handling the failure if it comes:

```python
# EAFP
try:
    value = some_dict["key"]
except KeyError:
    value = default
```

Python's culture favors EAFP, and for good reasons. EAFP is often cleaner. But the *decisive* reason is not aesthetics — it is correctness under concurrency, and this is the part that makes it a real design choice rather than a style preference.

LBYL contains a hidden flaw: a *race condition* between the check and the act. Consider checking that a file exists before opening it:

```python
# LBYL — racy
if os.path.exists(path):
    f = open(path)          # the file could have been deleted RIGHT HERE
```

Between the `os.path.exists` check and the `open`, another process — or another thread — could delete the file. The check said the file existed; by the time you act on that knowledge, it is stale. The check-then-act is two operations, and anything can happen in the gap. This class of bug has a name — TOCTOU, "time of check to time of use" — and it is a real source of both correctness bugs and security vulnerabilities.

EAFP does not have the gap:

```python
# EAFP — no race
try:
    f = open(path)
except FileNotFoundError:
    handle_missing(path)
```

There is no window. The `open` either succeeds atomically or raises `FileNotFoundError`, and you handle the failure. The check and the act are the *same* operation. For anything involving shared state — the filesystem, a database, anything another process or thread can touch — EAFP is not just cleaner, it is *more correct*, because LBYL's check-then-act is racy by construction.

So when is LBYL right? When the check is genuinely cheap and reliable and there is no shared-state race — validating your *own* in-process data before a computation, for instance — and when the "exceptional" case is actually common, since exceptions have a cost when raised and a hot loop that raises on most iterations is using exceptions as control flow, which is its own anti-pattern. The judgment: **EAFP by default, especially for anything touching shared or external state; LBYL when the check is race-free and the failure is genuinely expected and frequent.**

## 8.7 Designing Custom Exceptions

A library or a sizeable application should define its *own* exception types, and doing it well is a small but real piece of API design.

The principle: a custom exception hierarchy lets the *callers* of your code catch your failures at the granularity *they* need. Define a single base exception for your library or component, and derive specific exceptions from it:

```python
class PaymentError(Exception):
    """Base for all payment-related errors."""

class CardDeclined(PaymentError):
    """The card was declined by the issuer."""

class InsufficientFunds(CardDeclined):
    """Declined specifically for insufficient funds."""

class PaymentGatewayTimeout(PaymentError):
    """The payment gateway did not respond in time."""
```

This hierarchy gives callers a *choice of granularity*. A caller that wants to handle any payment failure the same way writes `except PaymentError`. One that wants to treat declines specially writes `except CardDeclined`. One that wants to offer a "top up your balance" flow specifically for insufficient funds writes `except InsufficientFunds`. Because the specific exceptions inherit from the general ones, a single `except PaymentError` still catches them all — the caller picks the level. A flat collection of unrelated exception types cannot do this; the hierarchy is what gives callers the dial.

Good practice in the design: always have a single base exception per component, so callers can catch everything from your component with one `except` if they want. Make exception *names* descriptive — the name is the first thing in the traceback and it should communicate. Carry useful *data* on the exception as attributes — a `CardDeclined` might carry the decline code, an error involving a record might carry the record ID — so the handler has what it needs without parsing the message string. And do not over-build it: a three-level hierarchy that mirrors a real taxonomy of failures is good design; a fifteen-class hierarchy invented speculatively is ceremony.

## 8.8 The Anti-Patterns

Some error-handling patterns are not merely suboptimal — they are bugs, or they cause bugs, and a staff engineer recognizes them on sight and removes them on sight.

**The swallowed exception.** This is the worst one:

```python
try:
    do_something_important()
except Exception:
    pass
```

This catches every error and *does nothing* — no log, no re-raise, no handling. The operation may have completely failed and the program continues as if it succeeded. The result is the most expensive category of bug: not a crash, but *silent wrongness* — the system produces incorrect results and gives no indication anything went wrong. A crash is debuggable; you get a traceback pointing at the problem. A swallowed exception gives you nothing — just wrong data, discovered much later, traced back with great effort. **An exception should be handled, logged, or propagated. It should never be silently discarded.** If you genuinely intend to ignore a *specific* expected exception, catch that *specific* type, narrowly, and add a comment saying why the ignoring is correct — and even then, consider `contextlib.suppress(SpecificError)`, which at least makes the intent explicit and unmistakable.

**The over-broad `try`.** Wrapping a large block in one `try` so that an exception from any of a dozen operations gets caught by one handler. The handler cannot know which operation failed, so it cannot respond meaningfully, and an unexpected failure from one operation gets misattributed as the expected failure of another. Keep `try` blocks *tight* — around the single operation that might fail in the way you are handling — and use the `else` clause (Section 8.3) to keep success-path code out of the `try`.

**Exceptions as control flow.** Using `raise`/`except` to implement ordinary, expected branching — looping until a `StopIteration`-style exception in your own non-iterator code, or raising to break out of nested loops as a routine matter. Exceptions have a real cost when raised, and more importantly, code that uses them for normal flow is hard to read because exceptions *signal the exceptional*. When `except` is catching something that happens on most runs, that thing is not exceptional and should not be an exception.

**The `assert` for validation.** This one is a genuine *security and correctness* bug, and it surprises people. `assert` statements are **removed entirely** when Python runs with the `-O` (optimize) flag. So code like `assert user.is_authorized, "access denied"` — using `assert` to enforce a real security check — *does nothing at all* when the program runs optimized, and production deployments often run optimized. The check silently vanishes. `assert` is for *internal sanity checks during development* — "this condition should be impossible; if it is not, I have a bug" — things you are content to have compiled out. It is *never* for validating input, enforcing authorization, or any check whose absence would be a real problem. Real validation raises a real exception, unconditionally.

> ### War Story — The Exception That Was Swallowed
>
> A data pipeline had a stage that enriched each record by calling an external service. The engineer who wrote it knew the external service was occasionally flaky, and wanted the pipeline to be "robust" — to not crash when one enrichment call failed. So they wrapped the call: `try: enrich(record) except Exception: pass`.
>
> The pipeline became extremely "robust." It never crashed. It also, when the external service had a bad day, silently produced records with the enrichment *missing* — and because the exception was swallowed with no log, there was no signal anywhere that this was happening. The pipeline reported success. Its dashboards were green. Downstream systems consumed the under-enriched records and made decisions on them. Weeks later, someone investigating a discrepancy in a downstream report traced it back, with considerable effort, through several systems, to a pipeline that had been quietly dropping enrichment data every time the external service hiccuped — for weeks, with no alert, because the exception that would have revealed it was being swallowed by a `pass`.
>
> The fix was to handle the failure *honestly*: catch the *specific* exception the flaky service could raise, *log it* with the record ID and the error, increment a metric so the rate of enrichment failures was visible on a dashboard, and then decide — deliberately, as a product decision — whether a record that failed enrichment should be retried, sent to a dead-letter queue, or passed through with an explicit "enrichment unavailable" marker. Every one of those is a legitimate choice. `except Exception: pass` is not a choice — it is the *absence* of a choice, disguised as robustness. The lesson the team adopted: an exception is information, and `pass` throws the information away; robustness is *handling* failure visibly, never *hiding* it.

> ### The Level Line — Errors and Failure
>
> **A senior engineer** uses `try`/`except`/`finally` correctly, catches specific exceptions rather than bare `except:`, knows not to swallow exceptions, and uses `with` for cleanup.
>
> **A staff engineer** treats error handling as design — keeps `try` blocks tight, uses `else` for scoping, translates exceptions across layer boundaries with `raise from` to preserve the cause, designs custom exception hierarchies that give callers a granularity dial, chooses EAFP versus LBYL deliberately (knowing the TOCTOU race), and knows the `assert`-removed-under-`-O` trap; recognizes the anti-patterns on sight.
>
> **A principal engineer** designs the *failure model* of a system as deliberately as its feature set — what fails, how it fails, what is retried, what degrades, what alerts — establishes the team's conventions for exception design and handling, and treats the quality of failure behavior as a first-class measure of the system's engineering. (Chapters 19, 20, and 23 extend this from a single program to a distributed system.)

## 8.9 At the Interview Table

Error handling is interview territory both as a technical topic and as a behavioral one — "tell me about an incident" is, very often, a question about someone's failure-handling judgment.

**The question: "What's wrong with `except: pass`?"**

A complete answer hits both problems. "Two things. The bare `except:` catches *everything* — including `KeyboardInterrupt` and `SystemExit`, so it can make the program impossible to Ctrl-C and can swallow a deliberate shutdown; you almost always want `except Exception` instead, which leaves those alone. And the `pass` *swallows* the exception — does nothing with it. That's the more dangerous half: the operation may have entirely failed and the program continues as if it succeeded, producing silent wrong results. A crash gives you a traceback; a swallowed exception gives you nothing, just wrong data discovered much later. An exception should be handled, logged, or propagated — never silently discarded. If I truly mean to ignore a *specific* expected exception, I catch that narrow type and comment why, or use `contextlib.suppress` to make the intent explicit."

**The question: "EAFP or LBYL — which, and why?"**

"Python favors EAFP, and the strongest reason isn't style — it's correctness under concurrency. LBYL is check-then-act, two separate operations, and anything touching shared or external state has a race in the gap between them: you check a file exists, and it's deleted before your `open` runs. That's the TOCTOU class of bug, and it's both a correctness and a security issue. EAFP — just attempt it, handle the exception — has no gap; the check and the act are one atomic operation. So: EAFP by default, definitely for anything involving the filesystem, a database, or other shared state. LBYL is fine when the check is genuinely race-free — validating my own in-process data — and when the failure case is common enough that raising exceptions for it would be using exceptions as control flow."

**The question: "How would you design exceptions for a library you're writing?"**

"A single base exception for the library, with specific exceptions deriving from it — so callers can catch at whatever granularity they need: the base class to handle anything from my library uniformly, a specific subclass to handle one failure mode specially. Descriptive names, since the name leads the traceback. Useful data carried as attributes on the exception — error codes, the relevant identifier — so handlers don't parse message strings. And I'd keep the hierarchy proportionate to the real taxonomy of failures, not invent fifteen speculative classes."

**The question (behavioral): "Tell me about a production incident."**

This is often a failure-handling question in disguise. The structure of a strong answer — and Chapter 23 develops this fully — is: the situation, what *you* specifically did, how it was *mitigated* (service restored), the *lasting* fix, and what you *changed* so it could not recur. Interviewers are listening for honesty about failure paired with learning from it. A candidate who can connect an incident to a root cause in error-handling judgment — "the underlying problem was an exception being swallowed, so we had no signal" — is demonstrating exactly the discipline this chapter teaches.

**The red flags.** Bare `except:` with no awareness of what it over-catches. Defending `except Exception: pass` as "making the code robust." Not knowing the EAFP/LBYL race argument — treating it as pure style. Not knowing `assert` is stripped under `-O`, especially using `assert` for validation. And, on the behavioral question, an incident story with no honest ownership and no lasting fix — which signals an engineer who has not yet learned that failure is a teacher.

## 8.10 The Forge

**Drill 8.1.** For each, state what is wrong and rewrite it correctly: a bare `except:` around a database call; `assert user.has_permission` used as an access check; a single `try` wrapping ten unrelated operations; `except Exception: pass` around a network call; catching `JSONDecodeError` and raising a plain `ConfigError` with no `from e`.

**Drill 8.2.** Write a `try` statement that uses all four clauses meaningfully — `try`, `except`, `else`, `finally` — for a realistic operation (e.g., opening and parsing a file), and write one sentence explaining what each clause is doing and why that code belongs in that clause rather than another.

**Build 8.1.** Design and implement a custom exception hierarchy for a small library of your choice (a payment processor, an HTTP client, a config loader — pick one). It should have a single base exception, at least three specific exceptions in a sensible inheritance structure, descriptive names, and useful data carried as attributes. Then write example calling code that catches your exceptions at two different granularities, demonstrating the dial the hierarchy provides.

**Build 8.2.** Take a function that calls an unreliable external operation. Implement *honest* failure handling for it: catch the specific exceptions it can raise, log each with context, expose a failure metric (a simple counter is fine), and implement a deliberate policy for a failed call — retry with a limit, or route to a dead-letter list, or pass through with an explicit marker. Contrast it, in a short written note, with the `except Exception: pass` version and the War Story.

**Investigate 8.1.** Demonstrate the TOCTOU race concretely. Write a LBYL check-then-act on a file (check it exists, then open it) and, using two threads or two processes, construct a scenario where the file is deleted in the gap and the code fails despite the check having passed. Then rewrite it EAFP and show the race is gone. Write up why.

**Investigate 8.2.** Demonstrate the `assert`-under-`-O` trap. Write a small program that uses `assert` for a validation check, run it normally and observe the check working, then run it with `python -O` and observe the check silently vanishing. Write up the implications for any code that uses `assert` for real validation.

**Design 8.1.** You are designing the failure model for a service that processes payments: it calls an external payment gateway, writes to a database, and emits events to a queue. Write a one-to-two-page document specifying, for each operation, what can fail, how that failure is detected, what the response is (retry, with what limits and backoff; fail the request; compensate; degrade), what is logged and what is alerted, and how a partial failure — payment succeeded but the database write failed — is handled. Define the exception hierarchy the service exposes. This is the staff-level exercise: designing failure as deliberately as features. (Chapter 19 will extend exactly this kind of thinking to the distributed case.)

---

# Part II Capstone — A Typed, Validated Configuration Loader

You have now completed Part II — the language as a tool of craft. The capstone integrates all five chapters into one small but complete and genuinely useful library, and it is deliberately the kind of thing you would be proud to have in a portfolio or to walk an interviewer through.

**The brief.** Build a configuration-loading library. It loads application configuration from a file (JSON or TOML), validates it against a schema the user of your library defines, applies defaults and environment-variable overrides, and presents the result as typed, attribute-accessible objects. It must fail clearly and early when the configuration is wrong.

**What it must demonstrate, by chapter:**

From **Chapter 4** — make deliberate, defensible data-structure choices internally, and be able to state the complexity of the operations that matter.

From **Chapter 5** — use at least one well-constructed decorator where it genuinely clarifies (a `@cached` accessor for an expensive derived value, perhaps), correctly with `functools.wraps`; and consider where `functools.partial` or `lru_cache` fits, with awareness of the latter's memory implications.

From **Chapter 6** — model the configuration as proper *types*. Decide deliberately between `dataclass`, a `frozen` dataclass, and a `Pydantic` model — and the trust-boundary rule should drive that decision, because configuration loaded from a file *is* untrusted data crossing a boundary. Get the `__eq__`/`__hash__` and immutability decisions right. Use a descriptor if a reusable validated field genuinely earns one.

From **Chapter 7** — if your loader handles a large or streaming source, or processes config entries through stages, do it lazily; and use context managers (`with`) for every file handle, never manual open/close.

From **Chapter 8** — design a real exception hierarchy: a single base `ConfigError`, specific subclasses for the distinct failure modes (file missing, malformed syntax, schema violation, missing required value), descriptive names, useful data on the exceptions. Translate lower-level exceptions (`JSONDecodeError`, `FileNotFoundError`) across your library's boundary with `raise ... from ...`, preserving the cause. Validate unconditionally — never with `assert`. Fail early and clearly: a configuration error should be caught at load time with a precise message, not surface as a mysterious failure deep in the application later.

**The deliverables:**

1. The library itself — clean, typed, with the structure of Part III's chapters foreshadowed (you will package it properly in the Part III capstone).
2. A short *design rationale* document — one to two pages — that defends each significant choice: why these types, why this exception hierarchy, where laziness was and was not used, where a decorator earned its place, where the trust boundary is and how it is enforced.

**On the interview.** This capstone is, deliberately, a thing you can present in an interview. "Walk me through something you've built" is a common prompt, and a small, complete, well-reasoned library — where you can articulate *why* every type is the type it is, *why* the failure model is shaped as it is, *where* the trust boundary sits — demonstrates staff-level thinking far better than a large, vague project. Build it as though you will be asked to defend every decision, because the skill of defending every decision is the skill the rest of this book is building toward.

---

[← Chapter 7: Iteration, Generators, and the Lazy Mindset](07-iteration-and-generators.md) · [Home](README.md) · [Chapter 9: Modules, Packages, and Imports →](09-modules-packages-imports.md)
