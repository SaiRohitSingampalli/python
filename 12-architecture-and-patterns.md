# Chapter 12 — Architecture and Design Patterns

*Part III — Structure*

[← Chapter 11: Packaging, Tooling, and the Developer Loop](11-packaging-tooling-loop.md) · [Home](README.md) · [Chapter 13: The GIL and Concurrency Models →](13-gil-and-concurrency-models.md)

---

## 12.1 The Question Behind the Chapter

Architecture is the most over-discussed and least-understood word in software. It is invoked to justify enormous up-front complexity and dismissed as ivory-tower abstraction; it is the subject of entire shelves of books and of cargo-culted patterns applied where they do harm. This chapter tries to cut through that with a single, deflationary definition that turns out to be genuinely useful:

**Architecture is the set of decisions about what can change independently of what.**

That is it. When you decide that the web layer and the business logic are separate, you are deciding that you can change one without touching the other. When you put a `Protocol` at a seam, you are deciding that either side can be swapped without the other noticing. When you choose a modular monolith over microservices, you are making a decision about the *granularity* at which things can change and deploy independently. Every architectural decision, underneath, is a decision about independence — about decoupling — and the measure of a good one is simple: does it make the *changes you will actually need to make* cheaper, without making everything else prohibitively expensive?

This definition does real work. It tells you architecture is not about elegance or about applying named patterns; it is about *anticipating change and arranging the code so the likely changes are cheap*. It tells you that you cannot do architecture well without a guess about what will change — which is why over-engineering (decoupling things that will never vary independently) and under-engineering (entangling things that will) are both failures of the same skill. And it gives you the lens for the rest of the chapter: design patterns are vocabulary, layered and hexagonal architectures are decoupling schemes, the monolith-versus-microservices question is a decoupling-granularity question, and all of it is judged by the same standard — cheaper change where change is needed.

This chapter is also the bridge into the rest of the book. It is the last chapter of "Structure," and it feeds directly into the system-design interview, which is taught across Chapters 12, 17, 18, and 19. The judgment built here is rehearsed in every system-design round.

## 12.2 Design Patterns: Vocabulary, Not Goals

The "Gang of Four" design patterns — the 1994 catalog of twenty-three patterns — are part of the field's common culture, and a staff engineer should know them. But they must be understood correctly, and the correct understanding has two parts: patterns are *vocabulary*, and many of the classic patterns are *artifacts of less expressive languages*.

The classic catalog was written for C++ and Smalltalk. Many of its patterns are *workarounds for things those languages could not do directly* — and Python *can* do those things directly, which means the pattern, in Python, collapses into a language feature. A few concrete examples, because they are the ones interviewers probe:

The **Strategy pattern** — encapsulating an interchangeable algorithm in an object — exists because in those languages a "behavior" had to be wrapped in a class to be passed around. In Python, functions are first-class objects (Chapter 5). A strategy *is* a function; you pass the function. The pattern, in Python, is "pass a function," and erecting a `Strategy` class hierarchy to do it is ceremony.

The **Singleton pattern** — ensuring one instance of something — exists because those languages had no simpler way to have one shared, globally-reachable object. In Python, a *module* is itself a singleton: it is imported once, cached in `sys.modules` (Chapter 9), and every importer gets the same one. A module-level object *is* a singleton, with no pattern required.

The **Visitor pattern** — adding operations to a type hierarchy from outside it — is an elaborate workaround for the absence of multiple dispatch. Python has `functools.singledispatch` (Chapter 5), and the structural `match` statement, either of which does the job directly.

This is not "design patterns are useless." It is the more precise point that **patterns are vocabulary**. Their genuine, lasting value is *communication*: saying "this is a Strategy" or "we'll use an Adapter here" transmits a design idea to another engineer in two words, and that shared vocabulary is real and worth having. What patterns are *not* is *goals*. A design is not better for containing more named patterns; it is better for making the likely changes cheaper. The failure mode — and it is common, especially in engineers who have just read the pattern book — is reaching for patterns as a sign of sophistication, wrapping a three-line function in a Factory and an Abstract Factory and a Strategy because the patterns feel like good engineering. They are not good engineering; they are ceremony, and ceremony is a cost. Know the patterns as vocabulary. Apply them only when they make a real change cheaper. And know which ones your language has already absorbed.

## 12.3 The Patterns That Still Earn Their Keep

