# Chapter 4 — Built-in Data Structures

*Part II — The Language*

[← Chapter 3: Names, Scope, and Binding](03-names-scope-binding.md) · [Home](README.md) · [Chapter 5: Functions and Decorators →](05-functions-and-decorators.md)

---

## 4.1 The Question Behind the Chapter

Every engineer knows Python has lists, dicts, sets, and tuples. Most can use them competently. So why a chapter — a long one — on something so apparently elementary?

Because the gap between "can use" and "chooses correctly under pressure" is enormous, and it is exactly the gap this book exists to close. The choice of data structure is the single highest-leverage performance decision in most programs, and it is made *before* a profiler is ever run — it is made at design time, from knowledge, or it is made wrong. An engineer who reaches for a `list` and writes `if x in my_list` in a hot path has made an O(n) decision that no amount of later micro-optimization will rescue; the fix is a `set`, and the fix had to be a *decision*, not a discovery. An engineer who calls `my_list.pop(0)` in a loop has written O(n²) code that looks innocent; a `deque` makes it O(n), and again, only knowledge prevents it.

This chapter is about knowing these structures the way Part I taught you to know the machine — from the inside, by their implementation, so that their performance characteristics are not facts to memorize but consequences you can derive. By the end you will be able to look at a problem and know, before writing a line, which structure fits — and defend the choice. That defense, incidentally, is a staple of the staff-level interview, and we will rehearse it.

## 4.2 The List Is a Dynamic Array

A Python `list` is, underneath, a *dynamic array*: a contiguous block of memory holding pointers to objects, plus the machinery to grow that block when it fills. "Contiguous block of pointers" is the entire key to the list's performance profile, and everything else in this section is derivation from it.

Because the storage is a contiguous array indexed by position, **indexing is O(1)**. `my_list[5000]` is one pointer-arithmetic step — base address plus five thousand times eight bytes — regardless of list size. Same for assigning to an existing index. This is the list's great strength.

Because the storage is contiguous, **inserting or deleting at the front is O(n)**. `my_list.insert(0, x)` cannot just place `x` at the front; there is no room. Every existing element must shift one slot toward the back to make space. A million-element list means a million pointer-copies for one front insertion. `my_list.pop(0)` is the same cost in reverse — remove the front, shift everything down. This is why `pop(0)` in a loop is the quiet O(n²) trap: each `pop` is O(n), done n times.

Because the storage is an array with a tracked length, **appending to the end is O(1) amortized** — and the word "amortized" carries real content worth unpacking, because it is a frequent interview probe.

When a list's underlying array is full and you `append`, CPython must allocate a larger array and copy every element over — an O(n) operation. If it grew the array by exactly one slot each time, every single append would be O(n) and building a list of n elements would be O(n²). It does not do that. When CPython grows a list, it *over-allocates* — it asks for meaningfully more room than is immediately needed, growing the capacity by roughly an eighth plus a small constant. The exact factor is an implementation detail, but the *pattern* is the point: capacity grows geometrically, not by a fixed step. Geometric growth means the expensive resize-and-copy happens rarely and at exponentially spaced intervals, and the cost of all those copies, *spread across* (amortized over) all the cheap appends in between, averages out to O(1) per append. Any individual append might be O(n) — the one that triggers a resize — but the *average* over many appends is constant. That is what "amortized O(1)" means, and being able to explain *why* — geometric over-allocation spreading the rare expensive operation thin — is the difference between reciting the term and understanding it.

One practical corollary of over-allocation: a list that has grown large and then had most elements removed still holds the large underlying array. If memory matters, rebuild the list (`new = list(old)`) to get a right-sized array.

### Timsort: the list's sort

`list.sort()` and the built-in `sorted()` use an algorithm called *Timsort*, written for Python by Tim Peters and since adopted by other languages' standard libraries — Java's, for instance. It is worth a paragraph because it is a small lesson in real-world engineering.

