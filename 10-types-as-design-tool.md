# Chapter 10 — Types as a Design Tool

*Part III — Structure*

[← Chapter 9: Modules, Packages, and Imports](09-modules-packages-imports.md) · [Home](README.md) · [Chapter 11: Packaging, Tooling, and the Developer Loop →](11-packaging-tooling-loop.md)

---

## 10.1 The Question Behind the Chapter

Python is a dynamically typed language. A name can refer to a string now and an integer later; a function can be passed anything; types are checked, if at all, only when an operation actually runs. This is, depending on whom you ask, Python's great freedom or its great liability — and for most of the language's history the argument was theoretical, because there was no other option.

Then, beginning with PEP 484 in 2014, Python grew an *optional* static type system: a syntax for annotating names, parameters, and return values with their intended types, and a set of external tools that *check* those annotations without running the code. The annotations do not change how the program executes — Python ignores them at runtime, with the narrow exceptions this chapter will note — but they let a tool tell you, before you ship, that you passed a `str` where an `int` was expected.

The reflexive engineer's questions are "is this worth the effort?" and "isn't this just bureaucracy bolted onto a language that was fine without it?" — and this chapter's thesis is a direct answer to both. **Type hints are not bureaucracy. They are a design tool.** They catch an entire class of bug at the speed of thought rather than the speed of a failing production request. They make a large, unfamiliar codebase *navigable* — the types are documentation that cannot go stale, because a tool verifies them. They make refactoring safe, because the checker finds every call site a signature change broke. And — the subtler benefit — the act of writing down a function's types *is* an act of design: it forces you to decide, precisely, what the function accepts and produces, and vague types are often the first visible symptom of vague thinking.

But the thesis has a crucial qualifier, and this chapter is honest about it: **gradual adoption is the only adoption that works.** A typing rollout that demands the whole codebase be annotated at once, perfectly, before anything ships, fails — every time. The value of types is real and the path to capturing it is incremental, and a staff engineer must understand both halves.

## 10.2 The Shape of the Type System

The Python type system arrived in layers, over a decade of PEPs, and that history shows — there is sometimes an old way and a new way to express the same thing. This section gives you the map.

The foundation is **annotation syntax**. You annotate a variable, a parameter, or a return type with `name: Type`:

```python
def greet(name: str, times: int = 1) -> str:
    return (f"Hello, {name}. " * times).strip()

count: int = 0
```

The annotations say: `name` is a `str`, `times` is an `int`, the function returns a `str`, `count` is an `int`. Python *stores* these annotations (they are accessible at runtime in `__annotations__`) but does not *enforce* them — running this code with a wrong type does not raise because of the annotation. Enforcement is the job of the external checker (Section 10.6).

The basics build up quickly. **Containers** are parameterized — `list[int]` is a list of integers, `dict[str, float]` is a dict from strings to floats, `tuple[int, str, bool]` is a three-element tuple of those types. (In older code you will see `List[int]`, `Dict[str, float]` imported from `typing` — these were necessary before Python 3.9 and are now legacy; the built-in lowercase forms are current.) **Optional and union** types express "this or that": the modern syntax is `int | None` for "an int or None" and `int | str` for "an int or a string." (The legacy forms `Optional[int]` and `Union[int, str]` mean the same; the `|` syntax, available from 3.10, is current.) **`Any`** is the escape hatch — a value typed `Any` is compatible with everything and the checker stops checking it; it is sometimes necessary and always a small admission of defeat, to be used sparingly.

## 10.3 Generics: Old Syntax and New

A *generic* is a type parameterized by another type — `list[int]` is the built-in `list` generic applied to `int`. You also need to write your *own* generic functions and classes: code that works uniformly over many types while preserving the type relationship. A function that returns the first element of a list of `T` should be known to return a `T` — not an `Any`, a `T`, the same type the list held.

The mechanism is the **type variable**. The older syntax, which you will see in existing code, declares one explicitly:

```python
from typing import TypeVar

T = TypeVar("T")

def first(items: list[T]) -> T:
    return items[0]
```

