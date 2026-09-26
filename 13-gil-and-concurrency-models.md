# Chapter 13 — The GIL and Concurrency Models

*Part IV — Concurrency and Performance*

[← Chapter 12: Architecture and Design Patterns](12-architecture-and-patterns.md) · [Home](README.md) · [Chapter 14: Asyncio in Depth →](14-asyncio-in-depth.md)

---

## 13.1 The Question Behind the Chapter

Ask a room of Python engineers about the Global Interpreter Lock and you will get a confident chorus: "Python can't do real threads," "the GIL makes Python single-threaded," "you have to use multiprocessing for anything parallel." Each of these statements is *almost* true and *importantly* wrong, and the gap between almost-true and precisely-true is exactly where engineers make expensive mistakes — reaching for `multiprocessing` and its heavy machinery when threads would have worked perfectly, or reaching for threads to speed up a computation that the GIL guarantees they cannot speed up.

This chapter replaces the folk story with a precise one. By the end you will know what the GIL actually is and the *exact* claim it makes — not the folklore claim. You will know that Python is in the middle of *removing* the GIL, what that means, and what its status is. And, most importantly, you will be able to look at a workload and choose — correctly, with reasons — between threading, multiprocessing, and async, because that choice is the real skill, and the chapter ends with a decision procedure for making it.

A note before we start: this chapter is about choosing *between* the concurrency models and about the threading and multiprocessing models specifically. The asyncio model is large enough to need its own chapter — Chapter 14 — so here we cover what async *is* and *when to choose it*, and the next chapter covers *how it works in depth*. The two chapters are one subject in two parts.

## 13.2 What the GIL Actually Is

The **Global Interpreter Lock** is a mutex — a lock — inside the CPython interpreter, and it makes one precise guarantee: **only one thread can be executing Python bytecode at any given moment.** A thread must hold the GIL to run Python bytecode; there is one GIL; therefore one thread runs Python bytecode at a time.

Sit with the precision of that statement, because the folklore distorts it in two directions.

First, the folklore says "Python can't do threads." False. Python has real, genuine operating-system threads — `threading.Thread` creates a real OS thread, the OS schedules it, it is not a simulation. What the GIL constrains is not the *existence* of threads but the *parallelism of bytecode execution*: the threads exist and run, but only one of them is executing Python bytecode at any instant; they take turns holding the GIL.

Second — and this is the part that the folklore most damagingly omits — **the GIL is released during I/O and during many C-extension operations.** When a thread does I/O — reads a file, waits on a network socket, queries a database — it *releases the GIL* while it waits, because waiting for I/O does not require executing Python bytecode. While that thread waits, another thread can take the GIL and run. Likewise, many C extensions — NumPy prominent among them — release the GIL during their heavy C-level computation. So during the I/O wait or the C computation, *other Python threads genuinely run*.

Now the consequence, which is the whole practical content of the GIL:

For a **CPU-bound** workload — a pure-Python computation that is constantly executing bytecode, never waiting — threads do **not** help. The threads cannot run bytecode in parallel; they serialize on the GIL; using four threads for a pure-Python computation gives you roughly the speed of one (in fact slightly *less*, because of the overhead of passing the GIL back and forth). This is the true core of the folklore.

For an **I/O-bound** workload — a workload that spends most of its time *waiting* for the network, for disk, for a database — threads **do** help, and substantially. While one thread waits on its I/O, it has released the GIL, and the other threads run. Four threads each spending 90% of their time waiting on network calls genuinely overlap their waiting and genuinely run several times faster than one. The GIL does not hurt I/O-bound threading because I/O-bound threads spend most of their time *not holding the GIL*.

That single distinction — **the GIL makes threads useless for CPU-bound work and useful for I/O-bound work** — is the real, precise content of the GIL, and it is the foundation of every model-selection decision in this chapter.

## 13.3 Why the GIL Exists

Engineers often treat the GIL as an embarrassing mistake, a thing the CPython developers simply failed to remove. That framing is wrong and it is worth correcting, because understanding *why* the GIL exists is what makes the free-threading effort (next section) comprehensible.

