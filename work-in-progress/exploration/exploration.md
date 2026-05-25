# Python Guidelines Exploration Consolidation

## Purpose And Baseline

This artifact consolidates the Phase 2 inputs for the Python guideline port. It
combines the C++ source disposition map, modern Python tooling research, and the
factors2 exploration into one writing brief for the first durable guide pass.

Source coverage:

- `work-in-progress/exploration/cpp-source-python-port.md` for the C++ guide
  disposition map and the Python replacement mechanisms.
- `work-in-progress/exploration/codex-research/` for Python tooling, typing,
  architecture, testing, debugging, and factors2-oriented research.
- `work-in-progress/exploration/factors-claude/` for concrete factors2
  patterns, anti-patterns, tooling, and boundary examples.

The governing baseline is Pythonic rather than mechanically parallel to C++.
The Python guides should preserve the transferable engineering pressure:
domain ownership, forward-only dependencies, declarative dataflow, explicit
invariants, idempotency, deterministic tests, root-cause debugging, and
tool-backed feedback. They should replace C++ mechanisms with Python mechanisms:
type checkers, runtime validation, exceptions, context managers, pytest, Ruff,
uv, async task groups, structured logging, warnings, profilers, and runtime
diagnostics.

The durable guides should use this consolidation as source material, not as a
linked dependency. Stable guidance belongs in the guide text. Temporary research
paths stay out of durable documentation.

## Modern Python Baseline

### Python Version

- Use Python 3.13 as the first-pass baseline and Python 3.14 as the forward
  target.
- Prefer modern syntax for Python 3.12 and newer code: built-in collection
  generics, `str | None`, `type Alias = ...`, PEP 695 generic functions and
  classes, and PEP 696 defaults only where they simplify a real public API.
- For Python 3.14-only code, avoid quoted annotations when quotes exist only to
  dodge eager annotation evaluation. Runtime annotation consumers should use
  supported inspection APIs.
- Keep compatibility guidance explicit for libraries that support older Python
  versions.

### Package And Command Workflow

- Use `pyproject.toml` as the project metadata and tool configuration surface.
- Use `.python-version` as the interpreter selector when the project relies on
  a specific baseline.
- Use uv for environment creation, dependency updates, lockfile maintenance,
  and command execution.
- Commit `uv.lock` for applications, data pipelines, operational tools, and
  other repositories where reproducibility matters.
- Put development-only dependencies in PEP 735 dependency groups such as
  `test`, `lint`, `typing`, and `docs`.
- Use PEP 723 inline metadata for durable single-file scripts with their own
  dependencies.
- Provide one command surface through `uv run` and a thin task facade when the
  repository benefits from short commands.

### Ruff

- Use Ruff for linting, import sorting, formatting, modernization, and common
  bug checks.
- Choose one formatter per repository. When Ruff format is chosen, do not keep
  Black as a parallel formatting policy.
- Treat Ruff as a fast syntax, import, style, and simple-defect tool. It is not
  a type checker.
- Select rules because they prevent real defects or preserve maintainability.
  Useful first-pass families include pyflakes, pycodestyle, bugbear, import
  sorting, naming, quotes, pyupgrade, comprehensions, unused arguments, numpy,
  pandas, performance, selected pylint rules, no production `print`, security,
  Ruff-specific checks, and simplification checks.
- Keep local commands, CI scopes, and configured includes aligned. Benchmarks,
  examples, generated code, and tests should have explicit inclusion or
  exclusion policy.

### Type Checker Policy

- Use Pyright as the conservative default because it has a strong standards and
  IDE story.
- Treat Pyrefly as a credible fast alternative for large repositories when its
  diagnostics and conformance fit the project.
- Treat mypy as an existing-project or legacy fallback, and as a useful companion
  when a project already relies on mypy-specific plugins.
- Pick one authoritative type checker policy per repository. Running two
  checkers can be useful, but the guide should require a reason and a clear
  response policy for disagreements.
- Start with standard mode when adopting typing in an existing codebase. Move
  stable application and library packages toward strict checking by package or
  directory.
- Keep developer-specific virtual environment paths out of committed type
  checker configuration.

### Pytest And CI

- Use pytest as the test runner.
- Put correctness tests under `tests/`, with pytest configured to import from
  `src/` deliberately.
