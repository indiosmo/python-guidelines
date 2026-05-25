# Python Guidelines Porting Plan

This is the active plan for turning the completed research pass into the first
durable Python guideline set. `GOAL.md` is the stable objective; this file is
the writing map.

The guide set ports the transferable C++ engineering guidance into idiomatic
Python. Preserve the intent: domain ownership, forward-only dependency graphs,
declarative dataflow, invariant ownership, idempotency, deterministic testing,
root-cause debugging, and tool-backed feedback. Replace C++ mechanisms with
Python mechanisms: type annotations, runtime validation, exceptions, context
managers, pytest, Ruff, uv, async task groups, structured logging, warnings,
profilers, and runtime diagnostics.

## Current State

Phase 4 is complete. The duplicate plan drafts and pre-consolidation
exploration files are superseded by this plan and the research inventory below.

Next action: start the durable writing pass at Batch 1.

## Research Inventory

Use these files as the research source map while writing guides:

- `GOAL.md`: governing objective and success criteria.
- `MONOREPO.md`: local Python monorepo shape and boundary philosophy.
- `/home/msi/repos/cpp-guidelines/`: upstream C++ guideline source.
- `/home/msi/python_workspace/factors2/`: concrete Python reference codebase.
- `work-in-progress/exploration/cpp-source-python-port.md`: C++ source
  disposition map and Python replacement mechanisms.
- `work-in-progress/exploration/exploration.md`: consolidated Python tooling,
  typing, architecture, testing, debugging, and operations research.
- `work-in-progress/exploration/antipatterns.md`: consolidated factors2
  anti-pattern briefing and durable guide placement.
- `work-in-progress/exploration/codex-research/architecture-and-patterns.md`:
  Python architecture, data boundaries, and factors2-oriented design research.
- `work-in-progress/exploration/codex-research/generalizable-cpp-design.md`:
  transferable design guidance and Python replacements.
- `work-in-progress/exploration/codex-research/generalizable-cpp-testing-debugging.md`:
  transferable testing and debugging guidance.
- `work-in-progress/exploration/codex-research/modern-python-tooling-and-typing.md`:
  modern Python tooling, typing, and checker research.
- `work-in-progress/exploration/codex-research/tooling-tests-docs.md`:
  project tooling, testing, documentation, and CI research.
- `work-in-progress/exploration/cpp-claude/research/exhaustive-matching.md`:
  `match`, discriminated unions, and `assert_never`.
- `work-in-progress/exploration/cpp-claude/research/observability-and-debugging.md`:
  Python diagnostics, warning policy, profilers, and runtime checks.
- `work-in-progress/exploration/cpp-claude/research/pep695-and-generics.md`:
  PEP 695 syntax and generic API policy.
- `work-in-progress/exploration/cpp-claude/research/ruff-rule-sets.md`:
  Ruff baseline and strict rule selection.
- `work-in-progress/exploration/cpp-claude/research/strong-types-and-data-classes.md`:
  `NewType`, dataclasses, attrs, Pydantic, and domain values.
- `work-in-progress/exploration/cpp-claude/research/structured-concurrency.md`:
  `asyncio.TaskGroup`, anyio, cancellation, and grouped failures.
- `work-in-progress/exploration/cpp-claude/research/typecheckers-2026.md`:
  Pyright, pyrefly, ty, and mypy comparison.
- `work-in-progress/exploration/factors-claude/tooling-baseline.md`: concrete
  factors2 uv, Ruff, pytest, pre-commit, and CI evidence.
- `work-in-progress/exploration/factors-claude/patterns/*.md`: concrete
  factors2 patterns for module layout, typing, Pydantic, Pandera, data
  pipelines, error handling, invariants, testing, logging, performance, and
  concurrency.
- `work-in-progress/exploration/factors-claude/research/pep-695-vs-legacy-aliases.md`:
  local PEP 695 alias research.

