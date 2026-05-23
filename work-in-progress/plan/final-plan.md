# Python guidelines: final port plan

This plan reconciles the Claude-side and Codex-side passes into a single
writing program. Its job is to name every guide to produce, fix the
policy decisions, and point each writing pass at the right source
artifacts. The plan is a prompt skeleton, not a draft of the guides
themselves; the section-by-section outlines live in the consolidation
files cited below.

## How to use this plan

1. Read this file end-to-end.
2. For each guide you are asked to write, jump to its row in the guide
   table and follow the four pointers: section outline, cpp source(s),
   factors2 patterns/anti-patterns, supporting research.
3. The cross-guide policy decisions (next section) override anything
   in the upstream artifacts that disagrees. They are the resolution of
   the Claude-vs-Codex divergences and the owner's open questions.
4. Writing agents should be prompted one guide at a time with the
   relevant slice of this plan plus the cited artifacts. Do not ask one
   agent to write more than one guide; cross-guide consistency comes
   from this plan, not from a single author.

## Inputs

Consolidation (read first):

- `work-in-progress/consolidation-claude/00-overview.md`
- `work-in-progress/consolidation-claude/design-guides.md`
- `work-in-progress/consolidation-claude/testing-guides.md`
- `work-in-progress/consolidation-claude/debugging-guides.md`
- `work-in-progress/consolidation-claude/agent-context.md`
- `work-in-progress/consolidation-claude/cross-cutting-themes.md`
- `work-in-progress/consolidation-claude/divergences-and-decisions.md`
- `work-in-progress/consolidation-claude/risks-and-open-questions.md`
- `work-in-progress/consolidation-codex/research-consolidation.md`
- `work-in-progress/plan-codex/python-guidelines-port-plan.md` (treat
  as input, superseded by this file where they disagree)

Per-cpp-guide analyses:

- `work-in-progress/exploration/cpp-claude/{design,testing,debugging,agent-context,research}/*.md`
- `work-in-progress/exploration/cpp-codex/{design,testing,debugging,agent-context,research}/*.md`

Factors2 patterns and anti-patterns:

- `work-in-progress/exploration/factors-claude/00-summary.md`
- `work-in-progress/exploration/factors-claude/patterns/*.md`
- `work-in-progress/exploration/factors-claude/anti-patterns.md`
- `work-in-progress/exploration/factors-claude/tooling-baseline.md`
- `work-in-progress/exploration/codex-research/architecture-and-patterns.md`
- `work-in-progress/exploration/codex-research/tooling-tests-docs.md`
- `work-in-progress/exploration/codex-research/generalizable-cpp-design.md`
- `work-in-progress/exploration/codex-research/generalizable-cpp-testing-debugging.md`
- `work-in-progress/exploration/codex-research/modern-python-tooling-and-typing.md`
- top-level `work-in-progress/{claude,codex}-factors-antipatterns.md` if
  the writer wants a second cut at the anti-pattern list.

Existing durable docs in the repo:

- `MONOREPO.md` (canonical monorepo shape)
- `README.md` (currently empty; the writing pass populates it)

External upstream:

- `/home/msi/repos/cpp-guidelines/` (the cpp sister repo; cited for
  parity, never copied verbatim)

## Cross-guide policy decisions (resolved)

Resolved from the divergence and open-question docs. Every guide
follows these unless a guide-specific note overrides.

