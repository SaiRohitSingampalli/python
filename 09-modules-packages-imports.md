# Chapter 9 — Modules, Packages, and Imports

*Part III — Structure*

[← Chapter 8: Errors and the Discipline of Failure](08-errors-and-failure.md) · [Home](README.md) · [Chapter 10: Types as a Design Tool →](10-types-as-design-tool.md)

---

## 9.1 The Question Behind the Chapter

`import` is the first keyword every Python programmer learns to *use* and one of the last they learn to *understand*. For most engineers it is a black box: you write `import something`, the something becomes available, and the mechanism is never examined — until the day it misbehaves. A circular import that crashes at startup with a baffling half-initialized module. An `ImportError` that depends on which directory you ran the program from. A package that works when installed one way and not another. Each of these is opaque under the black-box model and obvious under the real one.

This chapter opens the box. By the end you will know what a module object actually *is*, how Python *finds* the thing you import, the precise difference between a regular package and a namespace package, the three-phase protocol that finders and loaders implement, why circular imports happen and the four real ways to fix them, and how the entry-point mechanism turns a package into an extensible plugin host. None of this is exotic. All of it is the everyday machinery of how Python code is assembled into a program — and understanding it is the difference between an engineer who is mystified by import errors and one who reads them like sentences.

## 9.2 What a Module Actually Is

Strip away the syntax and a module is a simple thing: **a module is an object** — specifically, an object whose attributes are the names defined at the top level of a `.py` file. When Python imports `mymodule`, it does not perform some special incantation; it *runs the file* `mymodule.py` from top to bottom, in a fresh namespace, and then wraps that namespace in a module object. Every `def`, every `class`, every top-level assignment in the file becomes an attribute of that module object. `mymodule.some_function` is just attribute access on an object, exactly like any other.

This has a consequence engineers stumble over until they know it: **the top level of a module runs, in full, the first time the module is imported.** A `print` at module level prints on import. A database connection opened at module level opens on import. An expensive computation at module level happens on import. The module body is *code*, and importing the module *executes that code*. Most of the time you only put definitions at the top level — `def`, `class`, constants — so "running the module" just means "creating those definitions," and the distinction is invisible. But the moment you put *side-effecting code* at module level, you have made `import` do work, and that is occasionally what you want and occasionally a bug.

The other half of the picture is `sys.modules` — a dictionary, maintained by the interpreter, that maps the name of every already-imported module to its module object. `sys.modules` is a *cache*, and it is the reason the module body runs only *once*. The first time you `import mymodule`, Python runs the file, builds the module object, and stores it in `sys.modules["mymodule"]`. Every subsequent `import mymodule`, anywhere in the program, finds the module already in `sys.modules` and simply hands back the cached object — the file is not re-run. This is why importing the same module from ten different files does not execute it ten times, and it is why a module's top-level state is effectively *shared* across the whole program: there is one module object, in `sys.modules`, and every importer gets that same one. (It is also, as we will see, half the explanation of circular imports.)

## 9.3 How Python Finds a Module

When you write `import requests`, how does Python locate `requests`? The answer is `sys.path` — a list of directory paths that Python searches, *in order*, for the module you asked for.

You can print it: `import sys; print(sys.path)`. What it contains, roughly, and in priority order: the directory of the script being run (or the current directory, in interactive mode) — searched *first*; then any directories listed in the `PYTHONPATH` environment variable; then the standard library's directories; then the `site-packages` directory where third-party packages installed by `pip` or `uv` live. Python walks this list top to bottom and uses the *first* place it finds a match.