Durable guides must not link to `work-in-progress/`. Pull the stable reasoning
into the guide prose, or cite durable project files and external sources.

## Structure

Use the narrower C++-parallel structure. The structure mirrors the upstream
repository for navigation and citation, but Python idioms control the content.

Durable output folders:

- `python-projects-and-tooling/`
- `python-design-principles/`
- `python-testing-principles/`
- `python-debugging-principles/`
- `agent/`

Conditional outputs:

- `python-design-principles/data-pipeline-and-dataframes.md` is included in
  the first pass because the reference codebase is dataframe-heavy.
- `python-testing-principles/web-ui.md` is deferred until a target codebase has
  web UI tests.
- Qt guidance stays dropped unless a Python Qt surface appears.
- `agent/skills/repo-specific-python-guidelines/` is deferred until the first
  guide set exists.

## Policy Defaults

| Topic | Default |
|---|---|
| Python version | Python 3.13 baseline, Python 3.14 forward target. |
| Type checker | Pyright by default; pyrefly as the fast alternative; mypy as an existing-project fallback. |
| Strict typing | Strict checking is the target state; adoption may ratchet by package or directory. |
| Package workflow | uv workspaces, locked installs, and `uv run` command surfaces. |
| Lint and format | Ruff for linting and formatting. |
| Data carriers | Frozen slotted dataclasses for internal value objects; mutable aggregates when mutation is the domain operation; Pydantic v2 at trust boundaries; Pandera at dataframe boundaries. |
| Error handling | Idiomatic Python exceptions inside a domain; built-in exceptions when they communicate the failure clearly; custom exceptions only for meaningful domain failures or boundary translation. |
| Boundary failures | Translate failures explicitly at HTTP, CLI, job, queue, worker, and UI boundaries. |
| Result style | Mention only where a Python boundary benefits from type-checker-visible failure branches. Do not port C++ Result or LEAF patterns as the default. |
| Concurrency | `asyncio.TaskGroup` baseline; anyio when cancel scopes, timeout composition, or trio compatibility earn the dependency. |
| Cross-cutting services | Context-local facades for logger, clock, timer, settings, and tracing; explicit parameters for domain-specific dependencies. |
| Logging | One logging stack per project; stdlib logging for libraries, loguru for applications, structlog when strict key/value logs are required. |
| Approval tests | approvaltests plus pytest-approvaltests unless a project has a stronger existing convention. |
| Import boundaries | import-linter for CI contracts, pydeps for visualization, Ruff import rules for cheap checks. |
| Comments | Present-tense, positive comments and docstrings; no past-tense refactor notes or non-responsibility narration. |

## Focused Error-Handling Pass

The Python error-handling guide should be exception-first. It should describe
what current Python code should do, not map LEAF or `std::expected` into Python
syntax.

Use built-in exceptions when the built-in name communicates the failure:
`ValueError` for invalid values, `KeyError` for missing mapping keys,
`FileNotFoundError` for missing files, `TimeoutError` for elapsed timeouts, and
library exceptions when the dependency already exposes a precise contract.

Introduce a custom exception type when it earns its keep:

- The failure is a meaningful domain class, such as missing market data or a
  rejected order.
- Callers need to distinguish the failure by class.
- The exception carries structured attributes that support logging,
  translation, retries, or tests.
- A boundary needs a stable translation target for HTTP status, CLI exit code,
  queue behavior, job state, or UI response.

Keep custom hierarchies small. A domain base exception plus a few concrete
classes is useful; a mirror of every low-level failure is noise.

Use exception chaining at translation points:

- `raise DomainError(...) from exc` when adding domain context.
- `raise BoundaryError(...) from exc` when converting a dependency failure to
  a boundary failure.
- Suppress a cause only when the public contract intentionally hides the
  lower-level failure.