| Topic                       | Decision                                                                                                       | Origin                                         |
|-----------------------------|----------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| Python version              | 3.13 baseline, 3.14 forward target. One sidebar per relevant guide for PEP 649 implications.                   | `risks-and-open-questions.md` Q1               |
| Type checker                | pyright as the default; pyrefly as the documented alternative; mypy as legacy fallback only.                   | `divergences-and-decisions.md` D8, Q2          |
| Strictness                  | Global `pyright --strict` as the target state; ratchet from `standard` directory-by-directory.                 | D9, Q3                                         |
| Formatter / linter          | ruff format (replaces black); ruff check with documented baseline + strict variant.                            | Q4 of `plan-codex`, D7                         |
| Package manager             | uv with workspaces; `uv sync --locked` and `uv run` are the canonical entry points.                            | `factors-claude/tooling-baseline.md`           |
| Data carrier                | By layer: frozen kw_only slots dataclass for values; mutable for aggregates; pydantic at trust boundary; pandera at DataFrame boundary. | D4                          |
| Settings                    | pydantic-settings.BaseSettings.                                                                                | Q13                                            |
| Boundary validation         | pydantic v2 only; document attrs and msgspec as alternatives.                                                  | Q5, Q6                                         |
| Errors                      | Exceptions in-domain; boundary translation at HTTP / queue / job / CLI edges; document Result libraries (`returns` or `result`) as a library-neutral boundary option with one example. | D5, Q10 |
| Logger                      | loguru for applications; stdlib `logging` for libraries; structlog when strict key/value semantics required. Never mix two loggers in one project. | D13, Q7                          |
| Concurrency                 | `asyncio.TaskGroup` baseline; anyio documented swap-in when cancel scopes are load-bearing or trio support is needed. | D3, Q8                                  |
| Cross-cutting services      | contextvars-backed module singletons for logger / clock / timer / settings / tracing; explicit constructor parameters for domain-specific dependencies; `ContextVar` alone for request-scoped context (request_id, user, tenant). | D2, Q12 |
| Approval tests              | approvaltests + pytest-approvaltests with `PythonNativeReporter`; `pytest-approvaltests-geo` if geospatial.    | D6, Q11                                        |
| Coverage gate               | 80% line coverage required in CI; reducible only via PR comment + diff.                                        | Q18                                            |
| Pre-commit hook surface     | lint and format only; type-check in CI. `default_install_hook_types = [pre-commit, post-checkout, post-merge, post-rewrite]`. | Q17                                |
| Import-boundary enforcement | `import-linter` for declarative contracts in CI; `pydeps` SVG committed in `doc/`; ruff `TID252` everywhere.   | Q9                                             |
| Dataframe API in examples   | pandas first; polars and ibis as one-liners on the side.                                                       | Q4                                             |
| Docs renderer               | mkdocs-material default; sphinx for autodoc-heavy projects. Sidebar in `comments-and-docstrings.md`.           | Q19                                            |
| Security                    | pip-audit (or safety) for dep CVEs; gitleaks (or trufflehog) for secret scanning. Warn-only initially, gate after baseline cleanup. | Q21                              |
| CODEOWNERS / PR templates   | In scope; short section in `tooling-baseline.md`.                                                              | Q20                                            |
| Free-threaded interpreter   | In scope as sidebars in `invariants-and-rollback.md` and `runtime-and-concurrency.md`; no dedicated chapter.   | Q15                                            |
| Dev-dep policy              | Every dev dependency has a documented invocation (Makefile target, doc, or test) or gets pruned.               | D14                                            |
| Agent context in first pass | In scope. The skill packaging (`agent/skills/repo-specific-python-guidelines/`) is deferred.                   | D12, Q16                                       |
| Comments                    | Present-tense, positive statements only. The "negative documentation" rule from the user's CLAUDE.md applies with Python-specific docstring examples. | D15                              |

## Repository shape

Four sister folders plus agent context, mirroring the cpp shape with
one tooling addition. Guide count: 25 markdown files plus the two agent
files. One file is conditional on the target codebase (qt or web UI).

```
README.md                       (rewrite: list the four sister guides + agent pair)
MONOREPO.md                     (keep; cross-linked from project-layout.md)
python-projects-and-tooling/    NEW (3 files)
  README.md
  project-layout.md
  tooling-baseline.md
  python-version-and-typing-stack.md
python-design-principles/       PORT, file-per-topic (15 files)
  README.md
  architecture.md
  types-and-correctness.md
  comments-and-docstrings.md
  declarative-style.md
  functional-programming.md
  generics-and-protocols.md
  decorators-and-metaprogramming.md
  error-handling.md
  invariants-and-rollback.md
  state-machines.md
  pipelines.md
  runtime-and-concurrency.md
  cross-cutting-services.md
  performance.md
  data-pipeline-and-dataframes.md   (conditional on dataframe-heavy target codebase)
python-testing-principles/      PORT, file-per-topic (7 files; +1 conditional)
  README.md
  philosophy.md
  pytest-conventions.md
  test-patterns.md
  test-helpers.md
  error-path-testing.md
  condition-based-waiting.md
  approval-tests.md
  web-ui.md                     (conditional on Playwright/Selenium-using target)
python-debugging-principles/    PORT, file-per-topic (4 files)
  README.md
  root-cause-tracing.md
  defense-in-depth.md
  logging.md
  static-analysis-and-runtime-checks.md
agent/
  python-agent-context.md       (always-loaded rule sheet)
  python-agent-examples.md      (on-demand good/bad pairs)
```

Files dropped from the cpp set: `qt-gui.md` (unless the target codebase
owns a Qt surface).

Files renamed: `sanitizers.md` becomes `static-analysis-and-runtime-checks.md`.

