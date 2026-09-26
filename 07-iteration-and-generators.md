# Chapter 7 — Iteration, Generators, and the Lazy Mindset

*Part II — The Language*

[← Chapter 6: Objects and Classes](06-objects-and-classes.md) · [Home](README.md) · [Chapter 8: Errors and the Discipline of Failure →](08-errors-and-failure.md)

---

## 7.1 The Question Behind the Chapter

Here is a problem that sounds routine and is quietly a trap. You are asked to process a hundred-gigabyte log file — read every line, transform it, count something. The obvious code is `lines = open("huge.log").readlines()`, then a loop. That code is correct, it passes on the small test file, and it kills the process in production when `readlines()` tries to pull a hundred gigabytes into a machine with sixteen of memory.

The fix is not a bigger machine. The fix is a different *way of thinking* — the lazy mindset — and this chapter is about installing it. Lazy evaluation means describing a computation without performing it all at once: producing values one at a time, on demand, so that a hundred-gigabyte file flows through your program one line at a time and never occupies more than a line's worth of memory. The generator is the tool, but the tool is the small part. The large part is the *mindset*: an engineer who reaches for laziness by default writes code that is more memory-efficient, more composable, and often clearer — and an engineer who does not will keep writing `readlines()` and keep being surprised.

This chapter builds that mindset. We start with the iterator protocol — the foundation everything rests on — then generators as the easy way to make iterators, then `itertools` as a toolkit of composable lazy pieces, then the historical line from generators to `async`/`await` (because the history *is* the explanation), and finally context managers, which are a close cousin worth knowing well.

## 7.2 The Iterator Protocol

`for item in thing` is the most common loop in Python, and almost no one knows what it actually does. It is worth knowing, because the whole chapter is built on it and because the knowledge dissolves a category of confusion.

Two concepts, precisely distinguished. An **iterable** is anything you can iterate over — anything you can put after `in` in a `for` loop. A list is iterable, a string is iterable, a dict is iterable, a file is iterable. An iterable is an object that implements `__iter__`, a method that returns an *iterator*. An **iterator** is the object that actually does the work of producing values one at a time. It implements `__next__`, a method that returns the next value — and, when there are no more, raises a specific exception, `StopIteration`.

So when you write `for item in some_list`, Python does this, mechanically: it calls `some_list.__iter__()` to get an iterator; then it repeatedly calls that iterator's `__next__()`, binding each returned value to `item` and running the loop body; and when `__next__()` raises `StopIteration`, the loop ends. The `for` statement is syntactic sugar over exactly that — get an iterator, call `__next__` until `StopIteration`. Every `for` loop you have ever written is that protocol underneath.

The iterable-versus-iterator distinction is not pedantry; it has an observable consequence that confuses people who do not know it. A list is iterable but *not its own iterator* — each call to `iter(my_list)` produces a *fresh* iterator starting at the beginning, which is why you can loop over a list twice and both loops see all the elements. But many iterators *are* their own iterator and are *single-use* — once exhausted, they are done. A file object is like this. A generator (next section) is like this. This is the source of the classic bug: you iterate something once, it works; you try to iterate it again, and the second loop sees nothing, because the iterator was consumed and is not reusable. Knowing the distinction — iterable can be re-iterated, an exhausted iterator cannot — is knowing why that bug happens and is the difference between a senior and a staff understanding of the loop.

## 7.3 Generators: Lazy Iterators, Made Easy

You could implement the iterator protocol by hand — write a class with `__iter__` and `__next__`, manually track state, manually raise `StopIteration`. It works and it is tedious. The *generator* is Python's tool for getting an iterator without any of that ceremony.

A **generator function** is a function with `yield` in its body. The presence of `yield` changes the function fundamentally — and this is the conceptual leap, so take it slowly. A normal function, when called, runs to completion and returns a value. A generator function, when called, **runs none of its body**. It immediately returns a *generator object* — an iterator — with the body not yet executed. The body runs only as the generator is iterated, and it runs *in pieces*: each call to `__next__` runs the body until it hits a `yield`, at which point the function **pauses** — its entire state frozen, local variables intact, position in the code remembered — and the yielded value is handed back to the caller. The next `__next__` *resumes* the function from exactly where it paused, with all its state restored, until the next `yield`. When the function body finally ends, `StopIteration` is raised automatically.

That ability — to pause a function mid-execution, keep all its state, and resume later — is the entire magic, and it is what makes laziness easy:

```python
def count_up_to(limit):
    n = 1
    while n <= limit:
        yield n              # pause here, hand back n, resume here next time
        n += 1

counter = count_up_to(5)     # nothing runs yet; counter is a generator object
print(next(counter))         # runs to first yield -> 1
print(next(counter))         # resumes, runs to next yield -> 2
# ... and so on, until StopIteration after 5
```

Now the lazy-mindset payoff, the answer to the chapter's opening problem. Compare two ways to process that hundred-gigabyte file:

```python
# Eager — pulls the entire file into memory. Dies on a large file.
def get_lines_eager(path):
    with open(path) as f:
        return f.readlines()          # a list of ALL lines, all at once

# Lazy — yields one line at a time. Constant memory, any file size.
def get_lines_lazy(path):
    with open(path) as f:
        for line in f:
            yield line.strip()        # one line, then pause
```

The eager version builds a list holding every line — for a hundred-gigabyte file, a hundred gigabytes of list. The lazy generator holds *one line at a time*: the consumer asks for a line, the generator reads one, yields it, and pauses; the line is processed and discarded; the consumer asks again. Peak memory is one line, whether the file is one megabyte or one terabyte. The eager version's memory grows with the input; the lazy version's is constant. *That* is the lazy mindset, and it is why a generator is not just a convenience but, for large or unbounded data, a correctness requirement.

There is also the **generator expression** — a comprehension in parentheses instead of brackets — which is a lazy comprehension:

```python
squares_list = [x*x for x in range(1_000_000)]    # builds a million-element list now
squares_lazy = (x*x for x in range(1_000_000))    # builds nothing; yields on demand
```

The list comprehension materializes a million numbers immediately. The generator expression produces a generator that yields each square as asked. When you are going to consume something once and do not need it to persist, the generator expression is the memory-cheap choice — and it composes, which the next sections develop.

A generator also has a lifecycle beyond `__next__`: `send` can pass a value *into* a paused generator, `throw` can raise an exception at the pause point, and `close` can terminate it. These matter most for generator-based coroutines (Section 7.5) and are the seam where this material meets `async`. For ordinary iteration you rarely need them, but knowing they exist explains how generators became the foundation of async Python.

## 7.4 `itertools`: Composable Lazy Pieces

If generators are how you *make* lazy iterators, `itertools` is the standard library's box of pre-made ones — a set of building blocks, all lazy, all composable, all implemented efficiently in C. The point of `itertools` is not the individual functions; it is that they *chain*: the output of one is the input of the next, and because every piece is lazy, an entire pipeline of them processes data one element at a time, with constant memory, no matter how many stages.

The members worth knowing, grouped by what they do:

**Infinite generators:** `count(start, step)` counts forever; `cycle(iterable)` repeats an iterable forever; `repeat(value)` yields a value forever. They are usable precisely *because* the whole ecosystem is lazy — an infinite iterator is fine as long as something downstream stops taking from it.

**Combining iterables:** `chain(a, b, c)` yields from several iterables in sequence as if they were one; `zip_longest` zips but pads to the longest input rather than stopping at the shortest.

**Filtering and slicing:** `islice` takes a slice of an iterator *without materializing it* — `islice(huge_generator, 10)` gives the first ten elements lazily, where `list(huge_generator)[:10]` would build the whole list first; `takewhile` and `dropwhile` yield elements while or after a condition holds; `filterfalse` is the inverted `filter`.

**Grouping and mapping:** `groupby`, `accumulate`, `starmap`.

**Combinatorics:** `product`, `permutations`, `combinations` — the Cartesian product and the arrangements, generated lazily one at a time.

Two of these carry traps sharp enough that a staff engineer must know them, because they are exactly the kind of thing that passes a small test and fails in production.

**`groupby` requires its input to be already sorted by the grouping key.** It does not gather all matching elements from across the whole input the way SQL `GROUP BY` does — it only groups *consecutive* runs of equal keys. Hand it unsorted data and it will happily produce broken groups (the same key appearing as several separate groups) with no error. The fix is to `sorted()` the input by the same key first. The bug, when it happens, is silent and data-dependent — the classic profile of an expensive one.

**`tee` can quietly explode memory.** `itertools.tee` splits one iterator into several independent ones. It sounds cheap and lazy, and it is lazy — but if the copies are consumed at very different rates, `tee` must *buffer* every element produced since the slowest copy last read it. Consume one copy fully and then start the other, and `tee` has buffered the entire iterator in memory — defeating the whole point of laziness. `tee` is fine when the copies advance roughly together; it is a memory bomb when they do not.