- Keep long-running benchmarks under `benchmarks/` and run them explicitly.
- Use pytest marks, node IDs, `-k`, and direct file paths for focused loops.
- Use CI commands that match developer commands. If docs recommend
  `uv run pytest`, CI should run the same command shape.
- Add explicit CI checks for formatting, linting, typing, tests, lockfile
  freshness, documentation drift where applicable, and project metadata sanity.

### Documentation Drift Checks

- Check that package metadata names files that exist, especially `readme`,
  license files, Python version files, and package roots.
- Keep starter-template docs out of project-owned documentation.
- Prefer runnable setup commands and scripts over prose-only transcripts of
  setup work.
- Use ADRs for decisions with meaningful tradeoffs, especially operational
  scheduling, recoverability, data-source behavior, and tool policy.
- Put domain conventions near the code that depends on them: dataframe schemas,
  ordering assumptions, units, date conventions, and fill behavior.

## Architecture And Boundaries

### Package Layers

The recommended Python architecture mirrors the C++ dependency intent through
packages, modules, imports, and adapters:

- Domain packages own vocabulary, types, errors, invariants, and public
  contracts.
- Adapter packages translate vendor SDKs, wire formats, database rows, CLI
  inputs, queue messages, and UI inputs into domain values.
- Application and orchestration packages wire domains together and own runtime
  concerns such as event loops, threads, schedulers, logging configuration,
  process pools, and framework entry points.
- Dependency arrows should form a DAG. Import-linter, pydeps, and Ruff import
  rules can enforce or visualize the contract.

For monorepos, multiple top-level packages under `src/` are reasonable when
they name distinct layers such as core library, ingestion, orchestration,
dashboards, and SQL transforms. Avoid top-level buckets named `common`,
`shared`, or `utils` unless the package owns a real, stable domain.

### Adapters And Functional Core

- Keep integration code in adapters that translate foreign data into domain
  dataclasses, Pydantic models, typed dictionaries, attrs classes, enums, or
  dataframe schemas.
- Keep the core free of file I/O, network calls, database sessions, environment
  reads, thread ownership, event-loop ownership, and framework globals.
- Let framework entry points be the imperative shell. Move business decisions
  behind plain functions or service objects.
- Pass required data explicitly and return produced data explicitly. Mutating
  methods are fine when the receiver owns the state and the method name makes
  the mutation clear.

### Explicit Pipelines

Python code should keep pipeline order visible until graph behavior becomes a
real domain concept. factors2 demonstrates this with a direct sequence from
universe fetch through indicators, ranking, signals, weights, orders, trades,
and simulation results.

Use explicit functions, methods, callbacks, queues, or async functions for
in-process pipelines. Use Dagster, dbt, dlt, or a similar tool when the graph
has operational consequences: materialization, partitions, schedules,
recoverability, lineage, or warehouse schema management.

### Dagster, dbt, dlt, And Data Tools

- Use dlt for ingestion when incremental state, file metadata, write
  dispositions, or source resources matter.
- Use ibis or a query engine at messy ingest boundaries when column
  normalization and simple relational reshaping should stay declarative.
- Use dbt for warehouse transforms where SQL-first modeling, schema tests, and
  lineage are the natural contract.
- Use Dagster for durable asset orchestration, partitions, schedules, resource
  wiring, and asset selection.
- Keep Dagster asset bodies thin. Delegate domain work to library functions
  that do not import Dagster.
- Schedule durability concerns, not every pipeline. Irrecoverable vendor
  snapshots deserve schedules and alerts; replayable transforms can often be
  materialized on demand.

### Dataframe Boundaries

- Validate dataframes at system and function boundaries, not after every pandas
  expression.
- Use Pandera for dataframe shape, dtypes, nullability, and simple value
  constraints.
- Use named domain validators for cross-row and cross-field invariants that
  Pandera cannot express clearly.
- Keep pandas, NumPy, Polars, SQL, or ibis transformations readable and native
  to the tool. Avoid wrapping every expression in tiny abstractions.
- Prefer vectorized transformations and library-native query APIs over
  Python-level iterator pipelines for tabular workloads.

## Typing And Models

### Pydantic Unions

