# Chapter 14 — Asyncio in Depth

*Part IV — Concurrency and Performance*

[← Chapter 13: The GIL and Concurrency Models](13-gil-and-concurrency-models.md) · [Home](README.md) · [Chapter 15: Performance Engineering →](15-performance-engineering.md)

---

## 14.1 The Question Behind the Chapter

Asyncio is the concurrency model most often *reached for* and most often *misused*. It is reached for because it is the modern, fashionable choice and because the web frameworks engineers most want to use (Chapter 17) are async-native. It is misused because engineers adopt the *syntax* — `async def`, `await` — without the *model*, and the model is genuinely different from anything in synchronous code. The result is a class of bugs that synchronous Python simply cannot produce: a single blocking call that freezes an entire server, a swallowed cancellation that turns a clean shutdown into an outage, a coroutine that was created and never awaited and silently did nothing.

Chapter 13 placed asyncio among the concurrency models and told you *when* to choose it: I/O-bound work, at large concurrency scale, or in an async ecosystem. This chapter is the *how* and the *why* — the depth. By the end you will hold the event-loop model firmly enough to reason from it, you will know the modern structured-concurrency primitives, you will understand the cancellation contract that trips up so many engineers, and you will know the cardinal sins and how to avoid them. Chapter 7 already planted the seed — `async`/`await` is the pause-and-resume capability of generators pointed at I/O waiting — and this chapter grows the full system from that seed.

## 14.2 The Event Loop

Everything in asyncio rests on one idea, and if you hold this idea clearly the rest follows: **asyncio is a single thread running a single event loop that interleaves many coroutines by switching between them at their `await` points.**

Unpack that. There is *one thread* — asyncio does not use multiple threads, and so, within a single event loop, the GIL is simply not a factor; there is only ever one thread executing. There is one **event loop**, which is a scheduler: a loop that keeps a collection of coroutines and decides which one runs next. And there are many **coroutines** — the pausable functions from Chapter 7, defined with `async def` — which the loop runs *one at a time*, switching between them.

The switching is the key, and it happens at `await`. When a coroutine reaches an `await` on something that is not yet ready — an `await` on a network response, a database query, a timer — the coroutine *pauses itself* (the generator pause-and-resume of Chapter 7) and *yields control back to the event loop*. The loop, now free, looks at all its other coroutines and runs one that is *ready* to make progress. When the thing the first coroutine was waiting for becomes ready, the loop *resumes* that coroutine from exactly where it paused. The loop is constantly doing this — running a coroutine until it `await`s something not-ready, switching to another, coming back when the first one's wait is satisfied — and the *effect* is that many I/O operations are *in flight at once* even though only one thread, only one coroutine, is ever actually *executing* at any instant.

This is why asyncio is brilliant for I/O-bound work and useless for CPU-bound work, and you can now *derive* both facts rather than memorize them. It is brilliant for I/O because I/O is *waiting*, and `await` turns waiting into an opportunity for the loop to run something else — ten thousand coroutines each mostly waiting on the network overlap all their waiting, on one thread, with almost no per-coroutine cost. It is useless for CPU-bound work because a CPU-bound computation does not `await` anything — it just *computes*, holding the single thread, and while it holds the thread *nothing else in the entire program runs*. That last sentence is also the seed of the cardinal sin of Section 14.7; hold onto it.

The **cooperative** nature of this is worth stating explicitly, because it is the model's central trade-off. The event loop cannot *forcibly* take control away from a running coroutine the way an operating system can preempt a thread. A coroutine keeps the thread until *it chooses* to give it up — and it gives it up at an `await`. Concurrency in asyncio is *cooperative*: every coroutine must cooperate by `await`ing often enough to let others run. A coroutine that runs for a long time without `await`ing is *not cooperating*, and it starves every other coroutine for that whole time. This is the source of asyncio's power (switching is cheap and explicit, there are no surprise preemptions, far fewer race conditions than threads) and the source of its central hazard (one uncooperative coroutine stalls everything).

## 14.3 Coroutines, Tasks, and the Distinction

Two concepts are easy to conflate and must be distinguished, because the distinction is the source of a common bug.

A **coroutine** is what you get when you *call* an `async def` function. And — exactly as Chapter 7 said of generators — calling an `async def` function *runs none of its body*. It returns a coroutine object, inert, with the body not yet executed. The coroutine runs only when it is *awaited* or *scheduled on the loop*.

