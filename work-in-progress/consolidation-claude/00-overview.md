# Python guidelines: proposed structure and dispositions

This is Claude's independent synthesis of the cpp-guidelines port. It mirrors
the cpp layout where the principle survives untouched and replaces files where
the Python idiom is wholly different. Codex's parallel pass produced a shorter
list with broader files; we recommend the longer per-topic split because the
cpp guides have proven readable at that granularity and the agent-context
file we are also porting indexes the topics one-by-one.

## Three sister sub-repos plus the always-loaded agent context

Mirror the cpp shape: `python-design-principles/`, `python-testing-principles/`,
`python-debugging-principles/`, and `agent/`. Add one new top-level guide for
project layout and tooling, since the cpp side leaves this implicit in the
build-system files and Python has no comparable single source of truth.

```
python-projects-and-tooling/    NEW
python-design-principles/       PORT, file-per-topic
python-testing-principles/      PORT, file-per-topic
python-debugging-principles/    PORT, file-per-topic
agent/
  python-agent-context.md       PORT_WITH_ADAPTATION
  python-agent-examples.md      PORT_WITH_ADAPTATION
```

Disposition keys:

- `PORT_AS_IS`: principle and most prose survive; substitute Python syntax.
- `PORT_WITH_ADAPTATION`: principle survives; mechanics swap.
- `PYTHON_SPECIFIC`: rewrite around Python-native shape; cpp source is sparse.
- `NEW`: no cpp parent; needed for Python.
- `DROP`: cpp file does not earn its place in the Python set.

## `python-projects-and-tooling/`

| File                            | Scope                                                                                          | Disposition | Source notes                                                              |
|---------------------------------|------------------------------------------------------------------------------------------------|-------------|---------------------------------------------------------------------------|
| `project-layout.md`             | `src/` layout, uv workspaces, child `pyproject.toml` files, monorepo boundaries.                | NEW         | MONOREPO.md, factors-claude/patterns/module-layout.md, codex-research/tooling-tests-docs.md. |
| `tooling-baseline.md`           | uv, ruff config, pyright config, pre-commit hooks, Makefile/justfile, CI commands.              | NEW         | factors-claude/tooling-baseline.md, cpp-claude/research/ruff-rule-sets.md, codex-research/modern-python-tooling-and-typing.md. |
| `python-version-and-typing-stack.md` | Python 3.13/3.14 baseline, PEP 695 syntax, PEP 649 annotations, pyright vs pyrefly choice. | NEW         | cpp-claude/research/typecheckers-2026.md, codex-research/modern-python-tooling-and-typing.md. |

## `python-design-principles/`

| File                                       | One-line scope                                                                  | cpp source                                | Disposition          |
|--------------------------------------------|---------------------------------------------------------------------------------|-------------------------------------------|----------------------|
| `architecture.md`                           | Domain ownership, adapters, forward-only imports, functional core / shell.       | `architecture.md`                          | PORT_AS_IS           |
| `types-and-correctness.md`                  | NewType, frozen dataclass, pydantic at boundary, parse-don't-validate, pyright strict policy. | `compile-time-correctness.md`               | PYTHON_SPECIFIC      |
| `comments-and-docstrings.md`                | Guiding comments, docstring conventions, negative-documentation anti-pattern.    | `comments.md`                              | PORT_AS_IS           |
| `declarative-style.md`                      | Decomposition, named predicates, comprehensions, itertools, pandas pipe.         | `declarative-style.md`                     | PORT_AS_IS           |
| `functional-programming.md`                 | Pure functions, sum types via Literal, `match` dispatch, late-binding traps.     | `functional-programming.md`                | PORT_WITH_ADAPTATION |
| `generics-and-protocols.md`                 | PEP 695 generics, Protocol vs ABC, TypeIs/TypeGuard, assert_never.               | `templates.md`                             | PYTHON_SPECIFIC      |
| `decorators-and-metaprogramming.md`         | Decorators first, `__init_subclass__` next, metaclasses last; `ParamSpec` typing. | `preprocessor-macros.md`                   | PYTHON_SPECIFIC      |
| `error-handling.md`                         | Exception-first inside a domain, boundary translation, top-level catch-all.       | `error-handling.md`                        | PYTHON_SPECIFIC      |
| `invariants-and-rollback.md`                | Commit-at-end, ExitStack.callback, idempotency, free-threaded caveats.            | `invariants.md`                            | PORT_WITH_ADAPTATION |
| `state-machines.md`                         | `match` over discriminated-union states; `transitions` for large machines.       | `state-machines.md`                        | PORT_WITH_ADAPTATION |
| `pipelines.md`                              | Callable fields, wiring at the edge, defer re-entrance, async create_task.       | `pipelines.md`                             | PORT_AS_IS           |
| `runtime-and-concurrency.md`                | asyncio.TaskGroup vs anyio, call_soon_threadsafe, free-threaded sidebar.         | `runtime.md`                               | PORT_WITH_ADAPTATION |
| `cross-cutting-services.md`                 | contextvars-backed module singletons; clock/logger/timer catalog; pytest fixtures. | `cross-cutting.md`                         | PYTHON_SPECIFIC      |
| `performance.md`                            | Vectorisation first, `__slots__`, profiler choice, when to leave Python.         | `performance.md`                           | PYTHON_SPECIFIC      |
| `data-pipeline-and-dataframes.md`           | pandera at frame boundary, ibis/dbt split, no f-string SQL, indicator purity.    | (none)                                     | NEW                  |

`data-pipeline-and-dataframes.md` is genuinely new and only earns its place
because the target audience (factors2-shaped codebases) is dataframe-heavy. If
the first user codebase has no dataframes, defer it to a second pass.