Pydantic v2 discriminated unions are the strongest observed Python replacement
for C++ variant-style configuration:

- Each variant carries a frozen `Literal` discriminator field.
- The package-level registry exposes the union type as the supported public
  configuration surface.
- The wrapper model uses `Annotated[Union, Field(discriminator="tag")]`.
- The concrete strategy owns a domain verb such as `compute_scores`,
  `compute_signals`, `rebalance`, or `compute`, not a vague generic method.

Explicit registries are acceptable when the allowed variants are a supported
contract. Use dynamic discovery only when openness is part of the domain.

### Pandera Schemas

- Use `pandera.DataFrameModel` for dataframe values that cross module or
  function boundaries.
- Annotate function signatures with the schema type where it helps readers and
  type checkers.
- Compose schema validation with named invariant checks. Each invariant should
  be testable in isolation and should report useful context such as rule,
  count, and offending row samples.

### Dataclasses, attrs, TypedDict, And NewType

- Use frozen, slotted, keyword-only dataclasses for internal value objects and
  passive aggregates made from already validated parts.
- Use mutable dataclasses when mutation is the domain operation and the owner
  protects the invariant.
- Use Pydantic at trust boundaries such as HTTP, CLI, config, JSON, and
  cross-process records.
- Use attrs when the project already depends on it or needs converters and
  validators without the full Pydantic boundary model.
- Use `TypedDict` for dictionary-shaped boundaries when a class would add
  ceremony without behavior.
- Use `NewType` for identity-only primitive distinctions. Use a dataclass,
  Pydantic model, or validated constructor when the value carries proof.

### Protocol And Structural Typing

- Prefer `Protocol` over inheritance when the caller depends on behavior rather
  than class identity.
- Use `Protocol` for plugin contracts, fakes, clocks, serializers, repositories,
  callbacks with behavior, and capability-style interfaces.
- Use `Callable` for small policy functions, predicates, ID factories, clocks,
  and short test probes.
- Use `ParamSpec`, `Concatenate`, and `functools.wraps` for decorators that
  preserve or intentionally adjust call signatures.

### Any, cast, And type-ignore Policy

- `Any` is the deliberate dynamic escape hatch. It should appear at hostile or
  untyped boundaries and be normalized quickly.
- Use `object` when only universally valid operations are allowed.
- Use `cast` only after a nearby runtime check, trusted library contract, or
  documented invariant.
- Repeated casts around the same third-party library signal the need for a typed
  adapter or local stub.
- Keep type ignores narrow and tied to the exact checker complaint. A type
  ignore over project-owned model access usually means the model shape should
  change.
- Tests may use escape hatches more readily for deliberately invalid input, but
  reusable test helpers and fakes should still carry useful types.

### Type Alias Policy

- Prefer PEP 695 `type Alias = ...` in new Python 3.13+ code.
- Use legacy aliases when runtime reflection over the alias is central to the
  design, or when a dependency cannot handle `TypeAliasType` correctly.
- Avoid large aliases that obscure more than they clarify. A named data model is
  often clearer than a giant nested union or mapping type.

## Testing And Debugging

### Pytest-Native Patterns

- One behavior per test function is the default.
- Use `pytest.mark.parametrize` for table-shaped scenarios and provide `ids=`
  or labeled row objects for readable failures.
- Use fixtures for shared setup and teardown, not for hiding scenario-specific
  facts.
- Keep autouse fixtures rare. Use them when they establish a broad invariant
  for a whole module or suite.
- Use `tmp_path`, `monkeypatch`, `caplog`, `capsys`, and `pytest.raises`
  instead of custom harness code when pytest already owns the pattern.
- Mirror `src/` layout under `tests/` when that improves navigation.

### Test Data And Assertions

- Build expected values from domain rules, specifications, documented examples,
  or independent calculations.
- Avoid tautological tests that recompute expected values with the production
  helper being tested.
- Use domain-shaped factories for complex inputs. Vary one field or behavior per
  scenario.
- For dataframe transformations, assert full output when tractable using the
  dataframe library's testing helpers.
- For validation failures, assert the exception type and the stable contract:
  message, attributes, code, row count, or sample.

### Approval Tests