Files folded into a single Python guide: none. We considered Codex's
broader-file proposal and rejected it in `divergences-and-decisions.md`
D1 because the agent-context file maps one-to-one to the topical
guides; collapsing the guides breaks that index.

## Guide-by-guide writing briefs

Format for each row: file path, scope sentence, section outline source,
cpp source files, factors2 citations to include, supporting research,
and any guide-specific policy carveouts.

### `python-projects-and-tooling/project-layout.md`  (NEW)

- Scope: `src/` layout, uv workspaces, child `pyproject.toml` files,
  monorepo boundaries, where `docker/`, `db/`, and `dagster/` live.
- Section outline: `consolidation-claude/design-guides.md` does not
  cover this; the closest brief is `consolidation-codex/research-consolidation.md`
  "Project Layout and Tooling" section.
- Cross-link `MONOREPO.md` as the canonical local convention; do not
  duplicate its content.
- factors2 must-cite: `factors-claude/patterns/module-layout.md` (DAG
  shape, forward-only imports, `__init__.py` export discipline).
- Research: `codex-research/tooling-tests-docs.md`.

### `python-projects-and-tooling/tooling-baseline.md`  (NEW)

- Scope: uv, ruff config (baseline + strict variant), pyright config,
  pre-commit hooks (lint+format only), Makefile or justfile, CI command
  surface, security scanners, CODEOWNERS / PR templates.
- Section outline: `consolidation-claude/00-overview.md` row 39.
- factors2 must-cite: `factors-claude/tooling-baseline.md`. Quote the
  factors2 `pyproject.toml` ruff section and pre-commit hook-types
  pattern.
- factors2 anti-patterns to cite by reference: `factors-claude/anti-patterns.md`
  entries Q (declared-but-unused dev deps) and L (mixed loggers).
- Research: `cpp-claude/research/ruff-rule-sets.md`, `codex-research/modern-python-tooling-and-typing.md`.
- Policy note: every dev dep has a documented invocation or gets pruned
  (D14). The CODEOWNERS / PR template section is short - examples only.

### `python-projects-and-tooling/python-version-and-typing-stack.md`  (NEW)

- Scope: Python 3.13 baseline, 3.14 target. PEP 695 type-statement
  syntax. PEP 649 deferred annotations. pyright vs pyrefly comparison
  with explicit recommendation. mypy as legacy.
- Section outline: `consolidation-claude/00-overview.md` row 41.
- Research: `cpp-claude/research/typecheckers-2026.md` (numbers and
  conformance tables), `codex-research/modern-python-tooling-and-typing.md`.

### `python-design-principles/architecture.md`  (PORT_AS_IS)

- Scope: domain ownership, adapters at boundaries, forward-only
  imports, functional core / imperative shell.
- Section outline: `consolidation-claude/design-guides.md` "architecture.md".
- cpp sources: `/home/msi/repos/cpp-guidelines/cpp-design-principles/architecture.md`,
  per-file analyses at `cpp-claude/design/architecture.md` and `cpp-codex/design/architecture.md`.
- factors2 must-cite: `factors-claude/patterns/module-layout.md` for the
  forward-only DAG; `factors-claude/00-summary.md` for the
  domain-core-vs-imperative-shell split in `pyfactors/` vs
  `etl_factors/`.
- Quote the MONOREPO.md boundary-first framing as the design motivation;
  the filesystem shape is in `project-layout.md`, the conceptual shape
  is here (D10).

### `python-design-principles/types-and-correctness.md`  (PYTHON_SPECIFIC)

- Scope: NewType, frozen kw_only slots dataclass for values, pydantic
  at boundary, pandera at DataFrame boundary, parse-don't-validate,
  pyright strict ratchet, typing escape hatches (when `Any`, `cast`,
  `# type: ignore` are correct).
- Section outline: `consolidation-claude/design-guides.md`
  "types-and-correctness.md".
- cpp sources: `cpp-design-principles/compile-time-correctness.md`.
- factors2 must-cite: `factors-claude/patterns/typing-discipline.md`
  (heuristics for when typing is escaped, with justification);
  `factors-claude/patterns/pydantic-pandera-dataclasses.md` (layer
  split that has proven itself).
- factors2 anti-patterns to cite: the 40-arm `Function = A | B | C | ...`
  without a discriminator that forces `hasattr` + `inspect.signature`
  + three `# type: ignore` (factors-claude reports it at
  `indicators/computed_indicator.py:28-43`).
- Research: `cpp-claude/research/strong-types-and-data-classes.md`,
  `cpp-claude/research/pep695-and-generics.md`.