The "first match wins, and the script's own directory is searched first" rule is the explanation for a whole family of confusing bugs. The classic one: you name a file of your own `random.py` or `email.py` or `string.py` — the name of a standard-library module — and place it in your project directory. Now `import random` anywhere in your project finds *your* file first, because the script's directory is searched before the standard library, and your project breaks in bewildering ways as code that expected the real `random` module gets yours instead. This is *shadowing*, and it is why "do not name your files after standard-library modules" is real advice and not superstition. The same rule explains why an `import` can succeed when you run a program from one directory and fail from another: `sys.path[0]` depends on where you launched from, so what is importable depends on your working directory — which is exactly the fragility that the `src` layout in Chapter 11 exists to eliminate.

## 9.4 Packages: Regular and Namespace

A *module* is a single file. A *package* is a directory of modules — a way to group related modules under a shared dotted name (`mypackage.submodule.thing`). There are two kinds of package, and the distinction is worth precision because it appears in interviews and in real "why does this import work / not work" debugging.

A **regular package** is a directory containing an `__init__.py` file. The presence of `__init__.py` marks the directory as a package. When you import the package, Python runs its `__init__.py` — that file *is* the package's module body, the code that executes when the package itself is imported. The `__init__.py` can be empty (it then just marks the directory) or it can contain code (it then shapes what the package exposes — Section 9.6).

A **namespace package** (PEP 420) is a package directory *without* an `__init__.py`. Its defining and unusual feature: a single namespace package can be *split across multiple directories*, in multiple locations on `sys.path`, and Python will merge them into one logical package. The motivating use case is a large project — or a plugin ecosystem — where different teams, or different separately-installed distributions, each contribute sub-modules under one shared package name. Each ships its piece; Python assembles them.

For everyday work, the guidance is simple: **use a regular package — include an `__init__.py`** — unless you have a specific, deliberate reason to want the split-across-directories behavior of a namespace package. The `__init__.py`, even when empty, makes the package's boundary explicit and unambiguous, and it gives you a place to put package-level code if you later need one. Namespace packages are a real and useful feature for the plugin-ecosystem case, but a namespace package created *by accident* — because someone forgot the `__init__.py` — is a common source of confusion, and the `src` layout of Chapter 11 specifically helps catch that mistake.

## 9.5 The Import Protocol: Finders and Loaders

`import` is not a monolithic built-in operation — it is a *protocol*, an extensible three-phase process, and understanding the phases is what makes import errors legible and what makes advanced techniques (lazy loading, plugin discovery, even importing from unusual sources) possible.

The three phases, in order:

**Finding.** Given the name `mypackage.module`, Python must locate it. It does this by consulting a list of *finders* on `sys.meta_path` — objects whose job is to answer "do you know where to find this name?" The built-in finders handle the cases you know: one finds built-in and frozen modules, one finds modules on the filesystem via `sys.path`. A finder, asked about a name, either returns a *module spec* — a description of where and how to load the module — or declines, and Python moves to the next finder.

**Loading.** Given the spec the finder produced, a *loader* actually creates the module: it produces the module object, executes the module's code into it, and so on. For ordinary `.py` files the loader compiles the source to bytecode (Chapter 1) and executes it.

**Caching.** The freshly created module object is placed in `sys.modules` so that the next import of the same name skips finding and loading entirely.

Why does a working engineer need this? Because the protocol is *extensible*, and the extension points are where real capabilities live. `sys.meta_path` is a list you can add to — a custom finder can teach Python to import modules from a database, from an encrypted archive, from over a network, from a generated source. The plugin systems of large frameworks, the lazy-import machinery, the import hooks that testing and instrumentation tools rely on — all of them are finders and loaders plugged into this protocol. You will rarely *write* a finder, but knowing that `import` is a pluggable three-phase protocol — rather than an opaque built-in — is what lets you understand the tools that do, and what lets you read an import-related stack trace and know which phase failed.

## 9.6 Relative Imports, Absolute Imports, and `__init__.py`

Within a package, one module often needs to import another. There are two ways to write that import, and a clear convention for which to use.

An **absolute import** names the full path from the project's top-level package: `from mypackage.utils import helper`. An **explicit relative import** names the target relative to the current module's location, using leading dots: `from .utils import helper` (one dot — the same package), `from ..common import thing` (two dots — the parent package).