Use `ExceptionGroup` and `except*` when grouped concurrent work can fail in
several places. `asyncio.TaskGroup` makes this part of normal Python failure
handling. For expected partial failure, prefer an explicit result object or
per-item status so the caller cannot forget to inspect failures.

Use `contextlib` for cleanup and rollback:

- context managers for resources;
- `ExitStack` when cleanup callbacks are registered dynamically;
- `try` / `finally` when the cleanup is local and obvious;
- transactions and atomic renames for effects that must commit together.

Use warnings for recoverable conditions that callers may choose to escalate:
deprecations, questionable but accepted inputs, and runtime conditions where
continuing is safe. Tests and CI can turn project warnings into errors.

Use sentinel values when absence is the normal API contract. `None` is fine for
lookups, optional configuration, caches, and "not found" queries when the type
signature makes absence explicit. Use a private sentinel object when `None` is
a valid domain value.

Actively reject C++ error machinery as a default Python pattern. LEAF,
`BOOST_LEAF_CHECK`, `BOOST_LEAF_ASSIGN`, and `std::expected` are source
mechanisms, not destination mechanisms. Result-shaped values are acceptable
only for a Python boundary where typed, explicit failure branches make the
consumer clearer than exceptions.

## Durable Guide Set

Create these guides in the first writing pass:

- `python-projects-and-tooling/README.md`
- `python-projects-and-tooling/project-layout.md`
- `python-projects-and-tooling/tooling-baseline.md`
- `python-projects-and-tooling/python-version-and-typing-stack.md`
- `python-design-principles/README.md`
- `python-design-principles/architecture.md`
- `python-design-principles/types-and-correctness.md`
- `python-design-principles/comments-and-docstrings.md`
- `python-design-principles/declarative-style.md`
- `python-design-principles/functional-programming.md`
- `python-design-principles/generics-and-protocols.md`
- `python-design-principles/decorators-and-metaprogramming.md`
- `python-design-principles/error-handling.md`
- `python-design-principles/invariants-and-rollback.md`
- `python-design-principles/state-machines.md`
- `python-design-principles/pipelines.md`
- `python-design-principles/runtime-and-concurrency.md`
- `python-design-principles/cross-cutting-services.md`
- `python-design-principles/performance.md`
- `python-design-principles/data-pipeline-and-dataframes.md`
- `python-testing-principles/README.md`
- `python-testing-principles/philosophy.md`
- `python-testing-principles/pytest-conventions.md`
- `python-testing-principles/test-patterns.md`
- `python-testing-principles/test-helpers.md`
- `python-testing-principles/error-path-testing.md`
- `python-testing-principles/condition-based-waiting.md`
- `python-testing-principles/approval-tests.md`
- `python-debugging-principles/README.md`
- `python-debugging-principles/root-cause-tracing.md`
- `python-debugging-principles/defense-in-depth.md`
- `python-debugging-principles/logging.md`
- `python-debugging-principles/static-analysis-and-runtime-checks.md`
- `agent/python-agent-context.md`
- `agent/python-agent-examples.md`
- Root `README.md`

## Writing Batches

Each batch can be written in parallel. Later batches may cite vocabulary
created by earlier batches.

### Batch 1: Project Vocabulary

1. `python-projects-and-tooling/project-layout.md`
2. `python-projects-and-tooling/tooling-baseline.md`
3. `python-projects-and-tooling/python-version-and-typing-stack.md`

Primary sources: `MONOREPO.md`, `exploration.md`, `antipatterns.md`,
`codex-research/tooling-tests-docs.md`,
`codex-research/modern-python-tooling-and-typing.md`,
`factors-claude/tooling-baseline.md`,
`factors-claude/patterns/module-layout.md`,
`cpp-claude/research/ruff-rule-sets.md`, and
`cpp-claude/research/typecheckers-2026.md`.

### Batch 2: Design Foundations

