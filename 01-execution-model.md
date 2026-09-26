# Chapter 1 — The Execution Model

*Part I — The Machine*

[← Front Matter](00-front-matter.md) · [Home](README.md) · [Chapter 2: Objects, Identity, and Memory →](02-objects-identity-memory.md)

---

## 1.1 The Question Behind the Chapter

Here is a question that sounds trivial and is not: what happens when you run a Python program?

The folk answer is "the interpreter reads the file and does what it says." That answer is not wrong, exactly, the way "the car moves because I press the pedal" is not wrong. It is just not a model you can *use*. It will not help you understand why one loop is three times faster than another that looks identical, why a syntax error in a function you never call still crashes your program at startup, or why the same code runs at different speeds on Python 3.10 and 3.12.

By the end of this chapter you will have a model you can use. You will have read the actual bytecode of a Python program, you will know what the `.pyc` files cluttering your directories are for, and you will understand — at least in outline — why Python 3.11 was a genuine performance release and what the JIT arriving in 3.13 and 3.14 does and does not change.

The model, stated in one sentence up front so you know where we are going: **Python is a stack-based virtual machine that executes bytecode, and almost every surprising behavior you will ever encounter is a direct, explicable consequence of that fact.**

## 1.2 From Source to Output: The Pipeline

When you run `python myprogram.py`, the source code does not go straight to the CPU. It passes through a pipeline of four stages.

**Stage one: tokenizing.** The raw text of your file — a stream of characters — is broken into *tokens*: the atomic units of the language. The text `x = 1 + 2` becomes the tokens `NAME(x)`, `OP(=)`, `NUMBER(1)`, `OP(+)`, `NUMBER(2)`, `NEWLINE`. Indentation, which is syntactically significant in Python, becomes explicit `INDENT` and `DEDENT` tokens here. This is also where a file with a stray tab-versus-spaces mix fails.

**Stage two: parsing.** The flat token stream is assembled into a tree — the *Abstract Syntax Tree*, or AST. The tree captures the *structure* of the program: that `1 + 2` is a binary operation whose operands are `1` and `2`, and that the whole thing is the right-hand side of an assignment to `x`. Since Python 3.9, CPython uses a PEG parser (a "parsing expression grammar"), which replaced the older LL(1) parser and made certain previously-awkward syntax — like the structural `match` statement — feasible to add.

The AST is a real object you can inspect. We will do exactly that in a moment.

**Stage three: compiling.** The AST is walked and *compiled* into *bytecode* — a flat sequence of simple instructions for a virtual machine. This is the stage most engineers do not know exists, and it is the one that explains the most. The bytecode is the actual thing that gets executed.

**Stage four: interpreting.** The bytecode is fed, one instruction at a time, to the *evaluation loop* — a large loop in C, living in a file called `ceval.c` in the CPython source, that reads each instruction and performs it. This loop is the beating heart of CPython.

Here is the critical reframing. When you "run Python," you are not running your source code. You are running bytecode that was compiled from your source code, on a virtual machine implemented in C. Your `.py` file is the blueprint; the bytecode is the thing that is built; the evaluation loop is the machine that executes it.

This is why a `SyntaxError` in a function you never call still stops your program before it produces any output: the *whole file* must be compiled to bytecode before *any* of it runs. Compilation is not lazy. A syntax error anywhere is a compilation failure everywhere.

## 1.3 Looking at the Machine: `dis` and `ast`

Abstractions you cannot inspect are abstractions you will not trust. So let us inspect.

Take the trivial program `x = a + b`. Here is its AST:

```python
import ast

source = "x = a + b"
tree = ast.parse(source)
print(ast.dump(tree, indent=2))
```

```
Module(
  body=[
    Assign(
      targets=[Name(id='x', ctx=Store())],
      value=BinOp(
        left=Name(id='a', ctx=Load()),
        op=Add(),
        right=Name(id='b', ctx=Load())))],
  type_ignores=[])
```

Read that tree. It says: the module's body is a single `Assign`. The assignment's target is the name `x`, in a `Store` context — we are storing into it. Its value is a `BinOp` — a binary operation — whose operator is `Add`, whose left operand is the name `a` in a `Load` context, and whose right operand is the name `b`, also `Load`. The `ctx` field — `Load` versus `Store` — is the AST already distinguishing *reading* a name from *writing* one. Hold that thought; it returns in Chapter 3.

Now the bytecode. The `dis` module — "disassemble" — shows it:

```python
import dis

source = "x = a + b"
code = compile(source, "<example>", "exec")
dis.dis(code)
```

