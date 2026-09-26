# Chapter 11 — Packaging, Tooling, and the Developer Loop

*Part III — Structure*

[← Chapter 10: Types as a Design Tool](10-types-as-design-tool.md) · [Home](README.md) · [Chapter 12: Architecture and Design Patterns →](12-architecture-and-patterns.md)

---

## 11.1 The Question Behind the Chapter

Code that runs on your machine is not finished. It is finished when it can be *reliably built, depended upon, reproduced, and shipped* — when a colleague can clone the repository and have it working in minutes, when a CI server produces the same result your laptop does, when a deployment six months from now installs exactly the dependency versions that were tested. The gap between "works on my machine" and "works, verifiably, everywhere" is the subject of this chapter, and closing it is not glamorous work but it is staff-level work, because a team's *velocity* is gated by it.

This is also the chapter where the book is most explicitly a *2026* book. Python packaging spent two decades as the language's most criticized area — a confusing accretion of `setup.py`, `easy_install`, `requirements.txt`, `virtualenv`, `pip`, competing tools, and genuine ambiguity about the right way to do anything. That era is ending. A modern, standards-based stack has emerged — `pyproject.toml` as the single configuration file, `uv` as a fast unified tool, `ruff` as a consolidating linter and formatter — and it is genuinely good. This chapter teaches that modern stack, names the legacy it replaces so you can recognize it in older projects, and frames the whole thing around the concept that ties it together: the **developer loop**, the cycle of clone-install-edit-test-commit that every engineer on a team runs dozens of times a day, and which is therefore worth optimizing with real care.

## 11.2 `pyproject.toml`: The Single Source of Truth

For most of Python's history, a project's configuration was scattered: `setup.py` (an executable Python file that *was* the build configuration), often `setup.cfg` alongside it, `requirements.txt` for dependencies, `MANIFEST.in` for packaging data files, and a separate config file for nearly every tool — `.flake8`, `.isort.cfg`, `pytest.ini`, and more. The configuration of a single project was spread across half a dozen files in three formats.

The modern answer, established by a series of PEPs (518, 621, and others), is **`pyproject.toml`** — a single file, in the TOML format, that holds *all* of it. It has a defined structure:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mypackage"
version = "0.1.0"
description = "A short description"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "mypy", "ruff"]

[tool.ruff]
line-length = 100

[tool.mypy]
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Read the structure. The `[build-system]` table declares *how the project is built* — which build backend turns the source into an installable artifact. The `[project]` table holds the *project metadata* — name, version, description, the Python versions it supports, and crucially its *dependencies*. The `[project.optional-dependencies]` table groups extra dependencies under named *extras* — here a `dev` extra bundling the development tools. And every `[tool.*]` table holds the configuration of one tool — `ruff`, `mypy`, `pytest`, and the rest each reading their own table.

The significance is consolidation. One file, one format, the whole project's configuration in front of you. A new engineer opens `pyproject.toml` and sees what the project is, what it depends on, and how every tool is set up. The decisive point of guidance for the chapter: **`setup.py`, `setup.cfg`, and standalone tool-config files are legacy.** You will encounter them in older projects and you must be able to read them — but new projects use `pyproject.toml` for everything, and migrating an old project to it is a real and worthwhile piece of cleanup.

## 11.3 The `src` Layout

A project's *directory layout* is a small decision with a real consequence, and the chapter recommends a specific answer.

The two layouts. The **flat layout** places the package directory at the repository root, beside the other top-level files:

```
myproject/
    mypackage/
        __init__.py
        core.py
    tests/
    pyproject.toml
```

The **`src` layout** places the package one level down, inside a `src/` directory:

```
myproject/
    src/
        mypackage/
            __init__.py
            core.py
    tests/
    pyproject.toml
```