4. `python-design-principles/architecture.md`
5. `python-design-principles/types-and-correctness.md`
6. `python-design-principles/comments-and-docstrings.md`
7. `python-design-principles/declarative-style.md`
8. `python-design-principles/error-handling.md`

Primary sources: `cpp-source-python-port.md`, `exploration.md`,
`antipatterns.md`, upstream C++ design guides, `MONOREPO.md`,
`codex-research/generalizable-cpp-design.md`,
`codex-research/architecture-and-patterns.md`,
`cpp-claude/research/strong-types-and-data-classes.md`,
`cpp-claude/research/exhaustive-matching.md`,
`cpp-claude/research/pep695-and-generics.md`,
`factors-claude/patterns/module-layout.md`,
`factors-claude/patterns/typing-discipline.md`,
`factors-claude/patterns/pydantic-pandera-dataclasses.md`,
`factors-claude/patterns/data-pipeline.md`,
`factors-claude/patterns/error-handling.md`, and
`factors-claude/patterns/documentation.md`.

### Batch 3: Design Specifics

9. `python-design-principles/functional-programming.md`
10. `python-design-principles/generics-and-protocols.md`
11. `python-design-principles/decorators-and-metaprogramming.md`
12. `python-design-principles/invariants-and-rollback.md`
13. `python-design-principles/state-machines.md`
14. `python-design-principles/pipelines.md`
15. `python-design-principles/cross-cutting-services.md`

Primary sources: `cpp-source-python-port.md`, `exploration.md`,
`antipatterns.md`, upstream C++ design guides,
`codex-research/generalizable-cpp-design.md`,
`codex-research/architecture-and-patterns.md`,
`cpp-claude/research/exhaustive-matching.md`,
`cpp-claude/research/pep695-and-generics.md`,
`cpp-claude/research/structured-concurrency.md`,
`factors-claude/patterns/data-pipeline.md`,
`factors-claude/patterns/pydantic-pandera-dataclasses.md`, and
`factors-claude/patterns/state-invariants.md`.

### Batch 4: Runtime, Performance, And Data

16. `python-design-principles/runtime-and-concurrency.md`
17. `python-design-principles/performance.md`
18. `python-design-principles/data-pipeline-and-dataframes.md`

Primary sources: `cpp-source-python-port.md`, `exploration.md`,
`antipatterns.md`, upstream C++ runtime, pipeline, and performance guides,
`codex-research/generalizable-cpp-design.md`,
`codex-research/architecture-and-patterns.md`,
`cpp-claude/research/structured-concurrency.md`,
`cpp-claude/research/observability-and-debugging.md`,
`factors-claude/patterns/async-concurrency.md`,
`factors-claude/patterns/performance.md`,
`factors-claude/patterns/data-pipeline.md`, and
`factors-claude/patterns/pydantic-pandera-dataclasses.md`.

### Batch 5: Testing

19. `python-testing-principles/philosophy.md`
20. `python-testing-principles/pytest-conventions.md`
21. `python-testing-principles/test-patterns.md`
22. `python-testing-principles/test-helpers.md`
23. `python-testing-principles/error-path-testing.md`
24. `python-testing-principles/condition-based-waiting.md`
25. `python-testing-principles/approval-tests.md`

Primary sources: `cpp-source-python-port.md`, `exploration.md`,
`antipatterns.md`, upstream C++ testing guides,
`codex-research/generalizable-cpp-testing-debugging.md`,
`codex-research/tooling-tests-docs.md`,
`factors-claude/patterns/testing.md`,
`factors-claude/tooling-baseline.md`,
`factors-claude/patterns/error-handling.md`, and
`factors-claude/patterns/data-pipeline.md`.

### Batch 6: Debugging

26. `python-debugging-principles/root-cause-tracing.md`
27. `python-debugging-principles/defense-in-depth.md`
28. `python-debugging-principles/logging.md`
29. `python-debugging-principles/static-analysis-and-runtime-checks.md`