This produces the classic beginner bug: an engineer calls an async function — `do_the_thing()` — expecting it to do the thing, and it does nothing at all, because they created a coroutine object and never awaited it. (Python will usually warn — "coroutine was never awaited" — but the work silently did not happen.) A coroutine is a *description* of asynchronous work; it does nothing until something drives it.

A **Task** is a coroutine that has been *scheduled on the event loop to run*. You create one with `asyncio.create_task(some_coroutine())`, and from that moment the event loop is *driving* it — it will make progress whenever the loop runs, interleaved with everything else. A Task runs *concurrently*; a bare coroutine does not run at all until awaited.

The distinction governs concurrency. If you write `await do_a()` and then `await do_b()`, the two run *sequentially* — `do_a` fully completes before `do_b` starts — because each `await` waits for its coroutine to finish before the next line. There is no concurrency there at all. To run them *concurrently*, both must be scheduled on the loop *before* you wait — as tasks, or through `gather` (next section) — so that while one is paused awaiting its I/O, the other can run. "I used `async`/`await` so my code is concurrent" is false; sequential `await`s are sequential. Concurrency comes from having *multiple things scheduled on the loop at once*, and that is what tasks and `gather` are for.

## 14.4 Running Things Concurrently

A toolkit of constructs schedules multiple coroutines to run concurrently. A working set, in roughly the order you should reach for them.

**`asyncio.run(main())`** is the entry point — it starts an event loop, runs the given coroutine to completion, and shuts the loop down cleanly. It is how an async program begins; you write one `async def main()` and call `asyncio.run(main())`.

**`asyncio.gather(*coroutines)`** runs several coroutines *concurrently* and waits for *all* of them, returning their results as a list in order. This is the workhorse for "do all of these I/O operations at once": `await asyncio.gather(fetch(a), fetch(b), fetch(c))` runs all three fetches concurrently — while each waits on its network response, the others proceed — and gives you the three results when all are done. The total time is roughly the time of the *slowest* one, not the sum.

**`asyncio.TaskGroup`** (Python 3.11+) is the **modern, recommended** way to run a group of tasks concurrently, and you should prefer it to `gather` for new code. It is a *structured concurrency* primitive — and that phrase names a real and important idea. Structured concurrency means that concurrent tasks have a well-defined *scope*: they are created inside a block, and the block does not exit until *all* of them have finished. Tasks cannot "leak" out of the block and outlive it. You use it with `async with`:

```python
async with asyncio.TaskGroup() as tg:
    tg.create_task(fetch(a))
    tg.create_task(fetch(b))
    tg.create_task(fetch(c))
# all three tasks are guaranteed complete here
```

The `async with` block does not exit until every task created inside it has finished. This guarantees there are no orphaned, leaked tasks — a real failure mode with raw `create_task`, where a task created and forgotten runs (or fails) somewhere with nothing watching it. And `TaskGroup` has a second crucial property: **if one task fails, the group cancels the others and collects the exceptions**. If two of the three fetches raise, the `TaskGroup` cancels any still-running siblings and raises an `ExceptionGroup` — exactly the `ExceptionGroup` of Chapter 8, and now you see *why* that chapter introduced it: concurrent failure is plural, and `TaskGroup` is the primitive that produces it. The structured guarantees — tasks scoped to a block, failures collected, siblings cancelled — are why `TaskGroup` is the modern default.

**`asyncio.as_completed(coroutines)`** runs coroutines concurrently but lets you process results *as each one finishes*, rather than waiting for all — useful when you want to act on the fastest results first.

**`asyncio.wait_for(coroutine, timeout)`** and the `asyncio.timeout()` context manager apply a *timeout* to an awaitable — if it does not complete in time, it is cancelled and a timeout error is raised. Timeouts are not optional in real async code, for the same reason they are not optional anywhere (the principle recurs in Chapter 17): an `await` on a network operation with no timeout can wait *forever* if the other end never responds, and one such hung `await` ties up a coroutine indefinitely. Every `await` on something external should have a timeout around it.

## 14.5 Async Synchronization

Asyncio has its *own* synchronization primitives — `asyncio.Lock`, `asyncio.Semaphore`, `asyncio.Event`, `asyncio.Queue` — and there are two things a staff engineer must know about them.

First: **the asyncio primitives are not the `threading` primitives, and you must not mix them up.** `asyncio.Lock` is awaited (`async with async_lock:`); `threading.Lock` is not. Using a `threading.Lock` in async code is a bug — at best it does the wrong thing, at worst (if it blocks) it freezes the whole event loop. Async code uses the `asyncio.*` synchronization primitives, full stop.

