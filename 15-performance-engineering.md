# Chapter 15 — Performance Engineering

*Part IV — Concurrency and Performance*

[← Chapter 14: Asyncio in Depth](14-asyncio-in-depth.md) · [Home](README.md) · [Chapter 16: Testing as Engineering →](16-testing-as-engineering.md)

---

## 15.1 The Question Behind the Chapter

There is a sentence every engineer has heard, usually as half of itself: "premature optimization is the root of all evil." It is Donald Knuth's, and the truncation has done real damage, because the full sentence says something more precise and more useful: *"We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%."* The full quote is not an injunction against caring about performance. It is an injunction to find the *critical 3%* and to ignore the other 97% — and finding the 3% is a skill, a discipline, with a method.

This chapter teaches that discipline. Its thesis, stated plainly because it is the most violated principle in the field: **measure first; optimize only what measurement proves is slow; optimize the algorithm before the code; and stop when it is fast enough.** Almost every performance mistake an engineer makes is a violation of one of those four clauses — optimizing by guess, optimizing code that was never slow, micro-optimizing past an O(n²) algorithm, or polishing code that was already fast enough to no benefit. The chapter is the antidote: the profiling tools that let you *measure* rather than guess, the optimization hierarchy that tells you *where* the wins actually are, and the judgment of *when to stop*.

A staff engineer is not the person who knows the most micro-optimizations. A staff engineer is the person who, faced with "this is slow," reaches for a profiler instead of an opinion — and who, faced with "make this faster," asks "how fast does it need to be, and have we proven this is the slow part?" That instinct is the chapter.

## 15.2 The Discipline: Measure, Don't Guess

The single most important sentence in this chapter: **you cannot know what is slow by reading the code. You must measure.**

Engineers resist this, because reading code and forming a theory feels like engineering, and running a profiler feels like a chore. But human intuition about performance is *reliably wrong*, and it is wrong in a specific, well-documented way: the slow part of a program is almost never where you think it is. The function that *looks* expensive — the one with the nested loops, the one with the clever algorithm — is often not the bottleneck. The bottleneck is often something mundane and invisible: a function called far more times than anyone realized, a logging call in a hot path, a database query issued inside a loop, an accidental O(n²) hiding behind innocent-looking code. You will not find these by reading. You will find them by measuring, and you will be surprised by what you find *almost every time* — and being surprised almost every time is exactly the evidence that intuition cannot be trusted and measurement must be.

So the discipline has a fixed first step, and it is never skipped: **before optimizing anything, profile, and let the profiler tell you where the time actually goes.** Then optimize *that* — the thing the profile identified, not the thing you suspected. Then *measure again*, to confirm the optimization actually helped (optimizations sometimes do not, or help less than expected, or help one path while hurting another — only measurement tells you). The loop is: measure, change the thing measurement pointed at, measure again. Guessing is not a faster version of this loop; it is a different activity that happens to resemble it and usually wastes the time it appears to save.

There is a corollary about *when* to do this. The 97%/3% split means most code should *not* be optimized at all — it should be written clearly, correctly, and simply, and left alone, because it is not on a hot path and its performance is irrelevant. Optimization effort is spent only on code that *measurement* has shown to be both slow *and* on a path that matters. Optimizing the 97% is not just wasted effort; it is *negative*, because optimized code is almost always *less readable* than clear code, and you have traded readability — which you needed — for speed you did not. Clear-and-correct is the default; fast is the exception you earn with a profile.

## 15.3 CPU Profiling

To find where CPU time goes, you use a *profiler*. There are several, suited to different situations, and a staff engineer knows the toolkit.

