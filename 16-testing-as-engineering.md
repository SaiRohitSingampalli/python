# Chapter 16 — Testing as Engineering

*Part IV — Concurrency and Performance*

[← Chapter 15: Performance Engineering](15-performance-engineering.md) · [Home](README.md) · [Chapter 17: Network and Web →](17-network-and-web.md)

---

## 16.1 The Question Behind the Chapter

This chapter is short, and the brevity is deliberate, not dismissive. Testing is not a large body of mechanism — the mechanics of `pytest` fit in a few pages — but it is a large body of *judgment*, and judgment is taught by principle and example rather than by exhaustive API coverage. The chapter is short the way Chapter 8 was short: the topic is focused, and most of its weight is in *how to think*, not *what to type*.

The reframing the chapter is built on, and the reason it sits here at the end of Part IV rather than in some appendix: **tests are not a chore performed after the real work; tests are the executable specification of the system, and they are the thing that makes all the rest of this book's advice safe to act on.** Every chapter so far has urged you to *change* code — refactor toward better structure, optimize a hot path, swap a concurrency model, introduce types. Every one of those changes is *dangerous* without tests and *safe* with them, because a good test suite is what tells you, in seconds, whether a change broke something. The type checker in Chapter 10 caught a refactor's broken call sites; the test suite catches a refactor's broken *behavior*. Tests are not separate from engineering. They are the safety system that makes engineering — real, continuous, confident change — possible.

So the chapter's thesis: a test suite is a specification you can execute and a safety net you can trust, and the staff-level skill is not knowing every `pytest` feature but having the *judgment* — what to test, where to test it, what to mock and what not to, when coverage is meaningful and when it lies, and the absolute refusal to tolerate a flaky test.

## 16.2 Why `pytest`

Python has a standard-library testing framework, `unittest`, modeled on the older "xUnit" family. It works. But the Python community has, with near-unanimity, settled on a third-party framework, `pytest`, and a staff engineer should know why — because the *why* is itself a small lesson in good tool design.

`pytest` won on *friction*. A `unittest` test must be a method on a class that inherits from a framework base class, and assertions are made through named methods — `self.assertEqual(a, b)`, `self.assertTrue(x)`. A `pytest` test is just a *function* whose name starts with `test_`, and assertions are just the plain `assert` statement — `assert a == b`. The reduction in ceremony is real: a test is the smallest possible thing, a function with an `assert`. And `pytest` does something clever with that plain `assert` — it rewrites the assertion under the hood so that when `assert a == b` *fails*, the failure message shows you the actual values of `a` and `b`, the introspection that `unittest`'s named-method assertions provided explicitly and that plain `assert` would not give you on its own. You get the low ceremony of a bare `assert` *and* the rich failure output.

`pytest` also has a *fixture* system (Section 16.4) that is genuinely better than `unittest`'s setup/teardown methods, and a large *plugin ecosystem* (built on the entry-point mechanism of Chapter 9 — this is that mechanism's most visible real-world use). The verdict is simple and the chapter states it plainly: **for new Python projects, use `pytest`.** You should be able to *read* `unittest`, because plenty of existing code uses it, but `pytest` is the default, and the friction difference is large enough that it genuinely affects how many tests get written.

## 16.3 The Test Pyramid and the Honest Definition of "Unit"

Tests come in kinds, distinguished by *how much of the system they exercise*, and there is a well-known model for how to balance the kinds: the **test pyramid**.

At the base of the pyramid, broad, are **unit tests** — tests of a small, isolated piece of code, a single function or class, with its dependencies replaced by fakes or simply absent. They are *fast* (no I/O, no database, no network — they run in milliseconds) and *focused* (a failure points at a specific small thing). Because they are fast and focused you can have *thousands* of them and run them constantly.

In the middle are **integration tests** — tests of several pieces working *together*, often with *real* dependencies: a real database, a real call across a module boundary. They are slower (real I/O) and broader (a failure means "something in this interaction broke," less precisely located). You have fewer of them.

At the narrow top are **end-to-end tests** — tests of the *whole system*, exercised the way a user would, through its real external interface. They are the slowest and the broadest, and they are also the most *fragile* — they break for many reasons, including reasons that are not real bugs. You have *few* of them.

