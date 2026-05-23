# Research Consolidation for Python Guidelines

## Executive Synthesis

The research points toward Python guidelines that preserve the C++ guide's core engineering discipline while using Python-native mechanisms. The durable guides should emphasize domain vocabulary, explicit package boundaries, typed contracts at stable edges, runtime validation at untrusted boundaries, deterministic tests, and tool-backed feedback through uv, Ruff, pytest, and a chosen type checker. They should not promise C++-style compile-time guarantees, zero-cost abstractions, RAII, template machinery, or sanitizer workflows.

The existing durable docs are sparse. `README.md` is empty. `MONOREPO.md` establishes the current local philosophy: strict separation between infrastructure, source code, workspace tooling, and deployable units. Future guides should build from that boundary-first view instead of copying the C++ repository structure mechanically.

The Codex and Claude research artifacts mostly agree on principles. The important differences are policy strength. Claude artifacts often recommend firm defaults such as strict Pyright, contextvars-backed cross-cutting services, anyio for structured concurrency, and frozen slotted dataclasses as the default data carrier. Codex artifacts are more conservative: use strictness by directory, pick one formatter and primary type checker, validate at boundaries, and choose dataclasses, attrs, Pydantic, Pandera, or dynamic Python based on the layer. Treat those differences as explicit policy decisions for the final writing phase.

## Proposed Durable Guide Set

### Project Layout and Tooling

This guide should explain the default repository shape for modern Python projects and monorepos. Cover `src/` layout, workspace packages, child `pyproject.toml` files, committed lockfiles, dependency groups, `.python-version`, and a small command surface built around `uv sync` and `uv run`. It should also cover Ruff as lint and format when selected, CI using the same commands developers run locally, benchmark separation from correctness tests, and metadata checks for files named by packaging config. Use `MONOREPO.md`, `factors2-codex/tooling-tests-docs.md`, and `research-codex/modern-python-tooling-and-typing.md`.

### Architecture and Boundaries

This guide should define how Python code keeps domain logic separate from framework, vendor, orchestration, and infrastructure concerns. Cover domain vocabulary, adapters, forward-only imports, functional core and imperative shell, explicit data flow, orchestration at the edge, and explicit pipeline stages before introducing graph frameworks. It should connect monorepo package boundaries to application boundaries: core packages expose typed domain operations; Dagster, dbt, HTTP clients, filesystems, databases, and CLIs live at integration or application edges. Use `design-codex/generalizable-cpp-design.md`, `factors2-codex/architecture-and-patterns.md`, `cpp-claude/00-summary.md`, and `cpp-claude/design/architecture.md`.

### Types, Data Models, and Validation

This guide should present typing as a way to name contracts and catch mistakes early, not as runtime enforcement. Cover modern annotation syntax, `Protocol`, `Enum`, `Literal`, `TypedDict`, dataclasses, attrs, Pydantic, Pandera, `NewType`, `assert_never`, and selective generics. It should separate static typing, boundary parsing, schema validation, and domain invariant validation. For dataframe-heavy code, recommend typed configuration and scalar boundaries, Pandera or similar schema validation at system edges, and tests for transformations rather than elaborate column-level type aliases. Use `research-codex/modern-python-tooling-and-typing.md`, `design-codex/generalizable-cpp-design.md`, `factors2-codex/architecture-and-patterns.md`, `cpp-claude/design/compile-time-correctness.md`, and `cpp-claude/research/strong-types-and-data-classes.md`.

### Function and Dataflow Design

This guide should teach readable Python function design: work on simpler inputs, stage derived values before decisions, name predicates, keep core logic pure where practical, materialize iterators intentionally, and use domain-shaped return values when a step produces multiple outputs. It should include Python collection tools, generator expressions, `itertools`, vectorized dataframe APIs, and explicit pipeline sequencing. It should also warn against helper fragmentation, hidden mutation of nested configuration, and generic graph abstractions when a visible sequence is clearer. Use `design-codex/generalizable-cpp-design.md`, `factors2-codex/architecture-and-patterns.md`, and `cpp-claude/design/declarative-style.md`.

