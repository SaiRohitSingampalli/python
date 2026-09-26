# Chapter 6 — Objects and Classes

*Part II — The Language*

[← Chapter 5: Functions and Decorators](05-functions-and-decorators.md) · [Home](README.md) · [Chapter 7: Iteration, Generators, and the Lazy Mindset →](07-iteration-and-generators.md)

---

## 6.1 The Question Behind the Chapter

This is the longest chapter in Part II, and the length is earned. Python's object system is deep — deeper than most engineers who use classes daily ever explore — and the depth is not academic. The difference between an engineer who can write a class and an engineer who can *design a type* is the difference between senior and staff, and that difference lives in the material this chapter covers: the method resolution order, the descriptor protocol, the `__eq__`/`__hash__` contract, the honest trade-offs between `dataclass` and `Pydantic` and the alternatives, and the question of when structural typing beats nominal typing.

We will not cover the object system as a feature list. We will cover it as a set of *mechanisms* — `__new__` and `__init__`, the MRO, descriptors, the dunder protocols — because once you see the mechanisms, the features become consequences, and consequences are derivable while feature lists must be memorized. By the end you will know why `property` and `cached_property` and an ORM's column fields are all *the same thing*, why a broken `__hash__` produces a bug that is invisible until it is catastrophic, and why a principal engineer reaches for `Protocol` far more often than for an abstract base class.

This chapter assumes everything before it. The name model from Chapter 3 explains attribute binding. The memory model from Chapter 2 explains `__slots__`. The closures of Chapter 5 reappear as the mechanism inside descriptors. The book compounds, and here is where the compounding becomes obvious.

## 6.2 `__new__` Versus `__init__`: Two Steps, Not One

Most engineers think object creation is one step — `__init__` — and for most code that belief is harmless. But it is wrong, and the cases where it matters are real, so a staff engineer knows the truth: creating an object is *two* steps.

When you call `MyClass(args)`, Python does two distinct things. First it calls `MyClass.__new__(MyClass, args)` — `__new__` is responsible for *allocating and returning the new object*. Then, on the object `__new__` returned, Python calls `__init__(self, args)` — `__init__` is responsible for *configuring the already-created object*. `__new__` creates; `__init__` initializes. They are different methods with different jobs, and `self` exists in `__init__` only because `__new__` already made it.

For ordinary classes you never write `__new__` — the default one, inherited from `object`, allocates a plain instance and that is exactly what you want. You write `__init__` to set up attributes, and that is the whole story 95% of the time.

The 5% where `__new__` matters: when you need to control *allocation itself*. The most important case is *subclassing an immutable type*. An immutable type — `int`, `str`, `tuple`, `frozenset` — has its value fixed at the moment of creation; by the time `__init__` would run, the object already exists and its value is already locked. So if you subclass `tuple` and want to influence the tuple's contents, `__init__` is too late — you must intercept `__new__`, the creation step:

```python
class Point(tuple):
    def __new__(cls, x, y):
        return super().__new__(cls, (x, y))   # construct the tuple's value here

    @property
    def x(self): return self[0]
    @property
    def y(self): return self[1]
```

`__new__` also appears in some implementations of singletons and object pools and caching constructors — anything where you might *not* return a fresh object. But the headline, the thing to carry: object creation is allocate-then-initialize, `__new__` then `__init__`, and the distinction becomes load-bearing the moment you subclass something immutable.

## 6.3 The Three Kinds of Method

A function defined in a class body can be one of three kinds, and each exists for a different relationship to the class and its instances.

An **instance method** — the ordinary kind — receives the instance as its first parameter, `self`. It operates on a particular object's data. This is the default and the common case.

A **class method** — marked `@classmethod` — receives the *class itself* as its first parameter, conventionally `cls`. It does not need an instance. Its most important use is the *alternative constructor*: a method that builds and returns an instance in some particular way.

```python
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    @classmethod
    def from_dict(cls, data):
        return cls(name=data["name"], email=data["email"])

    @classmethod
    def anonymous(cls):
        return cls(name="anonymous", email="")
```

`User.from_dict(payload)` and `User.anonymous()` are alternative ways to construct a `User` — and note they use `cls`, not `User`, by name. That matters: if someone subclasses `User`, `SubUser.from_dict(payload)` will correctly build a `SubUser`, because `cls` is whatever class the method was called on. Hard-coding `User` would break that. Class methods are the right tool whenever you want a named, discoverable way to construct an instance beyond the plain `__init__`.