The GIL exists because of *reference counting* — and you already understand reference counting, from Chapter 2. Recall: every Python object has a reference count, and that count is incremented and decremented constantly, on nearly every operation, as references are created and destroyed. Now recall the flaw the chapter flagged and promised to return to: those increments and decrements are *writes to a shared counter*, and if multiple threads could run Python bytecode genuinely in parallel, they would be incrementing and decrementing those counts *simultaneously, without synchronization* — and concurrent unsynchronized writes to a shared counter is a data race. The count would be corrupted: an object's count could be wrong, leading to an object being freed while still in use (a crash) or never freed (a leak). Reference counting, as designed, is *not thread-safe*.

The GIL is the solution to that problem — a brutally simple solution, but a real one. If only one thread runs Python bytecode at a time, then only one thread touches any reference count at a time, and the race cannot happen. The GIL makes reference counting safe by making bytecode execution serial.

The alternative — making every single reference-count operation individually thread-safe, with atomic operations or fine-grained locks — is *possible*, but atomic operations are markedly slower than plain ones, and reference counting happens *constantly*, so making every refcount atomic imposes a real, pervasive performance tax on *all* Python code, including the overwhelmingly common single-threaded case. For decades the CPython project's judgment was that one coarse lock — fast in the common single-threaded case, limiting only for CPU-bound multithreading — was a better trade than a pervasive per-operation tax. The GIL was not an oversight. It was a deliberate engineering trade-off: simplicity and single-threaded speed, paid for with the loss of CPU-bound thread parallelism. Whether that trade is still the right one is exactly the question the free-threading effort reopens.

## 13.4 Free-Threaded Python

The trade-off is being revisited. **PEP 703**, accepted and implemented, makes the GIL *optional* — it introduces a build of CPython, called **free-threaded** Python, that runs *without* a GIL, so that Python threads can execute bytecode genuinely in parallel on multiple cores.

Making this work required solving the problem the GIL was solving — thread-unsafe reference counting — by other means: a combination of techniques including making refcount operations safe through specialized counting schemes, and protecting the interpreter's internal data structures with fine-grained locks instead of one global one. It is a large, deep change to the interpreter.

The status, as of this book's writing, is the part that requires honesty rather than hype. Free-threaded Python is *real* — the free-threaded build exists, it works, and it is the most significant change to Python's execution model in the language's history. But it is *not yet the default*, and the transition is genuinely in progress. There is a single-threaded performance cost to the free-threaded build — removing the one fast global lock and replacing it with finer-grained, more pervasive synchronization makes the single-threaded common case somewhat slower, and closing that gap is ongoing work. And the *ecosystem* must catch up: C extensions written assuming the GIL — assuming that their code and the reference counts they touch were protected by it — may not be safe under free-threading, and the vast world of compiled Python extensions is being audited and updated, which takes time.

So the honest staff-level summary, and the thing to actually carry: **free-threaded Python is coming, it is real, and it will eventually make CPU-bound multithreading viable in pure Python — but as of now it is a transition in progress, not a default you build on.** You should *track* it; you should know it exists and what it changes; you should, when designing new systems, keep half an eye on the fact that the GIL's CPU-bound limitation is on a path to disappearing. But you should not yet assume it. For now, the model-selection logic of this chapter holds: the GIL is present in the Python you are most likely deploying, and CPU-bound work still needs processes, not threads. The book will note this transition again where it is relevant; here, the point is simply that you understand what is changing and why, and are neither ignorant of it nor over-betting on it.

## 13.5 The Three Models

Python offers three models for doing more than one thing at a time, and the entire skill of this chapter is matching the model to the *shape* of the workload. First the three models, in brief; then, in the next sections, the two that this chapter owns in depth, and then the decision procedure.

**Threading.** Multiple OS threads within one process, sharing memory. Constrained by the GIL: useful for I/O-bound work (threads overlap their waiting), useless for CPU-bound pure-Python work (threads serialize on the GIL).

**Multiprocessing.** Multiple *processes*, each with its own Python interpreter and *its own GIL*. Because each process has its own interpreter, processes achieve *true parallelism* of CPU-bound Python code — four processes genuinely use four cores. The cost is that processes do *not* share memory, so getting data between them requires explicit, and not free, communication.

**Asyncio.** A single thread, running a single *event loop*, that interleaves many *coroutines* by switching between them at their `await` points. It is concurrency without threads — and therefore without the GIL being relevant, since there is only one thread. It excels at I/O-bound workloads with a *very large number* of concurrent operations. Chapter 14 covers its machinery; here we cover when to choose it.

