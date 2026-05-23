# Executive summary: porting the C++ guidelines to Python

## Coverage

Analysed every guide file in `/home/msi/repos/cpp-guidelines/`:

- 14 design files (`cpp-design-principles/`)
- 8 testing files (`cpp-testing-principles/`)
- 4 debugging files (`cpp-debugging-principles/`)
- 2 agent-context files (`agent/cpp-agent-{context,examples}.md`)

Total: **28 source guide files**, classified section-by-section.

## Disposition counts (by file)

| Disposition                | Count | Files                                                                                                                                                              |
|----------------------------|-------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| PORTS_AS_IS                | 9     | architecture, declarative-style, comments, philosophy, approval-tests, condition-based-waiting, root-cause-tracing, defense-in-depth, pipelines                     |
| PORTS_WITH_ADAPTATION      | 11    | invariants, functional-programming, state-machines, runtime, performance, error-path-testing, test-patterns, test-helpers, logging, agent-context (both), preprocessor-macros (principle) |
| PYTHON_SPECIFIC_VARIANT    | 7     | error-handling, compile-time-correctness, templates, cross-cutting, preprocessor-macros (mechanism), catch2-conventions, sanitizers                                 |
| DROP                       | 1     | qt-gui (drop unless the codebase actually has a Qt surface)                                                                                                         |

Note: many files are mixed-disposition at the section level; the table
above shows the dominant disposition. See the per-file analyses for
details.

## Recommended Python guide structure

Three sister repos, mirroring the C++ shape:

### `python-design-principles/`

| File                                     | Replaces (cpp)                       | Disposition summary                                            |
|------------------------------------------|--------------------------------------|----------------------------------------------------------------|
| `architecture.md`                        | `architecture.md`                    | Ports as-is; import-linter / pydeps for arrow enforcement.     |
| `types-and-correctness.md`               | `compile-time-correctness.md`        | Rewritten; NewType, frozen dataclass, pydantic, pyright strict.|
| `declarative-style.md`                   | `declarative-style.md`               | Ports as-is; generators / itertools / comprehensions / pipe.    |
| `functional-programming.md`              | `functional-programming.md`          | Ports as-is; drop type-erasure costs; add late-binding traps.   |
| `generics-and-protocols.md`              | `templates.md`                       | Rewritten; PEP 695, Protocol, TypeIs, assert_never.             |
| `decorators-and-metaprogramming.md`      | `preprocessor-macros.md`             | Rewritten; decorators-as-default, __init_subclass__, metaclasses-last. |
| `error-handling.md`                      | `error-handling.md`                  | Rewritten; exceptions, top-level catch-all, optional Result lib.|
| `invariants.md`                          | `invariants.md`                      | Adapted; ExitStack.callback, dataclasses.replace.               |
| `state-machines.md`                      | `state-machines.md`                  | Adapted; match over discriminated unions; transitions for large. |
| `pipelines.md`                           | `pipelines.md`                       | Ports as-is; Callable fields, asyncio bridging.                 |
| `runtime.md`                             | `runtime.md`                         | Adapted; asyncio/anyio primitives; free-threaded sidebar.       |
| `cross-cutting.md`                       | `cross-cutting.md`                   | Rewritten; contextvars singletons + pytest fixtures.            |
| `performance.md`                         | `performance.md`                     | Rewritten; vectorisation, __slots__, profiler choice.           |
| `comments.md`                            | `comments.md`                        | Ports as-is; `#` and docstring conventions.                     |

### `python-testing-principles/`

| File                                       | Replaces (cpp)                | Disposition summary                                            |
|--------------------------------------------|-------------------------------|----------------------------------------------------------------|
| `philosophy.md`                            | `philosophy.md`               | Ports as-is.                                                    |
| `pytest-conventions.md`                    | `catch2-conventions.md`       | Rewritten for pytest.                                           |
| `test-patterns.md`                         | `test-patterns.md`            | Adapted; parametrize, ids, subtests.                            |
| `test-helpers.md`                          | `test-helpers.md`             | Adapted; dataclass param structs, fixtures, contextvars providers, probe modules. |
| `error-path-testing.md`                    | `error-path-testing.md`       | Adapted; pytest.raises, optional Result-lib variant.            |
| `condition-based-waiting.md`               | `condition-based-waiting.md`  | Ports as-is; sync + async helpers.                              |
| `approval-tests.md`                        | `approval-tests.md`           | Ports as-is; approvaltests + pytest-approvaltests.              |
| (optional) `web-ui.md`                     | `qt-gui.md`                   | Drop the Qt content; add Playwright if the codebase has a web UI.|