Timsort is a hybrid: a merge sort fused with insertion sort, with a great deal of cleverness about exploiting structure that already exists in the data. Its worst case is O(n log n), the theoretical optimum for a comparison sort. But its *real-world* performance is often far better than that bound suggests, because real data is rarely random — it contains already-sorted stretches ("runs"), reversed stretches, and patterns. Timsort detects these runs and merges them, so partially-sorted data sorts much faster than fully-random data, sometimes approaching O(n). It is also *stable*: elements that compare equal keep their original relative order, which is what makes multi-key sorting work — sort by the secondary key, then by the primary, and the stability preserves the secondary ordering within primary-key groups.

The engineering lesson, beyond Python: a textbook-optimal algorithm and a *good* algorithm are different things. Timsort is optimal in the worst case *and* tuned for the inputs real programs actually have. That combination — provable bounds plus pragmatic tuning for real data — is what good systems work looks like, and it is a theme the book returns to.

## 4.3 The Dictionary Is the Crown Jewel

If the list is the workhorse, the `dict` is the crown jewel — and not only of Python's data structures. The dict is so central that the language is *built out of it*: a module's namespace is a dict, an object's attributes are (often) a dict, the global and built-in scopes are dicts. The keyword arguments to a function arrive as a dict. When you make dicts fast, you make Python fast, and CPython's dict has been optimized with corresponding intensity.

A dict is a *hash table*. The mechanism, in brief: to store a key, Python computes the key's hash — an integer, from the key's `__hash__` method — and uses that integer to choose a slot in an internal array. To look a key up, it computes the same hash, goes to the same slot, and finds the value. Because the slot is computed directly from the key rather than searched for, **lookup, insertion, and deletion are all O(1) on average.** That average-case constant time, across the three core operations, is the dict's reason for existing and the reason it is everywhere.

Two keys can hash to the same slot — a *collision*. CPython resolves collisions with *open addressing*: on a collision it probes a deterministic sequence of alternative slots until it finds a free one (for insertion) or the matching key (for lookup). As the table fills, collisions grow more frequent and probing grows longer, so when the table passes roughly two-thirds full, CPython allocates a larger internal table and re-inserts everything — a resize, analogous to the list's, and likewise amortized away across many cheap operations. The "O(1) average" comes with the standard hash-table asterisk: a pathological case where every key collides degrades to O(n). In practice, with well-behaved hash functions, it does not happen — though it is a real consideration for security, since adversarially-chosen colliding keys were once a denial-of-service vector, which is why CPython now randomizes string hashing per process.

### The ordering guarantee, and what it does and does not mean

Since Python 3.7, a dict *guarantees* that iterating it yields keys in insertion order. This was a real, specified language change — before 3.7 it was an implementation detail of CPython 3.6, and before that, dict order was simply undefined and you were warned never to rely on it.

This deserves precision because interviewers probe it. The guarantee came from a *memory optimization*, not from anyone setting out to make dicts ordered. The modern CPython dict is built from two arrays. One is a dense, compact array that stores the actual key-value entries in the order they were inserted. The other is the sparse hash table, which now holds not values but *indices* into that dense entry array. The motivation was memory: the sparse table — which must be kept two-thirds empty — now holds small indices instead of full entry structures, so the wasted space is much smaller. Insertion order being preserved fell out for free, because the entries array simply *is* in insertion order. The ordering guarantee is a side effect of a layout chosen for compactness.

