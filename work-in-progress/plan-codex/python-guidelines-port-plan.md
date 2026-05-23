# Python Guidelines Port Plan

This plan turns the C++ guideline research into a Python guideline writing
program. It points writers to the research artifacts and names the durable
guides to produce. It does not carry the full guide content.

## Inputs

- `work-in-progress/consolidation-codex/research-consolidation.md`
- `work-in-progress/exploration/design-codex/generalizable-cpp-design.md`
- `work-in-progress/exploration/testing-debugging-codex/generalizable-cpp-testing-debugging.md`
- `work-in-progress/exploration/factors2-codex/architecture-and-patterns.md`
- `work-in-progress/exploration/factors2-codex/tooling-tests-docs.md`
- `work-in-progress/exploration/research-codex/modern-python-tooling-and-typing.md`
- `work-in-progress/exploration/cpp-codex/00-summary.md`
- `work-in-progress/exploration/cpp-codex/design/*.md`
- `work-in-progress/exploration/cpp-codex/research/modern-python-tooling.md`
- `work-in-progress/exploration/cpp-claude/00-summary.md`
- `work-in-progress/exploration/cpp-claude/design/*.md`
- `work-in-progress/exploration/cpp-claude/testing/*.md`
- `work-in-progress/exploration/cpp-claude/debugging/*.md`
- `work-in-progress/exploration/cpp-claude/research/*.md`
- `MONOREPO.md`

Primary web sources checked during research:

- Python 3.14 docs: https://docs.python.org/3.14/
- Python typing concepts: https://typing.python.org/en/latest/spec/concepts.html
- PEP 649: https://peps.python.org/pep-0649/
- PEP 749: https://peps.python.org/pep-0749/
- uv workspaces: https://docs.astral.sh/uv/concepts/projects/workspaces/
- Ruff configuration: https://docs.astral.sh/ruff/configuration/
- Pyright docs: https://microsoft.github.io/pyright/
- Pyrefly configuration: https://pyrefly.org/en/docs/configuration/

## Recommended Durable Guide Set

### `python-projects/tooling-and-layout.md`

Write the project baseline guide first. Cover `src/` layout, uv projects and
workspaces, committed lockfiles, `.python-version`, dependency groups, child
`pyproject.toml` files, Ruff configuration, type checker configuration, CI
commands, benchmark separation, and packaging metadata checks.

Use:

- `MONOREPO.md`
- `factors2-codex/tooling-tests-docs.md`
- `research-codex/modern-python-tooling-and-typing.md`
- `cpp-claude/research/ruff-rule-sets.md`
- `cpp-claude/research/typecheckers-2026.md`

### `python-design/architecture-and-boundaries.md`

Cover domain vocabulary, adapters, forward-only imports, functional core and
imperative shell, orchestration at the edge, explicit data flow, and when to use
a real DAG tool. Emphasize that core packages expose domain operations, while
Dagster, dbt, HTTP clients, database sessions, filesystems, and CLIs live at
integration edges.

Use:

- `design-codex/generalizable-cpp-design.md`
- `factors2-codex/architecture-and-patterns.md`
- `cpp-claude/design/architecture.md`
- `cpp-claude/design/pipelines.md`

### `python-design/types-data-models-and-validation.md`

Cover typing as contract naming and early feedback, not runtime enforcement.
Include modern annotations, `Protocol`, `Enum`, `Literal`, `TypedDict`,
dataclasses, attrs, Pydantic, Pandera, `NewType`, selective generics, and
`assert_never`. Separate static types, boundary parsing, schema validation, and
domain invariant validation. Include dataframe guidance that keeps pandas code
dynamic where static typing adds ceremony without safety.

Use:

- `research-codex/modern-python-tooling-and-typing.md`
- `design-codex/generalizable-cpp-design.md`
- `factors2-codex/architecture-and-patterns.md`
- `cpp-claude/design/compile-time-correctness.md`
- `cpp-claude/research/strong-types-and-data-classes.md`
- `cpp-claude/research/pep695-and-generics.md`

### `python-design/functions-and-dataflow.md`

Cover declarative, readable Python functions: work on simpler inputs, stage
derived values before decisions, name predicates, keep core logic pure where
practical, return domain-shaped values, materialize iterators intentionally, and
use vectorized dataframe or SQL APIs where they are the native abstraction.
Warn against helper fragmentation, hidden mutation, and generic graph machinery
when a visible sequence is clearer.

Use:

- `design-codex/generalizable-cpp-design.md`
- `factors2-codex/architecture-and-patterns.md`
- `cpp-claude/design/declarative-style.md`
- `cpp-claude/design/functional-programming.md`
- `cpp-claude/design/performance.md`