`T` is a type variable; the annotation says "given a `list` of some type `T`, this returns a value of that same `T`." Call `first([1, 2, 3])` and the checker knows the result is an `int`; call `first(["a", "b"])` and it knows the result is a `str`. The type relationship is *preserved*, which `Any` would not do.

Python 3.12 introduced **new, cleaner syntax** (PEP 695) that declares the type parameter inline, with no separate `TypeVar`:

```python
def first[T](items: list[T]) -> T:
    return items[0]

class Box[T]:
    def __init__(self, item: T) -> None:
        self._item = item
    def get(self) -> T:
        return self._item
```

The `[T]` after the function or class name declares the type parameter right where it is used. For new code on Python 3.12+, this is the form to prefer — it is more readable and keeps the type parameter local to the thing it parameterizes. The older `TypeVar` form remains valid and is what you will encounter in any codebase predating 3.12, so you must read both; you should write the new one.

Type variables can be **bounded** — constrained to a type and its subtypes — when the generic code needs to *do* something with the value beyond passing it through. A function that needs to compare its arguments needs a `T` bounded to something comparable. The new syntax expresses this as `def f[T: SomeBound](...)`. Bounding is the difference between "any type at all" and "any type that supports the operations my code performs," and using it correctly is part of designing a generic that is both flexible and safe.

## 10.4 Protocols: Structural Typing

This section is the most important in the chapter, because `Protocol` is the single most *under-used powerful feature* of Python's type system, and a principal engineer reaches for it constantly.

Python has, before `Protocol`, *nominal* typing — a value is acceptable as type `X` if its class is `X` or inherits from `X`. The relationship is by *name and ancestry*: you must explicitly declare "this class is an `X`." That is how abstract base classes (ABCs) work, and it is how most statically typed languages work.

But Python's actual *runtime* behavior has always been *structural* — "duck typing." At runtime, Python does not care what class an object is; it cares what the object can *do*. A function that calls `obj.read()` works with *any* object that has a `read` method, regardless of its class or ancestry. The object does not need to inherit from a `Readable` base class; it needs to *have a `read` method*. For years, the type system could not express this — the most natural and most Pythonic style of code was the style the checker could not check.

`Protocol` (PEP 544) closes that gap. A `Protocol` defines an *interface by structure*: a set of methods and attributes a type must have. And — the key — a class *satisfies* a `Protocol` simply by *having those members*, with **no declaration, no inheritance, no import of the protocol at all**:

```python
from typing import Protocol

class Readable(Protocol):
    def read(self, size: int = -1) -> bytes: ...

def consume(source: Readable) -> bytes:
    return source.read()
```

Any object with a matching `read` method satisfies `Readable` — a file object, a network stream, a custom class, an in-memory buffer — and *none of them* needs to know `Readable` exists. The checker verifies the structure; the classes never reference the protocol. This is *static typing for duck typing*: it lets the type checker verify the structural, duck-typed code that is Python's natural idiom.

The contrast with abstract base classes is the design decision a staff engineer makes deliberately:

An **ABC** is *nominal*. A class must *explicitly inherit* from the ABC to be considered an instance. This creates a hard, declared coupling — the class must import and name the ABC. ABCs are right when you *want* that explicit declaration, when you are providing shared implementation alongside the interface (an ABC can have concrete methods), or when you need the ABC's runtime `isinstance` enforcement.

A **`Protocol`** is *structural*. A class satisfies it by shape alone, with no coupling — the implementing class and the protocol need not know about each other at all. This is right when you are defining an interface at a *seam* between parts of a system, and you want the parts decoupled: the consumer depends on the `Protocol`, the providers merely have the right shape, and neither imports the other. It is right when you want to type code that accepts "anything that can do X" without forcing every such thing to inherit from a common base — including third-party classes you cannot modify.

The guidance, and it is genuinely the view of experienced Python engineers: **for defining an interface at a boundary, prefer `Protocol`.** It matches Python's structural nature, it keeps the two sides of the boundary decoupled, and decoupled boundaries are exactly what Chapter 12's architecture is built on. ABCs remain right for the shared-implementation case and where you specifically want nominal, declared subtyping — but `Protocol` is the tool that has been most under-used relative to how often it is the better choice, and a staff engineer corrects that.

## 10.5 The Precision Tools