- Use approval tests for large generated artifacts such as SQL, structured
  reports, JSON, YAML, CLI output, generated configuration, and legacy
  characterization.
- Use direct assertions for scalar values, short strings, and narrow behavior.
- Canonicalize output before approval: sort unordered collections, normalize
  paths and line endings, scrub timestamps, scrub UUIDs, and stabilize platform
  differences.
- Pair approval files with focused assertions for important parameters or
  security-sensitive behavior.

### Benchmarks

- Keep performance benchmarks outside the normal correctness loop.
- Use pytest-benchmark for hot-path public functions and representative input
  sizes.
- Treat benchmark movement as a reviewable signal, not a substitute for a
  correctness test.
- Use profilers such as pyinstrument, py-spy, memray, tracemalloc, scalene, or
  cProfile according to the question being asked.

### Deterministic Waits

- Wait for the observable condition under test, not a guessed duration.
- Use `asyncio.Event`, `asyncio.wait_for`, queues, task groups, `threading.Event`,
  `threading.Condition`, subprocess readiness signals, or a reusable predicate
  wait helper.
- Fixed sleeps are appropriate only when timing behavior itself is under test.
  First wait for the timer or worker to start, then sleep for the documented
  timing window.
- Always set per-test timeouts for async, threaded, subprocess, and external
  readiness tests.

### Root-Cause Loop

- Start with the exact symptom: command, file, line, exception, value, and
  operation.
- Read the full traceback, including chained exceptions and grouped failures.
- Walk bad values backward through callers until the value first became wrong.
- State one hypothesis, test it with focused evidence, then fix the origin.
- If the fix attempt fails, return to investigation with the new evidence.
- For flaky tests, isolate order dependence, leaked process state, time, random
  seeds, filesystem ordering, environment, logging configuration, caches, and
  global registries.

## Operations And Reliability

### Recoverability Classes

Classify operational steps by whether they are recoverable:

- Vendor-rotated or externally expiring inputs need schedules, alerts, and
  explicit missed-run handling.
- Replayable upstream pulls can usually be rerun on demand.
- Warehouse transforms over persisted data belong in dbt or an equivalent
  transformation layer.
- Pure in-memory calculations should stay outside scheduling machinery unless
  they produce an operational artifact.

Capture non-obvious recoverability decisions in ADRs.

### Transactions And Rollback

- Build new state in locals first and commit at the end when possible.
- Use database transactions for multi-statement writes.
- Use context managers, `ExitStack`, temporary paths followed by atomic rename,
  or explicit compensating actions when intermediate effects must happen before
  later checks.
- Test failed operations by asserting direct post-failure state.
- Keep invariant-maintaining methods on the type or service that owns enough
  context to validate the invariant.

### Idempotency

- Queue consumers, webhook handlers, scheduled jobs, and message processors
  should define duplicate-delivery behavior.
- Use idempotency keys, processed-event markers, durable reservations,
  uniqueness constraints, `INSERT ... ON CONFLICT`, dlt merge dispositions, or
  atomic cache operations.
- Define what repeat processing returns or emits. Do not add idempotency keys
  without semantics.

### Logging And Observability

- Choose one logging stack per project.
- Use stdlib logging for libraries unless the project policy says otherwise.
- Use loguru for application ergonomics when the project standardizes on it.
- Use structlog when strict key/value logs are a product or operations
  requirement.
- Use static messages with structured values. Avoid building log messages with
  f-strings when the variable values should be separate fields.
- Use `logger.exception(...)` inside exception handlers when the boundary logs
  and stops or translates the failure.
- Avoid log-and-reraise. Propagate, or log and convert to the boundary response.
- Use level checks, lazy formatting, sampling, throttling, and counters for
  expensive or high-volume diagnostics.

### Failure Translation

- Inside a domain, prefer idiomatic Python exceptions.
- Use existing exception types when they communicate the failure clearly.
- Introduce custom exception types only when they carry real domain meaning,
  help callers distinguish a meaningful failure class, or support useful
  boundary translation.
- Translate failures at HTTP, CLI, job, queue, UI, and worker boundaries.
- Use exception chaining with `raise ... from exc` when adding context.
- Use `ExceptionGroup` and `except*` when grouped concurrent work can fail in
  several places.