## `python-testing-principles/`

| File                                       | One-line scope                                                                  | cpp source                                | Disposition          |
|--------------------------------------------|---------------------------------------------------------------------------------|-------------------------------------------|----------------------|
| `philosophy.md`                             | Test behavior not implementation, independent oracle, component vs integration.  | `philosophy.md`                            | PORT_AS_IS           |
| `pytest-conventions.md`                     | Collection, marks (registered in `pyproject.toml`), parametrize ids, xdist, plugin inventory. | `catch2-conventions.md`                    | PYTHON_SPECIFIC      |
| `test-patterns.md`                          | parametrize, fixtures, `assert_type`, table-driven testing, subtests when needed. | `test-patterns.md`                         | PORT_WITH_ADAPTATION |
| `test-helpers.md`                           | `_make_valid_X()` factories, dataclass params, fixtures, contextvars providers, probe modules. | `test-helpers.md`                          | PORT_WITH_ADAPTATION |
| `error-path-testing.md`                     | `pytest.raises`, exact-message via `re.escape`, rollback assertions, optional Result style. | `error-path-testing.md`                    | PORT_WITH_ADAPTATION |
| `condition-based-waiting.md`                | sync `wait_for`, anyio `fail_after`, the `time.sleep`-in-coroutine trap.         | `condition-based-waiting.md`               | PORT_AS_IS           |
| `approval-tests.md`                         | approvaltests + pytest-approvaltests, scrubbers, dataframe stabilisation, JSON over text. | `approval-tests.md`                        | PORT_AS_IS           |
| (drop) `qt-gui.md`                          | Drop unless the user codebase actually owns a Qt surface.                        | `qt-gui.md`                                | DROP                 |
| (optional) `web-ui.md`                      | Add only if the user codebase has Playwright/Selenium needs.                     | (none)                                     | NEW (conditional)    |

## `python-debugging-principles/`

| File                                       | One-line scope                                                                  | cpp source                                | Disposition          |
|--------------------------------------------|---------------------------------------------------------------------------------|-------------------------------------------|----------------------|
| `root-cause-tracing.md`                     | Trace bad value backward; pdb, pytest-randomly, py-spy, rich.traceback.          | `root-cause-tracing.md`                    | PORT_AS_IS           |
| `defense-in-depth.md`                       | Parse-at-boundary, business validation, environment guards, debug instrumentation. | `defense-in-depth.md`                      | PORT_AS_IS           |
| `logging.md`                                | One logger per project (loguru recommended), static message + kwargs, `logger.exception` in except. | `logging.md`                               | PORT_WITH_ADAPTATION |
| `static-analysis-and-runtime-checks.md`     | pyright/ruff CI gate, `-X dev` + `-W error` for tests, memray/py-spy cookbook, free-threaded caveat. | `sanitizers.md`                            | PYTHON_SPECIFIC      |

The `sanitizers.md` rename is deliberate: the cpp file's mental model (build
presets per defect class with version-controlled suppressions) ports to
Python as "static analysis + runtime warnings + profilers". Codex's own
proposal calls this `tooling.md`; we prefer the longer name because there is
already a `tooling-baseline.md` in `python-projects-and-tooling/` and the two
should not collide.

## `agent/`

| File                            | One-line scope                                                                | cpp source                | Disposition          |
|---------------------------------|-------------------------------------------------------------------------------|---------------------------|----------------------|
| `python-agent-context.md`       | The always-loaded rule sheet. Section ordering mirrors the cpp file.            | `cpp-agent-context.md`     | PORT_WITH_ADAPTATION |
| `python-agent-examples.md`      | The on-demand pair of good/bad examples per section.                            | `cpp-agent-examples.md`    | PORT_WITH_ADAPTATION |

## Files to drop and why

- `cpp-testing-principles/qt-gui.md`: drop unless the user codebase has Qt.
  The cpp content (offscreen platform, QSignalSpy, QAbstractItemModelTester)
  does not transfer; rewriting it for PySide6/pytest-qt is wasted work for
  the common case.

## Cross-cutting opinions baked in

- Target Python 3.13/3.14 with PEP 695 syntax everywhere new.
- pyright strict as the default CI gate; pyrefly noted as the swap-in once
  conformance stabilises.
- Exceptions inside a domain; Result libraries documented as a boundary
  option only.
- contextvars-backed module singletons for cross-cutting services.
- `@dataclass(frozen=True, kw_only=True, slots=True)` for in-domain values;
  pydantic at the trust boundary; pandera at the DataFrame boundary.
- `asyncio.TaskGroup` is the baseline; anyio recommended when cancel scopes
  pay for the extra dependency.
- loguru as the default logger with a stdlib-`logging` redirect.
- Gradual typing is doctrine, not aspiration: a typing-escape-hatch section
  documents when `Any`, `cast`, and `# type: ignore` are correct, not just
  acceptable.

## Where Codex and Claude diverge on structure

Codex proposes ~10 broader files in 3 top-level folders
(`python-design/`, `python-testing/`, `python-debugging/` plus `python-projects/`,
`python-runtime/`, `python-style/`). Claude proposes ~22 narrower files in
the cpp shape. We recommend the narrower shape because:

1. The cpp guides have proven scannable at this granularity.
2. The always-loaded agent-context already maps one-to-one to the topical
   guides; collapsing the topical guides into broader files breaks that
   mapping.
3. Each section is independently citable from CI failure messages, code
   review comments, and PR descriptions.

If the owner prefers Codex's broader shape, the section outlines in
`design-guides.md` / `testing-guides.md` / `debugging-guides.md` still apply;
they would just be folded into fewer files.