- Policy carveouts: pydantic v2 only; attrs documented as
  "when pydantic is overkill"; msgspec for hot deserialisation paths.

### `python-design-principles/comments-and-docstrings.md`  (PORT_AS_IS)

- Scope: positive present-tense comments only, Google-style docstrings,
  the negative-documentation anti-pattern with Python examples, sidebar
  on mkdocs-material vs sphinx.
- Section outline: `consolidation-claude/design-guides.md`
  "comments-and-docstrings.md".
- cpp source: `cpp-design-principles/comments.md`.
- Must cite the user's CLAUDE.md negative-documentation rule with the
  Python-specific "non-responsibility in a docstring" example
  (D15). Quote the past-tense / delta / non-responsibility tells.
- factors2 must-cite: any negative-doc examples surfaced in
  `factors-claude/patterns/documentation.md`.

### `python-design-principles/declarative-style.md`  (PORT_AS_IS)

- Scope: small composable functions, named predicates, comprehensions,
  itertools, `match` statement, pandas `.pipe`, dataframe vectorisation.
- Section outline: `consolidation-claude/design-guides.md` "declarative-style.md".
- cpp source: `cpp-design-principles/declarative-style.md`.
- factors2 must-cite: examples from `factors-claude/patterns/data-pipeline.md`
  and `factors-claude/patterns/typing-discipline.md`
  (`match` + `assert_never` on `Literal` fields at
  `scoring_strategies/binary.py:29-35` and `interval_indicator.py:30-36`).

### `python-design-principles/functional-programming.md`  (PORT_WITH_ADAPTATION)

- Scope: pure functions in the core, sum types via `Literal` discriminator,
  `match` dispatch with `assert_never`, late-binding traps in
  comprehensions and default args.
- Section outline: `consolidation-claude/design-guides.md`
  "functional-programming.md".
- cpp source: `cpp-design-principles/functional-programming.md`.
- factors2 must-cite: discriminated Pydantic unions assembled in
  `__init__.py` (`pipeline/scoring_strategies/__init__.py:9`), per
  `factors-claude/patterns/pydantic-pandera-dataclasses.md`.
- Research: `cpp-claude/research/exhaustive-matching.md`.

### `python-design-principles/generics-and-protocols.md`  (PYTHON_SPECIFIC)

- Scope: PEP 695 generics syntax, `Protocol` over ABC for structural
  typing, `TypeIs` / `TypeGuard` (PEP 742), `assert_never` for
  exhaustiveness, `ParamSpec` for decorator-preserving typing.
- Section outline: `consolidation-claude/design-guides.md`
  "generics-and-protocols.md".
- cpp source (informs principle, not syntax): `cpp-design-principles/templates.md`.
- Research: `cpp-claude/research/pep695-and-generics.md`,
  `cpp-claude/research/exhaustive-matching.md`.
- Policy: do not port C++ template-metaprogramming framing. State once
  that Python generics are structural and erased; they exist for
  type-checker feedback, not runtime behavior.

### `python-design-principles/decorators-and-metaprogramming.md`  (PYTHON_SPECIFIC)

- Scope: decorators first; `__init_subclass__` and class decorators
  next; metaclasses last and only when nothing else fits. Typing
  decorators correctly with `ParamSpec`. `functools.wraps`.
- Section outline: `consolidation-claude/design-guides.md`
  "decorators-and-metaprogramming.md".
- cpp source (informs principle): `cpp-design-principles/preprocessor-macros.md`.
- factors2 must-cite: Dagster component decorators if present in
  `factors-claude/patterns/data-pipeline.md`.
- Policy: do not import the cpp file's macro-hygiene examples; Python's
  problem shape is different.

### `python-design-principles/error-handling.md`  (PYTHON_SPECIFIC)

- Scope: exceptions are the default inside a domain; per-domain
  exception hierarchy with structured attributes; boundary translation
  at HTTP / queue / CLI / Dagster-op edges; `logger.exception` only
  inside `except`; do not swallow; Result libraries documented as a
  library-neutral boundary option with one example using `returns` or
  `result`.
- Section outline: `consolidation-claude/design-guides.md` "error-handling.md".
- cpp source: `cpp-design-principles/error-handling.md`. Do NOT port
  the LEAF / `std::expected` machinery (D16); cite the cpp file for the
  principle of boundary translation, not for the result-type vocabulary.
- factors2 must-cite: `factors-claude/patterns/error-handling.md` for
  the good patterns and `factors-claude/anti-patterns.md` for the
  catch-and-toast smell (8 instances in `portfolio_management/app.py`,
  one in `fetch_cnpj.py:38`).