**`cProfile`** is the standard-library CPU profiler. It is *deterministic* — it instruments the program and records every function call — and it produces, for every function, how many times it was called and how much time was spent in it. Two numbers in its output are essential and constantly confused, so learn them precisely. **Total time** (cProfile calls it "cumulative") for a function is the time spent in that function *including everything it called* — all its callees, recursively. **Self time** (cProfile's "tottime") is the time spent *in that function's own code*, excluding its callees. The distinction is the heart of profile-reading: a function with huge *total* time but tiny *self* time is not itself slow — it is just calling something slow, and you must follow the call chain downward to find the real culprit. A function with large *self* time *is* the culprit — the time is being spent in its own code. When you read a profile, you sort by self time to find the functions to optimize, and you use total time to understand the call structure.

`cProfile`'s cost is overhead — instrumenting every call slows the program down, sometimes substantially, which can distort the picture for very call-heavy code. Its output is also dense; tools like `snakeviz` visualize it.

**`py-spy`** is a *different kind* of profiler and a genuinely important tool. It is a *sampling* profiler — instead of instrumenting every call, it periodically peeks at what the program is doing and builds a statistical picture from the samples. This has two large advantages. It has *very low overhead*, because sampling is cheap. And — the killer feature — **it can attach to an already-running process without any code changes and without restarting it.** This means `py-spy` can profile a *production* service, live, as it serves real traffic, which `cProfile` (which must be wrapped around the code) cannot. When a production service is slow *right now* and you need to know why *right now*, `py-spy` is the tool: attach it, let it sample, get a picture of where the live process is spending its time.

**`pyinstrument`** is a sampling profiler oriented toward application code, with output that presents the call tree in a readable, intuitive form — often the most pleasant profiler for "why is this request slow" investigations.

**`timeit`** is not a profiler but a *micro-benchmark* tool — it runs a small snippet many times and reports how long it takes, with the statistical care (multiple runs, warm-up) that ad-hoc timing lacks. It is the right tool for the narrow question "which of these two small implementations is faster," and the wrong tool for "where is my program slow" (that is a profiler's job).

The workflow: use a profiler (`cProfile`, `py-spy`, or `pyinstrument`) to find *where* the time goes; use `timeit` to compare specific small alternatives once you know which code to focus on.

## 15.4 Reading a Profile and the Surprises It Holds

A profile is only useful if you can read it, and reading it well means knowing what to look for — and the things to look for are, characteristically, the things intuition missed.

**The function called far more often than expected.** A profile shows call counts. Very often the bottleneck is not a slow function but a *fast* function called an enormous number of times — a cheap operation, individually trivial, invoked in an inner loop or recursively, totalling to most of the runtime. You would never suspect it from reading the code, because each individual call is obviously cheap. The profile's call count reveals it.

**The accidental O(n²).** A profile can reveal that runtime grows quadratically with input — a function whose total time balloons disproportionately as data grows. The cause is usually an O(n) operation nested inside a loop: a membership test against a list inside a loop over another list (Chapter 4's exact lesson), a `list.pop(0)` in a loop, a string built by repeated concatenation. The profile points at the function; recognizing the quadratic shape and its cause is the chapter-4 knowledge applied.

**The I/O hiding in the compute.** A profile can show that a function which looks like pure computation is actually spending its time *waiting* — a database query, a network call, a file read, often issued inside a loop (the N+1 query problem, which Chapter 18 names). The fix is not to optimize the computation; it is to fix the I/O — batch the queries, cache the result, move the call out of the loop.

**The surprising hot spot.** And, very often, simply something *nobody expected* — a serialization step, a logging call, a regular expression, a deep-copy — quietly consuming a large fraction of the time. This is the general case of "the profile surprises you," and it is the entire argument for profiling: you found a thing you would never have found by reading, and now you can fix the *real* problem instead of the imagined one.

The skill of reading a profile is, in the end, the skill of letting it overturn your theory. You came in with a guess about the slow part; the profile shows you the actual slow part; the discipline is to *believe the profile and abandon the guess*. An engineer who reads a profile and then optimizes the thing they suspected anyway has wasted the profile.

## 15.5 The Optimization Hierarchy

When measurement *has* identified a genuine bottleneck, there is an *order* in which to attack it — a hierarchy from the highest-leverage changes to the lowest. Working the hierarchy top-down is the difference between a large speedup for modest effort and a tiny speedup for large effort.

**Level one — the algorithm and the data structure.** This is, by an enormous margin, the highest-leverage level, and it is Chapter 4's lesson cashed in here. A better *algorithm* — turning an O(n²) into an O(n log n), turning a repeated linear scan into a hash lookup — produces speedups that *no amount of code-level tuning can match*, because it changes how the runtime *scales*. A membership test changed from a list (O(n)) to a set (O(1)) is not 20% faster; it is asymptotically faster, a different curve. An accidental quadratic fixed is not tuned; it is *cured*. Before touching anything else, ask: is the algorithm right? Is the data structure right? Is there repeated work that could be done once? These questions, answered well, are where the real speedups live, and a staff engineer always asks them *first*.

**Level two — the structure of the work.** Above the level of individual code but below whole algorithms: are you doing work you do not need to do at all? Computing the same thing repeatedly when it could be computed once and *cached* (Chapter 5's `lru_cache`, with its memory caveat)? Doing work *eagerly* that could be done *lazily* and perhaps skipped entirely (Chapter 7's generators)? Issuing many small I/O operations that could be *batched* into one? Loading data you do not use? This level is about *eliminating* work rather than speeding it up, and eliminated work is infinitely fast.

**Level three — Python-level code tuning.** Now, and only now, the code itself. Python has real, known performance characteristics that can be exploited: built-in functions and operations implemented in C are faster than equivalent hand-written Python loops; comprehensions are faster than equivalent `append` loops (Chapter 1's bytecode showed why); local variable access is faster than global; some idioms are simply faster than others. These tunings are *real* but they are *modest* — they shave percentages, not orders of magnitude — and they often cost readability. They are worth doing in a genuine, measured, narrow hot path; they are not worth doing anywhere else, and scattering them through ordinary code is the readability-for-nothing trade of Section 15.2.

**Level four — drop out of Python.** When the bottleneck is genuinely CPU-bound, genuinely in the hot path, and Levels one through three have been exhausted, the remaining move is to take that specific hot piece *out of pure Python*. The options form their own small hierarchy. **NumPy** — for numerical and array work, NumPy's vectorized C operations are dramatically faster than Python loops, and "vectorize this with NumPy" is often the entire answer for numerical hot spots. **Numba** — a just-in-time compiler that compiles annotated numerical Python functions to fast machine code, often with a single decorator. **Cython** — compiles Python-like code to C, allowing gradual typing-for-speed of an existing module. And, for a genuinely demanding hot path, a **native extension in Rust** via PyO3, or in C — the modern Python performance ecosystem (Pydantic's core, the `ruff` and `uv` tools, much of the fast tooling) increasingly does its hot work in Rust and exposes it to Python. Dropping out of Python is powerful and it is *last*, because it is the most effort and the most complexity, and reaching for it before exhausting the higher levels is a sign of an engineer optimizing by reflex rather than by hierarchy.

The hierarchy is the chapter's central practical tool. Faced with a measured bottleneck, walk it from the top: fix the algorithm, eliminate unnecessary work, tune the code if a real hot path warrants it, and drop to native code only as a last resort for a genuine CPU-bound core. Working top-down gets the big wins first; working bottom-up — reaching straight for code tuning or native code — does enormous effort for small returns while the O(n²) at Level one sits unfixed.

## 15.6 Memory and Its Profilers

Performance is not only speed; it is also *memory*, and memory has its own profiling tools and its own characteristic failure mode.

The failure mode is the **memory leak** — in Python, almost never a true leak in the C sense (Python manages memory; Chapter 2), but rather *unintended retention*: objects that are no longer needed but are still *referenced* by something, so reference counting and the cycle collector cannot reclaim them, and memory grows without bound. The usual culprits are exactly the ones Chapter 2 and Chapter 5 flagged — an unbounded cache or `lru_cache` that only ever grows, a module-level or class-level collection that accumulates across requests, a registry of callbacks that is never cleaned up, the `lru_cache`-on-a-method that pins every instance. In a long-running service, unintended retention is a serious problem: memory climbs over hours or days until the process is killed by the operating system or its container's memory limit.

To find it, you measure — the same discipline as for CPU. **`tracemalloc`** is the standard-library memory profiler: it tracks where objects are allocated, and — critically — it can take *snapshots* and *compare* them. The technique for hunting a leak is to take a `tracemalloc` snapshot, let the program run a while (serve more requests, process more data), take another snapshot, and *diff* them: the diff shows which allocation sites *grew* between the snapshots, and a site that grows steadily across snapshots is your leak. **`memray`** is a more powerful, more recent memory profiler — it tracks all allocations, including those in C extensions that `tracemalloc` cannot see, and produces rich visualizations; for a serious memory investigation it is the heavier, more capable tool. And `sys.getsizeof`, from Chapter 2, answers the narrow question of how large an individual object is.

The discipline is identical to CPU profiling and bears the identical warning: do not *guess* where memory is being retained — *measure* it, with snapshots and diffs, and let the tool show you the growing allocation site. And the chapter's recurring theme applies once more: in a long-running service, set up memory measurement as a *habit* — watch the process's memory over time, take periodic snapshots — so that unintended retention is caught early, as a gentle upward trend on a graph, rather than late, as a 3 a.m. page when the process is killed.

## 15.7 Latency Versus Throughput

A piece of judgment that a staff engineer must hold, because optimizing for the wrong one of these is a real and common mistake: **latency and throughput are different things, and a system is often optimized for one at the expense of the other.**

**Latency** is *how long one operation takes* — the time from a single request arriving to its response leaving. **Throughput** is *how many operations the system completes per unit time* — requests per second, records processed per hour. They are related but they are not the same, and improving one can *worsen* the other. Batching is the cleanest example: processing requests in batches of a hundred rather than one at a time typically *raises throughput* (less per-request overhead, better use of resources) while *raising latency* (a request now waits for its batch to fill before being processed). You cannot optimize "performance" in the abstract; you must know *which* of these the system needs.

And which it needs depends entirely on what the system is *for*. A user-facing web request is a *latency* problem — the user is waiting, and a fast average is not even enough, because users experience the *slow* requests. This is why latency is measured in **percentiles**, not averages: the **p99 latency** — the latency below which 99% of requests fall, i.e. the experience of the slowest 1% — matters enormously, because in a system serving millions of requests, that "slowest 1%" is a vast number of real users having a bad experience, and an average comfortably hides it. A staff engineer designing a user-facing system optimizes and monitors *tail latency* — p95, p99, p99.9 — not the mean. A batch data-processing job, by contrast, is usually a *throughput* problem — no one is waiting on any individual record; what matters is that the whole batch finishes in time — and there, batching and the latency-for-throughput trade are exactly right.

The judgment, then: before optimizing a system's "performance," determine whether it is a latency-sensitive system or a throughput-sensitive one, because the right optimizations — and the right *measurements* — differ, and optimizing for the wrong one can actively harm the thing that mattered. (Chapter 20 returns to this: percentile latency and the four golden signals are core to production observability, and this section is the performance-engineering groundwork for it.)

> ### War Story — The Optimization That Wasn't the Problem
>
> A team had a service endpoint that was slow — noticeably, complained-about slow. An engineer was assigned to fix it. They opened the code, read it, and found what was obviously the expensive part: a function that did a genuinely involved computation, with nested loops and a non-trivial algorithm. *That*, clearly, was the bottleneck. They spent the better part of a week optimizing it — tightening the loops, applying code-level tunings, even partially rewriting the algorithm. The function became substantially faster in isolation.
>
> The endpoint was not noticeably faster. The complained-about slowness was essentially unchanged.
>
> Only then did the engineer profile the endpoint — which should have been the *first* step, not the step taken after a week of work. The profile showed, immediately and unambiguously, where the time actually went: not in the computation at all, but in the *data layer*. The endpoint was issuing a database query inside a loop — the N+1 problem of Chapter 18 — making dozens of separate round-trips to the database where one query would have done. The "obviously expensive" computation the engineer had spent a week on was, per the profile, a small fraction of the endpoint's time. The real bottleneck was the I/O pattern, it was invisible from reading the computation-heavy function, and it was fixable in an afternoon — replace the query-in-a-loop with a single batched query.
>
> The week of computation-tuning was not *wrong* exactly — the function was genuinely faster afterward — but it was a week spent optimizing the 97%, code that measurement would have shown was not the problem, while the actual 3% sat untouched. The lesson is the chapter's first and most violated principle, learned the expensive way: *profile first.* Human intuition about where time goes is reliably wrong, the bottleneck is almost never where it looks like it is, and the cost of skipping the measurement step is not just the wasted optimization effort — it is the whole stretch of time during which the real problem went unfixed because nobody had looked. The engineer had done real, skilled optimization work. They had simply done it before measuring, and so they had done it in the wrong place.

> ### The Level Line — Performance Engineering
>
> **A senior engineer** can use a profiler, knows that algorithms matter more than micro-optimizations, can speed up code that is identified as slow, and knows that built-ins and comprehensions are fast.
>
> **A staff engineer** treats "measure before optimizing" as an unbreakable discipline; reads a profile fluently — self versus total time, call counts, the characteristic surprises — and lets it overturn their guess; works the optimization hierarchy top-down, algorithm first and native code last; profiles memory with snapshots and diffs, not guesses; and distinguishes latency from throughput, optimizing and measuring the one the system actually needs, in percentiles where appropriate.
>
> **A principal engineer** establishes performance as a discipline on the team — measurement habits, profiling in the toolkit, performance budgets, monitoring of tail latency in production — resists optimization that measurement has not justified and protects the readability of the 97%; knows when "fast enough" has been reached and stops; and teaches the measure-first instinct so the team stops optimizing by guess.

## 15.8 At the Interview Table

Performance questions are common, and what they almost always test is not knowledge of micro-optimizations but the *discipline* — whether the candidate measures or guesses.

**The question: "This function is slow. How do you make it faster?"**

The interviewer is, above all, listening for whether you *profile first*. The strong answer leads with the discipline, not with a guess: "Before changing anything, I'd profile it — with `cProfile`, or `py-spy` if it's a running service — because where the time actually goes is reliably surprising, and optimizing by intuition usually means optimizing the wrong thing. Once the profile shows me the real bottleneck, I work an optimization hierarchy top-down. First the algorithm and data structures — that's the highest-leverage level by far; an O(n²) made O(n log n), or a list membership test made a set lookup, is an asymptotic win nothing else can match. Then: am I doing unnecessary work — repeated computation I could cache, eager work I could make lazy, many small I/O calls I could batch? Then, only for a genuine measured hot path, Python-level code tuning. And as a last resort, for a real CPU-bound core, drop that piece out of Python — NumPy, Numba, or a Rust extension. After each change I measure again to confirm it actually helped. And I'd ask first how fast it *needs* to be — there's no point optimizing past the requirement." That answer demonstrates the entire chapter.

**The question: "Explain self time versus total time in a profile."**

"Total — or cumulative — time for a function is the time spent in it *including everything it calls*, recursively. Self time is the time spent in *its own code*, excluding callees. The distinction is how you read a profile correctly: a function with huge total time but tiny self time isn't itself slow — it's just calling something slow, and I follow the call chain down. A function with large self time *is* where the time is being spent. So I sort by self time to find what to optimize, and use total time to understand the call structure."

**The question: "How would you find a memory leak in a long-running service?"**

"In Python it's almost never a true leak — it's unintended retention: objects still referenced by something, so they can't be collected. Usual suspects are an unbounded cache, module- or class-level state accumulating across requests, a registry never cleaned up. To find it I'd use `tracemalloc`: take a snapshot, let the service run and process more load, take another snapshot, and *diff* them — the diff shows which allocation sites grew, and a site that grows steadily across snapshots is the leak. `memray` is the heavier tool if I need to see C-extension allocations too. The key is the same as for CPU: measure with snapshots, don't guess. And in a long-running service I'd monitor process memory over time as a habit, so retention shows up as a trend on a graph rather than as an out-of-memory kill."

**The question: "Latency or throughput — what's the difference, and why does it matter?"**

"Latency is how long one operation takes; throughput is how many complete per unit time. They're different and they can trade against each other — batching usually raises throughput while raising latency, since a request waits for its batch. Which one to optimize depends on the system: a user-facing request is a latency problem, and specifically a *tail* latency problem — I'd measure p99, not the average, because the slow 1% is a huge number of real users and the average hides them. A batch job is a throughput problem — no one waits on an individual record. Optimizing for the wrong one can actively harm what mattered, so I'd establish which the system needs before optimizing anything."

**The red flags.** Jumping straight to a fix without mentioning profiling — the single biggest red flag, and the thing the question is designed to catch. Reaching first for micro-optimizations or native code while ignoring the algorithm. Not knowing self versus total time. Proposing to find a memory leak by reading code rather than by measuring. And no awareness of the latency/throughput distinction or of percentile latency.

## 15.9 The Forge

**Drill 15.1.** Take a provided (or self-written) slow program. *Before profiling*, write down your guess for where the time goes. Then profile it with `cProfile`. Compare the profile to your guess and write up the difference — the exercise's point is to experience, concretely, how wrong intuition is.

**Drill 15.2.** Given a profile output (real or provided), identify: the function with the highest *self* time; a function with high *total* but low *self* time and what that tells you; the most-called function; and any sign of an accidental O(n²) or hidden I/O. Write your reading of the profile and what you would optimize first.

**Build 15.1.** Take a deliberately slow program with several distinct performance problems — an O(n²) algorithm, an unnecessary repeated computation, a hidden I/O-in-a-loop, and some micro-inefficiency. Profile it, then optimize it by working the hierarchy top-down. After each change, re-measure. Produce a report: the profile, each change, the measured speedup of each, and the total — and note which level of the hierarchy gave the biggest win (it should be the algorithm).

**Build 15.2.** Take a numerical computation written as pure-Python loops. Measure it. Then optimize it by dropping out of Python: first vectorize it with NumPy and measure; then, optionally, try Numba and measure. Report the speedups and write up when each tool is the right choice.

**Investigate 15.1.** Hunt a memory leak. Take (or write) a long-running program that retains memory unintentionally — an unbounded cache is the easiest to construct. Use `tracemalloc` snapshots and diffs to *locate* the growing allocation site. Write up the technique and how the diff pointed at the leak.

**Investigate 15.2.** Profile a real running process with `py-spy` — attach it to something live, without modifying or restarting the process, and capture where it spends its time. Write up the experience and why the ability to profile a live process without code changes is valuable in production.

**Design 15.1.** You are the staff engineer on a team where performance work is done badly: engineers optimize by guessing, the codebase has micro-optimizations scattered through ordinary code hurting its readability, there is no profiling habit, and a recent "optimization" project sped up code that was never the bottleneck. Write a one-to-two-page plan to fix the *culture*: what discipline you establish, what tools you put in the team's standard kit and how, how you make profiling the reflexive first step, how you decide what is worth optimizing and what to leave alone, how performance gets monitored in production (tying to Chapter 20), and how you would push back on optimization that measurement has not justified. This is the staff-level exercise — performance not as a trick but as a team discipline.

---

[← Chapter 14: Asyncio in Depth](14-asyncio-in-depth.md) · [Home](README.md) · [Chapter 16: Testing as Engineering →](16-testing-as-engineering.md)