The distinction an interviewer wants to hear: a `dict` is now an *insertion-ordered mapping*, but it is not an *ordered-map type* in the sense of a sorted tree — it does not keep keys in *sorted* order, and it has no efficient "give me the smallest key" operation. If you need keys kept in sorted order, a dict is the wrong tool; you want a different structure (the `sortedcontainers` library's `SortedDict`, or a different approach entirely). Insertion-ordered and sorted-ordered are different properties, and conflating them is a small but real tell.

## 4.4 Sets: Membership as a First-Class Operation

A `set` is, structurally, a dict that stores only keys — the same hash table, the same O(1)-average membership test, insertion, and deletion, with no associated values. `frozenset` is its immutable sibling: same behavior, but unchangeable after creation, and therefore *hashable*, and therefore usable as a dict key or as an element of another set.

The set exists for one job, and it is a job the list does badly: answering "is this element present?" For a `list`, `x in my_list` is O(n) — it walks the list comparing elements. For a `set`, `x in my_set` is O(1) average — it hashes `x` and checks one slot. This is not a marginal difference. On a collection of a million elements, the set membership test is roughly a million times faster than the list one. An engineer who writes a membership-heavy algorithm against a list has not written slow code; they have written code with the wrong *asymptotic complexity*, and that is a different and more serious category of mistake.

The rule that follows is simple and absolute enough to carry as an instinct: **if your code asks "is X in this collection?" and the collection is more than tiny, the collection should be a set.** The cost of building the set is O(n), paid once; it is repaid the first few times you would otherwise have scanned a list.

Sets also give you the algebra of their mathematical namesake — union, intersection, difference, symmetric difference — and these are not only expressive but *fast*, implemented in C over the hash tables:

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}
a & b      # {3, 4}        intersection
a | b      # {1,2,3,4,5,6} union
a - b      # {1, 2}        difference
a ^ b      # {1,2,5,6}     symmetric difference
```

A small but genuine subtlety, the kind that appears in interviews precisely because it separates the careful from the approximate: the *operators* (`&`, `|`, `-`, `^`) require both operands to be sets, while the equivalent *methods* (`a.intersection(b)`, `a.union(b)`, and so on) accept any iterable on the right. `my_set.intersection(my_list)` works; `my_set & my_list` raises a `TypeError`. When you want to intersect a set with something that is not a set, the method is the tool. Knowing that distinction is a marker of someone who has read the documentation rather than guessed from the operators.

## 4.5 The `collections` Module: Specialized Tools

The four built-ins cover most needs. The `collections` module covers most of the rest, and each of its types exists because a specific recurring pattern is awkward or slow with the built-ins. Knowing them is knowing that the awkward thing you are about to hand-code already exists, written in C, tested by millions of programs.

**`deque`** — a double-ended queue. The list's weakness, recall, was the front: `insert(0, x)` and `pop(0)` are O(n). The `deque` is built — as a doubly-linked structure of blocks — so that **append and pop at *both* ends are O(1)**. Any time you need a queue (append one end, pop the other) or need to push and pop at the front, the `deque` is the correct structure and the list is the bug. Its trade-off is the mirror of the list's: `deque` indexing in the middle is O(n), because there is no contiguous array to index into. Ends fast, middle slow — the exact opposite of the list, and you choose between them by where your operations land.

**`Counter`** — a dict subclass for counting. Counting occurrences of things is one of the most common small tasks in programming, and the hand-coded version (`if key in counts: counts[key] += 1; else: counts[key] = 1`) is verbose and easy to get subtly wrong. `Counter` does it in one line — `Counter(some_iterable)` — and adds genuinely useful operations on top: `.most_common(n)` for the top-n by count, and arithmetic between counters (`counter_a + counter_b` merges counts). Whenever the task is "how many of each," `Counter` is the answer.

**`defaultdict`** — a dict that supplies a default for missing keys by calling a factory function you provide. The classic use is grouping: building a dict-of-lists where each value accumulates items.

```python
from collections import defaultdict

groups = defaultdict(list)
for name, team in people:
    groups[team].append(name)      # no "if team not in groups" needed