The shape of the decision, stated once before the detail: **the first question is always whether the workload is CPU-bound or I/O-bound.** CPU-bound — burning processor cycles — needs *multiprocessing*, because only separate processes escape the GIL. I/O-bound — mostly waiting — can use *threading* or *asyncio*, and the choice between those two is mostly about *how many* concurrent operations and how the surrounding code is written. Hold that frame; the next sections fill it in.

## 13.6 Threading in Depth

A `threading.Thread` is a real operating-system thread. You give it a function to run; you `start()` it; you can `join()` it to wait for it to finish. Multiple threads in one process share the process's memory — every thread sees the same variables, the same objects — and that shared memory is simultaneously threading's convenience and its central danger.

The danger is the **race condition**. When two threads access the *same* shared mutable data and at least one of them *modifies* it, and their accesses are not synchronized, the result depends on the unpredictable interleaving of their execution — and is therefore wrong, intermittently and irreproducibly. The classic illustration is two threads each incrementing a shared counter. `counter += 1` *looks* atomic, a single indivisible step, but it is not: it is *read the current value, add one, write the new value back* — three steps. If thread A reads the value, and then thread B reads the same value before A has written its result back, both compute the same new value and both write it, and one of the two increments is silently lost. Run two threads incrementing a shared counter a hundred thousand times each and the final value is reliably *less* than two hundred thousand, by an unpredictable amount. (A subtlety worth noting: the GIL does *not* save you here. The GIL guarantees one thread runs *bytecode* at a time, but `counter += 1` is *several* bytecode instructions, and the GIL can be handed to another thread *between* those instructions. The GIL prevents the interpreter's own internals from being corrupted; it does *not* make your multi-step operations atomic.)

The defense is **synchronization** — primitives that coordinate threads' access to shared data. The `threading` module provides them:

A **`Lock`** is the basic mutual-exclusion primitive: a thread *acquires* the lock, does its work on the shared data, and *releases* it; while one thread holds the lock, any other thread trying to acquire it waits. Wrapping the counter increment in a lock — acquired before the read-modify-write, released after — makes the three steps effectively atomic with respect to other threads, and the lost-update bug disappears. Locks are used with a `with` statement (Chapter 7's context managers — `with my_lock:`), which guarantees the lock is released even if the protected code raises.

Beyond the basic lock, the module provides an **`RLock`** (a reentrant lock, which the same thread may acquire multiple times), a **`Semaphore`** (which allows up to N threads through rather than just one — useful for limiting concurrency, such as capping the number of simultaneous connections), an **`Event`** (a simple flag threads can wait on and signal), and a **`Condition`** (for more complex "wait until some state holds" coordination).

Two hazards of locking deserve naming because they are real and they appear in interviews. A **deadlock** occurs when threads wait on each other in a cycle — thread A holds lock 1 and wants lock 2, thread B holds lock 2 and wants lock 1, and neither can proceed. The classic defense is **lock ordering**: establish a global order for acquiring locks and have every thread acquire them in that order, which makes the waiting cycle impossible. **Lock contention** is the subtler cost: if many threads frequently need the *same* lock, they spend their time waiting for it rather than working, and the program's parallelism collapses toward serial. The defenses are to hold locks for as *short* a time as possible and to make their *granularity* fine — many small locks protecting small pieces of data rather than one big lock protecting everything.

The genuinely staff-level guidance about all of this: **shared mutable state protected by locks is hard to get right** — the bugs are intermittent, irreproducible, and timing-dependent, the worst kind to debug — and the best defense is, wherever possible, *to not share mutable state at all*. Threads that communicate by passing messages through a thread-safe **`queue.Queue`** — one thread puts work in, another takes it out, and no mutable state is directly shared — are far easier to reason about and far less bug-prone than threads sharing data behind locks. When you must share, lock carefully and minimally; but the deeper move is to architect the concurrency so that there is little or nothing to share.

## 13.7 Multiprocessing in Depth

When the workload is **CPU-bound** — a genuine computation, burning processor cycles in pure Python — threading cannot help, because the GIL serializes bytecode. The answer is **multiprocessing**: running the work in multiple separate *processes*, each with its own Python interpreter and its own GIL, so that the computation genuinely runs in parallel on multiple cores.

The `multiprocessing` module mirrors the `threading` API closely — there is a `Process` analogous to `Thread` — but the underlying reality is fundamentally different, and the difference is *memory*. Threads share one memory space; processes do *not* — each process has its own, isolated. That isolation is precisely what gives processes their own GIL and their true parallelism, but it has a direct and important consequence: **getting data into a worker process and getting results back out requires explicit communication, and that communication is not free.**

The mechanism of that communication is *serialization* — specifically, `pickle` (Chapter 13's stdlib material, in the reference sense). When you send an object to another process, Python serializes it with `pickle` in the sending process, transmits the bytes, and deserializes it in the receiving process. This has three consequences a staff engineer must hold. First, it has a *cost*: serializing, transmitting, and deserializing data takes time and memory, and for *large* data that cost can be significant — large enough, sometimes, to erase the benefit of the parallelism. Second, it has a *constraint*: the data must be *picklable*; some objects cannot be pickled, and passing one to a process fails. Third, it shapes the *design*: multiprocessing pays off when each worker is handed a *modest* amount of input and does a *large* amount of computation on it — a high ratio of compute to data-transfer. It pays off poorly when workers must constantly exchange large volumes of data, because then the serialization cost dominates.

There is also a subtlety in *how a worker process is created*, the **start method**, and it has real consequences. The **fork** method (the historical default on Unix) creates a child process as a near-instant copy of the parent — fast, but it copies the parent's entire state, including held locks and open resources, and a child that inherits a lock the parent was holding can deadlock; `fork` also interacts badly with threads. The **spawn** method starts a fresh, clean Python interpreter and imports what it needs — slower to start, but clean and predictable, with none of `fork`'s inherited-state hazards. The honest staff-level guidance: `spawn` is the safer, more predictable choice and is the modern default on several platforms; if you use multiprocessing, prefer `spawn`, and be aware that the start method is a real setting with real failure modes attached, not an implementation detail to ignore.

For most multiprocessing work you do not manage `Process` objects directly. You use a **pool** — a `multiprocessing.Pool`, or, better, the unified interface of the next section — which manages a set of worker processes for you, hands them tasks, and collects results, hiding the process lifecycle.

## 13.8 The Unified Interface: `concurrent.futures`

The `threading` and `multiprocessing` modules have different APIs, and switching a piece of code from one to the other historically meant rewriting it. The standard library solves this with `concurrent.futures` — a *single, unified interface* over both, and the recommended way to do most threading and multiprocessing.

It provides two interchangeable executors. A **`ThreadPoolExecutor`** manages a pool of *threads*. A **`ProcessPoolExecutor`** manages a pool of *processes*. They present the *identical interface* — you submit work and get back results the same way for both — so the only thing that changes between "I/O-bound, use threads" and "CPU-bound, use processes" is which executor class you instantiate. One line.

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# I/O-bound work — threads
with ThreadPoolExecutor(max_workers=8) as pool:
    results = list(pool.map(fetch_url, urls))

# CPU-bound work — processes. Identical API; one class changed.
with ProcessPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(heavy_computation, datasets))
```

The interface gives you `submit` (schedule one callable, get back a **`Future`** — an object representing a result that is not ready yet, which you can later ask for its result), and `map` (apply a callable across an iterable, like the built-in `map` but executed concurrently across the pool). The `with` statement ensures the pool is properly shut down. A `Future`'s `.result()` call waits for and returns the result — and *re-raises any exception* the work raised, which means exceptions in pool workers are not silently lost; they surface when you collect the result.

The reason `concurrent.futures` is the recommended default is precisely its uniformity. It makes the model selection — the central skill of this chapter — into a *single, isolated, easily-changed decision*. You write your concurrent code once, against the unified interface; you choose `ThreadPoolExecutor` or `ProcessPoolExecutor` based on whether the workload is I/O-bound or CPU-bound; and if you got that judgment wrong, or the workload changes, you change one class name. For threading and multiprocessing, reach for `concurrent.futures` first; drop to the raw `threading` or `multiprocessing` modules only when you need a primitive the executors do not expose.

## 13.9 The Decision Procedure

Everything in this chapter converges on one decision, and here it is as an explicit procedure — the thing to run in your head, and in an interview, when you face a concurrency question.

**Step one — the first and most important question: is the workload CPU-bound or I/O-bound?** A CPU-bound workload spends its time *computing* — executing instructions, burning processor cycles (image processing, numerical simulation, parsing huge data, cryptography). An I/O-bound workload spends its time *waiting* — for the network, for disk, for a database, for another service. If you are not sure, *measure*: a quick profile (Chapter 15) tells you whether the time is in computation or in waiting. This question dominates everything else.

**Step two — if the workload is CPU-bound:** you need *multiprocessing* (a `ProcessPoolExecutor`), because only separate processes, each with its own GIL, give you true parallel execution of pure-Python computation. Threading and asyncio cannot help a CPU-bound pure-Python workload at all. (The two honest caveats: if the CPU-bound work is actually inside a *C extension that releases the GIL* — heavy NumPy operations, for instance — then threading *does* parallelize it, because the GIL is released during the C computation; and *free-threaded* Python, as it matures, will eventually make threading viable for pure-Python CPU work too. But for pure-Python CPU work on a standard GIL build, the answer is processes.)

**Step three — if the workload is I/O-bound:** you use *threading* or *asyncio*, and the choice between them turns on scale and on the surrounding code. Use a **`ThreadPoolExecutor`** when the number of concurrent operations is *moderate* — dozens, low hundreds — and especially when the I/O libraries you are calling are *synchronous* (ordinary blocking libraries), because threads work fine with blocking calls. Use **asyncio** when the number of concurrent operations is *very large* — many thousands of simultaneous connections — because a thread has a real memory and scheduling cost and ten thousand threads is impractical, whereas ten thousand coroutines on one event loop is routine; and use asyncio when you are working in an *already-async* codebase or with *async-native* libraries. The rough rule: moderate I/O concurrency with sync libraries, threads; massive I/O concurrency, or an async ecosystem, asyncio.

**Step four — a question that comes before all the others: do you need concurrency at all?** Concurrency is not free. It adds real complexity — race conditions, the difficulty of reasoning about interleaved execution, harder debugging, harder testing. A correct, simple, sequential program is *better* than a concurrent one that is faster but harder to understand and occasionally wrong, unless the performance difference genuinely matters. So before choosing a model, ask whether the simplest thing — just doing the work sequentially — is good enough. Often it is, and the most senior answer to "how would you make this concurrent" is sometimes "I would first check whether it needs to be."

That is the procedure: do you need concurrency at all; if so, CPU-bound or I/O-bound; CPU-bound means processes; I/O-bound means threads for moderate scale and async for massive scale or an async ecosystem. It is short, and running it correctly is the staff-level skill the whole chapter exists to build.

## 13.10 The Anti-Patterns

A few concurrency mistakes are common enough, and damaging enough, to name explicitly.

**Using threads for CPU-bound work.** An engineer has a slow computation, reaches for threads expecting a speedup, and gets none — or a slight slowdown — because the GIL serializes the bytecode and the threads cannot run the computation in parallel. The fix is *processes*. This is the single most common consequence of not understanding the GIL precisely.

**Sharing mutable state across threads without synchronization.** Two threads modifying the same data without a lock, producing the intermittent, irreproducible race-condition bugs of Section 13.6. The fix is to synchronize the access — or, better, to redesign so the state is not shared, communicating through a queue instead.

**Blocking the event loop.** In asyncio, calling a *blocking* synchronous operation directly inside a coroutine. Because asyncio is a *single thread*, a blocking call freezes the *entire* event loop — every other coroutine stops, the whole program stalls — until the blocking call returns. This is the cardinal asyncio sin, and Chapter 14 covers both the failure and the fix (`run_in_executor`) in depth; it is named here so the catalog is complete.

**Reaching for concurrency reflexively.** Adding threads or processes or async to a program that did not need them — paying the full complexity cost of concurrency to solve a performance problem that was not actually there, or that a simpler change would have solved. The fix is Step four of the decision procedure: ask whether sequential is good enough first.

**Unbounded concurrency.** Spawning a thread, a process, or a task *per item* with no limit — ten thousand items, ten thousand threads — exhausting memory or overwhelming a downstream service. The fix is a *bounded* pool (the `max_workers` of an executor) or a semaphore that caps the number of operations in flight.

> ### War Story — The Threads That Did Not Help
>
> A team had a data-processing job that had grown slow — it performed a heavy numerical-and-parsing computation over a large batch of records, and it took hours. An engineer was asked to speed it up. They knew the machine had many cores, they knew the job was "slow," and they reached for the obvious tool: they rewrote the job to use a pool of threads, one per core, expecting a roughly N-times speedup.
>
> The threaded version was not faster. It was, if anything, very slightly *slower* than the original. The engineer was baffled — the machine clearly had idle cores, the work was clearly divisible, the threads were clearly running — and spent a frustrating stretch of time convinced something was misconfigured.
>
> The cause was the GIL, understood imprecisely. The job was *CPU-bound* — it was a pure-Python computation, constantly executing bytecode, almost never waiting. The GIL guarantees that only one thread executes Python bytecode at a time, so the pool of threads could not run the computation in parallel; they simply took turns holding the GIL, achieving the throughput of a single thread, minus the overhead of passing the GIL around. The idle cores stayed idle because the GIL would not let more than one thread *use* a core for Python bytecode at once. Threading had been exactly the wrong tool, applied with confidence, because the engineer's model of the GIL was the folklore one — "Python has threads" — rather than the precise one — "the GIL makes threads useless for CPU-bound work."
>
> The fix was to switch the model: a `ProcessPoolExecutor` in place of the thread pool — a one-class change, because the code had (fortunately) been written against `concurrent.futures`. Each process had its own interpreter and its own GIL, the computation genuinely ran in parallel across the cores, and the job sped up close to the number of cores. The lesson is the chapter's spine. The concurrency model is not a detail; choosing it correctly *is* the work, and choosing it correctly requires the precise model of the GIL, not the folklore. The first question — CPU-bound or I/O-bound — would have produced the right answer immediately, and skipping it cost a day.

> ### The Level Line — Concurrency Models
>
> **A senior engineer** can write threaded and multiprocessing code, knows the GIL exists and roughly that it limits CPU-bound threading, uses locks to protect shared state, and uses `concurrent.futures`.
>
> **A staff engineer** knows the *precise* claim the GIL makes and *why* it exists (reference counting); knows the GIL is released during I/O and in many C extensions, and therefore that threading genuinely helps I/O-bound work; can run the CPU-bound-versus-I/O-bound decision procedure and choose correctly among threading, multiprocessing, and asyncio with reasons; understands race conditions, the synchronization primitives, deadlock and contention, and the multiprocessing serialization cost and start methods; and knows the status of free-threaded Python without over-betting on it.
>
> **A principal engineer** designs the concurrency architecture of a system — choosing models deliberately, minimizing shared mutable state by design rather than guarding it after the fact, bounding concurrency, and keeping the model choice isolated and changeable; asks whether concurrency is needed at all before adding it; tracks the free-threading transition and factors its trajectory into long-lived design; and teaches the precise GIL model so the team stops making the folklore mistake.

## 13.11 At the Interview Table

The GIL is one of the most frequently asked Python interview questions at every level, and concurrency model-selection is a staple of system-design and deep-dive rounds. The depth of the answer is the signal.

**The question: "Explain the GIL."**

A *senior* answer: "The Global Interpreter Lock is a lock in CPython that means only one thread runs Python code at a time, so threads don't give you parallelism for CPU-bound work — you use multiprocessing for that."

A *staff* answer adds precision and the I/O exception: "It's a mutex that ensures only one thread executes Python *bytecode* at any moment. The crucial precision is two things the folklore omits. First, Python *does* have real OS threads — the GIL constrains parallelism of bytecode execution, not the existence of threads. Second, the GIL is *released* during I/O and during many C-extension operations like NumPy's — so while a thread waits on the network, other threads run. That's why threading is useless for CPU-bound pure-Python work, where threads just serialize on the GIL, but genuinely useful for I/O-bound work, where threads spend most of their time not holding it."

A *principal* answer adds the *why* and the *future*: "It exists because of reference counting — every object has a refcount touched constantly, and concurrent unsynchronized updates to those counts would be a data race that corrupts memory. The GIL makes refcounting safe by serializing bytecode. The alternative — atomic refcounts everywhere — imposes a pervasive tax on all code including the common single-threaded case, so for decades one coarse lock was judged the better trade. That's now being revisited: PEP 703's free-threaded build removes the GIL, solving the refcount problem by other means. It's real but a transition in progress — there's a single-threaded cost and the C-extension ecosystem is still catching up — so I track it but don't yet build on it."

**The question: "I have a slow task. How do I speed it up with concurrency?"**

The strong answer *runs the decision procedure out loud*: "First I'd ask whether it's CPU-bound or I/O-bound — if I'm not sure, I'd profile to find out, because that question determines everything. If it's CPU-bound — a real computation burning cycles — I need multiprocessing, a `ProcessPoolExecutor`, because only separate processes escape the GIL and run pure-Python computation in parallel; threads would not help at all. If it's I/O-bound — mostly waiting on network or disk — then threads or asyncio: a `ThreadPoolExecutor` for moderate concurrency with synchronous libraries, asyncio if I need many thousands of concurrent operations or I'm in an async codebase. And before any of that, I'd ask whether it needs to be concurrent at all — concurrency adds real complexity and a simple sequential program is often good enough." That answer demonstrates the entire chapter.

**The question: "What is a race condition, and how do you prevent one?"**

"A race condition is when two threads access shared mutable data, at least one of them modifying it, without synchronization — and the result depends on the unpredictable interleaving of their execution, so it's intermittently wrong. The classic case is two threads incrementing a shared counter: the increment is really read-add-write, three steps, and if one thread reads before the other writes back, an increment is lost. Note the GIL doesn't save you — it serializes individual bytecode instructions, but the increment is several of them. The fix is synchronization — a `Lock` around the read-modify-write makes it atomic with respect to other threads. But the deeper fix is to *not share mutable state*: have threads communicate by passing messages through a `queue.Queue` instead, which removes the shared state entirely and is far less bug-prone."

**The red flags.** The folklore answer — "Python can't do threads" — with no awareness of the I/O exception. Recommending threading for a CPU-bound workload. Not knowing *why* the GIL exists (reference counting). Thinking the GIL makes `counter += 1` atomic. Being unaware that free-threaded Python exists — or, conversely, over-claiming that the GIL is "already gone." And, on a design question, jumping to a concurrency model without first asking the CPU-bound-versus-I/O-bound question.

## 13.12 The Forge

**Drill 13.1.** For each workload, state whether it is CPU-bound or I/O-bound and which concurrency model you would choose, with one sentence of reasoning: resizing ten thousand images; fetching five hundred URLs; serving fifty thousand simultaneous WebSocket connections; parsing a very large XML file; running a numerical simulation; querying a database two hundred times.

**Drill 13.2.** Demonstrate the GIL empirically. Write a CPU-bound function and run it (a) sequentially, (b) across a `ThreadPoolExecutor`, (c) across a `ProcessPoolExecutor`. Measure all three. Confirm threading gives no speedup and processes do, and write one paragraph explaining the result in terms of the GIL.

**Drill 13.3.** Demonstrate a race condition. Write two threads that each increment a shared counter a hundred thousand times *without* a lock, and show the final value is reliably wrong. Then add a `Lock` and show it correct. Explain why the GIL did not prevent the bug.

**Build 13.1.** Write the same I/O-bound task — fetching a list of URLs, say — three ways: with a `ThreadPoolExecutor`, with `asyncio`, and sequentially. Benchmark all three across different numbers of URLs (10, 100, 1000). Plot or tabulate the results and write up where each model wins and why.

**Build 13.2.** Build a producer/consumer system using threads and a `queue.Queue`: one or more producer threads generate work items, one or more consumer threads process them, and *no mutable state is shared directly* — all communication goes through the queue. Demonstrate it is correct under load, and write a paragraph on why the queue-based design is less bug-prone than a shared-state-with-locks design.

**Investigate 13.1.** Find out whether a free-threaded build of Python is available to you (the `--disable-gil` / free-threaded build). If so, run the Drill 13.2 CPU-bound benchmark under both a standard build and a free-threaded build, with threads, and report the difference. If not, research and write up the current status of free-threaded Python — its performance characteristics and the state of C-extension support.

**Investigate 13.2.** Construct a deadlock: two threads, two locks, each thread acquiring them in the opposite order. Reproduce the hang. Then fix it with lock ordering — both threads acquiring the locks in the same global order — and show the deadlock is gone. Write up why lock ordering works.

**Design 13.1.** You are designing a service that must do several different kinds of work: it serves a large number of concurrent HTTP requests (each mostly waiting on a database), it runs a periodic heavy CPU-bound report-generation job, and it processes a stream of incoming messages that each require a moderate amount of computation plus some I/O. Write a one-to-two-page design specifying the concurrency model for *each* kind of work, with the reasoning; how the models coexist in one service; how concurrency is bounded; and where, if anywhere, you would reconsider once free-threaded Python matures. This is the staff-level exercise — the chapter's decision procedure applied to a realistic, mixed workload.

---

[← Chapter 12: Architecture and Design Patterns](12-architecture-and-patterns.md) · [Home](README.md) · [Chapter 14: Asyncio in Depth →](14-asyncio-in-depth.md)