Both traps share a lesson: lazy tools are powerful but they have *operational characteristics*, and "it is lazy so it is memory-safe" is not always true. The mindset is not "laziness for free" — it is "laziness with awareness of where the buffering hides."

## 7.5 From Generators to `async`/`await`

The history of how Python got `async` is not trivia — it is the clearest possible explanation of what `async` *is*, so this section tells it briefly.

Recall the deep capability of a generator: it can *pause* mid-execution, holding all its state, and *resume* later. Now ask what that capability is good for beyond producing sequences of values. If a function can pause itself, then a function that is *waiting* — waiting for a network response, a database reply, a file read — could pause at the point of the wait, let *other* code run while it waits, and resume when the result is ready. One thread, cooperatively switching between many paused-and-waiting functions, none of them blocking the others. That is *concurrency without threads*, and the pause-and-resume machinery of generators is exactly what it needs.

Python's developers saw this. The early versions of asynchronous Python were built *literally on generators* — `yield` was repurposed as the "pause here and wait" point, and a decorator (`@asyncio.coroutine`) marked the generators that were really coroutines. It worked, but it was confusing: `yield` meant two completely different things depending on context, and a generator-that-produces-values looked identical to a generator-that-is-really-a-coroutine.

So Python 3.5 introduced *dedicated syntax* — `async def` and `await` — that does the same thing with its own keywords. `async def` defines a *coroutine*, a function built for the pause-and-resume-while-waiting pattern. `await` is the pause point — "pause here, wait for this to finish, let other code run meanwhile, resume when it is ready." Under the hood the machinery is still the generator machinery — the ability to suspend a function and resume it — but the syntax is now distinct and the intent is unambiguous.

That is the whole conceptual content of `async` and it is why this chapter contains it: **`async`/`await` is the pause-and-resume capability of generators, given dedicated syntax and pointed at the problem of waiting for I/O.** A coroutine is a special kind of pausable function; `await` is where it pauses. This chapter establishes that idea; **Chapters 13 and 14 build the full machinery on top of it** — the event loop that schedules all those paused coroutines, tasks, cancellation, the whole concurrency model. You are not learning async twice; you are learning the *idea* here, in its natural home alongside generators, and the *system* there.

Async iteration is the same idea applied to iteration itself. An `async def` function with `yield` is an **async generator** — an iterator whose production of each value may involve `await`ing something — and `async for` consumes it. This is the right tool for streaming from an asynchronous source: paginated API results, a streaming database cursor, messages off a queue. The one trap to flag now, developed in Chapter 14: async generators have *cleanup* concerns — if an async generator holds a resource and is not fully consumed, ensuring its cleanup code runs requires care (`aclose`, or structured patterns). Lazy plus async multiplies the subtlety; for now, just know the async generator exists and is the streaming-from-async tool.

## 7.6 Context Managers: Code That Wraps Code

Context managers are the chapter's last topic, and they belong here for a precise reason: a context manager, like a decorator and like a generator-coroutine, is *code that wraps other code* — it runs something before, runs something after, and guarantees the after even when the before's body fails.

The motivating problem is resource cleanup. You open a file; you must close it. You acquire a lock; you must release it. You begin a transaction; you must commit or roll back. And the cleanup must happen *even if the code in between raises an exception* — a file left open on an error path is a leak, a lock left held is a deadlock. Doing this with `try`/`finally` by hand is correct but verbose and easy to forget.

The `with` statement is the tool, and it rests on a two-method protocol — recall the dunders from Chapter 6. An object is a context manager if it implements `__enter__` (run on entering the `with` block; its return value is what `as` binds) and `__exit__` (run on leaving the block, *whether normally or by exception*). `with open(path) as f:` works because file objects implement this protocol — `__enter__` returns the file, `__exit__` closes it — and `__exit__` runs on the way out no matter what, so the file is closed on the happy path and on every error path alike.

You can write context managers as a class with those two methods, but the common, clean way is the generator-based form, using `contextlib.contextmanager` — and notice that this *reuses the generator's pause-and-resume*, tying this section back to the chapter's spine:

```python
from contextlib import contextmanager
import time

@contextmanager
def timed_block(label):
    start = time.perf_counter()
    try:
        yield                                    # <-- the with-block body runs here
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label}: {elapsed:.4f}s")

with timed_block("data load"):
    load_the_data()
```