```

The first time a given `team` is seen, `defaultdict` calls `list()` to create an empty list for it automatically, and `.append` proceeds. Without `defaultdict` this needs an explicit existence check on every iteration. A close relative for the same problem is the plain dict's `setdefault` method; `defaultdict` is usually cleaner when *every* access wants the default. (One sharp edge: merely *reading* `some_defaultdict[missing_key]` *creates* the key, with the default value. That is occasionally surprising. If you want to check for a key without creating it, use `in` or `.get()`.)

**`ChainMap`** — presents several dicts as one, searching them in order. Its natural home is layered configuration: command-line overrides, then environment, then a config file, then defaults — four dicts, queried as one, first match winning. It does this *without copying*, so it is cheap, and updates to the underlying dicts show through.

**`OrderedDict`** — historically *the* ordered dict, before the plain dict gained its ordering guarantee in 3.7. Today it is largely superseded, but not entirely: it still has a few methods the plain dict lacks (notably `move_to_end`), and its equality comparison is order-*sensitive* where a plain dict's is not. For new code, the plain dict is usually right; `OrderedDict` survives for those specific behaviors.

**`namedtuple`** — a tuple whose fields also have names, so `point.x` works alongside `point[0]`. It is a lightweight, immutable record type. In modern code it has largely been overtaken by `typing.NamedTuple` (a typed, class-syntax version) and by `dataclass`, which Chapter 6 covers properly — but `namedtuple` is still everywhere in existing code and still perfectly serviceable for a small immutable record.

## 4.6 `heapq` and `bisect`: Algorithms Over Lists

Two standard-library modules are not data structures themselves but *algorithms that operate on a plain list*, turning that list into something with new capabilities. They are easy to overlook and genuinely valuable.

**`heapq`** turns a list into a *binary heap* — a structure where the smallest element is always at index 0, retrievable in O(1), and where insertion (`heappush`) and smallest-removal (`heappop`) are O(log n). This is exactly the tool for a *priority queue*: a collection you repeatedly pull the smallest (or, with a sign trick, the largest) item from. It is also the engine behind `heapq.nlargest(n, iterable)` and `nsmallest(n, iterable)`, which find the top or bottom n items efficiently — far better than sorting the whole collection when n is small relative to the total. Any scheduling problem, any "process the most urgent item next" loop, any top-n computation is a `heapq` problem.

**`bisect`** maintains a list in *sorted order* and performs *binary search* on it. `bisect.bisect` finds, in O(log n), the index where a value would be inserted to keep the list sorted; `bisect.insort` performs that insertion. If you have a sorted list and need fast "where does this fall" or "is this present" queries, `bisect` provides them — with the caveat that `insort`'s insertion is still O(n) because of the list shift from Section 4.2, so `bisect` shines when reads dominate writes. It is also the clean tool for range bucketing: given sorted boundary values, `bisect` tells you which bucket a number falls into in logarithmic time.

## 4.7 Choosing: The Decision That Precedes the Profiler

Step back from the individual structures to the meta-skill, because the meta-skill is what the chapter is really teaching and what the interview is really testing.

Choosing a data structure is choosing an algorithm's *complexity class*, and it is done at design time, from knowledge. No profiler will tell you to change a list to a set — a profiler will tell you a function is slow, and you will then need to already know that the membership test inside it is O(n) and should not be. The profiler finds *where*; your knowledge of these structures supplies *why* and *what instead*. This is why the chapter insisted on internals: you cannot derive the right choice from memorized fact-lists, but you can derive it instantly from "a list is a contiguous array, a set is a hash table, a deque is fast at both ends."

A compact decision frame, the kind worth carrying into both real work and interviews:

- Need ordered elements, indexed by position, appending at the end — **list**.
- Need fast append *and* pop at the front, or a queue — **deque**.
- Need to test membership, or to deduplicate — **set**.
- Need to associate keys with values, with fast lookup — **dict**.
- Need an immutable record, or a hashable fixed sequence — **tuple** (or `namedtuple`/`dataclass` for named fields).
- Need to repeatedly extract the smallest or largest — **heapq** over a list.
- Need fast search or range queries over sorted data — **bisect** over a sorted list.
- Need to count occurrences — **Counter**.
- Need to group items into buckets — **defaultdict**.

> ### War Story — The Membership Check That Did Not Scale
>
> A data-processing job cross-referenced incoming records against a set of known identifiers — "for each incoming record, is its ID in the known set?" The known IDs were loaded into a `list`, because the engineer who wrote it loaded them from a file and a list is what you get from reading lines. The membership test was `if record_id in known_ids`.
>
> With a few thousand known IDs and a few thousand records, the job ran in seconds and shipped. A year later the known set had grown to several hundred thousand IDs and the daily input to a few million records. The job now took most of a day and was creeping toward not finishing before the next day's run began — the failure mode where a batch job quietly falls off a cliff.
>
> The "fix" was one line: `known_ids = set(known_ids)` after loading. The job went from hours to under a minute. Nothing else changed. The membership test had been O(n) all along — a few-thousand-element scan was invisible, a few-hundred-thousand-element scan multiplied by a few million records was a quarter-trillion comparisons. The structure had been wrong since the first day; it had simply been wrong at a scale small enough not to notice. The lesson the team took away was the one this chapter is built around: the data-structure choice *is* the complexity choice, it is made before any profiling, and "it was fast enough in testing" is not evidence that it was *correct* — only that it had not yet met the input that would expose it.

> ### The Level Line — Data Structures
>
> **A senior engineer** knows the four built-ins and the common `collections` types, and uses them correctly for everyday tasks; knows that a set is the right tool for membership tests.
>
> **A staff engineer** knows the *internals* — list as dynamic array, dict and set as hash tables, deque as a linked block structure — and therefore *derives* the complexity of any operation rather than recalling it; chooses structures deliberately at design time as complexity decisions; knows `heapq` and `bisect` and reaches for them when the problem fits; can explain "amortized O(1)" and the dict's two-array layout precisely.
>
> **A principal engineer** treats data-structure choice as an architectural lever — selecting representations, not just algorithms — anticipates the scale at which a structure choice will fail before it does, and teaches the derive-from-internals habit to the engineers around them so the right choice becomes the team's default rather than one person's expertise.

## 4.8 At the Interview Table

Data structures are interview bedrock at every level. The questions are concrete, the answers are checkable, and — crucially — the *internals* questions cleanly separate the engineer who memorized a complexity table from the one who understands why the table reads as it does.

**The question: "How is a Python dict implemented?"**

One of the most common staff-level Python questions, and a complete answer has several beats. The mechanism: "A dict is a hash table. Each key is hashed to an integer; the integer selects a slot in an internal array; lookup, insertion, and deletion are O(1) on average because the slot is computed from the key, not searched for." Collisions: "Two keys can land in the same slot; CPython uses open addressing — it probes a deterministic sequence of other slots. When the table gets about two-thirds full it resizes to keep probing short." The ordering: "Since 3.7, dicts preserve insertion order — guaranteed by the language. The modern dict is two arrays: a compact entries array in insertion order, and a sparse index array that holds positions into it. The layout was chosen for memory compactness; insertion-order preservation came along for free." And the precision that marks depth: "It's insertion-ordered, but not a *sorted* map — no efficient minimum-key operation. If you need sorted keys, a dict is the wrong structure."

**The question: "What's the difference between a list and a tuple, beyond mutability?"**

Beginners answer "tuples are immutable" and stop. The deeper answer: "Mutability is the headline, but it has consequences. Because a tuple is immutable it's *hashable* — it can be a dict key or a set element, which a list can't. Tuples signal *fixed structure* — a coordinate pair, a database row — where a list signals a *homogeneous, variable-length collection*; that's a semantic distinction good code respects. And tuples are slightly lighter in memory and construction, since they don't carry a list's growth machinery. So the choice isn't only 'do I need to mutate it' — it's also 'is this a record or a collection,' and 'does this need to be a key.'"

**The question: "Explain 'amortized O(1)' for list append."**

This is a probe of whether you understand a term you use. "Appending is usually O(1), but when the underlying array is full, append must allocate a bigger array and copy everything — that one append is O(n). The reason the *average* is still O(1) is that CPython grows the array *geometrically*, not by one slot — it over-allocates. So resizes happen rarely and at exponentially spaced intervals, and the total cost of all the copying, spread across all the cheap appends in between, averages to a constant per append. Any single append can be O(n); the amortized cost is O(1)." If you can say *why* — geometric growth spreading the rare expensive operation thin — you have shown understanding rather than recall.

**The question: "When would you use a `deque` over a `list`?"**

"When I need fast operations at the *front*. A list is a contiguous array — indexing is O(1) but inserting or popping at the front is O(n) because everything shifts. A deque is built for O(1) append and pop at *both* ends, so it's the right structure for a queue or anything front-heavy. The trade-off is that deque indexing in the middle is O(n) — no contiguous array — so if I need fast random access by index, the list wins. Ends versus middle is the deciding question."

**The red flags.** Reciting complexities with no ability to explain them — "append is O(1)" with no idea why, "dict lookup is O(1)" with no notion of a hash table. Not knowing that `pop(0)` on a list is O(n) — it is the single most common accidental-O(n²) bug. Believing the dict's ordering means it is *sorted*. And, on a design question, choosing a list for a membership-heavy task and not catching it — which signals an engineer who has not internalized that structure choice is complexity choice.

## 4.9 The Forge

**Drill 4.1.** For each operation, state the average-case complexity and one sentence of justification from the structure's internals: `list` indexing by position; `list.append`; `list.insert(0, x)`; `list.pop()`; `list.pop(0)`; `x in some_list`; `x in some_set`; `dict` lookup by key; `deque.appendleft`; `deque` indexing in the middle.

**Drill 4.2.** Without running it, predict the output and explain via the dict's internals: build a dict by inserting keys `c`, `a`, `b` in that order; print `list(d.keys())`. Then delete `a` and re-insert it; print the keys again. What does the result tell you about how insertion order interacts with deletion and re-insertion?

**Build 4.1.** Implement an LRU (least-recently-used) cache class with a fixed capacity, supporting O(1) `get` and `put`. Do it twice: once using `collections.OrderedDict` and its `move_to_end`; once using a plain `dict` plus a hand-built doubly-linked list. Write a paragraph comparing the two implementations — which is simpler, which exposes more of the mechanism, and what each teaches.

**Build 4.2.** Implement a function `top_k_frequent(items, k)` that returns the `k` most common elements of an iterable. Use `Counter`. Then implement it a second time using `heapq` directly without `Counter.most_common`. Benchmark both against a naive sort-the-whole-thing approach for large inputs and varying `k`, and explain the curves.

**Investigate 4.1.** Empirically measure the list's growth strategy. Append elements to a list one at a time, and after each append record `sys.getsizeof(the_list)`. Plot or tabulate size against length. Identify the points where the underlying array resized, and from the jumps, estimate the growth factor. Reconcile what you find with Section 4.2.

**Investigate 4.2.** Construct a dict whose keys are deliberately chosen (or whose key type has a deliberately bad `__hash__`) so that many keys collide. Measure lookup time as the number of colliding keys grows, and compare against a normal dict of the same size. Explain what you observe in terms of open addressing and probing.

**Design 4.1.** You are designing the in-memory core of a rate limiter: it must answer, for a given client ID, "has this client made more than N requests in the last 60 seconds?" — quickly, for many thousands of distinct clients, called on every incoming request. Write a one-page design: which data structure(s) hold the per-client request history, what the time and space complexity of a check is, how old entries are expired, and what trade-offs your choice makes. Consider at least two designs (for example, a per-client deque of timestamps versus a sliding-window counter) and justify your selection. Do not pick before you have compared.

---

[← Chapter 3: Names, Scope, and Binding](03-names-scope-binding.md) · [Home](README.md) · [Chapter 5: Functions and Decorators →](05-functions-and-decorators.md)