Primary sources: `cpp-source-python-port.md`, `exploration.md`,
`antipatterns.md`, upstream C++ debugging guides,
`codex-research/generalizable-cpp-testing-debugging.md`,
`cpp-claude/research/observability-and-debugging.md`,
`factors-claude/patterns/logging-observability.md`,
`factors-claude/patterns/error-handling.md`, and
`factors-claude/tooling-baseline.md`.

### Batch 7: Agent Context And READMEs

30. `agent/python-agent-context.md`
31. `agent/python-agent-examples.md`
32. Top-level `README.md` and folder `README.md` files.

Primary sources: `GOAL.md`, this plan, all produced durable guides,
`cpp-source-python-port.md`, `exploration.md`, `antipatterns.md`, upstream
agent context files, and the factors2 pattern and anti-pattern evidence that
best illustrates good and bad Python behavior.

## Per-Guide Notes

Use these notes to keep each guide scoped:

| Guide | Scope |
|---|---|
| `project-layout.md` | `src/` layout, uv workspaces, child `pyproject.toml` files, monorepo boundaries, package layering, and generated artifact placement. |
| `tooling-baseline.md` | uv, Ruff, Pyright, pre-commit, CI command surface, security scanners, dependency groups, CODEOWNERS, PR templates, and development dependency policy. |
| `python-version-and-typing-stack.md` | Python 3.13 baseline, Python 3.14 target, PEP 695, PEP 649 implications, Pyright vs pyrefly, and mypy fallback. |
| `architecture.md` | Domain ownership, adapters, import DAGs, functional core, imperative shell, and testability without framework mocks. |
| `types-and-correctness.md` | Static typing, `NewType`, dataclasses, Pydantic, Pandera, discriminated unions, `assert_never`, `Any`, `cast`, and type-ignore policy. |
| `comments-and-docstrings.md` | Present-tense comments, positive phrasing, precondition comments, Google-style docstrings, and the negative-documentation anti-pattern. |
| `declarative-style.md` | Named predicates, staged values, built-ins, comprehensions, generators, itertools, `match`, pandas `.pipe`, and vectorized dataframe style. |
| `functional-programming.md` | Pure functions in the core, sum types, `match`, `assert_never`, closures, late binding, and higher-order functions. |
| `generics-and-protocols.md` | PEP 695 syntax, `Protocol`, structural typing, `TypeIs`, `TypeGuard`, `ParamSpec`, and decorator-preserving signatures. |
| `decorators-and-metaprogramming.md` | Plain functions first, decorators, `functools.wraps`, `__init_subclass__`, class decorators, descriptors, code generation, and metaclasses last. |
| `error-handling.md` | Exception-first domain failures, structured exceptions, boundary translation, exception chaining, `ExceptionGroup`, warnings, sentinels, logging boundaries, and narrow Result use. |
| `invariants-and-rollback.md` | Commit-at-end mutation, transactions, `ExitStack`, context managers, atomic file writes, idempotency, rollback tests, and free-threaded caveats. |
| `state-machines.md` | When a state machine earns its place, `match` over states, state classes, transition tables, `transitions` for large graphs, and edge tests. |
| `pipelines.md` | Explicit stages, callbacks, queues, async boundaries, runtime wiring, Dagster composition, and graph tools when the graph has operational meaning. |
| `runtime-and-concurrency.md` | Event loops, `TaskGroup`, anyio triggers, thread-to-loop marshalling, queues, multiprocessing, cancellation, rate limiting, and shared-state ownership. |
| `cross-cutting-services.md` | Logger, clock, timer, settings, and tracing facades; `ContextVar` overrides; explicit domain dependencies; test substitution. |
| `performance.md` | Measure first, vectorization, database-side work, batching, caching, slots, profilers, benchmark placement, and when to leave Python. |
| `data-pipeline-and-dataframes.md` | Pandera schemas, dataframe mutation ownership, pandas vs Polars vs ibis vs SQL decisions, generated SQL safety, Dagster and dbt boundaries, and dataframe tests. |
| `philosophy.md` | Behavior-first testing, independent oracles, component versus integration scope, deterministic data, and public contracts. |
| `pytest-conventions.md` | Test layout, collection, fixtures, marks, parametrization, assertion rewriting, plugin policy, xdist, and focused command loops. |
| `test-patterns.md` | Table-driven tests, `ids=`, fixtures, subtests, type-checker tests, dataframe assertions, and one behavior per test. |
| `test-helpers.md` | Domain factories, fixture factories, keyword-only dataclass defaults, context-local providers, fake protocols, and integration probes. |
| `error-path-testing.md` | `pytest.raises`, stable message or attribute assertions, rollback assertions, predicate tests, cleanup tests, and optional Result-style helpers. |
| `condition-based-waiting.md` | Predicate waits, monotonic clocks, events, queues, async timeouts, subprocess readiness, and fixed sleeps only for timing behavior. |
| `approval-tests.md` | Approval fit, deterministic serialization, scrubbers, review discipline, received file diffs, and direct assertions for narrow facts. |
| `root-cause-tracing.md` | Trace bad values backward, read chained and grouped tracebacks, use focused probes, isolate flaky tests, and fix the origin. |
| `defense-in-depth.md` | Boundary parsing, domain validation, environment guards, assertions for internal impossibility, runtime checks, and diagnostic layering. |
| `logging.md` | One logging stack, stable messages, structured fields, disabled-level cost, `logger.exception` at stopping boundaries, and no log-and-reraise. |
| `static-analysis-and-runtime-checks.md` | Pyright, pyrefly, Ruff, pytest warnings, `python -X dev`, asyncio debug mode, faulthandler, tracemalloc, memray, py-spy, and suppression policy. |
| `python-agent-context.md` | Always-loaded rule sheet: package layout, domain ownership, import DAGs, typing, exceptions, pytest, debugging, tooling, and comments. |
| `python-agent-examples.md` | Good and bad Python pairs keyed to the agent-context sections, grounded in factors2 evidence where useful. |