The pyramid's *shape* is its lesson: **many fast unit tests at the base, fewer integration tests in the middle, few end-to-end tests at the top.** The shape matters because it determines whether the suite is *usable*. A suite that is mostly fast unit tests runs in seconds, so engineers run it constantly, so bugs are caught immediately. A suite that is mostly slow end-to-end tests — the "inverted pyramid" or "ice-cream cone" anti-pattern — takes many minutes to run, so engineers run it rarely, so bugs are caught late; and it is flaky, so its failures are distrusted; and a distrusted suite is a useless suite. The pyramid is not aesthetic; it is the shape that keeps the suite fast enough and reliable enough to actually be used.

One honesty the chapter insists on: **be honest about what is a "unit" test.** A test that hits a real database is *not* a unit test, regardless of what the file is named — it is an integration test, with an integration test's speed and an integration test's failure modes. Mislabeling it does not change what it is; it just means your "unit test suite" is secretly slow and you will be puzzled why. Call tests what they are, and the pyramid stays meaningful.

## 16.4 Fixtures: Dependency Injection for Tests

`pytest`'s **fixtures** are its best feature, and the right way to understand them is the one the chapter's title for this section gives away: **a fixture is dependency injection (Chapter 12) for tests.**

A fixture is a function that *produces something a test needs* — a configured object, a database connection, a sample piece of data, a temporary directory. A test that needs that thing simply declares it as a *parameter*, and `pytest` sees the parameter name, finds the matching fixture, runs it, and *injects* the result into the test. The test does not construct its dependencies; it declares them and receives them — exactly the dependency-injection principle of Chapter 12, applied to tests.

```python
import pytest

@pytest.fixture
def sample_user():
    return User(name="Ada", email="ada@example.com")

def test_user_greeting(sample_user):     # declares the fixture as a parameter
    assert sample_user.greeting() == "Hello, Ada"
```

Fixtures do two things that make them powerful. First, they handle **setup and teardown** cleanly. A fixture can `yield` its value (the generator-based pattern of Chapter 7) — the code before the `yield` is *setup*, run before the test; the code after the `yield` is *teardown*, run after the test, even if the test failed. A database fixture sets up a connection before `yield` and closes it after; a temporary-directory fixture creates the directory before and deletes it after. Setup and teardown are paired, in one place, and the teardown is guaranteed.

Second, fixtures have **scope**, and scope is the lever that keeps an integration suite fast. A `function`-scoped fixture (the default) runs *fresh for every test*. A `session`-scoped fixture runs *once for the entire test run*. This matters enormously for expensive setup: spinning up a database connection is slow, and doing it fresh for every one of a thousand tests is unbearable. The standard pattern is a `session`-scoped fixture for the expensive, *shared* thing (the database connection) and a `function`-scoped fixture for the *per-test isolation* — typically, each test runs inside a database *transaction* that is *rolled back* at the end, so every test gets a clean database state without the cost of rebuilding the database. The expensive connection is created once; the cheap per-test isolation is fresh each time. That pattern — expensive resource at session scope, isolation at function scope — is what makes an integration suite both fast *and* reliably isolated, and it is the single most useful fixture technique to know.

## 16.5 Parametrization

A frequent need: run *the same test logic* against *many different inputs*. The naive approach — copy the test, change the input, repeat — produces a dozen near-identical test functions, which is exactly the duplication every other chapter of this book has warned against.

`pytest`'s **parametrization** is the answer. `@pytest.mark.parametrize` lets you write the test logic *once* and supply a *list of input/expected pairs*; `pytest` runs the test once per pair, and — importantly — reports each as a *separate* test, so a failure tells you exactly *which* input broke.

```python
@pytest.mark.parametrize("value, expected", [
    (0, "zero"),
    (1, "one"),
    (-1, "negative"),
    (1000, "large"),
])
def test_classify(value, expected):
    assert classify(value) == expected
```

One test function, four cases, four separately-reported results. Parametrization is how you achieve thorough coverage of a function's input space — the edge cases, the boundary values, the empty and the huge — without duplicating the test body. When you find yourself about to copy a test to change its input, parametrize instead.

## 16.6 Mocking: The Hardest Judgment in Testing