A **static method** — marked `@staticmethod` — receives neither the instance nor the class. It is a plain function that lives in the class's namespace because it is *logically related* to the class, but it does not touch instance or class state. Static methods are the least common of the three and slightly suspect — a static method that does not use the class for anything is often a sign the function would be just as well, or better, as a module-level function. They are not wrong, but when you write one, ask whether it earns its place inside the class.

## 6.4 Inheritance, the MRO, and Cooperative `super()`

Inheritance is the part of OOP every engineer learns first and understands least, because the easy cases hide the mechanism and the mechanism only surfaces under multiple inheritance — which is exactly where interviewers go.

The mechanism is the **Method Resolution Order**, the MRO. When you access an attribute or method on an object, Python searches a *specific, ordered list of classes* — the object's class, then its bases, in a precise order — and uses the first match. That ordered list is the MRO, and you can see it directly: `SomeClass.__mro__` or `SomeClass.mro()`.

For single inheritance the MRO is obvious — the class, its parent, its grandparent, up to `object`. The interesting case is multiple inheritance, the "diamond": a class `D` inheriting from `B` and `C`, both of which inherit from `A`. In what order are `B`, `C`, and `A` searched? Python answers with an algorithm called **C3 linearization**, and its two guarantees are the ones worth knowing: a class always appears in the MRO *before* any of its base classes, and the order of base classes in a `class` statement is *preserved*. C3 produces a single, consistent, predictable ordering — and if no consistent ordering exists (a genuinely contradictory inheritance graph), Python *refuses to create the class* and raises an error at class-definition time, rather than silently picking something arbitrary.

The reason the MRO matters in practice — beyond passing the interview question — is `super()`. Engineers learn `super()` as "call the parent class's method," and under single inheritance that description is harmless. Under multiple inheritance it is *wrong*, and the wrongness causes real bugs. `super()` does not call "the parent." `super()` calls **the next class in the MRO** — and which class that is depends on the MRO of the *actual object*, which may be a subclass the current class has never heard of.

This is what "cooperative multiple inheritance" means and why it works only if every class cooperates. Consider:

```python
class Base:
    def __init__(self):
        print("Base")

class A(Base):
    def __init__(self):
        print("A")
        super().__init__()

class B(Base):
    def __init__(self):
        print("B")
        super().__init__()

class C(A, B):
    def __init__(self):
        print("C")
        super().__init__()

C()    # prints C, A, B, Base — each __init__ runs exactly once
```

When `C()` runs, `C.__init__`'s `super().__init__()` does *not* call `A` because `A` is "the parent" — it calls the next class in `C`'s MRO, which is `A`. Then `A`'s `super().__init__()` calls the next class in *`C`'s* MRO after `A`, which is `B` — not `Base`, even though `A` directly inherits from `Base`. Then `B`'s `super()` reaches `Base`. Each `__init__` runs exactly once, in MRO order, and the whole chain works *only because every class called `super()`*. If `A` had called `Base.__init__()` directly instead of `super().__init__()`, it would have skipped `B` entirely — `B`'s initialization would never run, and that is a real, MRO-shaped bug.

The rule that follows: **in any class that may participate in multiple inheritance, always use `super()`, never name a base class directly.** `super()` follows the MRO; a hard-coded base class name does not, and the mismatch is where cooperative inheritance breaks. And the broader piece of judgment, which the book will sharpen in Chapter 12: deep multiple-inheritance hierarchies are *hard* — the MRO is subtle, cooperative `super()` is fragile, and the cognitive cost is high. Composition (an object *holding* other objects) is very often the better design, and a staff engineer reaches for inheritance deliberately and shallowly, not reflexively.

## 6.5 The Dunder Protocols

The "dunder" methods — double-underscore methods like `__len__`, `__eq__`, `__getitem__` — are how a class hooks into Python's *syntax and built-in operations*. They are the mechanism by which `len(x)` knows what to do, by which `x + y` works, by which `for item in x` iterates. A class that implements the right dunders *becomes* a first-class participant in the language: indexable like a list, callable like a function, usable in a `with` statement. This is sometimes called "the Python data model," and it is the deepest expression of the language's consistency.

Rather than an alphabetical catalog — which would be a reference, not a teaching — group them by *purpose*:

**Construction and lifecycle:** `__new__`, `__init__`, `__del__` (Chapter 2 told you to avoid this one).

**Representation:** `__repr__` and `__str__`. `__repr__` is for *developers* — it should be unambiguous, ideally something that looks like the code to recreate the object, and it is what you see in a debugger or a REPL. `__str__` is for *users* — a readable, friendly rendering. If you write only one, write `__repr__`, because `__str__` falls back to it. A class with a good `__repr__` is dramatically easier to debug; a class with the default `<object at 0x7f...>` repr is a small, recurring tax on everyone who ever has to inspect it.

**Comparison:** `__eq__`, `__lt__`, `__le__`, `__gt__`, `__ge__`, and `__hash__`. The next section is devoted to the contract binding `__eq__` and `__hash__`, because breaking it is a genuine and dangerous bug.

**Container behavior:** `__len__`, `__getitem__`, `__setitem__`, `__delitem__`, `__contains__`, `__iter__`. Implement the right subset and your object behaves like a list or a dict — indexable, iterable, usable with `in` and `len`.

**Callable behavior:** `__call__`. A class with `__call__` produces *instances that can be called like functions* — useful for stateful "functions," for configurable callbacks, for objects that are conceptually behavior-with-memory.

**Context-manager behavior:** `__enter__` and `__exit__`, the protocol behind the `with` statement — Chapter 7's territory.

**Attribute access:** `__getattr__`, `__setattr__`, `__getattribute__`, and the descriptor methods — Section 6.8.

The unifying idea: Python's syntax is, to a remarkable degree, *protocol-based*. `len(x)` is not magic restricted to built-in types — it is "call `x.__len__()`," and any class that defines `__len__` gets it. This is why a well-designed custom class can feel exactly as natural to use as a built-in. It is also why the next chapter's iterators and the chapter after's data classes are not special cases — they are ordinary classes implementing the relevant protocols.

## 6.6 The `__eq__` / `__hash__` Contract

Here is a contract that is easy to break, breaks silently, and breaks *dangerously* — so it gets its own section, and it is one of the highest-value things in this chapter.

Two methods are involved. `__eq__` defines when two objects are considered equal. `__hash__` returns an integer hash, used to place the object in hash tables — recall from Chapter 4 that dicts and sets *are* hash tables, and that they find an object by hashing it to a slot.

The contract binding them, stated exactly: **if two objects are equal, they must have the same hash.** Equal objects, equal hashes. (The converse is *not* required — two unequal objects may share a hash; that is just a collision, and hash tables handle collisions. The required direction is equal-implies-same-hash.)

Why must this hold? Because of *how a set or dict finds things*. To check whether an object is in a set, Python hashes the object to locate the slot it *would* occupy, then looks in that slot. If two objects are equal but hash to *different* slots, the set will look in the wrong slot and conclude — wrongly — that the object is not present. Equality and hashing must agree, or hash-based lookup is broken at the foundation.

Now the trap, and it is a real one. **When you define `__eq__` on a class, Python automatically sets `__hash__` to `None`** — making your instances *unhashable*, unusable in a set or as a dict key. Python does this deliberately and defensively: the default `__hash__` (inherited from `object`) is based on the object's identity, and if you have just redefined equality to be value-based, an identity-based hash would *violate the contract* — two value-equal objects would have different identity-hashes. Rather than let you ship that latent bug, Python disables hashing entirely and forces you to make a decision.

So when you define `__eq__`, you must consciously do one of two things. If your objects should be *immutable and hashable* — usable as dict keys, as set members — you define `__hash__` too, computing it from *the same fields* `__eq__` uses, so the contract holds:

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        if not isinstance(other, Point):
            return NotImplemented
        return (self.x, self.y) == (other.x, other.y)

    def __hash__(self):
        return hash((self.x, self.y))      # SAME fields as __eq__