The difference looks cosmetic. It is not, and the reason connects directly to Chapter 9. Recall that `sys.path` includes the current working directory, and that this lets code be imported "by accident" — found because of where you happened to launch from rather than because it was properly installed. With the *flat* layout, `mypackage` sits at the repository root, so when you run anything from the repository root, `mypackage` is importable simply because it is *right there in the current directory* — whether or not it has been correctly installed. With the **`src` layout**, `mypackage` is inside `src/`, which is *not* on `sys.path` by default — so `import mypackage` works *only if the package has actually been installed* (including via an editable install, `pip install -e .` or `uv`'s equivalent, which is what you do during development).

That difference is exactly the point. The `src` layout *forces* the package to be properly installed before it can be imported, which means your tests, and your CI, exercise the package *as it will actually be installed for a real user* — not as it happens to sit in the source tree. It catches a real class of bug: a packaging configuration that forgets to include a sub-module, a missing `__init__.py`, a data file that is not packaged — all of these pass under the flat layout (because the source tree has the files regardless of packaging) and *fail loudly* under the `src` layout (because the installed package is what is imported, and the installed package is missing the piece). The recurring War-Story bug of Chapter 9 — the import that depended on the working directory — is structurally prevented by the `src` layout, because nothing is importable by accident.

The recommendation: **use the `src` layout for any package you intend to ship.** The cost is one extra `src/` directory and remembering to do an editable install once when you set up the project. The benefit is that "works in development" and "works when installed" stop being two different things. For an application that is never packaged as a distributable library, the flat layout is acceptable — but for a library, `src` is the sound default.

## 11.4 Virtual Environments and the Isolation Principle

A *virtual environment* is an isolated Python environment — its own interpreter link and its own set of installed packages — separate from the system Python and from every other project's environment. The principle it enforces is **isolation**: each project gets exactly the dependency versions it declares, and nothing else, and one project's dependencies cannot collide with another's.

Why this is non-negotiable: without isolation, every project on a machine shares one global set of installed packages. Project A needs version 2 of a library; project B needs version 3; they cannot coexist. Install something for one project and you may break another. And installing into the *system* Python — the one the operating system itself depends on — risks breaking the operating system's own tools. Modern Python actively defends against the last danger: PEP 668 lets a system Python mark itself "externally managed," and `pip` will *refuse* to install into it, precisely to stop engineers from corrupting the system environment. The error message that produces is not an obstacle to route around; it is the system correctly telling you to use a virtual environment.

The rule is absolute: **every project gets its own virtual environment, always.** The stdlib provides `python -m venv` to create one; the modern tools (next section) create and manage them for you, often invisibly. The mechanism matters less than the principle — per-project isolation, no exceptions — and an engineer who installs project dependencies into a shared or system environment is creating a class of "works on my machine" problem that the whole rest of this chapter exists to eliminate.

## 11.5 Dependencies, Lock Files, and Reproducibility

Here is the distinction that makes builds reproducible, and it is one a staff engineer must hold precisely, because it is the heart of "works on my machine."

Your `pyproject.toml` declares dependencies as *abstract constraints* — `httpx>=0.27`, meaning "any version of httpx at or above 0.27." That is the right thing to *declare*, because it expresses genuine compatibility: your code works with 0.27 and with later versions, and you do not want to over-constrain. But abstract constraints are not *reproducible*. "Any version >= 0.27" resolves to httpx 0.27 today and httpx 0.31 in three months, and if 0.31 has a behavior change, your colleague — or your CI server, or your production deploy — gets different code than you tested, from the *same* `pyproject.toml`. The constraint is satisfied; the build is not reproducible.

The solution is the **lock file**. A lock file records the *exact, concrete* version of *every* package that gets installed — not just your direct dependencies but their dependencies and theirs, the entire resolved tree, each pinned to a precise version (and usually a cryptographic hash). The workflow is two-layered: `pyproject.toml` holds the abstract constraints (the human intent — "I am compatible with these ranges"), and the lock file holds the concrete resolution (the machine fact — "*these exact versions* were resolved and tested"). When a colleague or a CI server installs *from the lock file*, they get *byte-for-byte the dependency set you tested*. The build is reproducible because the lock file removed the ambiguity.

The guidance has two parts, and the distinction matters. For an **application** — something you deploy — you **commit the lock file** to version control. The lock file is how every developer, every CI run, and every production deployment installs the identical, tested dependency set. For a **library** — something other projects depend on — you generally **do not** ship a lock file as the thing consumers install: a library should declare its abstract constraints and let the *consuming application's* resolver and lock file pin the concrete versions, because the consumer needs to resolve *your* library's constraints together with all its other dependencies' constraints. The library's own lock file is still useful for its *own* CI and development; it is just not the consumer's source of truth. Application: lock and commit and install from it. Library: declare ranges, let the consumer lock.

## 11.6 `uv`: The Modern Unified Tool

For most of Python's history, the workflow above required a collection of separate tools — `venv` to create the environment, `pip` to install, `pip-tools` or another tool to produce a lock file, `pyenv` to manage Python versions themselves. Each did one part, and stitching them together was its own small expertise.

`uv` is the tool — written in Rust, and very fast — that unifies the whole workflow into one. It creates and manages the virtual environment, installs dependencies, resolves and writes the lock file, and even installs and manages Python interpreter versions themselves. A modern project's developer loop with `uv` is short: `uv sync` reads `pyproject.toml` and the lock file and brings the environment to exactly the locked state; `uv add somepackage` adds a dependency, resolves it, updates `pyproject.toml`, and updates the lock file, in one command; `uv run somecommand` runs a command inside the project's environment without any explicit "activate the venv" step. The whole "create venv, activate it, pip install, freeze requirements" sequence collapses into a couple of fast commands.

`uv` is the reason this chapter can be a *2026* chapter. It has, in a short time, become a sound default for new Python projects, because it makes the correct workflow — isolated environment, locked dependencies, reproducible installs — also the *easy* and *fast* workflow. The book's recommendation is direct: for new projects, use `uv`, and the modern stack it anchors. Two things qualify that recommendation honestly. First, the packaging ecosystem genuinely does still move — `uv` is dominant as this is written, and you should always check the current state of the tooling rather than assuming a book's snapshot is eternal; the *principles* of this chapter (one config file, isolation, abstract-versus-locked, a fast developer loop) are stable even as the specific tool that best embodies them evolves. Second, `pip` and `venv` are not *wrong* — they are the stable, universal baseline, they are what an enormous amount of existing tooling and documentation assumes, and an engineer must be fluent in them. `uv` is the recommended modern default; `pip`/`venv` are the baseline you must also know. (A nice modern touch worth knowing: PEP 723 lets a *single-file script* declare its own dependencies in a comment block at the top, and `uv run` will execute that script in an environment with exactly those dependencies — no project, no manual setup, a self-contained runnable script.)

## 11.7 Building and Publishing

When a package is ready to be shared — on the public Python Package Index (PyPI) or a private index — it must be *built* into a distributable artifact. There are two artifact formats, and the difference is worth knowing.

A **wheel** (`.whl`) is a *built* distribution — the package already processed into the form that gets installed, so installing a wheel is essentially just unpacking it into place. It is fast to install and it is the format you want users to get.

An **sdist** (source distribution) is the package's *source*, archived. Installing from an sdist requires a build step on the installing machine. It exists as a fallback and for cases — notably packages with compiled C extensions — where a pre-built wheel for the user's exact platform may not exist and the package must be built locally.

The **build backend** — declared in that `[build-system]` table of `pyproject.toml` — is the component that actually performs the build, turning your source into a wheel and an sdist. Several exist (`hatchling`, `setuptools`, `flit`, and others); `hatchling` is a common modern choice. You configure which backend in `pyproject.toml` and then a build command (the standards-based `python -m build`, or `uv build`) invokes it to produce the artifacts in a `dist/` directory.

**Publishing** uploads those artifacts to an index. The traditional tool is `twine`; `uv` can publish as well. The detail worth knowing — because it is the modern, more secure way and a staff engineer should prefer it — is **trusted publishing**. The old way to publish from an automated pipeline was to store a long-lived API token as a secret in the CI system; that token is a credential that can leak, and a leaked publishing token lets an attacker push a malicious version of your package. Trusted publishing replaces the long-lived token with a short-lived, automatically-issued credential, established by a direct trust relationship between PyPI and your CI provider — there is no long-lived secret to leak. When you set up automated publishing for a package, trusted publishing is the right choice, and it connects forward to the supply-chain security material of Chapter 22.

## 11.8 `ruff` and the Consolidation of Tooling

Code *quality tooling* — the linters and formatters that catch mistakes and enforce consistency — was, like packaging, historically a collection of separate tools. A typical project ran `flake8` to lint, `isort` to sort imports, `pyupgrade` to modernize syntax, `black` to format, and perhaps `pylint` and others — each a separate tool, separate configuration, separate run, and the set of them was slow enough that engineers ran them only occasionally.

`ruff` is the consolidation, and it is the same story as `uv`: a single tool, written in Rust, *dramatically* faster than what it replaces — fast enough to run on every save in an editor and on every commit without anyone noticing the cost. `ruff` does in one tool what that whole collection did: it lints (catching likely bugs, questionable patterns, style violations — re-implementing the checks of `flake8` and many of its plugins, and `pylint`, and more), it sorts imports (replacing `isort`), it can apply modernizing fixes (replacing `pyupgrade`), and it *formats* code (a formatter compatible with `black`'s style). One tool, configured in one `[tool.ruff]` table of `pyproject.toml`, doing the work of six.

The *speed* is not a minor convenience — it changes behavior. A linter that takes thirty seconds to run gets run rarely, so problems pile up between runs. A linter that runs in well under a second runs *constantly* — on every keystroke in the editor, on every commit via a pre-commit hook, in CI — so problems are caught the instant they are introduced, when they are cheapest to fix. `ruff` made comprehensive linting and formatting fast enough to be *continuous*, and continuous is qualitatively better than occasional. For new projects, `ruff` is the recommended default; it is the quality-tooling half of the same modern stack that `uv` anchors.

## 11.9 The Developer Loop

Now the concept that ties the whole chapter together. The **developer loop** is the cycle every engineer on a team runs constantly: clone the repository, install its dependencies, make a change, run the tests and the linter, commit. Every engineer runs it dozens of times a day. Multiply that by the size of the team and the length of the project, and the loop is one of the largest consumers of engineering time in the whole endeavor — which means *optimizing the loop* is one of the highest-leverage things a staff engineer can do, and it is invisible work that pays a continuous dividend.

What a *good* developer loop looks like, assembled from the pieces of this chapter: a new engineer clones the repository and runs *one command* — `uv sync` — and has a correct, isolated, fully-installed environment in seconds, with byte-identical dependencies to everyone else's because of the lock file. They make a change; `ruff` runs on save and flags problems instantly; the tests run fast. They commit, and a **pre-commit hook** — an automated check that runs at commit time — runs `ruff` and the type checker and the fast tests, catching problems *before* they ever leave the engineer's machine and reach a colleague or CI. CI then runs the same checks again as the enforcing backstop, plus the slower tests. A **task runner** — whether a `Makefile`, a `justfile`, or the script-running features of the modern tools — gives the common operations memorable one-word names (`just test`, `just lint`, `just build`), so the loop is the same for everyone and no one has to remember a long incantation.

The friction in this loop is *tax* — a tax paid by every engineer, on every change, for the life of the project. A loop where setup is one fast command is dramatically more productive than one where a new engineer spends a day fighting environment problems. A loop where the linter is instant is more productive than one where it is a slow chore. A loop where pre-commit catches problems locally is more productive than one where they are caught in CI twenty minutes later, or by a colleague the next day. None of this is glamorous and none of it ships a feature directly — but a staff engineer understands that the team's *sustained velocity* is gated by the quality of this loop, and that investing in it is investing in everything the team will build afterward. That reframing — tooling and packaging as *velocity infrastructure*, not as chores — is the genuinely staff-level idea of the chapter.

> ### War Story — The Onboarding That Took a Week
>
> A team had a service with a packaging story that had grown by accretion: a `setup.py` partially superseded by a `setup.cfg`, a `requirements.txt` that was hand-maintained and *not* a true lock file (it pinned the direct dependencies but not their dependencies), a `README` with a dozen manual setup steps, and a set of environment assumptions that lived only in the heads of the engineers who had been there longest.
>
> A new engineer joined. Getting the project running on their machine took the better part of a week. The dependency set was not reproducible — the un-pinned transitive dependencies resolved to newer versions than the team had, and some of those newer versions had behavior changes, so the new engineer hit failures no one else was seeing. The manual setup steps were subtly out of date. Each problem was individually small and individually solvable, and solving all of them, with intermittent help from busy teammates, consumed days. And this was not a one-time cost: *every* new engineer paid a version of it, and even existing engineers periodically lost hours when their environment drifted from everyone else's.
>
> The staff engineer who eventually fixed it did not ship a feature that sprint. They migrated the project to a single `pyproject.toml`, adopted `uv` with a real, committed lock file so the dependency set was exact and reproducible, moved to the `src` layout so "installed" and "works" became the same thing, added a `justfile` so the common operations had one-word names, and added pre-commit hooks. Afterward, onboarding was one command and a few minutes, and environment drift simply stopped happening.
>
> The lesson is the chapter's thesis. The week of onboarding pain, multiplied across every past and future hire and every drift incident, was an enormous and entirely invisible cost — invisible because it never appeared as a line item, it was just diffuse friction that everyone had quietly accepted as normal. The developer loop *is* infrastructure, the friction in it *is* a tax, and a staff engineer is the person who notices the tax, calculates what it really costs, and does the unglamorous work of eliminating it — because every feature the team ships afterward is cheaper for it.

> ### The Level Line — Packaging and Tooling
>
> **A senior engineer** can set up a project with `pyproject.toml`, uses a virtual environment, installs dependencies, runs the linter and tests, and can build and perhaps publish a package.
>
> **A staff engineer** understands the modern stack and *why* it is shaped as it is — the single config file, the abstract-constraints-versus-locked-versions distinction and what it means for reproducibility, the `src` layout and the class of bug it prevents, trusted publishing; is fluent in both the modern tools and the `pip`/`venv` baseline; and treats the developer loop as a system to be deliberately optimized.
>
> **A principal engineer** owns the team's *velocity infrastructure* — establishes the project standards, drives the migration of legacy projects onto the modern stack, sets up the pre-commit and CI and task-runner conventions so the loop is fast and uniform for everyone — recognizes diffuse tooling friction as a real and quantifiable cost, and treats investment in the developer loop as investment in everything the team will build, because it is.

## 11.10 At the Interview Table

Packaging is less a source of trivia questions than a *competence signal* — the way a candidate talks about it reveals whether they have operated real projects or only written code in them.

**The question: "How do you ensure reproducible builds?"**

"The key distinction is abstract constraints versus locked versions. `pyproject.toml` declares dependencies as *ranges* — `httpx>=0.27` — which is the right thing to *declare*, because it expresses real compatibility. But ranges aren't reproducible: the same `pyproject.toml` resolves to different concrete versions over time. So you use a *lock file*, which records the exact, concrete version of every package in the entire resolved tree, usually with hashes. For an application, you commit that lock file and every developer, CI run, and deployment installs *from it* — so everyone gets byte-identical, tested dependencies. For a library you declare ranges and let the consuming application lock. The modern tooling — `uv` — manages the lock file as part of the normal workflow. That two-layer model, abstract intent plus concrete locked resolution, is what makes a build reproducible."

**The question: "Walk me through how you'd set up a new Python project."**

A confident answer threads the chapter together: "A single `pyproject.toml` for all configuration — project metadata, dependencies, and every tool's settings in their `[tool.*]` tables. The `src` layout, so the package has to be properly installed to be imported and tests exercise it the way a real user will, which catches packaging mistakes. A virtual environment, always — per-project isolation, no exceptions. `uv` to manage the environment, dependencies, and lock file, with the lock file committed. `ruff` for linting and formatting, configured in `pyproject.toml`. Pre-commit hooks running `ruff`, the type checker, and the fast tests, so problems are caught locally before they reach CI. A task runner — a `justfile` or similar — so the common operations have one-word names. The goal is a developer loop where a new person clones, runs one command, and is productive in minutes."

**The question: "What's the difference between a wheel and an sdist?"**

"A wheel is a *built* distribution — the package already in installable form, so installing it is essentially unpacking; it's fast and it's what you want users to get. An sdist is the *source*, archived, and installing it requires a build step on the installing machine. The sdist matters as a fallback and especially for packages with compiled extensions, where a pre-built wheel for the user's exact platform might not exist and it has to be built locally."

**The question: "Why use the `src` layout?"**

"Because of how `sys.path` works. With a flat layout, the package sits at the repo root, so it's importable just because it's *there* in the working directory — whether or not it's been correctly installed. With the `src` layout, the package is inside `src/`, which isn't on the path by default, so it's only importable if it's actually been *installed* — including via an editable install during development. That forces your tests and CI to exercise the package as it'll really be installed, which catches packaging bugs — a forgotten submodule, a missing `__init__.py`, an unpackaged data file — that the flat layout silently hides. It also eliminates the class of bug where an import works from one directory and fails from another."

**The red flags.** No grasp of the abstract-versus-locked distinction — "I just `pip install` the requirements" with no notion of reproducibility. Installing into the system Python, or not using virtual environments. Treating tooling as unimportant chore-work rather than as velocity infrastructure. And being unaware of the modern stack entirely — which is not disqualifying, since `pip`/`venv` work, but a staff candidate in 2026 should at least *know* the modern tools exist and what problem they solve.

## 11.11 The Forge

**Drill 11.1.** Take an existing project of yours that uses `setup.py` or a bare `requirements.txt` (or create a small one in that legacy style). Migrate it fully to a single `pyproject.toml` — metadata, dependencies, and tool configuration all consolidated. Confirm it still builds and installs.

**Drill 11.2.** Set up a brand-new project from scratch with the modern stack: `pyproject.toml`, the `src` layout, a virtual environment, `uv` managing dependencies with a committed lock file, and `ruff` configured. Add one dependency with `uv add` and observe what changes in `pyproject.toml` and the lock file.

**Build 11.1.** Write a complete, installable, publishable package — small but real (a useful utility library). Give it a proper `pyproject.toml` with a build backend, the `src` layout, dependencies and a `dev` extra, type annotations that pass a checker, and a test suite. Build it into a wheel and an sdist with `python -m build` or `uv build`, and install the built wheel into a fresh environment to confirm it works as a real user would receive it.

**Build 11.2.** Set up the full developer loop for a project: pre-commit hooks running `ruff` and a type checker, a task runner (`justfile` or equivalent) with one-word commands for the common operations (`install`, `test`, `lint`, `build`), and a one-command setup path for a new contributor. Write the project's `CONTRIBUTING` section describing the loop, and time how long a from-scratch setup actually takes.

**Investigate 11.1.** Take a real project's dependency tree and inspect it fully — the direct dependencies, and the transitive ones they pull in. Use the tooling (`uv tree`, `pip list`, or equivalent) to see the whole graph. Identify the depth of the tree, any dependency pulled in by multiple paths, and anything surprising. Write up what the exercise revealed about how much code the project actually depends on.

**Design 11.1.** You are the staff engineer joining a team whose project has accreted a legacy packaging setup — `setup.py`, a hand-maintained `requirements.txt` that is not a true lock file, manual setup steps in a `README`, a flat layout, no pre-commit hooks, slow and rarely-run linting. New-engineer onboarding takes days and environment drift causes recurring lost hours. Write a one-to-two-page migration plan: what you change and in what order; how you minimize disruption to ongoing work while you do it; how you bring the team along (this is a change to everyone's daily workflow); and how you would *quantify* the before-and-after — what the friction was costing and what the migration saved. This is the staff-level exercise: treating the developer loop as infrastructure worth measuring and investing in.

---

[← Chapter 10: Types as a Design Tool](10-types-as-design-tool.md) · [Home](README.md) · [Chapter 12: Architecture and Design Patterns →](12-architecture-and-patterns.md)