Read the control flow against the generator model. Everything before `yield` is the setup — it runs on entering the `with`. The `yield` is where the `with` block's body executes — the generator *pauses* there, exactly as a generator pauses, and the body runs in that pause. When the body finishes (or raises), the generator *resumes* past the `yield`, and the `finally` runs the teardown. It is the generator's pause-and-resume, repurposed: pause to let the wrapped code run, resume to clean up. The `try`/`finally` ensures the teardown runs even if the body raised — the same guarantee the class-based `__exit__` gives.

Two pieces of `contextlib` round this out and are worth knowing by name. `ExitStack` manages a *dynamic* number of context managers — when you need to open an unknown-at-write-time number of resources and guarantee all of them are cleaned up, `ExitStack` is the tool. `nullcontext` is a do-nothing context manager, useful as a placeholder when a code path conditionally needs a real context manager and the other path needs nothing — it lets both paths use the same `with` statement.

> ### War Story — The File Handles That Ran Out
>
> A service processed uploaded files. The processing function opened each file, did its work, and closed it — `f = open(path)`, work, `f.close()`. Plain, and it passed every test.
>
> The `close()` was not in a `finally`. On the happy path the file closed. But the processing work could raise — a malformed upload, an unexpected encoding — and on that exception path, execution jumped out of the function *before reaching `f.close()`*. The file handle leaked. One leaked handle is invisible. But the service ran for weeks between deploys, and a small fraction of uploads were malformed, and each one leaked a handle. Operating systems cap the number of open file descriptors a process may hold. Eventually the service hit that cap, and then *every* operation that needed a file descriptor — including opening *valid* uploads, including, on some systems, accepting network connections — failed at once. A service that had been processing files fine for weeks fell over completely, and the proximate error (`Too many open files`) pointed nowhere near the malformed-upload code path that had been slowly leaking for weeks.
>
> The fix was to replace `f = open(path)` ... `f.close()` with `with open(path) as f:`. The `with` statement's `__exit__` closes the file on *every* exit path, including the exception path the manual code missed. The lesson: resource cleanup that is not guaranteed against exceptions is not cleanup — it is a leak waiting for an error path to be taken. `with` exists precisely so that the guarantee is automatic, and the rule is simply that anything with a lifecycle — files, locks, connections, transactions — is acquired with `with`, never by hand.

> ### The Level Line — Iteration and Laziness
>
> **A senior engineer** uses `for` loops, comprehensions, and generators correctly; knows generator expressions save memory; uses `with` for files and locks.
>
> **A staff engineer** holds the iterator protocol explicitly — iterable versus iterator, why an exhausted iterator is single-use; reaches for laziness *by default* for large or unbounded data, treating it as a correctness concern not an optimization; knows the `itertools` toolkit and its traps (`groupby` needs sorted input, `tee` can buffer unboundedly); understands `async`/`await` as the generator pause-and-resume capability applied to I/O waiting; writes context managers, including the generator-based form, and knows `ExitStack`.
>
> **A principal engineer** designs data flows as composable lazy pipelines so that memory use is bounded by construction rather than by luck; recognizes when an interface should yield rather than return; teaches the lazy mindset — and the generators-to-async connection — so the team's default reach is toward streaming rather than materializing.

## 7.7 At the Interview Table

Iteration and generators are favored interview ground because the difference between memorized syntax and a genuine lazy mindset shows immediately — most visibly on the "process data too big for memory" question, which is almost a standard.

**The question: "What's the difference between a list and a generator?"**

The shallow answer is "a generator is lazy." The answer that lands states the mechanism and the *consequence*: "A list holds all its elements in memory at once. A generator holds none of them — it produces elements one at a time, on demand, by pausing and resuming its function body. The consequences: a generator uses constant memory regardless of how many elements it will produce, so it can handle data far larger than memory, and even infinite sequences; but a generator is single-use — once iterated it's exhausted — and you can't index it or take its length without consuming it. So I choose a generator when the data is large, or I'll consume it once, or it's a pipeline stage; a list when I need random access, length, or to iterate it more than once."

**The question (near-standard): "How would you process a file too large to fit in memory?"**

This question *is* the lazy mindset, and the interviewer is checking whether you have it. "I'd never call `readlines()` — that materializes the whole file. I'd iterate the file object directly, which yields one line at a time, or write a generator that does, so peak memory is one line regardless of file size. And I'd build the whole processing chain as lazy stages — a generator that reads lines, feeding a generator that transforms them, feeding one that filters — so the entire pipeline processes one element at a time, end to end, constant memory. `itertools` helps compose those stages. If I needed batching — say, writing results a thousand at a time — I'd add a batching generator, never an accumulating list." A candidate who answers this way has demonstrated the chapter; one who reaches for `readlines()` and a list has demonstrated its absence.