Some patterns are *not* language-feature artifacts — they describe genuine structural ideas that remain useful in Python, expressed Pythonically. A working set worth knowing:

**Adapter.** An Adapter wraps an interface you have to make it look like the interface you need. This is genuinely useful and not absorbed by any language feature — it is the pattern you reach for when integrating a third-party library, a legacy component, or an external service whose interface does not match what your code expects. You write a small wrapper class that presents *your* expected interface and translates calls through to the foreign one. The Adapter is also the concrete mechanism behind the "ports and adapters" architecture of Section 12.5 — so it is worth holding firmly.

**Observer.** The Observer pattern — objects registering interest in an event and being notified when it occurs — describes the publish/subscribe relationship, and that relationship is real and recurring: event systems, callbacks, the decoupling of "something happened" from "here is who cares." In Python it is expressed lightly — a list of callback functions, an event-emitter object — rather than as the heavy class hierarchy of the original. The *idea* (decouple the emitter from the listeners) earns its keep; the heavyweight form does not.

**Factory.** A Factory is, at bottom, "a function or method whose job is to construct and return an object, possibly choosing *which* object based on input." Stripped of the elaborate class hierarchies the classic catalog wrapped it in, this is genuinely useful — a `create_storage(config)` function that returns a database-backed or in-memory storage object depending on configuration is a factory, and it is good design. Often, in Python, the factory is just a function, or a `classmethod` alternative constructor (Chapter 6). The *idea* — centralize and abstract construction — survives; the ceremony does not.

**State machine.** When an object moves through a defined set of *states* with defined *transitions* — an order that is pending, then paid, then shipped, then delivered; a connection that is connecting, open, closing, closed — modeling that explicitly as a state machine is genuinely valuable. It makes the legal transitions explicit, makes illegal ones catchable, and makes the object's behavior comprehensible. This is a pattern that earns its keep precisely because it makes a real category of change and a real category of bug cheaper to handle.

The common thread: the patterns that survive are the ones describing a *structural idea* — wrap a foreign interface, decouple emitter from listener, abstract construction, model explicit states — rather than the ones working around a missing language feature. Know the survivors, express them Pythonically and lightly, and apply them by the change-cost test.

## 12.4 Layered Architecture

Now from patterns (small-scale structure) to *architecture* (the large-scale structure of a whole application). The most common, and the right *default*, large-scale structure is the **layered architecture**.

The idea: divide the application into horizontal layers, each with a defined responsibility, and impose a strict rule that *dependencies point in one direction only*. A typical three-layer scheme:

The **presentation layer** handles the *interface to the outside world* — HTTP request handling, the API surface, the CLI, the rendering of responses. Its job is to translate between the outside world's formats and the application's internal calls.

The **business-logic layer** (sometimes called the service or domain layer) holds the *actual logic of the application* — the rules, the workflows, the decisions, the things the application fundamentally *does*. This is the valuable core; it should be the largest and most carefully designed layer.

The **data layer** handles *persistence* — talking to the database, to caches, to external storage.

The rule that makes this an architecture rather than just three folders: **dependencies point downward only.** Presentation may depend on business logic; business logic may depend on the data layer; *nothing* depends upward. The business logic does not know about HTTP; the data layer does not know about business rules.

Connect this to the chapter's definition. The layering is a decision about *what can change independently*. Because the business logic does not depend on the presentation layer, you can change the API — add a CLI, swap the web framework, change the request format — without touching the business logic. Because the business logic does not depend on the data layer's specifics, you can change the database, add a cache, alter the persistence strategy, without touching the rules. The layering *buys* independence along exactly the seams where change is likely. And it buys *testability*: the business logic, depending on nothing above it and only on an abstraction below it, can be tested without a web server and without a real database — which is the difference between a fast, reliable test suite and a slow, flaky one.

There is a common anti-pattern the layering exists to prevent, and it has a name worth knowing: **the "fat controller"** or, more bluntly, business logic *leaking into the presentation layer*. It happens when a web request handler — which should only translate HTTP to an internal call and back — instead contains the actual business rules: the validation, the calculation, the workflow, all written inline in the route function. Now the business logic is *coupled to HTTP*; it cannot be tested without a web request, cannot be reused from a CLI or a background job, and cannot survive a change of web framework. The layered architecture's discipline — handlers stay thin, logic lives in the service layer — is what prevents this, and a staff engineer reviewing code watches for the fat controller specifically.