Beyond the basics and generics, the type system has a set of *precision tools* — constructs for expressing tighter, more specific contracts. A fluent engineer knows them by name and reaches for them when the situation fits.

**`TypedDict`** types a *dictionary with a known set of string keys and a type per key* — it describes the shape of a dict the way a class describes the shape of an object, without making it a class. It is the right tool for typing JSON-shaped data, API payloads, and configuration dicts where the data genuinely is and must stay a `dict`.

**`Literal`** restricts a value to *specific constant values* rather than a whole type. `Literal["red", "green", "blue"]` is a type accepting *only* those three strings — not any `str`, those three. It turns "this parameter is a string" into "this parameter is one of these exact strings," and the checker enforces it. It is excellent for mode flags, status values, and small fixed vocabularies, and it pairs well with `match` statements.

**`Annotated`** attaches *metadata* to a type without changing the type itself. `Annotated[int, SomeMetadata]` is, to the type checker, still just `int` — but the metadata rides along and can be read at runtime by libraries that look for it. This is the mechanism behind a great deal of modern Python: it is how FastAPI (Chapter 17) carries validation rules and request-source information on a parameter's type, how Pydantic carries field constraints. `Annotated` is the bridge between the static type and runtime-library behavior.

**`Callable`** types a *function* — `Callable[[int, str], bool]` is "a function taking an `int` and a `str`, returning a `bool`." For functions whose argument lists are themselves generic — a decorator that must preserve the signature of whatever it wraps — there is **`ParamSpec`**, which captures and replays a parameter list, letting you type a decorator that genuinely preserves the wrapped function's signature (the typing-level companion to Chapter 5's `functools.wraps`).

**`Self`** is the type "an instance of the enclosing class" — exactly what you want as the return type of a method that returns `self` (a fluent builder, a method-chaining API). And **`@override`** (PEP 698) marks a method as intended to override a base-class method, so the checker can catch the bug where you *think* you are overriding but a typo means you have silently defined a new, unrelated method.

You do not need every one of these in your head at all times. You need to know they *exist*, and the category each addresses, so that when you hit "I need to type a dict with known keys" or "I need to restrict this to three values" or "I need a decorator that preserves signatures," you reach for the right tool instead of falling back to `Any` and losing the checking.

## 10.6 The Toolchain

Annotations do nothing on their own — they are checked by *external tools*, and the toolchain is part of what a staff engineer must know.

**`mypy`** is the original and reference type checker — mature, strict-configurable, the standard against which the type system's behavior is often defined.

**`pyright`** is Microsoft's checker, written for speed and powering the Python experience in many editors (it is the engine inside the popular Pylance extension). It is fast enough to run continuously as you type, which is a meaningful part of its value — the feedback is immediate.

The two are largely compatible — code typed for one generally checks under the other — with occasional differences in strictness and inference. Many teams run one in CI and let developers use whichever their editor prefers.

**`ruff`**, the linter and formatter you met in Chapter 11's neighborhood (and will meet properly there), is *not* a type checker — it does not do the deep type inference `mypy` and `pyright` do — but it catches type-annotation *mistakes* and enforces annotation *style*, complementing the real checkers.

And there is a category beyond static checking: **runtime enforcement**. The static checkers verify annotations *without running the code*; some tools verify them *while it runs*. **Pydantic** (Chapter 6, Chapter 17) does this for its models — it checks, at object-construction time, that the data matches the declared types. **`beartype`** is a library that adds fast runtime type-checking to ordinary annotated functions. The distinction matters: static checking catches bugs before deployment, for free, at no runtime cost — but only the bugs it can prove from the code. Runtime checking catches bugs the static checker could not know about — most importantly, *bad data arriving from outside the program* — at the cost of some runtime work. They are complementary, and the trust-boundary rule from Chapter 6 applies: static checking everywhere; runtime checking (Pydantic) at the boundaries where untrusted data enters.

## 10.7 Variance: The Subtle Part

Variance is the part of the type system that confuses everyone, and interviewers know it — so it earns a careful, from-scratch treatment.