### Configuration, Strategies, and Extension Points

This guide should cover user-selectable behavior as declarative configuration plus small implementation objects. Use the factors2 pattern: Pydantic models for configuration crossing YAML, API, CLI, or storage boundaries; discriminated unions for closed strategy sets; explicit registries when the registry is the supported public surface; and domain verbs on concrete strategies. It should also cover dynamic dispatch as a deliberate adapter boundary, plugin registries, decorators, protocols, and when runtime discovery is better than static registration. Use `factors2-codex/architecture-and-patterns.md`, `research-codex/modern-python-tooling-and-typing.md`, `cpp-claude/design/templates.md`, and `cpp-claude/design/preprocessor-macros.md`.

### Error Handling, Invariants, and Reliability

This guide should define Python-native failure policy. Cover domain-owned exceptions or result objects, structured error attributes, top-level translation at HTTP handlers, queue consumers, CLIs, jobs, and framework callbacks, and consistent failure vocabulary inside a domain. It should also cover commit-at-end state mutation, context managers, transactions, temporary files plus atomic rename, idempotent handlers, and aggregate-boundary invariant checks. Use `design-codex/generalizable-cpp-design.md`, `testing-debugging-codex/generalizable-cpp-testing-debugging.md`, `factors2-codex/architecture-and-patterns.md`, `cpp-claude/design/error-handling.md`, and `cpp-claude/design/invariants.md`.

### Testing with Pytest

This guide should establish behavior-first pytest guidance. Cover reading interfaces before implementations, component versus integration scope, deterministic test data, parametrization with useful IDs, focused fixtures, domain-specific factories, failure and cleanup tests, dataframe output assertions, approval tests for generated SQL or structured text, and test isolation for process-global state. Include async and external readiness patterns based on observable conditions and timeouts. Use `testing-debugging-codex/generalizable-cpp-testing-debugging.md`, `factors2-codex/tooling-tests-docs.md`, and `cpp-claude/testing/*.md`.

### Debugging and Observability

This guide should teach the debugging loop: reproduce, trace backward to the bad value's origin, state one hypothesis, fix the origin, and verify. Cover traceback reading, focused pytest commands, flaky-test isolation, logging before likely failures, `caplog`, process-global cleanup, environment guards, and structured observability at boundaries. Translate the sanitizer idea into reproducible Python diagnostic commands: type checker, Ruff, pytest warning policy, `python -X dev`, `PYTHONASYNCIODEBUG`, `tracemalloc`, memray, py-spy, and framework-specific checks. Use `testing-debugging-codex/generalizable-cpp-testing-debugging.md`, `cpp-claude/debugging/*.md`, and `cpp-claude/research/observability-and-debugging.md`.

### Concurrency, Async, and Runtime Boundaries

This guide should treat threads, event loops, worker pools, multiprocessing, and framework-managed lifecycles as runtime concerns at the edge of domain logic. Cover single-owner mutable state, queues and task groups, explicit marshalling, cancellation discipline, deferred work for re-entrance hazards, condition-based synchronization in tests, and documentation for classes tied to a loop or thread. Use `design-codex/generalizable-cpp-design.md`, `cpp-claude/design/runtime.md`, `cpp-claude/design/pipelines.md`, and `cpp-claude/research/structured-concurrency.md`.

### Performance

This guide should reframe performance around Python's actual cost model. Cover requirements before optimization, profiling before changes, vectorized libraries, database-side work, batching, caching, avoiding needless materialization, bounded logging, and import or serialization costs. It should explain that type annotations do not speed CPython by themselves and that C++ hot-path allocation and inlining advice does not port directly. Use `design-codex/generalizable-cpp-design.md`, `factors2-codex/architecture-and-patterns.md`, and `cpp-claude/design/performance.md`.