### `python-design-principles/invariants-and-rollback.md`  (PORT_WITH_ADAPTATION)

- Scope: commit-at-end mutation, `ExitStack.callback` for rollback,
  context managers, atomic file rename, idempotency, free-threaded
  interpreter sidebar.
- Section outline: `consolidation-claude/design-guides.md`
  "invariants-and-rollback.md".
- cpp source: `cpp-design-principles/invariants.md`.
- factors2 must-cite: `factors-claude/patterns/state-invariants.md`
  (idempotent upserts, dedup patterns from
  `etl_factors/sources/economatica/`).
- Sidebar on the free-threaded interpreter (`dict.setdefault` no
  longer atomic, etc.); cite Q15.

### `python-design-principles/state-machines.md`  (PORT_WITH_ADAPTATION)

- Scope: `match` over discriminated-union states; named state classes;
  `transitions` library for large machines; testing with state-table
  approval tests.
- Section outline: `consolidation-claude/design-guides.md` "state-machines.md".
- cpp source: `cpp-design-principles/state-machines.md`.
- factors2 must-cite: any state-machine pattern surfaced in
  `factors-claude/patterns/data-pipeline.md` (Dagster sensor or backtest
  state).

### `python-design-principles/pipelines.md`  (PORT_AS_IS)

- Scope: callable fields composed at the edge, deferred re-entrance with
  `asyncio.create_task`, queue-backed pipelines, Dagster ops as
  pipeline composition.
- Section outline: `consolidation-claude/design-guides.md` "pipelines.md".
- cpp source: `cpp-design-principles/pipelines.md`.
- factors2 must-cite: `factors-claude/patterns/data-pipeline.md`
  (explicit pipeline stages in `pipeline/stages.py:62-183`).

### `python-design-principles/runtime-and-concurrency.md`  (PORT_WITH_ADAPTATION)

- Scope: `asyncio.TaskGroup` baseline, `call_soon_threadsafe` for
  thread-to-loop, anyio swap-in triggers, blocking-in-async smells,
  free-threaded interpreter sidebar.
- Section outline: `consolidation-claude/design-guides.md` "runtime-and-concurrency.md".
- cpp source: `cpp-design-principles/runtime.md`.
- factors2 must-cite: aiolimiter rate-limiting and aiohttp/httpx
  patterns from `factors-claude/patterns/async-concurrency.md`.
- Research: `cpp-claude/research/structured-concurrency.md`. Document
  the three anyio swap-in triggers from D3.

### `python-design-principles/cross-cutting-services.md`  (PYTHON_SPECIFIC)

- Scope: contextvars-backed module singletons for logger / clock /
  timer / settings / tracing; the decision matrix from D2 (singleton
  vs explicit param vs `ContextVar` vs pytest fixture); test-time
  substitution via `monkeypatch.setattr`.
- Section outline: `consolidation-claude/design-guides.md` "cross-cutting-services.md".
- cpp source: `cpp-design-principles/cross-cutting.md`.
- Must include the D2 decision matrix verbatim.

### `python-design-principles/performance.md`  (PYTHON_SPECIFIC)

- Scope: vectorisation first (numpy / pandas / polars); `__slots__`;
  profiler choice (cProfile baseline, py-spy for sampling, memray for
  allocation, pyinstrument for call-stack visualisation); when to leave
  Python (numba, cython, rust-via-pyo3).
- Section outline: `consolidation-claude/design-guides.md` "performance.md".
- cpp source: `cpp-design-principles/performance.md`. Drop the
  hot-path allocation framing; cite for the principle of measure-first.
- factors2 must-cite: `factors-claude/patterns/performance.md`
  (pytest-benchmark patterns in `benchmarks/`).
- State explicitly: type annotations do not speed CPython by themselves.

### `python-design-principles/data-pipeline-and-dataframes.md`  (NEW, CONDITIONAL)

- Scope: pandera at the DataFrame boundary, ibis vs pandas vs duckdb vs
  raw SQL decision matrix, no f-string SQL (use `psycopg.sql` or
  parameterised queries), indicator purity (no I/O in compute
  functions), Dagster asset shape, dbt model boundaries.
- Section outline: `consolidation-claude/design-guides.md`
  "data-pipeline-and-dataframes.md".
- cpp source: none.
- factors2 must-cite: `factors-claude/patterns/data-pipeline.md` and
  `factors-claude/patterns/pydantic-pandera-dataclasses.md`. Cite the
  layered validation pattern at `backtest/models.py:1022-1179`.