The question variance answers: if `Cat` is a subtype of `Animal`, what is the relationship between `list[Cat]` and `list[Animal]`? Intuition says `list[Cat]` should be usable wherever `list[Animal]` is expected — a list of cats is a list of animals, surely. Intuition is *wrong*, and the reason it is wrong is the whole of variance.

Suppose a function takes a `list[Animal]` and is allowed to *append* to it. If you could pass a `list[Cat]` to that function, the function could append a `Dog` to it — appending a `Dog` to a `list[Animal]` is perfectly valid — and now your `list[Cat]` contains a `Dog`. Type safety is broken. So `list[Cat]` is *not* a subtype of `list[Animal]`: a *mutable* container is **invariant** in its element type — `list[Cat]` and `list[Animal]` have *no* subtype relationship, in either direction.

The three possibilities, named:

**Covariant** — the subtype relationship is *preserved*. If `Cat` is a subtype of `Animal`, then `Producer[Cat]` is a subtype of `Producer[Animal]`. This is safe for things you only *read from* (only *produce* values). A read-only sequence of cats genuinely is a read-only sequence of animals — you can only take cats out, and a cat is an animal. Immutable containers and producer types are covariant.

**Contravariant** — the subtype relationship is *reversed*. If `Cat` is a subtype of `Animal`, then `Consumer[Animal]` is a subtype of `Consumer[Cat]`. This is the counterintuitive one, and it is safe for things you only *write to* (only *consume* values). A function that can handle *any* `Animal` can certainly be used where a handler of `Cat`s is needed — it handles cats and more. Consumer types — function argument positions — are contravariant.

**Invariant** — *no* subtype relationship, as with the mutable `list` above. A type that is *both* read and written must be invariant, because covariance would break reads-after-writes and contravariance would break the other direction.

For most code you do not declare variance explicitly — the type system and the standard library handle it: `list` is invariant, the read-only `Sequence` is covariant, and so on. With the new PEP 695 syntax, the checker often *infers* variance for your own generics. You need to *declare* it only when writing your own generic container or interface and the checker cannot infer it. But you must *understand* it, because the rule "a mutable container is invariant — `list[Cat]` is not a `list[Animal]`" is exactly the kind of thing that produces a confusing checker error, and the only way to read that error is to know variance. The mnemonic worth carrying: **producers are covariant, consumers are contravariant, anything both is invariant.**

## 10.8 Gradual Adoption

The chapter's thesis had two halves; this section is the second. Types are valuable — *and* the only adoption strategy that works is gradual. A staff engineer who is going to introduce typing to an existing untyped codebase must understand this, because the all-at-once approach is a well-documented way to fail.

Why all-at-once fails: a large untyped codebase, run through a strict type checker for the first time, produces thousands of errors. Fixing all of them before any of the value is realized is a multi-month project with no incremental payoff, it blocks other work, and it is exactly the kind of initiative that gets abandoned half-done — leaving the codebase in a worse state than before, partially typed and inconsistently checked.

The strategy that works is *incremental*, and it has a recognizable shape:

Start by **getting the checker running at all**, configured *leniently* — so that the existing untyped code passes (untyped code, to a lenient checker, is simply unchecked, not erroneous). The bar is "the checker runs in CI and is green," not "everything is typed."

Then **type the boundaries first**. The highest-value annotations are on the *interfaces* between components — the public function signatures, the data crossing between modules. These are where type mismatches cause the worst bugs and where the annotations most help a reader. Internal implementation details can stay untyped longer.

Then **ratchet**. Configure the checker so that *newly written and newly modified* code is held to a stricter standard than old code — a per-module or per-directory strictness setting, tightened over time. New code is fully typed; old code is typed as it is touched, opportunistically (the Boy Scout Rule from Chapter 29, applied to types). The typed fraction of the codebase rises monotonically, and at no point is there a giant blocking project.

Finally, **make the checker part of CI** so the ratchet cannot slip backward — once a module is checked strictly, it stays checked strictly, enforced automatically.

The result is that the codebase becomes progressively more typed, the value is captured continuously as it goes, and there is never a single demoralizing all-or-nothing push. This — not the syntax of `TypeVar` or the rule of variance — is the genuinely staff-level knowledge in this chapter: the *adoption strategy*, the understanding that the technical merit of typing is necessary but not sufficient, and that *how* you roll it out determines whether the team ever sees the benefit.