## 12.5 Hexagonal Architecture: Ports and Adapters

The layered architecture has a refinement, sometimes called **hexagonal architecture** or, more descriptively, **ports and adapters**. It takes the layering's central insight — protect the core from the edges — and makes it sharper.

The idea: place the *business logic* — the valuable core — at the center, and have it depend on *nothing concrete*. Everything external — the database, the web framework, the message queue, the third-party APIs, the filesystem — sits at the *edges*. Between the core and each edge is a **port**: an *interface*, defined *by the core*, in the core's own terms, describing what the core *needs* — "I need to be able to save and load an order," "I need to be able to send a notification." And for each port there is an **adapter**: a concrete implementation, living at the edge, that satisfies the port's interface using a real external thing — a Postgres adapter satisfying the storage port, an email-service adapter satisfying the notification port.

The decisive property: **the core depends only on the ports — the interfaces — never on the adapters.** The dependency points *inward*: the adapters depend on (implement) the core's ports; the core depends on nothing outside itself. The database adapter knows about the core's storage port; the core does not know there is a database at all.

This is where Chapter 10's `Protocol` becomes architecturally central, and the chapters connect. A *port* is, in Python, naturally expressed as a `Protocol` — a structural interface defined by the core, describing what it needs. The *adapters* are classes at the edge that *satisfy* that protocol — and because `Protocol` is structural, they satisfy it without importing it, without inheriting from it, without the core and the adapter knowing about each other at all. The core defines `StoragePort` as a `Protocol`; a `PostgresStorage` class at the edge simply *has the right methods*; a dependency-injection step (Section 12.8) wires the concrete adapter into the core at startup. The core is testable in complete isolation, because a test can supply a trivial in-memory adapter that satisfies the same protocol. The web framework can be swapped, the database can be swapped, an external service can be swapped — each is just a different adapter behind the same port, and the core never changes.

Hexagonal architecture is, in the chapter's terms, the maximal expression of "what can change independently": it makes *every external dependency* independently swappable, by routing every one of them through a port the core owns. Its cost is the indirection — every external interaction goes through an interface and an adapter rather than being called directly — and that cost is real, so it is not free and not always warranted. But for an application of meaningful size and lifespan, where the business logic is valuable and the external dependencies are likely to change, it is an excellent structure, and it is the natural place the layered architecture grows toward.

## 12.6 Domain-Driven Design as a Modeling Discipline

**Domain-Driven Design** (DDD) is a large body of ideas about building software around a deep model of the business *domain* it serves. It has a reputation as a heavyweight methodology with an intimidating vocabulary, and that reputation causes engineers to either cargo-cult its patterns or dismiss it entirely. The useful framing — the staff-level framing — is that **DDD's lasting value is as a modeling discipline, not as a pattern catalog.**

The single most valuable idea in DDD is **ubiquitous language**: the principle that the code should use *exactly* the vocabulary the business uses for its domain, with no translation layer. If the business says "policy" and "claim" and "premium," the code has classes named `Policy`, `Claim`, `Premium` — not `Record`, `Item`, `Amount`. This sounds trivial and is not: when the code's vocabulary matches the domain's, a conversation between an engineer and a domain expert maps directly onto the code, miscommunication drops, and the code itself becomes a form of documentation of how the business works. When the vocabularies diverge, every conversation requires translation, and translation loses things.

A second genuinely useful idea is the **bounded context**: the recognition that a large domain does not have *one* model — it has several, each valid within its own area, and the same word can mean different things in different contexts. A "customer" in the billing context (an account, a payment method, an address) is a different model from a "customer" in the support context (a contact, a history of tickets, a sentiment). DDD's insight is to *stop trying to build one universal `Customer` class* that serves every context — that class becomes a bloated, contradictory mess — and instead let each bounded context have its *own* model, connected at well-defined boundaries. This maps directly onto module and service boundaries, and it is one of the most useful tools for deciding *where to draw lines* in a large system.