### Comments and Documentation

This guide should codify present-tense comments and durable docs. Cover comments for non-obvious domain rules, ordering, concurrency preconditions, retry behavior, and performance constraints. Prohibit comments that narrate old behavior, moved responsibilities, obvious assignments, or absent behavior. For docs, recommend ADRs for decisions with tradeoffs, domain conventions near the code that depends on them, command references that call stable project entry points, and avoidance of starter-template durable docs. Use `design-codex/generalizable-cpp-design.md`, `factors2-codex/tooling-tests-docs.md`, `cpp-claude/design/comments.md`, and the project `AGENTS.md` instructions.

## Cross-Cutting Principles to Repeat

- Use practitioner vocabulary for core concepts, tests, fixtures, and errors.
- Keep domain code independent from vendor SDKs, frameworks, orchestration tools, and process-global runtime state.
- Parse untrusted input once at the boundary, then pass refined domain values inward.
- Put invariants on the object or aggregate that has enough context to validate them.
- Make dependency arrows visible and forward-only; avoid circular imports and hidden output channels.
- Prefer explicit pipelines and named stages before introducing generic graph or plugin machinery.
- Use static typing for stable contracts and runtime validation for values that arrive from outside the process or from dynamic libraries.
- Keep dynamic Python where it improves clarity, especially in dataframe-heavy, reflective, or plugin-heavy code, but isolate dynamic behavior behind typed or validated boundaries.
- Make tests deterministic by controlling time, random seeds, generated IDs, paths, environment, logging state, registries, and ordering.
- Verify failure paths directly, including rollback, cleanup, rejection, and post-call state.
- Choose tools deliberately and keep local commands, CI, docs, and configuration aligned.
- Write comments and docs in present tense about current behavior, current contracts, and current rationale.

## Decisions and Tradeoffs Requiring Explicit Policy

- Type checker policy: choose Pyright, Pyrefly, mypy, or another checker as the primary gate. Decide whether strict mode is global, directory-scoped, or aspirational.
- Formatter policy: decide whether Ruff format is the formatter. If yes, avoid maintaining Black as a parallel formatting authority.
- Ruff rule policy: decide whether the guide gives a baseline rule set, a strict rule set, or principles for selection only.
- Python version target: decide whether durable guidance targets Python 3.12+, 3.13+, or 3.14+. This affects PEP 695 syntax, annotation quoting, and PEP 649 guidance.
- Data carrier default: decide whether to recommend dataclasses by default, frozen slotted dataclasses by default, attrs for validators, or Pydantic for boundary objects only.
- Boundary validation stack: decide how explicitly to standardize Pydantic, attrs, Pandera, or project-specific validators.
- Cross-cutting services: decide whether to recommend contextvars-backed services as a default pattern or present them only for projects with async, tests, or request-local state.
- Concurrency library: decide whether to recommend anyio as the default structured-concurrency layer or keep the guide centered on stdlib asyncio.
- Failure style: decide whether exceptions are the default inside domains, whether result objects get a first-class pattern, and how boundary translation should look.
- Approval tests: decide whether to standardize a pytest snapshot or approval plugin, use plain expected files, or describe the pattern without naming a library.
- Dataframe guidance: decide how much the general Python guides should cover pandas and Pandera versus putting dataframe-specific guidance in a separate data-engineering guide.
- Dynamic registries: decide when explicit registries are required public surfaces and when runtime plugin discovery is acceptable.
- Monorepo depth: decide whether the final guide set should mirror C++ design, testing, and debugging repos, or consolidate into fewer Python-focused guides.

## Material to Avoid Porting from C++