**The question: "Explain `yield`. What does calling a generator function return?"**

"`yield` makes a function a *generator function*. Calling it runs *none* of the body — it immediately returns a generator object, an iterator. The body runs only as the generator is iterated, and in pieces: each `next()` runs the body until the next `yield`, then *pauses* the function with all its state — locals, position — frozen; the following `next()` resumes from exactly there. When the body ends, `StopIteration` is raised. That pause-and-resume is the whole mechanism, and it's also — historically — the foundation `async`/`await` was built on, since a coroutine is fundamentally a function that can pause while it waits for I/O."

**The question: "What's a context manager, and how does `with` relate to exceptions?"**

"A context manager is an object implementing `__enter__` and `__exit__` — `__enter__` runs on entering the `with` block, `__exit__` on leaving it. The key property is that `__exit__` runs *whether the block finishes normally or raises an exception* — so it's the right tool for cleanup that must happen on every path: closing files, releasing locks, rolling back transactions. Doing that by hand needs `try`/`finally` and is easy to get wrong — a `close()` not in a `finally` leaks on the error path. `with` makes the guarantee automatic. I can write a context manager as a class, or more cleanly as a generator decorated with `contextlib.contextmanager`, where the code before `yield` is setup and the code after is teardown."

**The red flags.** Reaching for `readlines()` or building a full list on the large-file question — the clearest sign the lazy mindset is missing. Not knowing a generator is single-use and being unable to explain the "second loop sees nothing" bug. Thinking `async` is unrelated to generators — a missed connection that signals shallow knowledge of both. And manual resource cleanup with no `finally` and no `with` — an engineer who has not yet been burned by a leak on an error path.

## 7.8 The Forge

**Drill 7.1.** Write a generator function `fibonacci()` that yields Fibonacci numbers forever. Use `itertools.islice` to print the first twenty without ever building an unbounded list. Then explain why an *infinite* generator is safe here when an infinite list would be impossible.

**Drill 7.2.** For each, predict the behavior and explain: iterating a list twice in two separate `for` loops; iterating a generator twice the same way; calling `len()` on a generator; calling `next()` on a generator that is already exhausted; using `itertools.tee` to make two copies of an iterator, fully consuming one, then consuming the other.

**Build 7.1.** Build a lazy log-processing pipeline as a chain of generators: one reads lines from a file; one parses each line into a structured record; one filters records by a predicate; one extracts a field. Compose them so the whole pipeline processes one line at a time. Demonstrate, by measuring memory, that processing a large file uses roughly constant memory regardless of file size.

**Build 7.2.** Write `batched(iterable, n)` — a generator that yields successive lists of up to `n` items from an iterable — without materializing the whole input. Then write a context manager `timed_section(label)` using `contextlib.contextmanager` that prints how long its block took. Use both together to process a large iterable in timed batches. (Note: Python 3.12+ has `itertools.batched`; write your own first, then compare.)

**Investigate 7.1.** Demonstrate the `groupby` trap. Take a list of records, run `itertools.groupby` on it *without* sorting first, and show that the grouping is broken — the same key appearing in multiple separate groups. Then sort by the key and show it correct. Write up exactly why `groupby` behaves this way and how SQL `GROUP BY` differs.

**Investigate 7.2.** Demonstrate the `tee` memory trap empirically. Create a large iterator, `tee` it into two, consume one of the copies completely while measuring memory, then consume the other. Show that memory grew to hold the whole iterator, and explain why in terms of `tee`'s buffering.

**Design 7.1.** You are designing the data-ingestion layer for a service that consumes records from several sources — a large file, a paginated HTTP API, a streaming database cursor — transforms them through several stages, and writes results in batches to a database. Some sources are synchronous, one is naturally asynchronous. Write a one-to-two-page design: where laziness is a correctness requirement versus a convenience; how the pipeline stages compose; where the async source fits and what the async generator's cleanup concern is; how batching is done without accumulating; and what the memory profile of the whole system is by construction. Justify the shape — do not just draw it.

---

[← Chapter 6: Objects and Classes](06-objects-and-classes.md) · [Home](README.md) · [Chapter 8: Errors and the Discipline of Failure →](08-errors-and-failure.md)