DDD has more — entities versus value objects (which Chapter 6's type-design exercise already touched), aggregates, repositories, domain events — and these are useful when a domain is genuinely complex. But the staff-level judgment is this: **take DDD's modeling discipline — ubiquitous language, bounded contexts, modeling the domain seriously — and apply it widely; treat DDD's full pattern catalog as something to reach for only when the domain's complexity genuinely warrants it.** A simple CRUD application does not need aggregates and domain events, and forcing them on it is exactly the ceremony Section 12.2 warned against. A genuinely complex domain — insurance, logistics, finance — benefits from the full toolkit. The discipline scales down to every project; the heavyweight patterns do not, and applying them indiscriminately is a failure of judgment, not a sign of sophistication.

## 12.7 The Monolith and the Microservice

The most consequential architecture decision a team makes about a system's large-scale shape is whether it is *one deployable unit* or *many*, and this section gives the chapter's clear position.

A **monolith** is a single deployable application — one codebase, one process (or a set of identical processes), deployed as a unit. A **microservices** architecture decomposes the system into many small, independently deployable services, each owning a slice of the functionality, communicating over the network.

The microservices model has real benefits, and they are mostly *organizational*: independent teams can own, develop, and deploy their services without coordinating with each other; each service can scale and be resourced independently; a failure can be isolated to one service. For a *large organization* — many teams, who would otherwise contend over one codebase and one deployment — these benefits are genuine and significant.

But microservices have severe costs, and they are *technical*: the moment a function call becomes a network call, you have inherited the entire catalog of distributed-systems problems — network failures, latency, partial failure, the need for retries and timeouts and circuit breakers, eventual consistency, the difficulty of a transaction that spans services, the operational burden of running and observing many services instead of one. Chapter 19 is devoted to those problems precisely because they are hard. A microservices architecture *signs the team up for all of them*.

The chapter's position, stated plainly because the industry spent a decade learning it the expensive way: **the default should be a monolith — specifically a well-structured, modular monolith — and microservices should be adopted deliberately, when a specific problem demands them, not reflexively because they are the prestigious choice.**

A **modular monolith** is the synthesis. It is a single deployable unit — so it has none of the distributed-systems costs; a call between modules is a function call, a transaction is a real transaction, there is one thing to deploy and observe. But *internally* it is rigorously modular — organized into well-bounded modules (Chapter 9's package architecture, Section 12.4's layering, Section 12.6's bounded contexts) with clear interfaces between them and an enforced acyclic dependency graph. It gets the *structural* benefits of decomposition — independent reasoning, clear ownership boundaries, the ability to change one module without disturbing others — without paying the *operational and distributed-systems* costs of actually splitting into separate deployables.

The decisive insight, the one a staff engineer carries into this decision: a well-structured modular monolith *can be split into microservices later* if and when a specific module genuinely needs independent deployment or scaling — because the module boundaries are already clean, drawn along the right seams. The reverse is far harder. So the modular monolith is not a compromise or a stepping stone you are embarrassed by; it is, for most systems and most team sizes, simply the *right* architecture — it keeps the option of microservices open while not paying their cost until a real need arrives. Microservices solve an *organizational* problem (many teams contending over one thing); if you do not have that organizational problem, adopting microservices means paying a large technical cost to solve a problem you do not have. Decide on the evidence, default to the modular monolith, and split deliberately.

## 12.8 Dependency Injection

One more pattern, because it is the practical mechanism that makes the layered and hexagonal architectures *work*: **dependency injection** (DI).

The idea is simple and the name makes it sound grander than it is. A component that needs another component — a service that needs a storage adapter, a handler that needs a service — should *receive* that dependency from outside (have it "injected"), rather than *constructing it itself*.

```python
# Without DI — the service constructs its own dependency. Coupled.
class OrderService:
    def __init__(self):
        self.storage = PostgresStorage(connection_string="...")

# With DI — the dependency is passed in. Decoupled.
class OrderService:
    def __init__(self, storage: StoragePort):
        self.storage = storage
```

The difference looks small and is architecturally large. In the first version, `OrderService` is *welded* to `PostgresStorage` — it names it, constructs it, knows its connection string. You cannot test `OrderService` without a real Postgres database; you cannot swap the storage; the dependency points the wrong way (the business-logic service depends on a concrete data-layer class). In the second version, `OrderService` depends only on the *abstraction* — the `StoragePort` protocol — and the *concrete* implementation is handed to it from outside. Now you can test `OrderService` with a trivial in-memory adapter, you can swap Postgres for anything else that satisfies the port, and the dependency points inward as the hexagonal architecture requires. Dependency injection is the *technique* that makes ports and adapters real: the port is the protocol, the adapter is the implementation, and DI is the act of supplying the adapter to the core.

Where do the dependencies get constructed and wired together, if not inside the components? At the application's *composition root* — a single place, typically at startup, where the concrete adapters are constructed and injected into the components that need them. The whole object graph is assembled in one place; everything else just receives what it needs.

Python does not need a framework for this. Because the technique is just "pass dependencies as constructor arguments," plain Python — constructing objects and passing them in, at the composition root — is entirely sufficient and is what most projects should do. There *are* dependency-injection *frameworks* and *containers* for Python, and they have a place in very large applications where wiring the object graph by hand becomes unwieldy — but they are not necessary, and reaching for one on a small or medium project is, again, the ceremony the chapter keeps warning against. (FastAPI, which Chapter 17 covers, has a notably elegant built-in dependency-injection mechanism, which is one of the reasons it is pleasant for structured applications — that is DI provided by the framework, used well.) The principle is what matters: components receive their dependencies; they do not construct them; the wiring happens at one composition root. That principle, not any framework, is what makes the architecture of this chapter actually testable and actually decoupled.

> ### War Story — The Service That Could Not Be Tested
>
> A team had a service whose core business logic was genuinely valuable and genuinely complex — and almost entirely untested, because it *could not* be tested. Every class in the business logic constructed its own dependencies directly: the order service created its own database connection, the pricing logic instantiated its own HTTP client to a third-party tax API, the notification logic built its own connection to the email provider. There was no dependency injection anywhere; every component reached out and grabbed what it needed.
>
> The consequence was that testing *any* piece of business logic required *all of its dependencies to be real and available*: a real database, the real third-party tax API, the real email provider. A "unit test" of the pricing logic actually called the external tax service. The tests were therefore slow, they were flaky (they failed whenever any external dependency had a bad moment), they could not run in CI without a fragile web of test infrastructure, and — predictably — engineers stopped writing them. The valuable, complex core had almost no test coverage, not because the team did not value testing but because the *architecture made testing impractical*.
>
> The fix was dependency injection, applied throughout. Each component was changed to *receive* its dependencies — typed against `Protocol` ports — rather than construct them. The concrete adapters (real database, real tax client, real email client) were constructed and wired together at a single composition root at startup. And now the business logic could be tested with trivial in-memory fakes that satisfied the same protocols: the pricing logic tested against a fake tax adapter that returned canned numbers, the order service tested against an in-memory storage adapter. The tests became fast, deterministic, and runnable anywhere — and the coverage of the valuable core climbed from almost nothing to thorough, because testing had become *easy*.
>
> The lesson reaches past testing. The team had not had "a testing problem" — they had had an *architecture* problem, and poor testability was its most visible symptom. Code that constructs its own dependencies is coupled to them, and coupled code is hard to test, hard to change, and hard to reason about. Dependency injection — the simple discipline of passing dependencies in rather than reaching out for them — is not primarily a testing trick; it is the technique that makes the decoupled, layered, port-and-adapter architecture of this whole chapter *real* instead of aspirational. Untestable code is usually trying to tell you that its architecture is wrong.

> ### The Level Line — Architecture
>
> **A senior engineer** structures an application into sensible layers, knows the common design patterns and applies them where they fit, separates business logic from the web layer, and writes testable code.
>
> **A staff engineer** understands architecture as decisions about what can change independently; knows which classic patterns are language-feature artifacts in Python and which still earn their keep; applies layered and hexagonal architecture deliberately, using `Protocol` ports and dependency injection to decouple the core from its edges; takes DDD's modeling discipline without cargo-culting its full catalog; and holds a clear, defensible position on the modular-monolith-versus-microservices question.
>
> **A principal engineer** makes the consequential architecture decisions for a system and can defend them in terms of the changes they make cheap and the costs they accept; resists both over-engineering and under-engineering by reasoning explicitly about what will and will not change; sets the architectural standards a whole team or organization builds within; and can teach the deflationary, change-cost view of architecture so that the engineers around them stop cargo-culting patterns and start reasoning about decoupling.

## 12.9 At the Interview Table

Architecture is the substance of the *system-design interview*, and this chapter — together with Chapters 17, 18, and 19 — is the book's preparation for it. The system-design round is not testing whether you can recite patterns; it is testing whether you can reason about structure, trade-offs, and change.

**The question: "How would you structure this application?" (a system-design opener)**

A strong answer reasons from the chapter's principles rather than naming patterns. "I'd start with a layered architecture — a thin presentation layer that just translates the outside world to internal calls, a business-logic layer holding the actual rules, a data layer for persistence — with dependencies pointing strictly downward, so the business logic doesn't know about HTTP or about the specific database. As it grows, I'd push toward ports and adapters: the business logic at the center depending only on `Protocol` interfaces it defines, and the database, external services, and framework as swappable adapters at the edges, wired in by dependency injection at a composition root. That makes the core independently testable and makes the edges independently swappable. The whole thing I'd build as a *modular monolith* — one deployable, rigorously modular inside — unless there's a specific reason it needs to be distributed." That answer demonstrates the reasoning the interviewer wants.

**The question: "Microservices or a monolith — which, and why?"**

"My default is a well-structured *modular monolith*, and I'd adopt microservices only for a specific reason. Microservices' benefits are mostly *organizational* — independent teams deploying independently, independent scaling — and they're real if you have a large organization with many teams contending over one system. But the *costs* are technical and severe: every cross-service call is a network call, so you inherit the whole distributed-systems problem set — partial failure, retries, timeouts, eventual consistency, distributed transactions, the operational burden of running many services. A modular monolith gets the *structural* benefits of decomposition — clear module boundaries, independent reasoning, clean ownership — without those costs, because internally it's modular but it deploys as one unit. And critically, a well-structured modular monolith can be split *later*, along boundaries that are already clean, if a specific module genuinely needs independent deployment — whereas premature microservices are very hard to merge back. So: modular monolith by default, split deliberately when there's evidence of need."

**The question: "Why don't we see the Singleton pattern much in Python?"**

"Because a Python module is already a singleton. A module is imported once, cached in `sys.modules`, and every importer gets that same module object — so a module-level object is a shared single instance with no pattern required. More broadly, several of the classic Gang of Four patterns are workarounds for things less expressive languages couldn't do directly — Strategy is just passing a function, since functions are first-class objects; Visitor is `singledispatch` or `match`. The patterns are valuable as *vocabulary* for communicating designs, but a lot of the catalog dissolves into language features in Python, and the ones that remain — Adapter, Observer, Factory, state machines — are the ones describing a real structural idea rather than a missing feature."

**The question: "What is dependency injection and why does it matter?"**

"It's the discipline of a component *receiving* its dependencies from outside rather than constructing them itself — passing them in as constructor arguments. It matters because constructing your own dependencies *couples* you to them: you can't test the component without the real dependency, you can't swap the dependency, and the dependency points the wrong way. With DI, a component depends only on an abstraction — a `Protocol` port — and the concrete implementation is injected, so it's testable with a fake and the implementation is swappable. It's the technique that makes ports-and-adapters real. In Python you don't need a framework for it — passing dependencies in and wiring the object graph at one composition root is plain Python and is what most projects should do."

**The red flags.** Reaching for microservices reflexively, as the obviously-sophisticated choice, with no account of the costs — a major signal of someone who has read about architecture but not operated it. Cargo-culting patterns — proposing a Factory and an Abstract Factory for a trivial construction. Business logic welded to the web framework with no instinct to separate it. Not knowing what dependency injection is, or thinking it requires a framework. And, generally, treating architecture as the application of named patterns rather than as reasoning about what must change independently.

## 12.10 The Forge

**Drill 12.1.** For each classic pattern, state whether it is largely absorbed by a Python language feature or still earns its keep, and explain: Strategy, Singleton, Visitor, Adapter, Observer, Factory, Iterator, State.

**Drill 12.2.** Given a small application described in prose (a web service with a handful of endpoints, a database, and one external API call), sketch its layered architecture: name the layers, state what goes in each, and draw the dependency arrows. Identify where the "fat controller" anti-pattern would creep in and how the layering prevents it.

**Build 12.1.** Take an application (or a piece of one) that currently has its business logic entangled with its web framework and its database — fat controllers, classes constructing their own dependencies. Refactor it into a layered, port-and-adapter structure: extract the business logic into a core that depends only on `Protocol` ports, push the database and the framework to the edges as adapters, and wire them with dependency injection at a composition root. Then write tests for the core using in-memory fake adapters, and note how much easier testing became.

**Build 12.2.** Implement a small system using ports and adapters from the start: a core piece of logic, a `Protocol` defining what it needs from storage, and *two* adapters satisfying that port (an in-memory one and a real-database one). Demonstrate that the core is swapped between them with a one-line change at the composition root and that the core's tests run against the in-memory adapter with no database.

**Investigate 12.1.** Find a real open-source Python project of moderate size and analyze its architecture. Identify its layers (or lack of them), where its business logic lives, how coupled it is to its framework and database, whether it uses dependency injection, and whether it is a monolith or microservices. Write up what is well-structured and what you would change, in the language of this chapter — what can and cannot currently change independently.

**Design 12.1.** You are the staff engineer designing a new system of moderate complexity — pick a concrete domain (an e-commerce order system, a content-publishing platform, a booking system). Write a three-to-four-page architecture document: the layered or hexagonal structure and why; where the bounded contexts and module boundaries fall and why there; the major `Protocol` ports and what is on each side of them; the monolith-versus-microservices decision with its justification; how dependencies are injected and where the composition root is; and — explicitly, because it is the chapter's whole thesis — *what changes this architecture makes cheap, and what costs it accepts to do so*. This is the system-design interview rehearsed on paper; the rest of the book's system chapters will deepen exactly this exercise.

---

# Part III Capstone — A Properly Architected, Packaged, Plugin-Extensible Library

Part III has been about *structure* — organizing code for the humans and organizations that must live with it. The capstone takes the configuration-loading library you built at the end of Part II and turns it into a *properly structured, properly packaged, extensible product* — the difference between "code that works" and "code an organization can build on."

**The brief.** Take the Part II configuration loader and elevate it. It must now be properly architected, properly packaged with the modern stack, fully typed, and *extensible by third parties* through a plugin mechanism — for example, supporting additional configuration *sources* (a JSON file, a TOML file, environment variables, a remote source) where new sources can be added by separate plugin packages without modifying the core.

**What it must demonstrate, by chapter:**

From **Chapter 9** — a deliberate package and module structure with an acyclic dependency graph, a curated `__init__.py` defining the library's public API, and a plugin system built on **entry points**, so a separately-installed package can register a new configuration source and the core discovers it via `importlib.metadata`.

From **Chapter 10** — full type annotations that pass a strict type checker. The seam between the core and the configuration-source plugins must be a `Protocol` — a structural port that any source adapter satisfies. Use the precision tools where they fit (`TypedDict` or a typed model for the loaded configuration shape, `Literal` for any fixed vocabularies).

From **Chapter 11** — the complete modern packaging setup: a single `pyproject.toml`, the `src` layout, a build backend, dependencies and a `dev` extra, a committed lock file, `ruff` configured, pre-commit hooks, and a task runner. The library must build cleanly into a wheel. Set up the developer loop so a new contributor is productive in one command.

From **Chapter 12** — the architecture must be deliberate and defensible. The *core* configuration logic depends only on `Protocol` ports — a source port, perhaps a validation port — and the concrete sources are *adapters* at the edge. Dependencies are *injected*: the core receives its sources rather than constructing them, wired at a composition root. The structure must be layered or hexagonal, the business logic decoupled from the I/O, and the whole thing testable with in-memory fake adapters.

**The deliverables:**

1. The library itself — architected, packaged, typed, plugin-extensible — plus at least one example *plugin package* (a separately-packaged configuration source) demonstrating that the extension mechanism genuinely works across package boundaries.
2. A complete test suite, including tests of the core that use in-memory fake adapters and run without any real I/O.
3. An *architecture document* — three to four pages — defending every significant structural decision: the module boundaries, the public API surface, the ports and where they sit, the plugin mechanism, the dependency-injection wiring, and — in the chapter's own terms — what changes this architecture makes cheap and what it costs.

**On the interview.** This capstone is, again, deliberately interview-grade. It is the kind of project that lets you answer "tell me about something you've architected" with substance — a small but genuinely well-structured system where you can defend the layering, point to the ports, explain the plugin mechanism, and articulate the change-cost reasoning behind every boundary. Build it as though an interviewer will ask you, of every decision, "why did you do it that way" — because that question is exactly what the system-design rounds, which the rest of the book now prepares you for, are made of.

---

[← Chapter 11: Packaging, Tooling, and the Developer Loop](11-packaging-tooling-loop.md) · [Home](README.md) · [Chapter 13: The GIL and Concurrency Models →](13-gil-and-concurrency-models.md)