```
  1           0 RESUME                   0
              2 LOAD_NAME                0 (a)
              4 LOAD_NAME                1 (b)
              6 BINARY_OP                0 (+)
             10 STORE_NAME               2 (x)
             12 LOAD_CONST               0 (None)
             14 RETURN_VALUE
```

(The exact output varies slightly by Python version — this is 3.12 — but the shape is stable.)

This is the machine laid bare. Read it instruction by instruction, and keep one fact in mind: **this is a stack machine.** There is a stack — a last-in-first-out pile of values — and almost every instruction pushes values onto it or pops values off it.

`RESUME` is bookkeeping; ignore it for now (it matters for generators, in Chapter 7). Then:

`LOAD_NAME 0 (a)` — look up the name `a` and push its value onto the stack. Stack now: `[a]`.

`LOAD_NAME 1 (b)` — look up `b`, push it. Stack: `[a, b]`.

`BINARY_OP 0 (+)` — pop the top two values, add them, push the result. Stack: `[a+b]`.

`STORE_NAME 2 (x)` — pop the top value, bind the name `x` to it. Stack: `[]`.

`LOAD_CONST 0 (None)` and `RETURN_VALUE` — every code block implicitly returns `None`; this is that.

That is the entire mechanism. A stack, and instructions that move values on and off it. Every Python program you have ever written is, at this level, just a longer version of this. The reason this matters is not trivia. It is that once you can see the bytecode, performance differences stop being mysterious. When we get to Chapter 4 and claim that a list comprehension is faster than the equivalent `for` loop with `.append()`, you will not have to take it on faith — you will be able to disassemble both and *count the instructions*. The comprehension has a dedicated, fused instruction for the append; the loop does not. The bytecode shows you exactly where the time goes.

## 1.4 The `.pyc` File: Caching the Compilation

Compilation — source to bytecode — takes real time. For a large program with many imported modules, it is not negligible. And it is wasteful: if the source has not changed, the bytecode will be identical every time.

So CPython caches it. The first time a *module* is imported, its compiled bytecode is written to disk in a `__pycache__` directory, in a file named something like `mymodule.cpython-312.pyc`. On the next import, if the source file is unchanged, CPython skips stages one through three entirely and loads the bytecode straight from the `.pyc`.

"If the source is unchanged" is the part that matters. How does CPython know? Historically, by comparing the source file's modification timestamp against a timestamp stored in the `.pyc`. Since Python 3.7 there is also an optional hash-based mode (PEP 552) that compares a hash of the source instead — more robust, because timestamps can lie.

And timestamps *do* lie. This is the chapter's first War Story.

> ### War Story — The Bug That Came Back
>
> A service had a bug. A engineer found it, fixed it, tested the fix locally, and deployed. The bug was gone. Three days later it was back — same stack trace, same behavior — with no deploy in between.
>
> The deployment process built the application into a container image. The build copied the source files in, and — crucially — the build also ran a step that touched some files, resetting modification times. An *older* `.pyc` file, from a layer cached earlier in the build, had a timestamp that the build process had inadvertently made *newer* than the fixed source. CPython compared timestamps, concluded the cached bytecode was current, and loaded the *old, buggy* bytecode. The fixed source sat right there on disk, never compiled, never run.
>
> The fix was twofold: set `PYTHONPYCACHEPREFIX` so cache files lived outside the source tree and could not be staled by it, and set `PYTHONDONTWRITEBYTECODE` in the container build so the image never carried `.pyc` files at all — they would be generated fresh, from the real source, on first run. The deeper lesson: a `.pyc` is a cache, every cache can be stale, and a stale cache is a class of bug that does not look like a bug. It looks like a haunting.

For day-to-day work the takeaway is small but real: if you ever see Python behavior that contradicts the source code in front of you — *delete the `__pycache__` directories and try again.* It is the Python equivalent of "turn it off and on," and like that advice, it is dismissed right up until the moment it is the answer.

## 1.5 CPython Is One Implementation

We have been saying "Python does this" and "Python does that." That is a useful shorthand and also a small lie. There is no single thing called Python that does anything. There is a *language specification*, and there are several *implementations* of it.

**CPython** is the reference implementation — the one you get from python.org, the one almost everyone means by "Python." It is written in C. It is the one this book is about, because it is the one you will deploy. Everything we have described — the bytecode, `ceval.c`, the stack machine — is CPython specifically.

**PyPy** is an alternative implementation written (mostly) in Python itself, with a *tracing just-in-time compiler*. For long-running, pure-Python, compute-heavy workloads it is often four to ten times faster than CPython. Its cost is C-extension compatibility — the vast NumPy-and-friends ecosystem is built against CPython's C API, and while PyPy emulates that API, the emulation has a performance cost and occasional gaps.