```

The clean trick: hash a *tuple of the fields you compare* — `hash((self.x, self.y))` — which guarantees, automatically, that equal objects hash equally, because equal field-tuples hash equally. If instead your objects are *mutable* — their fields change over their lifetime — they generally should *not* be hashable at all, because an object's hash must never change while it sits in a set (a changed hash means it is now in the wrong slot and effectively lost). For a mutable class, leaving `__hash__` as the `None` that defining `__eq__` set is the *correct* outcome, not a bug.

> ### War Story — The Hash That Lied
>
> A team modeled a domain entity as a class. The class had a value-based `__eq__` — two entities with the same ID were "equal." Someone, wanting the instances usable in a set for deduplication, added a `__hash__`. But they wrote it carelessly: `__hash__` returned a hash derived from one set of fields, and `__eq__` compared a *different* set of fields. The two methods disagreed about what made two objects "the same."
>
> For a long time nothing went wrong, because the deduplication set was usually small and the specific objects flowing through it happened not to expose the disagreement. Then a data shift caused the set to fill with objects where `__eq__` said "equal" but `__hash__` said "different slot." The deduplication *silently stopped working* — duplicates sailed straight through, because the set looked for each incoming object in the slot its hash pointed to, did not find the equal-but-differently-hashed object sitting in another slot, and concluded the object was new. No exception. No error. Just duplicates in a system that had been promised there would be none, and a slow, expensive hunt downstream for where they were coming from.
>
> The fix was to make `__hash__` and `__eq__` use the *same fields* — the tuple-hash trick. The lesson: the `__eq__`/`__hash__` contract is not a style guideline, it is a correctness invariant of every hash-based collection, and breaking it does not crash — it *lies*, quietly, until the data is unlucky enough to make the lie visible. The defense, beyond knowing the contract, is to not hand-write these methods at all when you do not have to — which is the cue for the next section.

## 6.7 `dataclass`, and the Honest Comparison

Writing `__init__`, `__repr__`, `__eq__`, and a correct `__hash__` by hand, for every small data-holding class, is tedious and — as the War Story shows — error-prone. The tedium is also *unnecessary*, because for the common case Python generates all of it for you.

The `@dataclass` decorator, given a class with annotated fields, generates the boilerplate:

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float
```

From those three lines, `@dataclass` generates `__init__(self, x, y)`, a useful `__repr__` (`Point(x=1.0, y=2.0)`), and an `__eq__` that compares the fields. The flags extend it: `@dataclass(frozen=True)` makes instances immutable *and* generates a correct, field-based `__hash__` — the contract handled for you, correctly, by the same tuple-of-fields logic the previous section taught by hand. `@dataclass(slots=True)` adds `__slots__` (Section 6.9) for memory savings. `@dataclass(order=True)` generates the comparison methods so instances can be sorted. `field()` gives per-field control — default factories (the sentinel pattern, built in), excluding a field from comparison, and more. For the overwhelmingly common task of "a class that holds some data," a `dataclass` is the right tool, it is less code, and — critically — the code it generates is *correct*, including the `__eq__`/`__hash__` contract that is so easy to break by hand.

But `dataclass` is not the only tool, and a staff engineer chooses among the alternatives knowingly. The honest comparison:

A **plain class** — you write the methods yourself. Use it when the class is mostly *behavior* rather than data, or when you need full control over construction. For a data-holder, a plain class is just `dataclass` with the boilerplate re-typed by hand and the chance to get `__hash__` wrong.

A **`dataclass`** — generated boilerplate, standard library, zero dependencies, no runtime validation. Use it for internal data structures whose data you *trust* — values your own code produced. This is the default for a data-holder.

A **`typing.NamedTuple`** — like a `dataclass` but it *is* a tuple: immutable, indexable, iterable, unpackable, and very lightweight. Use it for a small immutable record that benefits from tuple behavior, especially one returned from a function.

A **`Pydantic` model** — a third-party library, and the important one for a specific job: it *validates and coerces data at runtime*. A Pydantic model checks, when an instance is created, that the data matches the declared types — and raises a clear error if not. That is exactly what you want at a *trust boundary*: parsing an incoming API request, loading a config file, reading anything from outside your program. `dataclass` trusts; `Pydantic` *verifies*. The cost is a dependency and some runtime overhead. The decision rule is clean and worth carrying: **for data your own code produced and trusts, a `dataclass`; for data crossing a trust boundary from the outside world, a `Pydantic` model.** We will see Pydantic doing exactly this job in Chapter 17 (validating web requests) and Chapter 22 (validating configuration).

An **`attrs`** class — the library that predates and inspired `dataclass`, still with more features than the standard-library version. Use it when you have hit a specific limit of `dataclass` and need what `attrs` adds; otherwise `dataclass` is the lower-dependency default.

## 6.8 Descriptors: The Mechanism Behind `property`