- Use warnings, context managers, or sentinel values such as `None` when that is
  the idiomatic contract for the API.

## Promote And Avoid

| Area | Promote | Avoid |
|---|---|---|
| Project layout | `src/` layout, layer-named packages, tests that import through package roots, thin command facade over uv. | Top-level `common` or `utils`, metadata pointing at missing files, tool scopes that disagree between CI and local commands. |
| Domain modeling | Practitioner vocabulary, dataclasses for internal values, Pydantic at trust boundaries, explicit strategy registries. | Loose dicts through the core, primitive tuples for domain values, dynamic discovery hiding the supported public surface. |
| Strategy selection | Pydantic discriminated unions with frozen `Literal` tags and explicit package registries. | Giant untagged unions, `hasattr` dispatch over project-owned variants, trailing unknown-case raises after closed dispatch. |
| Closed sets | `Enum`, `Literal`, tagged unions, `match`, and `assert_never`. | Stringly typed closed values and default branches that hide missing cases from the checker. |
| Dataframes | Pandera at boundaries, named cross-row validators, vectorized pandas or query-engine operations. | Pretending dataframe internals are fully statically typed, defensive `.copy()` in every stage, hidden mutation of shared frames. |
| SQL | Parameterized values, SQL composition APIs for identifiers, approval tests for generated SQL. | f-string interpolation of values into raw SQL, regex plus `eval` for generated maintenance workflows. |
| Pipelines | Explicit in-process sequences and orchestration tools for durable asset graphs. | Generic graph abstractions before the graph is a real domain concept, unbounded dependency expansion without deduplication or cycle checks. |
| Error handling | Built-in or domain exceptions, boundary translation, exception chaining, focused failure tests. | Broad catches away from a boundary, swallowed exceptions, log-and-reraise, custom exception hierarchies without domain meaning. |
| Logging | One logging stack, static messages with structured context, `logger.exception` at stopping boundaries. | Production `print`, ad hoc `timeit` output, mixed logging stacks without policy, f-string log payloads. |
| Concurrency | `asyncio.TaskGroup`, anyio where cancel scopes earn the dependency, `httpx` clients scoped by resource, explicit rate limits. | `asyncio.gather(return_exceptions=True)` as a default, blocking calls inside async paths, broad catches that hide cancellation. |
| Testing | Pytest-native fixtures and parametrization, domain factories, approval tests for generated text, deterministic waits. | Tautological expected values, arbitrary sleeps, `cast` to fake invalid test data, broad catches in tests. |
| Performance | Profile first, vectorize, benchmark public hot paths, move heavy setup into worker initialization. | Micro-optimizing Python loops first, timing via `print`, adding parallelism before measuring. |
| Documentation | ADRs for decisions, algorithm notes for non-obvious math, present-tense comments explaining current intent. | Starter-template docs, comments narrating obvious code, past-tense refactor notes, placeholder durable docs. |

## Open Decisions

1. Decide whether dataframe-heavy guidance earns a required
   `python-design-principles/data-pipeline-and-dataframes.md` guide in the
   first pass, or remains distributed across architecture, types, testing, and
   performance.
2. Decide whether `asyncio.TaskGroup` stays the default concurrency baseline
   with anyio as the advanced option, or whether anyio becomes the default for
   applications with meaningful async orchestration.
3. Finalize the logging decision tree: stdlib logging for libraries, loguru for
   applications, structlog for strict structured logging, and how mixed
   third-party logging is bridged.
4. Decide how strongly the first guide set recommends global strict typing
   versus a directory-by-directory ratchet. The current plan sets strict as the
   target and ratcheting as the migration path.
5. Decide whether the agent examples should include dataframe-specific
   examples from factors2 or remain domain-neutral until the repo-specific
   agent skill exists.
6. Decide how much free-threaded Python material belongs in the first pass.
   Mention explicit locks and ownership where guidance relies on GIL-era
   assumptions, but keep normal CPython deployment guidance readable.
7. Decide the security baseline for the first tooling guide: Ruff security
   rules, dependency audit, secret scanning, SQL lint, and how much should be
   mandatory versus recommended.
8. Recheck Pyright, Pyrefly, ty, and mypy before freezing long-lived tool
   comparisons in durable documentation.