**GraalPy** runs on the Java virtual machine's GraalVM, with that platform's JIT. **MicroPython** is a tiny implementation for microcontrollers — Python on a device with kilobytes of RAM. **Jython** (on the JVM) and **IronPython** (on .NET) exist and are largely historical now.

Why does this matter to you? Two reasons. First, when someone says "Python is slow," the honest reply is "CPython is slower than C at pure-Python loops; PyPy often is not; and most real Python programs spend their time in C extensions where the question barely applies." Implementation matters. Second, the *language* and the *implementation* are different things, and a few behaviors you might think are "Python" are really "CPython." The small-integer cache we will meet in the next chapter is one. Relying on a CPython implementation detail as if it were a language guarantee is a real and avoidable bug.

## 1.6 The Specializing Adaptive Interpreter

Now to the most important recent change in how CPython runs, and the reason Python 3.11 was not just a normal release.

For most of CPython's history, the evaluation loop was *static*. The instruction `BINARY_OP +` did the same thing every time it ran: a fully general "add these two objects" operation, which has to check the types of both operands, dispatch to the right addition logic, and handle every possible case — two ints, two floats, two strings, a list and a list, a custom class with an `__add__` method. That generality is not free. Every single `+` paid the cost of being able to add anything to anything.

But here is the thing about real programs: a given `+` in a given line of code almost always adds the *same types* every time. A `+` inside a loop summing integers adds two ints on iteration one, and on iteration two, and on all million iterations. The generality is paid for and never used.

PEP 659, shipped in Python 3.11, exploits this. The interpreter is now *specializing* and *adaptive*. It watches each bytecode instruction as the program runs. When it notices that a particular `BINARY_OP +` has added two integers several times in a row, it *rewrites that instruction, in place,* to a specialized version — call it `BINARY_OP_ADD_INT` — that skips the general type-checking and dispatch and just adds two integers directly. The instruction has *adapted* to the data flowing through it.

If the types later change — if that `+` suddenly sees a string — the specialized instruction *deoptimizes*: it notices the mismatch and falls back to the general version. The system is adaptive in both directions.

This is most of why 3.11 was 10–60% faster than 3.10 across typical workloads, with no code changes required. But it has a consequence you should internalize, because it is a genuine shift in how to think about Python performance:

**Type stability is now a performance characteristic.** Code where each operation sees consistent types runs on specialized, fast instructions. Code that is wildly polymorphic — where the same line processes ints and floats and strings and custom objects in unpredictable sequence — keeps deoptimizing, never benefits from specialization, and runs on the slow general path. This is a large part of *why* NumPy is fast: not only because it drops into C, but because the data is rigidly, uniformly typed, and uniform types are exactly what the modern interpreter optimizes for. The same principle scaled down applies to your own hot loops.

## 1.7 The JIT, and What It Does Not Change

The specializing interpreter sets up the next step. Python 3.13 shipped, as an experimental and off-by-default build option, a *just-in-time compiler* — PEP 744. Python 3.14 continues that work toward making it a default.

CPython's JIT uses a technique called "copy-and-patch." Without the full machinery: it takes sequences of those specialized bytecode instructions and stitches together small pre-compiled chunks of machine code for them, so that hot code runs as native machine instructions rather than being interpreted instruction-by-instruction. The early performance gains are modest — single-digit to low-double-digit percentages — with more expected as the implementation matures over subsequent releases.

Here is the part to hold onto, because it is where engineers get the wrong idea. The JIT is an *incremental* improvement to interpreter overhead. It is not a transformation of Python into a fast language for the things Python is genuinely slow at. It does not change algorithmic complexity — an O(n²) algorithm JIT-compiled is still O(n²), now with a slightly faster constant factor. It does not eliminate the reasons people reach for NumPy or Rust extensions for serious numerical work. And it does not remove the value of the most important performance skill, which is *measuring before optimizing* and *fixing the algorithm first.*

What the JIT and the specializing interpreter together *do* mean is that the claim "Python is slow" needs an asterisk it did not used to need. CPython is getting meaningfully faster, release over release, for free. That is worth knowing. It is not worth betting your architecture on.

> ### The Level Line — The Execution Model
>
> **A senior engineer** can describe the pipeline at a high level — source becomes bytecode, the interpreter runs it — and knows that `.pyc` files are a compilation cache. They can use `dis` when they need to.
>
> **A staff engineer** has the stack-machine model firmly enough to reason from it: they can predict, and verify with `dis`, why one construct outperforms another, and they understand the GIL's place in this picture (Chapter 13). They know CPython is one implementation among several and do not confuse implementation details for language guarantees.
>
> **A principal engineer** understands the specializing interpreter well enough that it informs design — they know type stability is a performance lever, they can reason about what the JIT will and will not do for a given workload, and when a team is weighing a Python-version upgrade or an alternative implementation, they are the one who can frame the decision with real numbers instead of folklore.