Second, and more interesting: **you need synchronization in asyncio much *less* than you do in threading, and understanding why is understanding the model.** Recall that asyncio is *one thread*, and that switching between coroutines happens *only at `await` points*, cooperatively. This means a stretch of code in a coroutine that contains *no `await`* runs *atomically with respect to other coroutines* — nothing else can interleave into it, because nothing else gets to run until this coroutine `await`s. The read-modify-write race that plagues threading (Chapter 13's lost-update counter) simply *cannot happen across an await-free stretch of async code*. You need an `asyncio.Lock` only when a critical section *spans an `await`* — when a coroutine reads some shared state, then `await`s something (giving another coroutine the chance to run and modify that state), then acts on its now-possibly-stale read. That specific shape — shared state, an `await` in the middle, action after — is where async synchronization is needed. It is a much *narrower* set of cases than threading's, because the cooperative single-threaded model has already eliminated the rest. `asyncio.Semaphore` has a particularly common and important use that is not about correctness but about *bounding*: limiting how many operations run concurrently — capping concurrent outbound requests so you do not overwhelm a downstream service — which is the async answer to the unbounded-concurrency anti-pattern.

## 14.6 The Cancellation Contract

Cancellation is the part of asyncio that engineers most often get wrong, and getting it wrong produces real production incidents — so it gets a careful treatment.

A task can be **cancelled** — by `task.cancel()`, by a `TaskGroup` cancelling siblings after a failure, by a timeout firing. Cancellation in asyncio works by a specific mechanism: a special exception, **`asyncio.CancelledError`**, is *raised inside the coroutine* at its current `await` point. The coroutine, in effect, gets an exception thrown into it that says "stop."

This mechanism creates a **contract**, and the contract is the thing to internalize: **`CancelledError` must be allowed to propagate.** When a coroutine receives a `CancelledError`, the *correct* behavior is to do any necessary cleanup and then let the exception continue upward — to *not* swallow it. A coroutine *may* catch `CancelledError` briefly, to perform cleanup (close a connection, release a resource, roll back), but having done the cleanup it must **re-raise** it. Cancellation that is caught and not re-raised is cancellation *defeated* — the task that was supposed to stop simply does not stop.

Here is where the contract breaks in real code, and it breaks because of a pattern that looks completely innocent. An engineer writes a broad exception handler around some async work — `try: await something() except Exception: handle()` — intending to handle errors robustly. But recall the exception hierarchy from Chapter 8: in modern Python `asyncio.CancelledError` inherits from `BaseException`, *not* from `Exception` — *precisely so that* an ordinary `except Exception` does **not** catch it. If, however, the engineer writes `except Exception` and the code is older, or they write the broader `except BaseException`, or they write a bare `except:` — then the `CancelledError` *is* caught by that broad handler, *handled* as if it were an ordinary error, and *not re-raised*. The cancellation is silently swallowed. The task that was asked to stop continues running.

The consequences are real. Graceful shutdown depends on cancellation: when a server shuts down, it cancels its in-flight tasks and waits for them to wind down. If those tasks swallow their `CancelledError`, they do not wind down — the shutdown hangs, waiting forever for tasks that will never stop, until something forcibly kills the process. Timeouts depend on cancellation: a timeout fires by cancelling the timed-out operation. If the operation swallows the `CancelledError`, the timeout does *not* take effect — the operation runs on past its deadline. A "robust" `except Exception` (or worse) wrapped around async work, written with the best of intentions, can defeat both the shutdown path and every timeout in the system.

The rules, then, stated as a contract a staff engineer holds firmly: **catch specific exceptions in async code, not broad ones; if you must catch broadly, explicitly re-raise `CancelledError`; treat `CancelledError` as something to be allowed through, with at most a cleanup pause, never as an error to be handled and absorbed.** This is the single most important piece of async-specific judgment in the chapter, and the War Story below is what ignoring it costs.

## 14.7 The Cardinal Sin: Blocking the Event Loop

Section 14.2 planted this and now we harvest it. Asyncio is *one thread*. The event loop makes progress by running coroutines and switching between them at `await` points. The entire model depends on coroutines *yielding control back to the loop* regularly, by `await`ing.

The cardinal sin is therefore: **calling a blocking, synchronous operation inside a coroutine.**

A blocking operation — a synchronous file read, a synchronous network call with an ordinary library like `requests`, a `time.sleep()`, a heavy CPU-bound computation — does *not* yield control to the event loop. It just *runs*, holding the single thread, for as long as it takes. And because there is only one thread, while that blocking call holds it, **the event loop cannot run, and therefore no other coroutine runs, and therefore the entire program is frozen** until the blocking call returns. One coroutine making one blocking three-second database call with a synchronous driver freezes *every* request the server is handling, *every* background task, *everything*, for three seconds. The async server that was meant to handle ten thousand concurrent connections handles *zero* for the duration of one careless blocking call.

This is the most common and most damaging asyncio bug, and it is insidious because the blocking call *looks completely normal* — it is just ordinary synchronous code, the kind that is correct everywhere else. The problem is purely contextual: ordinary blocking code is fine in a thread (the OS preempts it and other threads run), and catastrophic in a coroutine (nothing preempts it and nothing else runs).

There are two fixes, and which one depends on what is blocking.

If the blocking thing is **I/O**, the right fix is to **use an async-native library for it.** Do not call `requests` (synchronous) in a coroutine — call `httpx` or `aiohttp` (async), which `await` properly and yield to the loop while waiting. Do not use a synchronous database driver — use `asyncpg` or an async ORM. The async ecosystem exists precisely so that I/O in a coroutine can be `await`ed rather than blocked-on. Section 14.8 surveys it.

If the blocking thing is **CPU-bound work**, or a synchronous library you genuinely cannot replace, the fix is to **move it off the event-loop thread**: `asyncio.to_thread(blocking_function, args)` runs the blocking call in a *separate thread* (from a thread pool) and gives you an awaitable for its result — so the coroutine `await`s, the loop stays free, and the blocking work happens elsewhere. (For CPU-bound work specifically, recall Chapter 13: a thread will not give you CPU parallelism because of the GIL, so genuinely heavy computation should go to a `ProcessPoolExecutor`, reached via the loop's `run_in_executor`. `to_thread` is right for *blocking I/O in a stubborn synchronous library*; a process pool is right for *heavy CPU work*.)

The rule that prevents the cardinal sin: **in async code, every operation that waits or computes for a non-trivial time must either be `await`ed through an async library or be explicitly moved off the loop thread. Nothing slow and synchronous may run directly in a coroutine.**

## 14.8 The Async Ecosystem

Async code needs *async libraries* — synchronous libraries block the loop, as Section 14.7 just established — so a staff engineer must know the async ecosystem at least by name.

For **HTTP**, `httpx` (which offers both sync and async, and is broadly the modern default) and `aiohttp` (async client and server). For **databases**, `asyncpg` (a fast async PostgreSQL driver), the async support in SQLAlchemy (Chapter 15's territory), and async drivers for other databases. For **web frameworks**, FastAPI and Starlette and others — Chapter 17's subject — are async-native and built directly on the event loop.

One library deserves a specific mention: **`anyio`**. Beyond `asyncio` itself there is an alternative async framework called `trio`, which pioneered several of the structured-concurrency ideas (`TaskGroup` is, in part, `trio`'s nursery concept brought into the standard library). `anyio` is a compatibility layer that lets library code be written *once* and run on *either* `asyncio` or `trio`. If you are writing an async *library* meant for broad use, `anyio` is worth knowing; for ordinary application code on `asyncio`, you do not strictly need it, but a staff engineer should know it exists and what problem it solves.

The single operational rule of the ecosystem: when you are writing async code, every library you use for I/O must be an *async* library. The moment a synchronous library sneaks into a coroutine, you have the cardinal sin of Section 14.7. Choosing async-native libraries throughout is not a preference; it is a correctness requirement of the model.

> ### War Story — The Cancellation That Was Swallowed
>
> A team ran an async service — a web server handling a large number of concurrent requests, built on the async stack, well-regarded internally as solid. It had one recurring, baffling operational problem: deploying a new version sometimes took far longer than it should, and occasionally a deploy would *hang* entirely, the old process refusing to exit, until someone forcibly killed it.
>
> The deploy process worked the normal way: send the running process a signal to shut down gracefully, and the process would stop accepting new requests, *cancel its in-flight request tasks*, wait for them to wind down, and exit. Most of the time this worked. Sometimes it hung, and the hang seemed random — correlated with nothing the team could identify, so it was tolerated as a quirk.
>
> The cause was a single broad exception handler. Somewhere in the request-handling path was a piece of code wrapped in a `try: ... except Exception: log_and_continue()` — written, long before, by an engineer who wanted that step to be "robust" and not let an error there fail the whole request. In most of the codebase the `CancelledError` would have sailed past such a handler, because it inherits from `BaseException` and not `Exception`. But this particular handler — older code — caught broadly enough, and in a way, that the `CancelledError` raised into the coroutine at shutdown was caught by it, *logged as an ordinary error*, and *not re-raised*. The cancellation was swallowed. The task that the graceful-shutdown path had asked to stop did not stop — it caught its "stop" signal, logged it, and carried on. The shutdown logic then waited, correctly and forever, for an in-flight task that had been told to finish and had quietly declined. The deploy hung. It hung "randomly" only because it hung exactly when a request happened to be passing through that handler at the moment shutdown began.
>
> The fix was small and the lesson was large. The fix: make the handler catch the *specific* exceptions it actually meant to handle, and — the belt-and-suspenders rule for any broad handler in async code — explicitly re-raise `CancelledError` before doing anything else. The lesson: cancellation in asyncio is a *contract*, and the contract says `CancelledError` must be allowed to propagate. A broad `except` in async code is not merely the ordinary code smell of Chapter 8 — it is, additionally, a thing that can silently defeat cancellation, and defeating cancellation breaks graceful shutdown and breaks every timeout in the system. The team's "random" deploy hangs were a broad exception handler quietly breaking the cancellation contract, and the cure was to respect the contract: catch narrowly, and if you must catch broadly, let the cancellation through.

> ### The Level Line — Asyncio
>
> **A senior engineer** writes correct `async`/`await` code, uses `asyncio.run` and `gather`, knows async needs async libraries, and knows not to block the event loop.
>
> **A staff engineer** holds the event-loop model explicitly — single thread, cooperative switching at `await`, derives from it why async fits I/O and not CPU; distinguishes a bare coroutine from a scheduled Task and knows sequential `await`s are not concurrent; uses `TaskGroup` and structured concurrency; understands the cancellation contract and why a broad `except` can break it; knows why async needs *less* synchronization than threading; and reaches correctly for `to_thread` versus a process pool when something must come off the loop.
>
> **A principal engineer** designs async systems where the event loop is never blocked by construction, where cancellation and graceful shutdown are correct because the cancellation contract is respected everywhere, where concurrency is bounded with semaphores and timeouts are universal; chooses async deliberately against threading (Chapter 13's decision) rather than by fashion; and teaches the event-loop model so the team stops committing the cardinal sin.

## 14.9 At the Interview Table

Asyncio is increasingly central to Python interviews, and the questions separate the engineer who learned the syntax from the one who holds the model.

**The question: "What is the event loop?"**

"It's a single-threaded scheduler at the heart of asyncio. Asyncio runs one thread, with one event loop, and many coroutines. The loop runs a coroutine until it `await`s something not yet ready — at which point that coroutine pauses and yields control back to the loop — and the loop then runs another coroutine that's ready to make progress, coming back to the first one when its awaited thing is ready. The effect is many I/O operations in flight at once on a single thread. It's *cooperative* — the loop can't preempt a running coroutine; a coroutine keeps the thread until it chooses to `await`. That's why asyncio is excellent for I/O-bound work, where `await` turns waiting into a chance to run something else, and useless for CPU-bound work, where a computation never `await`s and just holds the one thread."

**The question: "What happens if you call a blocking function inside a coroutine?"**

"You freeze the entire program. Asyncio is a single thread, and the event loop only makes progress when coroutines yield control by `await`ing. A blocking synchronous call — `requests.get`, `time.sleep`, a heavy computation, a synchronous database query — doesn't `await`; it just runs, holding the one thread, and while it holds the thread the event loop can't run, so *no other coroutine runs* — every request, every background task, frozen until the blocking call returns. It's the cardinal asyncio sin, and it's insidious because the blocking code looks completely normal — it's fine in a thread, catastrophic in a coroutine. The fix: for blocking *I/O*, use an async-native library — `httpx` not `requests`, `asyncpg` not a sync driver. For a stubborn synchronous library or CPU work, move it off the loop thread — `asyncio.to_thread` for blocking I/O, a process pool via `run_in_executor` for heavy CPU work."

**The question: "How does cancellation work in asyncio?"**

"A task is cancelled by raising `asyncio.CancelledError` inside it at its current `await` point. The key thing is that this creates a *contract*: `CancelledError` must be allowed to propagate. A coroutine can catch it briefly to do cleanup, but it must then re-raise it — swallowing it defeats the cancellation. And here's the trap: `CancelledError` inherits from `BaseException`, not `Exception`, *specifically so* an ordinary `except Exception` won't catch it. But a broad `except BaseException`, a bare `except:`, or older code, *will* catch it — and if it's caught and not re-raised, the cancellation is silently swallowed. That breaks things badly: graceful shutdown cancels in-flight tasks and waits for them, so a swallowed cancellation makes shutdown hang forever; and timeouts fire by cancelling, so a swallowed cancellation defeats every timeout. The rule is: catch specific exceptions in async code, and if you ever catch broadly, explicitly re-raise `CancelledError`."

**The question: "Why does `await a()` then `await b()` not run concurrently?"**

"Because each `await` waits for its coroutine to fully complete before the next line runs — so `a` finishes entirely before `b` starts; that's sequential. Calling an async function just creates an inert coroutine object; it doesn't run until something drives it. For concurrency, both have to be *scheduled on the loop at once* before you wait — `await asyncio.gather(a(), b())`, or creating them as tasks in a `TaskGroup`. Then while one is paused awaiting its I/O, the other runs. Just using `async`/`await` doesn't make code concurrent; having multiple things scheduled on the loop simultaneously does."

**The red flags.** Thinking sequential `await`s run concurrently. Not knowing that a blocking call freezes the whole loop — or not knowing the fixes. Swallowing `CancelledError`, or not knowing the cancellation contract exists. Using `threading.Lock` in async code. Thinking asyncio gives CPU parallelism. And reaching for asyncio reflexively without being able to say *why* it fits the workload (Chapter 13's decision).

## 14.10 The Forge

**Drill 14.1.** Write two async functions that each `asyncio.sleep` for one second. Call them three ways and time each: sequentially with two `await`s; concurrently with `asyncio.gather`; concurrently with a `TaskGroup`. Explain why the sequential version takes two seconds and the others take one.

**Drill 14.2.** Demonstrate the cardinal sin. Write an async program with several coroutines, and have one of them call a *blocking* `time.sleep` (not `asyncio.sleep`). Show that the blocking call freezes the others. Then fix it with `asyncio.to_thread` and show concurrency restored. Explain.

**Drill 14.3.** Write a coroutine that wraps its work in a broad `except`. Cancel it from outside. Show that the cancellation is swallowed and the task does not stop. Then fix the handler — catch specifically, or re-raise `CancelledError` — and show cancellation working.

**Build 14.1.** Build an async web scraper: given a list of several hundred URLs, fetch them all concurrently with `httpx.AsyncClient`, *bounded* by an `asyncio.Semaphore` so no more than N requests are in flight at once, with a per-request timeout, using a `TaskGroup` so a failure in one fetch is collected rather than lost. Report results and failures. Benchmark against a sequential version.

**Build 14.2.** Build an async producer/consumer pipeline using `asyncio.Queue`: producers generate work items, consumers process them, all on one event loop. Add graceful shutdown — on a signal, stop the producers, let the consumers drain the queue, and exit cleanly — and verify the cancellation path works correctly.

**Investigate 14.1.** Take a piece of async code and deliberately introduce a blocking call somewhere non-obvious. Use a tool or technique to *detect* the event-loop block (asyncio's debug mode can warn about slow callbacks; or measure loop responsiveness). Write up how you would find such a bug in a real codebase.

**Investigate 14.2.** Explore structured concurrency. Compare `asyncio.TaskGroup` with the older pattern of bare `create_task` calls. Construct a scenario where the bare-`create_task` approach leaks a task or loses an exception, and show that `TaskGroup` prevents it. Write up what "structured concurrency" guarantees and why those guarantees matter.

**Design 14.1.** You are designing an async service that handles a large number of concurrent client connections, calls several downstream services per request, has background tasks, and must shut down gracefully on deploy. Write a one-to-two-page design: how the event loop is kept unblocked (what is async-native, what must go to `to_thread` or a process pool); how concurrency to downstreams is bounded; how timeouts are applied universally; how the cancellation contract is respected so graceful shutdown actually works; and how you would test the shutdown path. This is the staff-level exercise — an async system designed so the model's hazards are prevented by construction.

---

[← Chapter 13: The GIL and Concurrency Models](13-gil-and-concurrency-models.md) · [Home](README.md) · [Chapter 15: Performance Engineering →](15-performance-engineering.md)