## Material Not To Port

Do not port these C++ mechanisms directly:

- Compile-time correctness as a literal Python guarantee.
- `std::expected`, LEAF, templates, concepts, `constexpr`, and type-erasure
  terminology.
- Custom exception hierarchies created merely to imitate C++ typed error
  channels.
- `std::variant` visitor mechanics.
- RAII and destructor cleanup as the primary model.
- Friend-based test probes, Catch2, CTest, Qt mechanics, or C++ sanitizer
  workflows.
- Container-choice, allocation, cache-locality, inlining, move-semantics, and
  undefined-behavior framing.
- Header, translation-unit, include, macro-expansion, and preprocessor framing.

Use Python replacements: type checkers, protocols, tagged unions, context
managers, fixtures, transactions, pytest, runtime warnings, profilers, and
diagnostic tooling.

## Verification

Before calling the first writing pass done:

- Check every durable guide against `GOAL.md`.
- Check that the root README points to durable guides, not temporary research.
- Check that each guide uses Python vocabulary and Python mechanisms.
- Check that code examples are plausible under the selected Python baseline.
- Check that C++-specific terms survive only where they are explicitly rejected
  or translated.
- Check that comments and documentation follow the positive, present-tense
  rule.
- Check that no durable guide links to `work-in-progress/`.
- Run `git diff --check`.
- Run an ASCII-only scan over edited markdown files.

## Deferred Work

- Re-survey Pyright, pyrefly, ty, and mypy before freezing long-lived tool
  comparisons in durable documentation.
- Validate the Ruff baseline against a representative project.
- Add `web-ui.md` only after there is enough web UI testing material to justify
  it.
- Package the agent skill after the guide set stabilizes.
- Write a PR review checklist after the guide sections exist.