**Mocking** — replacing a real dependency with a fake, controllable stand-in for the duration of a test — is the part of testing where engineers most often go wrong, and the wrongness is a *judgment* failure, not a mechanical one. The mechanics (Python's `unittest.mock`, the `pytest-mock` plugin, `Mock` and `patch`) are simple. The judgment — *what* to mock and, far more importantly, *what not to* — is hard, and a staff engineer must have it.

Start with *why* you mock. You mock a dependency to make a test *fast*, *deterministic*, and *isolated* — you replace the real payment provider so the test does not make a real payment, you replace the real clock so a time-dependent test is deterministic, you replace a real external service so the test does not depend on that service being up. These are legitimate, necessary uses.

Now the judgment, in two principles.

**The first principle: mock at the boundary, not in the middle.** The right place to mock is the *edge* of your system — the seam where your code talks to something *external and outside your control*: the third-party API, the external service, the system clock, the filesystem. Those are the things you legitimately want to replace. The *wrong* place to mock is *inside* your own system — mocking your own classes, your own functions, the interactions between your own modules. When you mock your own internals, you produce a test that is *coupled to the implementation*: it verifies that function A called function B with certain arguments — and now, if you *refactor* (change how A and B collaborate internally, without changing the externally-observable behavior at all), the test *breaks*, even though nothing a user could see has changed. The test is testing the *wiring* rather than the *behavior*, and a test that breaks on a behavior-preserving refactor is worse than no test — it is a tax on exactly the kind of change this book keeps urging you to make. Mock the boundary; let your own internals be exercised for real.

**The second principle: prefer fakes to mocks.** A *mock* typically verifies *interactions* — "was this method called, with these arguments." A *fake* is a *real, working, simplified implementation* — an in-memory version of the thing. An in-memory dictionary-backed implementation of a `StoragePort` (Chapter 12's term) is a fake: it genuinely stores and retrieves, it just does so in a dict instead of a database. A fake, used in a test, lets the test exercise *real behavior* — store something, then retrieve it, and assert you got it back — rather than asserting that specific methods were called in a specific way. Behavior-based tests with fakes survive refactoring; interaction-based tests with mocks often do not. This is also where Chapter 12 pays off again: a system built with dependency injection and `Protocol` ports is *easy* to test with fakes, because you can write an in-memory adapter satisfying the same port and inject it — the architecture and the testability are the same property, which is exactly what Chapter 12's War Story showed.

The summary judgment: mock sparingly, mock only at the true external boundary, and prefer a real in-memory fake to an interaction-verifying mock wherever you can. Over-mocking — mocking deep inside your own code — produces a test suite that is green, gives a feeling of safety, and yet breaks constantly under refactoring while failing to catch real bugs. That is the worst of all outcomes, and avoiding it is a staff-level judgment.

## 16.7 Testing Async Code

Async code (Chapters 13–14) needs testing too, and there is one mechanical thing and one judgment thing to know.

The mechanical thing: a coroutine cannot be tested by an ordinary test function, because — recall Chapter 14 — calling a coroutine just produces an inert coroutine object; it must be *driven by an event loop*. The `pytest` plugin `pytest-asyncio` handles this: it lets you write `async def test_...` test functions and runs them on an event loop. You also need an *async-aware mock* — `unittest.mock` provides `AsyncMock` — because a mock standing in for an `async` function must itself be awaitable; an ordinary `Mock` is not, and using one to replace a coroutine produces a confusing failure.

The judgment thing is the more important: **the parts of async code most worth testing are the parts the model makes hazardous** — and Chapter 14 named them. Test that *cancellation* is handled correctly — that a cancelled task actually stops, that `CancelledError` is not swallowed, that graceful shutdown works. Test the *timeout* paths — that an operation that exceeds its deadline is actually cancelled. Test *concurrent* behavior where it matters. These async-specific hazards are exactly the things that do not show up in a casual happy-path test and exactly the things that cause the production incidents of Chapter 14's War Story, so they are exactly the things a thorough test suite must cover.

## 16.8 Property-Based Testing

Most testing is *example-based*: you, the engineer, think of specific inputs and write a test for each. This works, but it has a structural weakness — you only test the cases *you thought of*, and the bugs that survive to production are very often in the cases you *did not* think of, the edge case that did not occur to you.

**Property-based testing** attacks that weakness directly, and the tool in Python is **`hypothesis`**. The idea: instead of writing specific examples, you state a *property* — a general truth that should hold for *all* valid inputs — and `hypothesis` *generates* large numbers of inputs, including weird and extreme ones, trying to find an input that *violates* the property.

The classic example is a *round-trip* property: if you serialize a value and then deserialize it, you should get the original value back, *for any value*. You do not write examples; you state "for any value, deserialize(serialize(x)) == x," and `hypothesis` generates hundreds of values — empty ones, huge ones, ones with strange characters, edge-case numbers — and tries to break it. Other properties: "sorting a list produces a list of the same length with the same elements"; "this function never raises for any valid input"; "this operation is idempotent."

`hypothesis` has one more feature that makes it genuinely powerful: **shrinking**. When it finds an input that violates the property, it does not just hand you that input — which might be a large, messy, randomly-generated value that is hard to reason about. It *shrinks* the failing input, automatically searching for the *smallest, simplest* input that still triggers the failure. Instead of "your function failed on this 500-element list of random data," you get "your function failed on the list `[0]`" — a minimal, comprehensible reproduction that points almost directly at the bug.

Property-based testing does not replace example-based testing — specific examples remain valuable, especially for specific known cases and regressions. It *complements* it, and it is especially powerful for exactly the kinds of code where edge cases hide: parsers, serializers, encoders, anything with a clean general property like a round-trip or an invariant. A staff engineer knows `hypothesis` and reaches for it when the code under test has a property worth stating.

## 16.9 Coverage: A Tool That Can Lie

**Code coverage** measures *which lines of your code were executed by your tests* — typically reported as a percentage, "85% coverage." It is a useful tool and it is a tool that *lies*, and a staff engineer must hold both halves of that.

It is useful because *low* coverage is a genuine, reliable signal: a module at 20% coverage has large stretches of code that *no test exercises at all*, and that is a real, actionable gap. Coverage is good at telling you where you have tested *nothing*.

It lies because *high* coverage does not mean *good* tests. Coverage measures whether a line was *executed*, not whether its behavior was *verified*. A test can *run* a function — executing every line, achieving 100% coverage of it — while *asserting nothing meaningful* about what the function actually did. You can reach 100% coverage with tests that would not catch a serious bug. Coverage tells you a line *ran*; it does not tell you the line *works*, because it cannot see whether your assertions actually check anything. So "we have 95% coverage" is *not* the same claim as "our tests are good," and a team that chases the coverage number — writing assertion-light tests purely to make the percentage climb — has optimized the metric and not the thing the metric was a proxy for. That is Goodhart's law, and it is a real and common failure.

The staff-level use of coverage, then: **use it to find the *untested* code — the genuine gaps — and ignore it as a measure of test *quality*.** Treat a low number as a real signal to investigate; treat a high number as necessary-but-not-sufficient, never as proof. The quality of a test is in whether its assertions would actually catch a real bug, and no coverage tool can measure that. (For teams that want a real measure of test quality, there is *mutation testing* — tools that deliberately introduce small bugs into your code and check whether your tests *catch* them; a test suite that does not notice an injected bug has just been proven weak regardless of its coverage number. Mutation testing is slower and less common, but it measures the thing coverage only pretends to.)

## 16.10 The Unforgivable Sin: The Flaky Test

The chapter ends with the one thing in testing a staff engineer must be *absolute* about: **a flaky test — a test that sometimes passes and sometimes fails without any change to the code — is a serious problem and must never be tolerated.**

The reason flakiness is unforgivable is not that a flaky test is annoying, though it is. The reason is that **flakiness destroys the value of the entire suite.** A test suite is useful only because a failure *means something* — because a red result tells the team "you broke something, stop and fix it." A flaky test produces *red results that mean nothing* — a failure that is just the flake, not a real bug. And once the suite produces failures that mean nothing, the team learns to *re-run it* and *ignore the red*. And a team that ignores red has lost the suite entirely — because the *real* failure, the genuine bug, now also gets re-run and ignored, indistinguishable from the noise. One tolerated flaky test does not cost you one test; it teaches the team to distrust *all* of them, and a distrusted suite is a useless suite no matter how many tests it contains.

So the rule is absolute and a staff engineer enforces it: a flaky test is found, and it is *fixed* or it is *removed* — it is never left in the suite to "mostly pass." Fixing it means finding the *cause*, and the cause is almost always one of a small set: a hidden dependence on *timing* (a test that assumes an operation finishes within some duration), a hidden dependence on *test order* (a test that only passes if another test ran first and left some state), a hidden dependence on *shared state* not properly isolated, a real *race condition* in the code under test (in which case the flaky test has found a genuine bug and is doing its job), or a dependence on something *external and unreliable* (a test that calls a real network service). Each of these has a fix — proper waiting instead of timing assumptions, proper isolation, fixing the race, mocking the external boundary. The fix is sometimes work. The alternative — tolerating the flake — is the slow death of the suite, and that is not an acceptable trade. Flakiness is not a minor quality issue; it is a direct threat to the safety system that makes all of this book's advice about *changing* code safe to follow.

> ### War Story — The Suite Nobody Trusted
>
> A team had a large test suite — thousands of tests, built up over years, genuinely a lot of work. It also had, accumulated over those same years, a few dozen tests that were *flaky*: they failed intermittently, for reasons no one had ever fully chased down. Each individual flake had seemed minor at the time — a test that failed maybe one run in twenty — and the team's informal practice had become: when CI goes red, glance at *which* tests failed, and if it is "one of the known flaky ones," just hit re-run.
>
> This worked, in the sense that the team kept shipping. But it had quietly destroyed the suite. Because there were always *some* flaky failures, *every* red CI run looked, at first glance, like "probably just the flakes" — and so the team's trained reflex on a red run was *re-run it*, not *investigate it*. And one day a *real* regression — a genuine bug, introduced by a genuine change — produced a genuine test failure. And the team did what the flakes had trained them to do: glanced at the red, assumed flake, hit re-run. The re-run, by chance, happened to pass the flaky tests *and* — because the regression was in a test that was itself slightly non-deterministic — happened to pass that one too. Green. The change shipped. The regression reached production, where it did real damage, and the post-incident review found the uncomfortable truth: *the test suite had caught the bug.* A test had genuinely failed on the genuine regression. The failure had been *ignored*, because years of tolerated flakiness had trained the entire team that red meant "re-run," not "stop."
>
> The fix was a deliberate, unglamorous campaign: every flaky test was hunted down and *either fixed or deleted* — no exceptions, no "mostly passes" left in the suite — and a hard rule was adopted that a newly-flaky test was a stop-the-line problem, fixed before anything else. It took weeks. Afterward, a red CI run *meant something* again, because there was no longer any noise for a real failure to hide in, and the team's reflex on red went back to what it must be: stop and look.
>
> The lesson is the chapter's closing rule, learned at the cost of a production incident. A flaky test does not cost you one test. It teaches the team to ignore red, and a team that ignores red has lost the entire suite — every test in it — because the real failure is now indistinguishable from the noise. Flakiness is not tolerable. It is not a minor quality issue to get to later. It is a direct attack on the trustworthiness of the safety net, and the safety net is the thing that makes every other kind of confident change possible. Find the flake; fix it or kill it; never let it live.

> ### The Level Line — Testing
>
> **A senior engineer** writes tests with `pytest`, uses fixtures and parametrization, can mock a dependency, knows the test pyramid, and uses coverage.
>
> **A staff engineer** treats tests as the executable specification and the safety net for change; keeps the pyramid's shape and is honest about what "unit" means; uses the session-resource / function-isolation fixture pattern; has the *mocking judgment* — boundary not middle, fakes over mocks — and knows over-mocking produces refactor-fragile tests; tests async code's hazardous paths; uses `hypothesis` where a property exists; treats coverage as a gap-finder not a quality measure; and is absolute about flakiness.
>
> **A principal engineer** establishes the testing culture and standards for a team — the pyramid balance, the mocking conventions, the flakiness rule enforced as stop-the-line — ensures the suite stays fast and trustworthy enough to actually be used, and teaches that tests are not a chore appended to engineering but the safety system that makes continuous, confident engineering possible at all.

## 16.11 At the Interview Table

Testing questions probe judgment far more than mechanics — interviewers want to know *how you think about* tests, because that reveals whether you have operated a real suite over time.

**The question: "How do you decide what to test, and at what level?"**

"I think in terms of the test pyramid. The base is many fast, isolated unit tests — a single function or class, dependencies faked or absent, running in milliseconds — and that's the bulk of the suite, because fast tests get run constantly. The middle is fewer integration tests, with real dependencies like a real database, exercising pieces together. The top is a few end-to-end tests through the real interface. The shape matters: a suite that's mostly fast unit tests runs in seconds and stays trusted; an inverted pyramid of slow end-to-end tests runs in minutes, gets run rarely, is flaky, and gets distrusted. As for *what* to test — the public behavior, not the implementation; the edge cases and failure paths, not just the happy path; and I'm honest about labels: a test that hits a real database is an integration test no matter what the file is called."

**The question: "What should you mock, and what should you not?"**

This is the judgment question and the strong answer is precise about both halves. "Mock at the *boundary* — the seam where my code meets something external and outside my control: a third-party API, the system clock, the filesystem, an external service. That's legitimate; it makes the test fast, deterministic, and isolated. Do *not* mock inside my own system — my own classes, my own modules' interactions — because that couples the test to the implementation: it verifies that A called B a certain way, so a behavior-preserving refactor breaks the test even though nothing observable changed. A test that breaks on a clean refactor is worse than no test. And where I can, I prefer a *fake* — a real in-memory implementation — over an interaction-verifying mock, because a fake lets the test check real behavior and survives refactoring. A system built with dependency injection and `Protocol` ports makes this easy: I inject an in-memory adapter."

**The question: "What's wrong with chasing high code coverage?"**

"Coverage measures which lines *ran* during the tests, not whether their behavior was *verified*. You can hit 100% coverage with tests that execute every line but assert nothing meaningful — running a function isn't testing it. So high coverage doesn't mean good tests; it's necessary but not sufficient. Chasing the number leads to assertion-light tests written to move the percentage — optimizing the metric instead of the thing it was a proxy for. I use coverage the other way: a *low* number is a real signal of genuinely untested code worth investigating; a high number I treat as a floor, not proof. If I want a real measure of test quality, that's mutation testing — inject small bugs and see if the tests catch them."

**The question: "A test in your suite fails intermittently. What do you do?"**

The answer must convey that this is *not minor*. "A flaky test is a serious problem, because it destroys trust in the whole suite — once some failures are 'just flakes,' the team learns to re-run and ignore red, and then a *real* failure gets ignored too, because it's indistinguishable from the noise. So a flaky test is not something to live with — it gets fixed or removed, never left to 'mostly pass.' I'd find the cause, which is almost always one of: a timing assumption, a test-order or shared-state dependence, a real race condition in the code (in which case the flaky test found a genuine bug), or a dependence on an unreliable external service. Each has a real fix. And I'd treat a newly-flaky test as a stop-the-line issue."

**The red flags.** Treating tests as a chore done after the real work. Over-mocking — mocking one's own internals — with no sense that it produces refactor-fragile tests. Equating high coverage with good tests. Inverting the pyramid. And, the most serious, being casual about flaky tests — "we just re-run those" — which signals an engineer who has not understood that flakiness is a threat to the whole suite.

## 16.12 The Forge

**Drill 16.1.** Take an untested function and write a thorough `pytest` test for it: cover the happy path, the edge cases, the boundary values, and the error cases. Use `@pytest.mark.parametrize` so the many input cases share one test body.

**Drill 16.2.** Write a fixture that sets up and tears down a resource using the `yield` pattern. Then make it `session`-scoped and add a separate `function`-scoped fixture that provides per-test isolation. Explain, in writing, why this two-fixture pattern keeps an integration suite both fast and isolated.

**Build 16.1.** Take a piece of code that talks to an external service. Write tests for it two ways: once mocking the external service at the boundary, and once with a real in-memory *fake* implementation behind a `Protocol` port. Then make a behavior-preserving change to the code's internals and observe which set of tests survives. Write up what the exercise demonstrated about mocking judgment.

**Build 16.2.** Add property-based tests with `hypothesis` to a piece of code that has a clean property — a serializer/deserializer with a round-trip property, or a function with an invariant. State the property, let `hypothesis` generate inputs, and — if it finds a failure — observe the shrinking produce a minimal reproduction. Compare what `hypothesis` found against the example-based tests you would have written by hand.

**Investigate 16.1.** Take a real test suite (yours or an open-source one) and measure its coverage. Then look *past* the number: find a file with high coverage but weak assertions — code that runs in tests but is not really verified — and a file with genuinely low coverage. Write up the difference between what coverage claims and what the tests actually check.

**Investigate 16.2.** Run a mutation-testing tool against an existing test suite. See how many injected bugs the suite catches and how many survive undetected. For a few surviving mutations, work out why the tests missed them. Write up what mutation testing revealed that coverage did not.

**Design 16.1.** You are the staff engineer joining a team whose test suite is in trouble: it is slow (mostly end-to-end tests — an inverted pyramid), it is heavily over-mocked so it breaks on every refactor, it has a couple dozen tolerated flaky tests, and the team has started ignoring red CI runs. Write a one-to-two-page plan to rehabilitate it: how you rebalance toward the pyramid; how you address the over-mocking; how you eliminate the flakiness and restore trust in red; how you bring the team along (this changes their daily workflow); and how you would know the suite had become trustworthy again. This is the staff-level exercise — testing as a team culture and a safety system, not as a set of files.

---

# Part IV Capstone — A Profiled, Concurrent, Fully-Tested Service Component

Part IV has been about making Python *fast and parallel* — and about the judgment of when to, and the testing that makes all change safe. The capstone integrates all four chapters into one exercise that is, deliberately, the shape of real staff-level work: take something that is slow, sequential, and untested, and make it fast, correctly concurrent, and trustworthy — and *prove*, with measurement, that you did.

**The brief.** You are given (or you write) a service component that is slow, sequential, and has little or no test coverage — for example, a batch processor that fetches data from several external sources, performs a non-trivial computation on it, and writes results. It is too slow, and it cannot be changed with confidence because it is untested.

**What it must demonstrate, by chapter:**

From **Chapter 13** — analyze the workload: which parts are I/O-bound, which are CPU-bound. Choose the right concurrency model for each part, with the decision procedure applied explicitly and the reasoning written down. Use `concurrent.futures` so the model choice is isolated and the threading-versus-processes decision is one line.

From **Chapter 14** — if any part of the system is I/O-bound at meaningful concurrency, an async implementation of that part is appropriate: built so the event loop is never blocked, concurrency is bounded, timeouts are universal, and the cancellation contract is respected so shutdown is clean.

From **Chapter 15** — *measure first*. Profile the original before changing anything; let the profile, not intuition, identify the bottlenecks. Work the optimization hierarchy top-down — fix any algorithmic or data-structure problem and eliminate unnecessary work *before* reaching for concurrency or native code. Profile memory too. After every change, measure again. The deliverable includes a *before-and-after performance report* with real numbers.

From **Chapter 16** — *before* changing the code, write a test suite for it — because the existing behavior must be captured so that you can prove your changes preserved it. The suite should follow the pyramid, mock only at the boundaries, prefer fakes (the external sources are an obvious place for in-memory fakes behind `Protocol` ports), use parametrization, and — since the result is concurrent — test the concurrency's hazardous paths. The suite must be fast and free of flakiness.

**The deliverables:**

1. The rehabilitated component — faster, correctly concurrent, fully tested.
2. A test suite written *first* (capturing the original behavior) and maintained through the changes (proving the behavior was preserved).
3. A *performance report* — the before profile, each change and the level of the hierarchy it came from, the measured effect of each, and the total — with real numbers, not estimates.
4. A short *design note* defending the concurrency-model choices: which model for which part, and why, per Chapter 13's decision procedure.

**On the interview.** This capstone is the shape of a very common senior/staff interview prompt — "here is some slow code; make it faster" — and the point of building it is that the *process* it forces is exactly the process the interview is testing. The interviewer asking that question is not looking for a fast typist who reaches for threads; they are looking for the engineer who profiles first, who fixes the algorithm before reaching for concurrency, who chooses the concurrency model with reasons, who writes tests so the change is safe, and who can show the numbers. Build the capstone as a rehearsal of that process, because the process *is* the staff-level skill, and the rest of the book — now turning, in Part V, to whole systems — assumes you have it.

---

[← Chapter 15: Performance Engineering](15-performance-engineering.md) · [Home](README.md) · [Chapter 17: Network and Web →](17-network-and-web.md)