> ### War Story — The Refactor the Types Caught
>
> A team maintained a service whose codebase had, over a couple of years, been gradually and incompletely typed — the boundaries were annotated, much of the core was, the edges were not. A staff engineer needed to make a breaking change to a widely-used internal function: a parameter's type was changing from a plain `dict` to a proper typed object, and the parameter order was changing too.
>
> In an untyped codebase this is a frightening change — the function was called from dozens of places, and a missed call site would not fail until that code path ran in production, perhaps weeks later, perhaps in an edge case no test covered. The usual mitigations are a careful grep, a lot of hope, and a slow nervous rollout.
>
> Because the function and its callers were typed, the engineer changed the signature and ran the type checker. The checker reported, precisely, *every* call site that the change had broken — each one, with file and line, because each now passed an argument of the wrong type or in the wrong position. The "frightening" refactor became mechanical: fix each reported site, re-run the checker, repeat until green. The change shipped with confidence, in an afternoon, and not one broken call site reached production — because the type checker had found all of them before the code ran.
>
> The lesson is the chapter's thesis made concrete. The types were not bureaucracy and they were not decoration. They were the thing that turned a dangerous, slow, hope-driven refactor into a safe, fast, checker-driven one. The value of types is most visible exactly here — at the moment of change — and a codebase that has been gradually typed has bought itself the ability to change safely. That is what the design tool *is*.

> ### The Level Line — Typing
>
> **A senior engineer** writes annotations on functions, uses the basic types and containers, runs a type checker, and understands `Optional`/union types.
>
> **A staff engineer** uses the type system as a design tool — writes generics (and knows the new PEP 695 syntax), reaches for `Protocol` to type interfaces structurally at seams, knows the precision tools (`TypedDict`, `Literal`, `Annotated`, `ParamSpec`) and when each fits, understands variance well enough to read the errors it causes, and knows the static-versus-runtime-checking distinction and where Pydantic belongs.
>
> **A principal engineer** owns the *typing adoption strategy* for a codebase — leads the gradual, ratcheting rollout, configures the toolchain, sets the boundary-first priority — understands that the technical merit of typing is necessary but not sufficient and that adoption is where it succeeds or fails, and can articulate to skeptical engineers *why* types are a design tool rather than ceremony, using the safe-refactor argument that lands.

## 10.9 At the Interview Table

Typing is increasingly central to Python interviews, and the questions probe both technical knowledge and the judgment around adoption.

**The question: "Are type hints worth it?"**

This question is really probing whether you have *judgment* about cost and value, so a one-word answer fails it. "Yes, with a qualifier. The value is real: types catch a class of bug before deployment for free, they make a large codebase navigable because they're documentation a tool keeps honest, and — most concretely — they make refactoring safe, because the checker finds every call site a signature change broke. Writing the types is also an act of design; it forces you to decide precisely what a function accepts and returns. The qualifier is adoption: on an existing untyped codebase, you cannot do it all at once — that's a months-long blocking project with no incremental payoff, and it usually gets abandoned. You roll it out gradually: get the checker running leniently, type the boundaries first, ratchet strictness up for new and modified code, enforce it in CI so it can't slip back. So: worth it, definitely — but the rollout strategy is what determines whether the team ever sees the benefit."

**The question: "What's the difference between a `Protocol` and an abstract base class?"**

"Both define an interface, but the typing is opposite. An ABC is *nominal* — a class must explicitly inherit from it to count, which creates a declared coupling: the class imports and names the ABC. A `Protocol` is *structural* — a class satisfies it just by *having the right methods and attributes*, with no inheritance, no declaration, no import; the implementing class and the protocol need not know about each other at all. `Protocol` is static typing for duck typing — it lets the checker verify the structural, duck-typed code that's Python's natural idiom. For an interface at a *seam* between components, I prefer `Protocol`, because it keeps the two sides decoupled — the consumer depends on the protocol, the providers just have the right shape. ABCs I keep for when I want explicit declared subtyping or shared implementation, since an ABC can carry concrete methods."

**The question: "Explain variance."**