The convention, and it is close to universal: **prefer absolute imports.** They are unambiguous — `from mypackage.utils import helper` means exactly one thing regardless of where the importing file sits or how the program was launched. They are robust to a file being moved. They are clearer to a reader, who sees the full provenance of the imported name. Relative imports have a legitimate, narrower place — they keep imports *within* a tightly-coupled package concise, and they make a package easier to rename as a whole — but absolute imports are the sound default, and a project that uses them consistently has one less source of fragility.

The `__init__.py` file deserves its own treatment, because it is more than a marker. It is the package's *public face* — the code that runs when the package is imported, and the place where you decide what the package *presents* to the outside world. Three common, legitimate uses:

The **empty `__init__.py`** — it just marks the directory as a regular package. Perfectly fine, and the right choice when the package has no public-API shaping to do.

The **curated `__init__.py`** — it imports selected names from the package's internal modules up to the package level, so that users can write `from mypackage import MainClass` instead of `from mypackage.internal.module import MainClass`. This *defines the package's public API*: the names you surface in `__init__.py` are what you are telling users to depend on; the names buried in submodules are, by convention, internal. Often paired with an `__all__` list, which declares the package's public names explicitly. This is good API design — it lets you reorganize the internals freely as long as the `__init__.py` keeps presenting the same surface.

The **lazy `__init__.py`** — for a package with *heavy* optional dependencies, importing everything eagerly at package-import time can be slow or can pull in dependencies a particular user does not need. PEP 562 lets a module define a module-level `__getattr__` that is consulted when an attribute is *not* found normally — so the package can defer importing a heavy submodule until the moment someone actually accesses it. A large library that does not want `import biglib` to take two seconds and load a dozen optional dependencies uses exactly this pattern.

## 9.7 Circular Imports: Why, and the Four Fixes

A circular import is the situation where module A imports module B, and module B imports module A. It produces one of Python's more confusing failures, and a staff engineer both understands precisely *why* it happens and knows that it is usually a *design signal* — not just a mechanical problem to patch, but a hint that the module boundaries are wrong.

The *why* follows directly from Section 9.2. Recall: importing a module *runs its body top to bottom*, and the module is registered in `sys.modules` as soon as it *starts* (so that the cache works), which means a module can be present in `sys.modules` while still *partially executed*. Now trace a cycle. Module A begins executing. Partway down, A's body hits `import B`. Python starts executing B. Partway down, B's body hits `import A` — and finds A already in `sys.modules`, so it does not re-run A; it hands B the A module object *as it currently is* — which is **half-built**, because A's execution was paused partway down to import B. If B's body now tries to use a name from A that A had not yet defined at the point it paused, B gets an `ImportError` or an `AttributeError`, and the failure message — something about a name not being available, or a partially initialized module — points at a symptom far from the actual cause. That is the circular import: not a mysterious bug, but the entirely logical result of two module bodies each pausing partway through to run the other.

There are four real fixes, and they are listed roughly in order of preference, because the *best* fix is usually the one that addresses the design rather than the mechanics:

**Fix one — restructure.** A circular import very often means your module boundaries are drawn wrong: two modules are mutually dependent, which means they are really *one* concern that has been split, or they share something that belongs in a *third* module. The best fix is frequently to extract the shared piece — the class or function both modules need — into a new module that both import, breaking the cycle by removing the mutual dependency entirely. When a circular import appears, the first question is not "how do I make this import work" but "why are these two modules entangled, and should they be."

**Fix two — import inside the function.** Move the offending `import` from the top of the module *into the function or method that actually uses it*. A function body does not execute at import time — it executes when the function is *called*, by which point both modules have finished loading. The cycle is broken because the import no longer happens during the module-body execution. This is a legitimate, common fix; its small cost is that the import runs on every call (cheap, since `sys.modules` makes repeated imports a dict lookup) and that the dependency is less visible at the top of the file.