Descriptors are the part of the object system most engineers never learn — and learning them is high-leverage, because descriptors are the single mechanism that explains `property`, `cached_property`, `classmethod`, `staticmethod`, and every ORM's column field. Learn one mechanism, understand five features.

A **descriptor** is an object that defines what happens when it is accessed *as a class attribute* — it customizes attribute access itself. It does this by implementing one or more of three methods: `__get__` (called when the attribute is read), `__set__` (called when it is assigned), and `__delete__` (called when it is deleted).

The motivating example everyone has used is `property`. When you write:

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("radius cannot be negative")
        self._radius = value
```

`radius` *looks* like a plain attribute to anyone using a `Circle` — `circle.radius` reads it, `circle.radius = 5` writes it — but reading runs the getter and writing runs the validating setter. The magic is not special syntax. **`property` is a descriptor.** It is an object with `__get__` and `__set__`, and when you access `circle.radius`, Python sees that the class attribute `radius` is a descriptor and calls its `__get__` instead of just handing back a stored value. `property` is not a language keyword with privileged behavior — it is an ordinary class, in the standard library, implementing the descriptor protocol. You could write it yourself, and writing your own descriptor is exactly how you would build, say, a reusable typed-and-validated attribute used across many classes:

```python
class Positive:
    def __set_name__(self, owner, name):
        self._name = f"_{name}"          # learn the attribute name we were assigned to

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self._name)

    def __set__(self, obj, value):
        if value < 0:
            raise ValueError(f"{self._name} must be non-negative")
        setattr(obj, self._name, value)

class Account:
    balance = Positive()                 # reusable validated attribute
    interest = Positive()
```

`balance` and `interest` are descriptor *instances* on the class; every read and write of `account.balance` runs through `Positive.__get__` and `Positive.__set__`. The validation logic is written once and reused — which is the thing a `property` cannot do, because a `property` is bound to one class.

There is one more piece of the mechanism worth knowing, because it answers a real "why does this happen" question and appears in deep-dive interviews: the distinction between a **data descriptor** (one that defines `__set__` or `__delete__`) and a **non-data descriptor** (one that defines only `__get__`). They have *different lookup priority*. A data descriptor takes precedence over an instance's own `__dict__`; a non-data descriptor does not. Concretely: a `property` (a data descriptor) cannot be shadowed by an instance attribute of the same name, but a plain method (a non-data descriptor — functions implement `__get__`) *can* be. This is why you can do `instance.some_method = something_else` and override the method on that one instance, but you cannot do the same to a `property`. It is an obscure-sounding rule with a concrete, observable consequence, and knowing it marks genuine depth.

The headline to retain: `property`, `cached_property`, `classmethod`, `staticmethod`, bound methods, and ORM fields are *all descriptors*. They are not six special features; they are six uses of one protocol. That is the kind of unifying insight this chapter exists to deliver.

## 6.9 `__slots__` and the Memory of Instances

By default, every Python instance stores its attributes in a per-instance dictionary — `instance.__dict__`. That dict is what makes Python objects so flexible: you can add an attribute to an instance at any time, because there is a dict to put it in. But a dict has overhead — recall Chapter 2 — and when you have a very large number of instances, that per-instance dict, multiplied across millions of objects, is real memory.

`__slots__` is the opt-out. A class that declares `__slots__` gives up the per-instance `__dict__` in exchange for a fixed, pre-declared set of attribute slots:

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x, y):
        self.x = x
        self.y = y
```

This `Point` stores `x` and `y` in fixed slots rather than a dict. The savings are substantial — often 40–50% less memory per instance — and attribute access is marginally faster too. The cost is the flexibility you gave up: you can no longer add an attribute that is not in `__slots__` (`point.z = 3` raises `AttributeError`), and there are interactions to know about — a class with `__slots__` whose parent does *not* have `__slots__` still gets a `__dict__` from the parent, so the saving requires `__slots__` down the whole hierarchy; and `__slots__` complicates multiple inheritance.

When is `__slots__` worth it? When you have *many* instances of a class — tens of thousands and up — and memory matters. For a class you instantiate a handful of times, `__slots__` is pointless ceremony. For the modern way to get it, you do not write `__slots__` by hand: `@dataclass(slots=True)` generates it, combining the boilerplate generation of Section 6.7 with the memory optimization here. That is the staff-level move — recognize the high-instance-count situation, and reach for `dataclass(slots=True)`.