- Compile-time correctness as a literal guarantee.
- `std::expected`, LEAF, `BOOST_LEAF_ASSIGN`, and C++ result machinery as default Python patterns.
- Template, concept, `constexpr`, dependent-false, and type-erasure terminology.
- `std::variant` visitor mechanics as a direct model for Python dispatch.
- RAII and destructor-based cleanup as the explanatory model; use context managers, fixtures, transactions, and `try`/`finally`.
- Friend-based test probes and private member access patterns as a default testing strategy.
- Catch2, CTest, `REQUIRE`, `CHECK`, tags, and `RUN_SERIAL` mechanics.
- Qt GUI testing guidance unless a future Python guide targets Qt for Python specifically.
- ASan, TSan, MSan, build preset, suppression-file, and C++ sanitizer workflows as general Python defaults.
- Container-choice advice based on C++ allocation, cache locality, and iterator invalidation.
- "No allocations on the hot path" as broad Python advice.
- Header, translation unit, include, macro expansion, move semantics, undefined behavior, and preprocessor framing.
- C++ callback storage cost advice around `std::function`, `function_ref`, and inplace functions.

## Gaps and Follow-Up Research Needed

- Decide the Python version baseline and update modern typing guidance against that baseline.
- Compare Pyright, Pyrefly, and mypy in the repo's intended project types before setting a default.
- Validate a Ruff baseline against real projects and check conflict with the formatter.
- Research uv workspaces for the exact syntax and tradeoffs that should appear in durable monorepo guidance.
- Decide whether dependency groups should be split by `test`, `lint`, `typing`, and `docs`, or kept as one `dev` group for small projects.
- Research import-boundary enforcement tools such as import-linter and pydeps before recommending them.
- Decide whether anyio belongs in baseline guidance or only in async-heavy projects.
- Research snapshot and approval-test libraries in current pytest practice before naming one.
- Add a focused data-engineering slice for pandas, Polars, Pandera, SQL generation, dbt, and Dagster if the final guidelines target analytics codebases.
- Research security-sensitive typing and validation topics such as `LiteralString`, SQL value parameters, secrets handling, and unsafe `eval`.
- Review packaging metadata checks that can prevent missing README, version drift, and tool-scope drift in CI.
- Decide whether agent-context docs are in scope for the first durable guide set.

## Artifact Pointers

- `work-in-progress/exploration/design-codex/generalizable-cpp-design.md`: broad C++ design principles translated into Python guide candidates.
- `work-in-progress/exploration/testing-debugging-codex/generalizable-cpp-testing-debugging.md`: pytest, deterministic tests, approval tests, debugging loop, logging, and validation layers.
- `work-in-progress/exploration/factors2-codex/architecture-and-patterns.md`: concrete Python architecture patterns from factors2, including explicit pipelines, Pydantic strategy models, Pandera dataframe validation, and orchestration boundaries.
- `work-in-progress/exploration/factors2-codex/tooling-tests-docs.md`: uv, Ruff, pytest, CI, Makefile, documentation, ADR, benchmark, and operational-script observations from factors2.
- `work-in-progress/exploration/research-codex/modern-python-tooling-and-typing.md`: current Python typing and tooling research, including Python 3.14 annotations, uv, Ruff, Pyright, Pyrefly, PEP 723, and PEP 735.
- `work-in-progress/exploration/cpp-claude/00-summary.md`: Claude's high-level proposed guide structure and opinionated defaults.
- `work-in-progress/exploration/cpp-claude/design/*.md`: per-topic C++ design translation notes.
- `work-in-progress/exploration/cpp-claude/testing/*.md`: per-topic C++ testing translation notes.
- `work-in-progress/exploration/cpp-claude/debugging/*.md`: per-topic C++ debugging translation notes.
- `work-in-progress/exploration/cpp-claude/research/*.md`: supporting research on type checkers, Ruff rules, strong types, generics, exhaustiveness, observability, and structured concurrency.
- `MONOREPO.md`: durable local guidance on source, infrastructure, workspace, Docker, Dagster, and dbt boundaries.
- `README.md`: currently empty; future root guidance should not assume existing content there.