### `python-debugging-principles/`

| File                                                      | Replaces (cpp)               | Disposition summary                                            |
|-----------------------------------------------------------|------------------------------|----------------------------------------------------------------|
| `root-cause-tracing.md`                                   | `root-cause-tracing.md`      | Ports as-is; traceback / loguru / rich / pytest-randomly.        |
| `defense-in-depth.md`                                     | `defense-in-depth.md`        | Ports as-is.                                                    |
| `logging.md`                                              | `logging.md`                 | Adapted; stdlib logging / loguru / structlog choice.            |
| `static-analysis-and-runtime-checks.md`                   | `sanitizers.md`              | Rewritten; pyright + ruff + dev mode + memray + py-spy stack.   |

### `agent/`

| File                            | Replaces (cpp)              | Disposition summary                                         |
|---------------------------------|------------------------------|-------------------------------------------------------------|
| `python-agent-context.md`       | `cpp-agent-context.md`       | Same shape; section ordering preserved; vocabulary swapped.  |
| `python-agent-examples.md`      | `cpp-agent-examples.md`      | Same shape; good/bad pairs per section.                      |
| `agent/skills/repo-specific-python-guidelines/` | matching C++ skill | Out of scope for this exploration; flag for later.           |

## Highest-impact recommendations (top 5)

1. **Make `pyright --strict` (or `pyrefly --strict`) the default CI
   gate.** This is the single biggest "make a bug class structurally
   impossible" lever the Python ecosystem offers, and it is the
   closest analogue to the C++ "compile-time correctness" rules.
2. **Default to exceptions inside a domain, not a Result library.**
   Document the Result-library pattern as an option for codebases that
   want type-checker-visible failure branches at the boundary, but
   the in-domain story is plain raise/except. This avoids the LEAF/
   expected machinery wholesale.
3. **Adopt `contextvars`-backed module singletons for cross-cutting
   services** (clock, logger, timer, settings). Compose naturally with
   asyncio (per-task), threads (per-thread), and pytest fixtures
   (per-test). This replaces the C++ `std::variant`-backed-global
   pattern with a simpler Python-native shape.
4. **Use `@dataclass(frozen=True, kw_only=True, slots=True)` as the
   default data-carrying shape**, with pydantic models reserved for
   trust-boundary parsing. NewType for cheap identity wrappers; frozen
   dataclass for the in-domain value object; pydantic at the boundary.
5. **Replace `sanitizers.md` with a "static analysis + runtime
   checks + profilers" chapter.** The C++ sanitizer model does not
   transfer; what does transfer is the discipline of running checkers
   on a schedule, version-controlling suppressions, and triaging
   before silencing. The Python stack is pyright + ruff + `python -X dev`
   + `pytest -W error` + memray + py-spy.

## Cross-cutting opinions baked into the analyses

- Target **Python 3.13+ / 3.14**; use **PEP 695 type syntax** by default;
  reach for `typing.Self`, `assert_never`, `Literal[...]` discriminators,
  `TypeIs` user-defined narrowing, `@override`.
- **pyright** is the recommended type checker; **pyrefly** is the
  drop-in faster alternative once it stabilises typing-spec conformance.
- **ruff** is the linter and formatter; recommended rule set documented
  in `research/ruff-rule-sets.md`.
- **uv** is the recommended package manager; `pyproject.toml` is the
  single source of truth.
- **dataclasses** are the default; **pydantic** at trust boundaries;
  **attrs** when validators / converters are needed and pydantic is
  overkill.
- **`match` / `case`** is the recommended dispatch shape where it
  improves readability; `assert_never` for exhaustiveness; `Protocol`
  for structural typing; generics only where they pay off.
- **anyio** is the recommended structured-concurrency library; the
  `asyncio.TaskGroup` stdlib form is the fallback for codebases that
  do not want the extra dependency.

## Artifact map

- `00-summary.md` -- this file.
- `design/*.md` (14 files) -- per-design-file analysis.
- `testing/*.md` (8 files) -- per-testing-file analysis.
- `debugging/*.md` (4 files) -- per-debugging-file analysis.
- `agent-context/*.md` (2 files) -- agent-context-file analyses.
- `research/*.md` (7 files) -- research notes on typecheckers, strong
  types, PEP 695, exhaustive matching, structured concurrency, ruff
  rule sets, observability.

Total: **35 markdown files**, all under
`/home/msi/repos/python-guidelines/work-in-progress/exploration/cpp-claude/`.