## 6.10 A Note on Metaclasses

Metaclasses are the deep end of the object system, and a chapter this length must at least name them — but the most useful thing a staff-level book can tell you about metaclasses is *when not to use one*.

Briefly: a metaclass is the class *of* a class. Just as an ordinary object is an instance of a class, a class is an instance of a metaclass — `type`, by default. A custom metaclass lets you intercept and customize *class creation itself*, running code every time a class using that metaclass is *defined*. This is genuine power, and frameworks have used it — older ORMs, for instance, used metaclasses to do things when you defined a model class.

But metaclasses are also subtle, they interact badly with each other (a class can have only one metaclass, so two metaclass-using frameworks collide), and they make a codebase harder to understand. And — the key point — for almost everything metaclasses were once used for, Python now has *simpler tools*. `__init_subclass__` is a method that runs whenever a class is *subclassed*, and it handles the most common metaclass use case (a base class that wants to do something each time someone extends it — like registering subclasses in a plugin table) with a fraction of the complexity. Class decorators handle most of the rest. The guidance, which is genuinely the consensus among experienced Python engineers and worth stating plainly: **you almost certainly do not need a metaclass.** Know what they are, recognize them in a framework's source, and reach for `__init_subclass__` or a class decorator instead. If you have concluded you truly need a metaclass, the bar is to have first ruled out both of those — and that is rare.

> ### The Level Line — Objects and Classes
>
> **A senior engineer** writes correct classes, uses inheritance and `super()`, knows the common dunders, uses `dataclass` for data holders, and can write a `property`.
>
> **A staff engineer** understands the *mechanisms*: the two-step creation, the MRO and why cooperative `super()` is mandatory under multiple inheritance, the `__eq__`/`__hash__` contract and why breaking it is dangerous, descriptors as the one thing behind `property` and friends; chooses deliberately among `dataclass`, `Pydantic`, `NamedTuple`, and a plain class by the trust-boundary rule; knows when `__slots__` earns its place; and knows to reach for `__init_subclass__` instead of a metaclass.
>
> **A principal engineer** designs *types*, not just classes — modeling a domain so the structure of the code mirrors the structure of the problem; favors composition and shallow hierarchies, and can articulate *why*; sets the team's conventions for these choices; and can teach the unifying mechanisms — "these five features are all descriptors" — so the team reasons from mechanism rather than memorized feature lists.

## 6.11 At the Interview Table

The object system is the richest vein of staff-level Python interview questions, because it has clear depth gradients — every topic here has a senior answer and a deeper staff answer.

**The question: "Explain the method resolution order. What does `super()` actually do?"**

"The MRO is the ordered list of classes Python searches for an attribute or method — you can see it as `Cls.__mro__`. For multiple inheritance it's computed by C3 linearization, which guarantees a class comes before its bases and preserves the order bases are listed; if no consistent order exists, Python refuses to create the class. And `super()` — this is the part people get wrong — does *not* call 'the parent class.' It calls *the next class in the MRO of the actual instance*. Under single inheritance that's the same thing, so the distinction never shows. Under multiple inheritance it's crucial: `super()` in class `A` might call `B`, a class `A` doesn't even inherit from, because `B` is next in the MRO of the object that's actually running. That's why cooperative multiple inheritance works only if *every* class uses `super()` — one class calling a base directly by name skips part of the chain." Naming C3 and explaining the next-in-MRO subtlety is a clear staff-level signal.

**The question: "What's the `__eq__`/`__hash__` contract?"**

"If two objects are equal, they must have the same hash — equal implies equal-hash. The reason is mechanical: sets and dicts find an object by hashing it to a slot; if two equal objects hash to different slots, lookup checks the wrong slot and wrongly reports the object as absent. There's a trap: defining `__eq__` automatically sets `__hash__` to `None`, making instances unhashable — Python does that on purpose, because the default identity-based hash would violate the contract once you've made equality value-based. So when you define `__eq__`, you decide: if the object is immutable and should be hashable, define `__hash__` from the *same fields* — the clean way is `hash` of a tuple of those fields. If it's mutable, leave it unhashable, because a hash that changes while the object is in a set effectively loses it." If you can add what breaking the contract looks like — silent wrong answers from sets and dicts, not a crash — you have shown you understand the *consequence*.

**The question: "When would you use a dataclass versus a Pydantic model versus a plain class?"**