### `python-design/configuration-strategies-and-extension-points.md`

Cover declarative configuration plus small implementation objects. Use the
factors2 pattern: Pydantic models for configuration crossing YAML, API, CLI, or
storage boundaries; discriminated unions for closed strategy sets; explicit
registries when the registry is the public surface; domain verbs on concrete
strategies; and dynamic dispatch only as a deliberate adapter boundary.

Use:

- `factors2-codex/architecture-and-patterns.md`
- `research-codex/modern-python-tooling-and-typing.md`
- `cpp-claude/design/templates.md`
- `cpp-claude/design/preprocessor-macros.md`
- `cpp-claude/research/exhaustive-matching.md`

### `python-design/error-handling-invariants-and-reliability.md`

Cover Python-native failure policy: domain exceptions or result objects,
structured error attributes, boundary translation, top-level containment,
commit-at-end mutation, context managers, transactions, atomic file writes,
idempotent handlers, aggregate-boundary invariant checks, and direct tests for
rollback and cleanup.

Use:

- `design-codex/generalizable-cpp-design.md`
- `testing-debugging-codex/generalizable-cpp-testing-debugging.md`
- `factors2-codex/architecture-and-patterns.md`
- `cpp-claude/design/error-handling.md`
- `cpp-claude/design/invariants.md`

### `python-testing/pytest-principles.md`

Cover behavior-first pytest guidance: read interfaces before implementations,
component versus integration scope, deterministic data, parametrization with
useful IDs, focused fixtures, domain-specific factories, failure-path tests,
cleanup assertions, dataframe output assertions, approval tests for generated
SQL or structured text, and isolation for process-global state.

Use:

- `testing-debugging-codex/generalizable-cpp-testing-debugging.md`
- `factors2-codex/tooling-tests-docs.md`
- `cpp-claude/testing/philosophy.md`
- `cpp-claude/testing/test-patterns.md`
- `cpp-claude/testing/test-helpers.md`
- `cpp-claude/testing/error-path-testing.md`
- `cpp-claude/testing/approval-tests.md`

### `python-debugging/debugging-and-observability.md`

Cover the debugging loop: reproduce, trace backward to the bad value's origin,
state one hypothesis, fix the origin, and verify. Include traceback reading,
focused pytest commands, flaky-test isolation, logging before likely failures,
`caplog`, environment guards, and structured observability at boundaries.
Translate C++ sanitizer discipline into reproducible Python diagnostics: type
checker, Ruff, pytest warnings policy, `python -X dev`, asyncio debug mode,
`tracemalloc`, memray, py-spy, and framework-specific checks.

Use:

- `testing-debugging-codex/generalizable-cpp-testing-debugging.md`
- `cpp-claude/debugging/root-cause-tracing.md`
- `cpp-claude/debugging/defense-in-depth.md`
- `cpp-claude/debugging/logging.md`
- `cpp-claude/debugging/sanitizers.md`
- `cpp-claude/research/observability-and-debugging.md`

### `python-runtime/concurrency-and-runtime-boundaries.md`

Cover threads, event loops, task groups, worker pools, multiprocessing, and
framework-managed lifecycles as edge concerns. Include single-owner mutable
state, queues, explicit marshalling, cancellation discipline, deferred work for
re-entrance hazards, condition-based synchronization in tests, and documentation
for classes tied to a loop or thread.

Use:

- `design-codex/generalizable-cpp-design.md`
- `testing-debugging-codex/generalizable-cpp-testing-debugging.md`
- `cpp-claude/design/runtime.md`
- `cpp-claude/design/pipelines.md`
- `cpp-claude/research/structured-concurrency.md`

### `python-design/performance.md`

Cover Python's real cost model: requirements before optimization, profiling
before changes, vectorized libraries, database-side work, batching, caching,
avoiding needless materialization, bounded logging, import cost, serialization
cost, and benchmark placement. State that type annotations help readers and
tools but do not speed CPython execution by themselves.

Use:

- `design-codex/generalizable-cpp-design.md`
- `factors2-codex/architecture-and-patterns.md`
- `factors2-codex/tooling-tests-docs.md`
- `cpp-claude/design/performance.md`

### `python-style/comments-and-documentation.md`

Cover present-tense comments and durable docs. Comments should explain
non-obvious domain rules, ordering, concurrency preconditions, retry behavior,
and performance constraints. Docs should use ADRs for tradeoff-heavy decisions,
put domain conventions near the code that depends on them, call stable project
entry points, and avoid starter-template content.

Use:

- `design-codex/generalizable-cpp-design.md`
- `factors2-codex/tooling-tests-docs.md`
- `cpp-claude/design/comments.md`
- project `AGENTS.md` instructions

## Cross-Guide Principles

- Use practitioner vocabulary for core concepts, tests, fixtures, and errors.
- Keep domain code independent from vendor SDKs, frameworks, orchestration
  tools, and process-global runtime state.
- Parse untrusted input at the boundary, then pass refined values inward.
- Put invariants on the object or aggregate that has enough context to validate
  them.
- Make dependency arrows visible and forward-only.
- Prefer explicit pipelines and named stages before generic graph or plugin
  machinery.
- Use static typing for stable contracts and runtime validation for external or
  dynamic data.
- Keep dynamic Python where it improves clarity, especially in dataframe,
  reflective, or plugin-heavy code, but isolate it behind typed or validated
  boundaries.
- Make tests deterministic by controlling time, random seeds, generated IDs,
  paths, environment, logging state, registries, and ordering.
- Verify failure paths directly, including rollback, cleanup, rejection, and
  post-call state.
- Keep local commands, CI, docs, and tool configuration aligned.
- Write comments and docs in present tense about current behavior and current
  rationale.

## Policy Decisions Before Writing

Resolve these before drafting final durable docs:

- Python baseline: choose Python 3.12+, 3.13+, or 3.14+.
- Primary type checker: choose Pyright, Pyrefly, mypy, or a documented
  combination.
- Strictness policy: choose global strict, directory-scoped strict, or staged
  adoption.
- Formatter policy: decide whether Ruff format replaces Black.
- Ruff baseline: choose a recommended rule set or write only rule-selection
  principles.
- Data carrier default: choose dataclasses, frozen slotted dataclasses, attrs,
  or Pydantic by layer.
- Boundary validation stack: decide how strongly to standardize Pydantic,
  attrs, Pandera, or local validators.
- Cross-cutting services: decide whether contextvars-backed providers are a
  default pattern or an advanced pattern.
- Concurrency baseline: choose stdlib `asyncio` first or recommend anyio for
  structured concurrency.
- Failure style: choose exceptions as the domain default, result objects as a
  first-class pattern, or both with clear boundaries.
- Approval test library: choose a plugin, expected files, or a library-neutral
  pattern.
- Data engineering scope: decide whether pandas, Polars, Pandera, SQL, dbt, and
  Dagster deserve a separate guide.
- Agent docs scope: decide whether Python agent-context docs belong in the first
  writing batch.

## Material to Drop or Reframe

- Do not promise C++-style compile-time correctness.
- Do not port `std::expected`, LEAF, templates, concepts, `constexpr`,
  `std::variant` visitor mechanics, or type-erasure terminology directly.
- Do not use RAII or destructor cleanup as the Python explanatory model.
- Do not port friend-based test probes, Catch2, CTest, Qt, or C++ sanitizer
  mechanics.
- Do not carry over C++ container, allocation, cache-locality, inlining, move
  semantics, undefined behavior, header, translation unit, or preprocessor
  framing.
- Reframe hot-path advice around measured Python costs: I/O, dataframe work,
  database queries, serialization, imports, batching, and unnecessary
  materialization.

## Suggested Writing Order

1. `python-projects/tooling-and-layout.md`
2. `python-design/architecture-and-boundaries.md`
3. `python-design/types-data-models-and-validation.md`
4. `python-design/functions-and-dataflow.md`
5. `python-testing/pytest-principles.md`
6. `python-design/error-handling-invariants-and-reliability.md`
7. `python-debugging/debugging-and-observability.md`
8. `python-design/configuration-strategies-and-extension-points.md`
9. `python-runtime/concurrency-and-runtime-boundaries.md`
10. `python-design/performance.md`
11. `python-style/comments-and-documentation.md`

This order establishes project and architecture vocabulary before writing the
more specialized guides.

## Follow-Up Research

- Validate Pyright, Pyrefly, and mypy against target project shapes before
  setting the default.
- Test a Ruff baseline against factors2 or another representative repository.
- Research import-boundary enforcement tools before recommending one.
- Decide whether anyio belongs in baseline guidance.
- Research current pytest snapshot and approval-test libraries before naming
  one.
- Add a focused data-engineering research slice if the guidelines will target
  analytics codebases explicitly.
- Research security-sensitive validation topics: `LiteralString`, SQL
  parameters, secrets handling, and unsafe `eval`.
- Add CI checks for packaging metadata drift, Python version drift, missing
  README files, and tool-scope drift if the tooling guide recommends them.