- factors2 anti-patterns to cite: f-string SQL in
  `database_queries.py` (lines 28, 92, 118, 144, 188, 216, 237) next
  to the correct `psycopg.sql` pattern at lines 285-294.
- Skip this guide entirely if the first target codebase does not have
  dataframes.

### `python-testing-principles/philosophy.md`  (PORT_AS_IS)

- Scope: behavior over implementation, independent oracle, component
  vs integration scope, test data determinism.
- Section outline: `consolidation-claude/testing-guides.md` "philosophy.md".
- cpp source: `cpp-testing-principles/philosophy.md`.

### `python-testing-principles/pytest-conventions.md`  (PYTHON_SPECIFIC)

- Scope: test collection (`tests/` layout, `conftest.py` discipline),
  registered marks in `pyproject.toml`, parametrize ids, xdist for
  parallelism, plugin inventory (pytest-cov, pytest-randomly,
  pytest-asyncio, pytest-benchmark, pytest-approvaltests, pyfakefs).
- Section outline: `consolidation-claude/testing-guides.md` "pytest-conventions.md".
- cpp source (informs principle): `cpp-testing-principles/catch2-conventions.md`.
- factors2 must-cite: `factors-claude/patterns/testing.md` for the
  fixture style and `factors-claude/tooling-baseline.md` for the
  `pyproject.toml` test config.

### `python-testing-principles/test-patterns.md`  (PORT_WITH_ADAPTATION)

- Scope: parametrize over duplicated tests, `assert_type` for
  type-level tests, table-driven tests, subtests when parametrize
  doesn't fit, fixture composition.
- Section outline: `consolidation-claude/testing-guides.md` "test-patterns.md".
- cpp source: `cpp-testing-principles/test-patterns.md`.

### `python-testing-principles/test-helpers.md`  (PORT_WITH_ADAPTATION)

- Scope: `_make_valid_X()` factories with dataclass parameters, fixture
  factories, contextvars providers as fixtures, probe modules for
  integration tests.
- Section outline: `consolidation-claude/testing-guides.md` "test-helpers.md".
- cpp source: `cpp-testing-principles/test-helpers.md`.
- factors2 must-cite: factory patterns surfaced in
  `factors-claude/patterns/testing.md`.

### `python-testing-principles/error-path-testing.md`  (PORT_WITH_ADAPTATION)

- Scope: `pytest.raises` with exact-message matching via `re.escape`,
  rollback assertions (state unchanged after a failure), per-rule
  predicate tests, optional Result-style coverage.
- Section outline: `consolidation-claude/testing-guides.md` "error-path-testing.md".
- cpp source: `cpp-testing-principles/error-path-testing.md`.
- factors2 must-cite: the one-assertion-per-test pattern at
  `tests/pyfactors/backtest/test_validate_step_result.py` per
  `factors-claude/patterns/testing.md`.

### `python-testing-principles/condition-based-waiting.md`  (PORT_AS_IS)

- Scope: synchronous `wait_for(predicate, timeout)`; anyio
  `fail_after`; the `time.sleep`-inside-an-async-test trap.
- Section outline: `consolidation-claude/testing-guides.md` "condition-based-waiting.md".
- cpp source: `cpp-testing-principles/condition-based-waiting.md`.

### `python-testing-principles/approval-tests.md`  (PORT_AS_IS)

- Scope: approvaltests + pytest-approvaltests with
  `PythonNativeReporter`; scrubbers for non-deterministic fields
  (timestamps, ids); dataframe-output stabilisation; JSON over text
  where possible.
- Section outline: `consolidation-claude/testing-guides.md` "approval-tests.md".
- cpp source: `cpp-testing-principles/approval-tests.md`.
- factors2 must-cite: the approvaltests config in factors2 `pyproject.toml`
  per `factors-claude/tooling-baseline.md`.

### `python-debugging-principles/root-cause-tracing.md`  (PORT_AS_IS)

- Scope: trace the bad value backward to its source; pdb / pudb /
  ipdb; pytest-randomly for order-dependent failures; rich.traceback
  for readable stacks; py-spy for live processes.
- Section outline: `consolidation-claude/debugging-guides.md` "root-cause-tracing.md".
- cpp source: `cpp-debugging-principles/root-cause-tracing.md`.

### `python-debugging-principles/defense-in-depth.md`  (PORT_AS_IS)

- Scope: parse at the boundary (pydantic / pandera); business
  validation in domain code; environment guards; debug-only
  instrumentation behind a flag.
- Section outline: `consolidation-claude/debugging-guides.md` "defense-in-depth.md".
- cpp source: `cpp-debugging-principles/defense-in-depth.md`.