"Variance is about whether a subtype relationship survives being put inside a generic. If `Cat` is a subtype of `Animal`, is `list[Cat]` a subtype of `list[Animal]`? It is *not* — and that's the key example. If it were, a function taking `list[Animal]` could append a `Dog` to your `list[Cat]`, breaking type safety. So a *mutable* container is **invariant** — no relationship in either direction. The three cases: covariant means the relationship is preserved, which is safe for things you only read from — a read-only sequence of cats is a read-only sequence of animals. Contravariant means it's reversed, which is safe for things you only write to — a function handling any `Animal` can stand in for one handling `Cat`s. Invariant means neither, which is required when a type is both read and written. The mnemonic: producers covariant, consumers contravariant, both invariant."

**The question: "How would you type a dict that always has the keys `name`, `age`, and `email`?"**

"`TypedDict` — it types a dict with a known set of string keys and a type per key, describing the dict's shape the way a class describes an object's, without making it a class. It's the right tool for JSON-shaped data and config dicts where the value genuinely needs to stay a `dict`. If it didn't need to *be* a dict, I'd consider a `dataclass` or a Pydantic model instead — and Pydantic specifically if the data is coming from outside the program and needs runtime validation."

**The red flags.** Treating typing as pure ceremony with no sense of its value at refactor time. Not knowing `Protocol` exists, or not knowing the structural-versus-nominal distinction — a sign of someone who has used types superficially. Advocating an all-at-once typing rollout — which signals no real adoption experience. And getting variance backwards, or not understanding why a mutable `list` is invariant — the single most common typing-depth miss.

## 10.10 The Forge

**Drill 10.1.** Annotate a set of provided untyped functions — including one that takes a callback, one that returns a tuple of mixed types, one with an optional parameter, and one that should be generic. Run a type checker (`mypy` or `pyright`) and make it pass with no `Any`.

**Drill 10.2.** For each, write the correct type annotation: a function taking a list of any one type and returning a single element of that type; a parameter that must be exactly one of the strings `"json"`, `"yaml"`, `"toml"`; a dict that always has keys `host` (str) and `port` (int); a parameter that is a function taking two ints and returning a bool.

**Build 10.1.** Write a fully generic, fully typed container class — a `Stack[T]` or a `Result[T, E]` (a value-or-error type) — using the new PEP 695 syntax. Make it pass a strict type checker. Then write a small amount of calling code that demonstrates the checker *catching* a misuse — pushing the wrong type, unwrapping incorrectly.

**Build 10.2.** Define an interface at a seam using `Protocol` — for example, a `Storage` protocol with `get` and `put` methods — and write two unrelated classes that satisfy it (an in-memory implementation and a file-based one), *without either class inheriting from or importing the protocol*. Write a consumer function typed against the protocol, and demonstrate the checker accepting both implementations and rejecting a class that is missing a method.

**Investigate 10.1.** Take a real untyped module of your own (or a small open-source one) and add type annotations to it. Run a type checker as you go. Write up: how many genuine bugs or latent issues the checker surfaced, what was awkward to type and why, and where you were tempted to reach for `Any` and whether you could avoid it.

**Investigate 10.2.** Construct a small example that demonstrates variance concretely: write a generic class, use it in a way the checker rejects on variance grounds, and from the error work out whether the type should be covariant, contravariant, or invariant. Write up the reasoning, tying it back to read-versus-write.

**Design 10.1.** You are the staff engineer on a team with a 150,000-line, almost entirely untyped Python codebase. Leadership has agreed to invest in typing. Write a one-to-two-page adoption plan: how you get the checker running without blocking anyone; what you type first and why; how the strictness ratchet works in practice (per-module configuration, CI enforcement); how you handle the engineers who are skeptical that this is worth it; what milestones mark progress; and how you would know, in six months, whether it had been worth it. This is the genuinely staff-level exercise — the technical knowledge of typing is assumed; the deliverable is the rollout judgment.

---

[← Chapter 9: Modules, Packages, and Imports](09-modules-packages-imports.md) · [Home](README.md) · [Chapter 11: Packaging, Tooling, and the Developer Loop →](11-packaging-tooling-loop.md)