**Fix three — import the module, not the name.** Instead of `from B import thing` at the top of A, write `import B` and refer to `B.thing` at the point of use. The difference is timing: `from B import thing` needs `thing` to *exist in B right now*, at import time; `import B` only needs the module object, and `B.thing` is resolved later, when the code actually runs — by which time B is fully built.

**Fix four — `TYPE_CHECKING`.** Often the circular import exists *only* because of type hints — module A imports a class from module B purely to annotate a parameter, and B imports from A for the same reason, and nothing but the annotations actually needs the cross-import. The `typing.TYPE_CHECKING` constant is `False` at runtime but `True` when a type checker analyzes the code. Guarding the import with `if TYPE_CHECKING:` means the import *does not happen at runtime at all* — so no cycle — while the type checker still sees it and can check the annotations. (The annotations themselves must then be strings, or the file must use the `from __future__ import annotations` deferred-evaluation behavior — Chapter 10's territory.) This is the clean fix for the very common "circular import caused purely by typing" case.

## 9.8 `__main__`, `python -m`, and Dynamic Imports

A few remaining pieces of the import machinery are worth knowing because they recur in real projects.

The `if __name__ == "__main__":` idiom is the most common piece of import-related code in Python and is widely used without being understood. `__name__` is a variable Python sets in every module. When a module is *imported*, `__name__` is set to the module's name. When a module is *run directly* (`python myfile.py`), `__name__` is set to the string `"__main__"`. So `if __name__ == "__main__":` is the test "am I being run directly, as opposed to being imported?" — and the code under it runs only in the direct-run case. This is what lets a file be *both* an importable module *and* a runnable script: its definitions are always available to importers, and its "do the thing" code runs only when the file is executed directly. It exists precisely because importing a module runs its body (Section 9.2) — without this guard, importing a script would *execute* the script.

`python -m packagename` runs a package as a program — and when you do, Python looks for and executes a `__main__.py` file inside the package. This is the clean, professional way to make a package runnable: a command-line tool shipped as a package puts its entry logic in `__main__.py`, and `python -m thetool` runs it. Running `-m` also fixes a `sys.path` subtlety in the running-a-script-from-inside-a-package case, which is a further reason it is preferred for anything beyond a trivial single file.

`importlib` is the standard library's interface to the import system *as an API* — it lets you import by name *dynamically*, at runtime, when the name is not known until then: `importlib.import_module(name_from_config)`. This is the mechanism behind plugin loading, behind frameworks that import the modules a configuration file names, behind any "import whatever the user asked for" capability. It is `import` as a function call rather than a statement.

## 9.9 Entry Points and the Plugin Pattern

The chapter closes with the most genuinely powerful structural capability of the packaging system: **entry points**, the mechanism that turns a package into an *extensible host* — a system that third parties can plug into without the host knowing they exist.

The idea: a package, in its packaging metadata (the `pyproject.toml` of Chapter 11), can *declare* that it provides something under a named group. A separately-installed package can declare that it, too, provides something under that *same* group. And the host application can, at runtime, *ask the installed environment* "what is registered under this group?" and get back everything any installed package has declared — discovering, dynamically, plugins it was never built with knowledge of.

Concretely: a tool defines a plugin group, say `mytool.plugins`. Anyone can write a plugin package that, in its own `pyproject.toml`, declares an entry point in the `mytool.plugins` group pointing at the plugin's implementation. When that plugin package is `pip install`ed alongside the tool, the tool — using `importlib.metadata.entry_points()` to query the group — finds it automatically. The user installs a plugin; the tool discovers and loads it; no code in the tool changed, and the tool's authors never heard of the plugin. This is precisely how `pytest` discovers its plugin ecosystem, how many extensible CLIs work, how a great deal of the Python tooling world is architected.

Why this belongs in a structure chapter: entry points are the structural pattern for *extensibility across package boundaries*. When you are designing a system that other teams — or the open-source world — should be able to extend without modifying your code, entry points plus the `importlib.metadata` query are the Pythonic mechanism, and it is built on exactly the import machinery this chapter has been describing.

> ### War Story — The Import That Depended on the Working Directory
>
> A team's service ran fine in production and fine on most engineers' machines. But one engineer, and the CI pipeline under one particular configuration, hit an `ImportError` on startup — a module that "obviously existed" could not be found. The error was intermittent across environments in a way that made it look like a flaky infrastructure problem, and it was treated as one for some time.
>
> The cause was `sys.path[0]`. The project used a flat layout — the package directory sat at the repository root alongside the scripts — and several imports were *implicitly relying on the current working directory being the repository root*, because that made the script's directory (which is `sys.path[0]`) the right place to find the package. Engineers who launched the service from the repository root never saw a problem. The one engineer who launched it from a subdirectory, and the CI job that did the same, had a different `sys.path[0]`, the package was no longer on the path, and the import failed. The "flaky" error was perfectly deterministic — it depended entirely on the working directory at launch, which simply varied between environments.
>
> The fix was structural: adopt the `src` layout (Chapter 11), where the package lives under a `src/` directory and the project is *installed* (even in development, via an editable install) rather than imported by accident from the working directory. With the package properly installed, it is on `sys.path` because it was *installed there*, not because of where someone happened to launch from — and the import works identically everywhere. The lesson: an import whose success depends on the working directory is a latent bug, and the defense is to make code importable because it was *installed*, never because the filesystem happened to line up. The import system is deterministic; the fragility was in relying on `sys.path[0]`, and the cure is to not rely on it.

> ### The Level Line — Modules and Imports
>
> **A senior engineer** organizes code into modules and packages sensibly, uses absolute imports, knows `if __name__ == "__main__"`, and can usually resolve a circular import by moving an import into a function.
>
> **A staff engineer** understands that a module is an object whose body runs once on first import and is cached in `sys.modules`; can explain *exactly* why a circular import fails (the half-built module) and chooses among the four fixes deliberately, recognizing the cycle as a possible design signal; knows the regular-versus-namespace package distinction; uses `__init__.py` to deliberately shape a package's public API; and knows that `import` is a pluggable three-phase protocol.
>
> **A principal engineer** designs the *module architecture* of a system — where boundaries fall, what each package exposes, how the dependency graph is kept acyclic — treats the structure as something maintained, not accreted; uses entry points to design genuine cross-boundary extensibility; and ensures the project's layout (Chapter 11) makes imports deterministic rather than working-directory-dependent.

## 9.10 At the Interview Table

The import system is a favored deep-dive topic because it cleanly separates the engineer who only *uses* `import` from the one who *understands* it.

**The question: "What happens, mechanically, when you write `import something`?"**

"Python first checks `sys.modules`, the cache of already-imported modules — if it's there, you get the cached module object and nothing else happens. If not, Python finds the module: it consults the finders on `sys.meta_path`, which locate the module — on the filesystem via `sys.path`, or elsewhere — and produce a spec. A loader then creates the module object and *executes the module's body top to bottom* into it — that's the part people forget: importing a module runs its code. Finally the module object is stored in `sys.modules` so the next import is just a cache hit. So `import` is really a three-phase protocol — find, load, cache — and it's extensible: you can plug in custom finders." Mentioning that the body *runs*, and that the protocol is pluggable, signals real understanding.

**The question: "Explain circular imports. Why do they fail, and how do you fix them?"**

This is the staff-level favorite. "A circular import is A imports B, B imports A. It fails because of how import works: importing a module runs its body, and the module is registered in `sys.modules` as soon as it *starts* — so a module can be in the cache while still half-executed. When A's body, partway down, imports B, and B's body imports A, B finds A already in `sys.modules` and gets it *as it currently is* — half-built, because A paused partway to import B. If B then uses a name A hasn't defined yet, you get an `ImportError` or `AttributeError`. The fixes, best first: restructure — a circular import usually means the module boundaries are wrong, and extracting the shared piece into a third module is the real fix. Or move the import inside the function that uses it, so it runs at call time when both modules are loaded. Or import the module rather than the name, deferring the name resolution. Or, if the cycle exists only for type hints, guard the import with `if TYPE_CHECKING:` so it doesn't happen at runtime at all. I treat a circular import as a design signal first and a mechanical problem second."

**The question: "Absolute or relative imports?"**

"Absolute, as the default — `from mypackage.utils import helper` is unambiguous regardless of where the file sits or how the program was launched, it survives a file being moved, and it shows a reader the full provenance. Relative imports have a place — keeping imports concise *within* a tightly-coupled package, and making a whole package easy to rename — but absolute is the sound default."

**The question: "How would you design a plugin system?"**

"Entry points. The host defines a named entry-point group; plugin packages declare, in their own `pyproject.toml`, an entry point in that group pointing at their implementation; the host queries `importlib.metadata.entry_points()` for that group at runtime and discovers every installed plugin — without the host having any compile-time knowledge of them. Install a plugin package, and it's found. That's how `pytest` does it. The alternative for a simpler case is a registration decorator plus dynamic import via `importlib`, but entry points are the mechanism when plugins live in *separate, separately-installed packages*."

**The red flags.** Not knowing that importing a module *runs its body*. Treating circular imports as purely mechanical with no sense that they signal a design problem. Believing `import` is an opaque built-in rather than a protocol. And not knowing why an import can succeed from one directory and fail from another — which signals an engineer who has never been bitten by the `sys.path[0]` fragility and does not know the `src`-layout cure.

## 9.11 The Forge

**Drill 9.1.** Create a module with a `print` statement and an expensive-looking computation at its top level. Import it twice from two different files in the same program. Confirm the top-level code runs exactly once, and explain why in terms of `sys.modules`.

**Drill 9.2.** For each, predict the result and explain via `sys.path`: a project containing a file named `random.py` that does `import random` elsewhere; a program run from its own directory versus from a parent directory, importing a sibling module; `import` of a package directory with no `__init__.py` versus one with.

**Build 9.1.** Construct a deliberate circular import — two modules that import each other — and reproduce the failure. Then fix it four ways, one at a time: by restructuring (extract the shared piece), by moving an import into a function, by importing the module instead of the name, and (for a typing-only version of the cycle) with `TYPE_CHECKING`. Write a short note on which fix you would actually choose for this case and why.

**Build 9.2.** Build a small plugin system using entry points. Create a host package that defines a plugin group and discovers plugins via `importlib.metadata.entry_points()`, and a separate plugin package that registers an entry point in that group. Install both (editable installs) and demonstrate the host discovering and running the plugin with no code in the host referring to the plugin.

**Investigate 9.1.** Use `python -X importtime yourprogram.py` to get a breakdown of how long each import takes. Identify the slowest imports in a real project of yours. For the slowest, investigate *why* — is the module doing expensive work at top level? pulling in heavy dependencies? — and write up what you found and whether lazy importing (PEP 562) would help.

**Design 9.1.** You are designing the package and module structure for a new, medium-sized service that three teams will work on simultaneously. Write a one-to-two-page document: the top-level package layout and the rationale for each boundary; how you keep the inter-module dependency graph acyclic; what each package's `__init__.py` exposes as public API and what stays internal; where, if anywhere, a namespace package is warranted; the import conventions the teams will follow; and how the project layout (foreshadowing Chapter 11) makes imports deterministic across every team member's environment. Justify the structure as something that makes the *next change cheap*.

---

[← Chapter 8: Errors and the Discipline of Failure](08-errors-and-failure.md) · [Home](README.md) · [Chapter 10: Types as a Design Tool →](10-types-as-design-tool.md)