## 1.8 At the Interview Table

The execution model is interview territory at every level, because it is a clean probe of depth. The same opening question — "walk me through what happens when you run a Python script" — is asked of a senior and a principal candidate, and the *depth of the answer* is the signal.

**The question: "What happens when you run a Python file?"**

A *senior-level* answer covers the pipeline at altitude: "Python reads the source, compiles it to bytecode, and the interpreter executes that bytecode. Bytecode gets cached in `.pyc` files so unchanged modules don't recompile." That is correct and sufficient for senior.

A *staff-level* answer adds the machine: "The source is tokenized, parsed into an AST, and compiled to bytecode. The bytecode runs on a stack-based virtual machine — the evaluation loop in `ceval.c`. It's worth saying that compilation isn't lazy: the whole file compiles before any of it runs, which is why a syntax error in an uncalled function still crashes startup." The interviewer is listening for the stack machine and for a *consequence* drawn from the model — the syntax-error point shows you can reason *from* the model, not just recite it.

A *principal-level* answer reaches the modern interpreter: "...and since 3.11 the interpreter is specializing and adaptive — PEP 659 — so hot instructions rewrite themselves to type-specialized fast paths. That's most of why 3.11 was a real performance jump, and it means type stability in hot code is now a performance characteristic. The 3.13-onward JIT builds on that, though it's an incremental constant-factor win, not a change to Python's complexity story." This answer demonstrates that you track the platform's evolution and translate it into design judgment.

**The follow-up: "Why is Python slower than C?"**

This question is partly technical and partly a *temperament* test — interviewers want to see whether you get defensive or answer like an engineer. The strong answer is matter-of-fact: "CPython interprets bytecode rather than executing native machine code, so there's per-instruction dispatch overhead. Every value is a heap-allocated object with a type and a refcount, so even integer arithmetic involves pointer indirection and allocation that C does in a register. And dynamic typing means operations resolve types at runtime. But two caveats: the gap is mostly about pure-Python tight loops — most real Python spends its time in C extensions like NumPy where it isn't slow — and the specializing interpreter and JIT are narrowing the interpreter-overhead part of the gap release over release."

**The red flags** an interviewer is watching for: claiming Python "compiles to machine code" (it compiles to bytecode); claiming it is "purely interpreted" with no compilation step at all (the bytecode compile is real); getting defensive about the speed question; or — the subtle one — reciting the specializing-interpreter material verbatim without being able to say *why it matters*, which reads as memorized rather than understood. The cure for the last one is the same as the cure for everything in this chapter: actually run `dis` on real code until the model is yours.

## 1.9 The Forge

**Drill 1.1.** For each of the following snippets, predict the bytecode *before* running `dis` on it, then check: `x = 5`; `x = 5 + 3` (note what the compiler does with two constants — this is "constant folding"); `print(x)`; `x = [i for i in range(3)]`. For the comprehension, identify which instruction does the per-element append.

**Drill 1.2.** Disassemble a `for` loop that builds a list with `.append()`, and disassemble the equivalent list comprehension. Count the instructions executed per element in each. Write one sentence explaining the difference.

**Build 1.1.** Write a function `count_ops(func)` that takes a function, disassembles it with `dis.get_instructions()`, and returns a `collections.Counter` of how many times each opcode appears. Use it on three functions of your choice and comment on what the counts reveal.

**Investigate 1.1.** Find a snippet of Python whose measured performance changed noticeably between two CPython versions (3.10 vs 3.12 is a good pair). Benchmark it on both with `timeit`. Then form a hypothesis — using this chapter's material — for *why* it changed, and find supporting evidence in the relevant PEP or release notes. There is no single right answer; the quality is in the reasoning.

**Design 1.1.** Your team runs a large Python service on Python 3.10. Leadership asks whether to invest a sprint in upgrading to the current release. Write a one-page framing of that decision: what the benefits plausibly are, what the risks and costs are, how you would *measure* whether the upgrade actually helped, and what would make you recommend for or against. Do not give a yes/no — give the decision framework. (This is the kind of question that separates a staff engineer from a senior one: the senior answer is "is it faster?", the staff answer is a framework.)

---

[← Front Matter](00-front-matter.md) · [Home](README.md) · [Chapter 2: Objects, Identity, and Memory →](02-objects-identity-memory.md)