"`dataclass` for data your own code produced and trusts — internal structures; it generates the boilerplate correctly, including the hash contract, with no dependency and no runtime validation. `Pydantic` when the data crosses a trust boundary from outside — an API request, a config file, anything external — because Pydantic *validates and coerces* at construction and fails loudly on bad data, which is exactly what you want at the edge. A plain class when the type is mostly behavior rather than data, or you need full control of construction. The one-line rule I use: trusted data, `dataclass`; untrusted data, `Pydantic`."

**The question: "How does `property` work?"**

"`property` is a *descriptor* — an object that customizes attribute access by implementing `__get__` and `__set__`. When you access `obj.x` and the class attribute `x` is a descriptor, Python calls its `__get__` instead of returning a stored value. `property` isn't special syntax — it's an ordinary standard-library class implementing that protocol. And the same protocol is behind `cached_property`, `classmethod`, `staticmethod`, bound methods, and ORM fields — they're all descriptors. If I needed validation logic reused across many classes, I'd write my own descriptor, because a `property` is bound to a single class and a descriptor isn't."

**The red flags.** Describing `super()` as "calls the parent" with no awareness of the MRO. Not knowing that defining `__eq__` disables `__hash__`, or not understanding *why*. Reaching for a metaclass when `__init_subclass__` would do — a sign of someone who has read about metaclasses but not internalized the modern guidance. And reflexively defaulting to deep inheritance for code reuse where composition would be cleaner — the instinct that most marks an engineer as not yet thinking like a designer.

## 6.12 The Forge

**Drill 6.1.** Given a diamond hierarchy — `A`, then `B(A)` and `C(A)`, then `D(B, C)` — write out `D.__mro__` by hand using the C3 rules (class before bases, declared order preserved), then verify with `D.mro()`. Then add `print` and `super().__init__()` calls to every `__init__` and confirm the execution order matches the MRO.

**Drill 6.2.** For each, predict whether instances are hashable, and explain: a plain class with no `__eq__`; a class with `__eq__` but no `__hash__`; a class with both; a `@dataclass`; a `@dataclass(frozen=True)`.

**Build 6.1.** Build a reusable, validated typed-attribute system using descriptors. Write descriptor classes `Typed(expected_type)` and `Bounded(min, max)` that validate on assignment, and use them on a couple of model classes. Make the descriptors learn their own attribute name via `__set_name__`. Demonstrate that invalid assignments raise clearly and that the descriptors are genuinely reused across classes.

**Build 6.2.** Model a small domain — a `Money` value object and an `Order` containing line items — using the right tools for each: `Money` should be an immutable, hashable value object (a `frozen` dataclass); `Order` is a mutable entity. Implement value-based equality where appropriate, a useful `__repr__` everywhere, and the comparison needed to sort line items by price. Write a paragraph defending each type choice.

**Investigate 6.1.** Take a class with a value-based `__eq__` and a *deliberately inconsistent* `__hash__` (different fields than `__eq__`). Put many instances into a set and into a dict, and construct a sequence of operations that makes the inconsistency *observable* — a membership test that wrongly returns `False`, a deduplication that lets a duplicate through. Then fix `__hash__` and show the bug closing. Write up exactly why the broken version fails, in terms of hash slots.

**Investigate 6.2.** Measure the memory cost of `__slots__`. Create a class with several attributes, instantiate a large number of objects, and measure total memory. Repeat with `__slots__` (or `@dataclass(slots=True)`). Report the saving as a percentage and per instance, and explain it using the object model from Chapter 2.

**Design 6.1.** You are designing the core domain types for an e-commerce system: products, a shopping cart, orders, customers, money, addresses. For each type, decide and *defend in writing*: is it an entity (identity matters, mutable) or a value object (equality by value, immutable)? Which tool — plain class, `dataclass`, `frozen` dataclass, `NamedTuple`, `Pydantic` model? Where are the trust boundaries (what data comes from outside and must be validated)? Should any of them be hashable, and if so why? Where, if anywhere, is inheritance appropriate, and where is composition better? Produce a one-to-two-page design with the reasoning, not just the answers. This is the exercise that most directly rehearses the staff-level skill: designing types, not writing classes.

---

[← Chapter 5: Functions and Decorators](05-functions-and-decorators.md) · [Home](README.md) · [Chapter 7: Iteration, Generators, and the Lazy Mindset →](07-iteration-and-generators.md)