### `python-debugging-principles/logging.md`  (PORT_WITH_ADAPTATION)

- Scope: one logger per project (loguru for apps, stdlib for libs,
  structlog when key/value strict); static message + kwargs, never
  f-strings; `logger.exception` inside `except`; log level discipline;
  stdlib-logging interception for libraries via loguru.
- Section outline: `consolidation-claude/debugging-guides.md` "logging.md".
- cpp source: `cpp-debugging-principles/logging.md`.
- factors2 anti-patterns must-cite: 35+ `print()` calls in
  `src/pyfactors/` (`pipeline/stages.py:307-329`, `backtest/server.py`);
  mixed-logger failure mode at `factors-claude/anti-patterns.md` entry L.

### `python-debugging-principles/static-analysis-and-runtime-checks.md`  (PYTHON_SPECIFIC)

- Scope: pyright / ruff CI gate; pytest with `-W error` + `python -X dev`;
  `PYTHONASYNCIODEBUG=1` for async work; `tracemalloc` for memory
  attribution; memray and py-spy for live processes; faulthandler;
  framework-specific runtime checks; free-threaded interpreter sidebar.
- Section outline: `consolidation-claude/debugging-guides.md`
  "static-analysis-and-runtime-checks.md".
- cpp source: `cpp-debugging-principles/sanitizers.md`. Drop the
  sanitizer / build-preset mental model; reframe as "static analysis +
  runtime warnings + profilers".
- Research: `cpp-claude/research/observability-and-debugging.md`.

### `agent/python-agent-context.md`  (PORT_WITH_ADAPTATION)

- Scope: the always-loaded rule sheet. Section ordering mirrors the
  cpp file. Length target ~450 lines (cpp file is ~400; Python adds
  typing-escape-hatches and import-discipline sections).
- Section outline: `consolidation-claude/agent-context.md` is the
  spec; follow its section-by-section breakdown.
- cpp source: `/home/msi/repos/cpp-guidelines/agent/cpp-agent-context.md`.
- Must include: the typing-escape-hatch heuristics (when `Any`, `cast`,
  `# type: ignore` are correct, not just acceptable). The import-DAG
  discipline (no parent-relative imports; cycle bans). The
  negative-documentation rule.
- Cross-reference every topical guide once; do not duplicate the topical
  guide content.

### `agent/python-agent-examples.md`  (PORT_WITH_ADAPTATION)

- Scope: the on-demand companion. One good / bad pair per agent-context
  section.
- Section outline: `consolidation-claude/agent-context.md` "examples"
  section.
- cpp source: `/home/msi/repos/cpp-guidelines/agent/cpp-agent-examples.md`.
- factors2 anti-patterns are the best source of "bad" examples; cite
  with `# from factors2:path/to/file.py:line` comments to make the
  origin traceable.

### Top-level `README.md` and per-folder `README.md` files

- Scope: each lists the guides in its folder with a one-sentence scope
  and the read order. Mirror the cpp `README.md` shape.
- The top-level `README.md` mirrors `/home/msi/repos/cpp-guidelines/README.md`
  (table of four sister folders), with the agent-context note from the
  cpp version.
- The four sub-folder `README.md` files mirror their cpp equivalents.

## Cross-cutting themes (repeat across guides)

Source: `consolidation-claude/cross-cutting-themes.md`.

1. **Domain ownership.** Each domain owns its types, errors, and
   invariants. Cross-domain communication flows through adapters at the
   edge; domain types never cross domain boundaries unchanged.
2. **DAG-shaped imports and call graphs.** Imports flow one way.
   Cycles are bugs. Enforce with `import-linter` + ruff `TID252`.
3. **Push effects to the edge.** Functional core, imperative shell.
   I/O, time, randomness, and threads live at integration boundaries.
4. **Types carry the proof, with gradual-typing caveats.** A typed
   value is, by construction, a valid one - up to the boundary where
   it was parsed. Document the escape hatches explicitly.
5. **Invariants and idempotency.** Operations that may be replayed
   produce the same outcome. State mutations commit at the end of an
   operation, not in the middle.
6. **Root-cause over symptom.** When debugging, trace the bad value
   backward to its source; do not patch the symptom.

Every guide should reference at least one cross-cutting theme by name
and quote no more than one sentence from the canonical theme statement.

## Material to NOT port from cpp

Combined from D16 and the codex consolidation's "Material to Avoid
Porting" section. Apply across all guides.

Files to drop:

- `cpp-testing-principles/qt-gui.md` unless the target codebase has Qt.

Content NOT to port inside surviving guides:

- C++-style compile-time correctness as a literal guarantee.
- `std::expected`, LEAF, `BOOST_LEAF_ASSIGN` mechanics. Cite the
  *principle* of boundary translation; do not port the syntax.
- Template, concept, `constexpr`, dependent-false, type-erasure terminology.
- `std::variant` visitor mechanics as a model for Python dispatch
  (Python uses `match` + `assert_never`).
- RAII / destructor-cleanup as the explanatory model. Use context
  managers, fixtures, transactions, and `try / finally`.
- Friend-based test probes.
- Catch2 / CTest / `REQUIRE` / `CHECK` / `RUN_SERIAL` mechanics.
- ASan / TSan / MSan and build-preset framing.
- Container choice based on allocation, cache locality, iterator
  invalidation.
- "No allocations on the hot path" as broad Python advice.
- Header / translation unit / move semantics / undefined behavior
  framing.
- `std::function` / `function_ref` / inplace_function callable cost
  framing.

## Writing order

The cpp pass shipped these as siblings, but the writing pass benefits
from a dependency order so later guides can cite earlier vocabulary.
Each batch can be written in parallel; cross-batch dependencies are
serial.

**Batch 1 (vocabulary):**

1. `python-projects-and-tooling/project-layout.md`
2. `python-projects-and-tooling/tooling-baseline.md`
3. `python-projects-and-tooling/python-version-and-typing-stack.md`

**Batch 2 (design foundations):**

4. `python-design-principles/architecture.md`
5. `python-design-principles/types-and-correctness.md`
6. `python-design-principles/comments-and-docstrings.md`
7. `python-design-principles/declarative-style.md`
8. `python-design-principles/error-handling.md`

**Batch 3 (design specifics):**

9. `python-design-principles/functional-programming.md`
10. `python-design-principles/generics-and-protocols.md`
11. `python-design-principles/decorators-and-metaprogramming.md`
12. `python-design-principles/invariants-and-rollback.md`
13. `python-design-principles/state-machines.md`
14. `python-design-principles/pipelines.md`
15. `python-design-principles/cross-cutting-services.md`

**Batch 4 (runtime and performance):**

16. `python-design-principles/runtime-and-concurrency.md`
17. `python-design-principles/performance.md`
18. `python-design-principles/data-pipeline-and-dataframes.md` (skip if
    target codebase is not dataframe-heavy)

**Batch 5 (testing):**

19. `python-testing-principles/philosophy.md`
20. `python-testing-principles/pytest-conventions.md`
21. `python-testing-principles/test-patterns.md`
22. `python-testing-principles/test-helpers.md`
23. `python-testing-principles/error-path-testing.md`
24. `python-testing-principles/condition-based-waiting.md`
25. `python-testing-principles/approval-tests.md`

**Batch 6 (debugging):**

26. `python-debugging-principles/root-cause-tracing.md`
27. `python-debugging-principles/defense-in-depth.md`
28. `python-debugging-principles/logging.md`
29. `python-debugging-principles/static-analysis-and-runtime-checks.md`

**Batch 7 (agent + READMEs):**

30. `agent/python-agent-context.md`
31. `agent/python-agent-examples.md`
32. Top-level `README.md` + four folder `README.md` files

Each batch is internally parallelisable. Batch N+1 may start when
Batch N has finished (so later guides can cite the vocabulary the
earlier guides establish).

## Verification checklist (after writing)

For each guide produced, verify:

- All `consolidation-claude/divergences-and-decisions.md` policy
  decisions for the guide's topic are reflected in the prose, not
  contradicted.
- Every factors2 citation in this plan appears in the guide (or has an
  explicit reason for omission in a PR comment).
- Cross-cutting themes are referenced by name where they apply.
- The "Material to NOT port" list is observed.
- The guide reads as idiomatic Python, not as C++ with Python syntax
  pasted over. Apply the same rubric the cpp guides use ("does this
  read like it was written by someone whose native language is Python?").

## Follow-up work after the first writing pass

Deferred from this pass:

- `agent/skills/repo-specific-python-guidelines/` skill packaging
  (Q16): the on-disk skill that walks the topical guides and writes
  per-repo mapping docs. Defer until the topical guides have shipped to
  a real codebase.
- PR review checklist (Q22): one bullet per guide section; write it
  once the guides exist.
- Web UI testing guide (conditional): only if a target codebase uses
  Playwright or Selenium.
- Qt testing guide (conditional, currently dropped): only if a target
  codebase owns a Qt surface.
- Periodic re-survey of type checker landscape (pyright vs pyrefly vs
  ty); the recommendation is correct as of 2026-05 but the field is
  moving.
