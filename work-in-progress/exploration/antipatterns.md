# Antipatterns Consolidation

## Purpose

This briefing consolidates the factors2 anti-pattern passes into one source
map for the Python guidelines port. It keeps concrete factors2 evidence where
that evidence clarifies the failure mode, but phrases the candidate guidance in
domain-neutral Python terms.

The briefing is not a durable guide. It is a writing input for the first guide
set, especially the guides on project layout, architecture, typing, dataframe
boundaries, error handling, concurrency, testing, debugging, and comments.

## Selection Rules

Include an anti-pattern when it teaches a general Python engineering rule that a
reviewer can apply outside factors2. Prefer entries that cross file boundaries,
encode implicit contracts, hide runtime dependencies, make tests less
trustworthy, or create operational ambiguity.

Do not promote every local code smell into guide material. Exclude issues that
are only one-off cleanup, mechanical lint findings, or facts that belong next
to a specific implementation. Use linter findings as evidence only when the
guideline value is the design rule behind the finding.

Each entry uses one of these statuses:

- Observed: found directly in factors2 and suitable as evidence.
- Research-derived: drawn from modern Python practice and useful for the
  guideline set even when factors2 only shows nearby symptoms.
- Enforcement-only: a rule that should appear as tooling policy or review
  criteria, not as a full prose section.

## Package Boundaries And Tool Metadata

### Collapsed deployable boundaries

Status: Observed.

Evidence: `pyproject.toml` defines one project whose runtime dependencies cover
the core library, ETL sources, Dagster, dbt, dashboards, portfolio management,
backtesting, HTTP clients, AWS clients, pandas, DuckDB, Ray, FastAPI, and
Streamlit. `src/` contains several import roots with different deployability
profiles, including `pyfactors`, `etl_factors`, `dagster_factors`,
`dashboards`, and `dbt_factors`.

Guideline candidate: split Python systems by deployable boundary. Core domain
packages declare only domain runtime dependencies. ETL jobs, orchestration
locations, dashboards, operational apps, and shared libraries get their own
package metadata or workspace member with a dependency set they own.

### Runtime and tool dependency drift

Status: Observed.

Evidence: factors2 has tool packages in runtime dependency lists, duplicate or
conflicting tool configuration, and command surfaces that do not describe the
same file scope. Examples include unused development tools listed as project
dependencies, Black configuration alongside Ruff formatting, and Ruff versions
that differ between project metadata and CI.

Guideline candidate: make one command surface authoritative. CI, pre-commit,
local Make targets, and documentation invoke the same checks over the same
file scopes and tool versions. Keep type stubs, profilers, visualization tools,
and test helpers in dependency groups rather than application runtime
dependencies.

### Upper-bound dependency pins in libraries

Status: Research-derived.

Evidence: the anti-pattern pass identifies upper-bound constraints as a
common Python packaging failure mode. Python has a flat dependency model, so a
library cap such as `<3` can block an application from resolving a compatible
environment long after the library still works.

Guideline candidate: library packages generally declare tested minimum
versions and avoid speculative upper bounds. Applications may pin exact
versions through a lock file because the application owns its environment.

### Flat layout masking import errors

Status: Research-derived.

Evidence: the Python Packaging User Guide recommends `src` layout for many
projects because flat layout can make tests import the local checkout rather
than the installed package.

Guideline candidate: use `src` layout for packages that need packaging
confidence. Tests should exercise the installed import path, not accidental
current-working-directory imports.

### Generated artifacts in source-like trees

Status: Observed.

Evidence: factors2 has generated Python caches, package metadata, dbt target
artifacts, and dbt logs under source-like directories.

Guideline candidate: generated artifacts stay in ignored output locations
unless they are intentionally versioned inputs. Source trees contain source
files, stable configuration, tests, and reviewable fixtures.

### Tool metadata without an invocation

Status: Observed.

Evidence: factors2 lists tools such as `pyfakefs`, `pyinstrument`, and
`pydeps` without a corresponding test, Make target, guide, or documented
command surface.

Guideline candidate: every development dependency earns its place through a
named invocation. If a tool supports a recurring workflow, expose it through
the project command surface. If no workflow uses it, remove it.

## Configuration, Environment, And Process-Global State

### Environment reads inside business paths

Status: Observed.

Evidence: factors2 reads environment variables, loads dotenv files, attaches
databases, inspects proxy settings, and calls `date.today()` inside work paths
such as database connection helpers, dataset fetchers, dashboards, and entry
points.

Guideline candidate: read environment variables, dotenv files, clocks, proxy
settings, and service endpoints at process boundaries: CLI `main`, app
factory, worker initializer, Dagster resource, or test fixture. Pass validated
settings, connections, and clocks inward as explicit objects.

### Hand-rolled settings objects

Status: Observed.

Evidence: `pyfactors/database/postgres_settings.py` uses a dataclass-like
settings object with a custom initializer that conditionally reads environment
variables and command-line arguments. The construction order determines which
source wins.

Guideline candidate: use a typed settings model for environment, CLI, and file
configuration. Construction should return a complete settings object with clear
precedence rules before domain work starts.

### Defaults that hide required configuration

Status: Research-derived.

Evidence: the research pass calls out defaults such as local database or cache
URLs for configuration values that are required in deployed environments.

Guideline candidate: required production configuration has no production
default. Fail during startup validation when a value is missing. Keep local
development defaults in local configuration files or explicit test fixtures.

### Secrets in dotenv files

Status: Observed.

Evidence: factors2 has a `.env` file in the working tree containing database
credentials next to an `.env.example`.

Guideline candidate: dotenv files are local-development inputs, not a secrets
boundary. Production secrets come from the deployment secret store, and
settings models use secret-aware field types where accidental logging is a
risk.

### Process-global mutable state as dataflow

Status: Observed.

Evidence: factors2 coordinates work through module globals, pandas global
options, DuckDB's module-level default connection, and notebook-style scripts
with import-time setup. `generate_one_pagers.py` uses globals to feed child
processes, and multiple modules mutate `pd.options.mode.copy_on_write`.

Guideline candidate: keep dataflow explicit through parameters, returned
values, context managers, dependency objects, and framework-owned resources.
Use process-local or context-local state only where a runtime boundary requires
it, and reset it in tests.

### Import-time side effects

Status: Research-derived.

Evidence: the research pass identifies import-time database connections,
client construction, registry population, and executable script cells as a
recurring Python failure mode. Factors2 shows nearby symptoms in notebook-style
scripts and process-global setup.

Guideline candidate: module import defines names and lightweight constants.
Runtime setup belongs in factories, `main` functions, app constructors,
fixtures, or explicit registration calls.

## Data And Type Contracts

### God modules for unrelated domain models

Status: Observed.

Evidence: `src/pyfactors/pipeline/models.py` groups trade dates, interest
rates, benchmarks, market prices, quarterly indicators, security lists,
blacklists, orders, positions, position books, and reference data in one file.

Guideline candidate: group domain types by concept, pipeline stage, or bounded
context. Use package exports for migration compatibility when needed, but keep
independent concepts in files that can evolve and test independently.

### God functions that mix orchestration and computation

Status: Observed.

Evidence: `src/pyfactors/backtest/server.py` has a simulation entry point that
computes lookback windows, builds portfolio configuration, scores inputs,
dispatches Ray work, and collects results in one function.

Guideline candidate: separate pure computation from orchestration shells.
Pure functions compute values from inputs. Runtime functions wire external
systems, scheduling, workers, logging, and result collection.

### Stringly typed domain values

Status: Observed.

Evidence: signal values such as `long`, `neutral`, and `short` are represented
as strings and validated only through a Pandera field.

Guideline candidate: promote closed domain vocabularies to enums, literals, or
small value types. Boundary schemas may parse and validate external strings,
but internal code should use names the type checker and IDE can track.

### Broad dictionaries for structured configuration

Status: Observed.

Evidence: chart configuration uses `dict[str, Any]` for trace and layout
configuration even when the application likely uses a known subset of Plotly.

Guideline candidate: use `TypedDict`, dataclasses, Pydantic models, or a
library-provided type for structured configuration that has a stable subset.
Use `dict[str, Any]` only at deliberately dynamic boundaries.

### `Any` as a permanent escape hatch

Status: Research-derived.

Evidence: the research pass calls out `Any` propagation as a common gradual
typing failure mode. Factors2 has nearby symptoms in broad dictionary
configuration and type ignores around dynamic dispatch.

Guideline candidate: treat `Any` as temporary boundary pressure. Narrow it at
the edge with validation, a typed mapping, a protocol, or a small adapter
before the value reaches domain code.

### Excessive type ignores and casts

Status: Observed.

Evidence: computed indicator dispatch uses several `# type: ignore` comments
around reflection, and tests cast an empty DataFrame to a schema type.

Guideline candidate: type ignores and casts require a local reason and a small
scope. In tests, prefer typed factories such as `WeightedSignals.empty()` over
casts that can drift when a schema changes.

### Closed enum dispatch with unreachable fallback raises

Status: Observed.

Evidence: `SeriesPeriod.annualization_factor` dispatches on a closed enum with
`if` branches and a trailing `raise ValueError("Unknown period")`.

Guideline candidate: dispatch closed enums with a lookup table, `match`, or
`assert_never`-backed exhaustiveness. Reserve runtime "unknown value" errors
for values parsed from an open boundary.

### Modern typing syntax left unused

Status: Research-derived.

Evidence: the research pass notes that Python 3.12 and later support native
generic class syntax and the `type` statement, while Python 3.13 adds `TypeIs`
for two-branch narrowing.

Guideline candidate: for a Python 3.13 baseline, prefer current typing syntax
where it clarifies contracts. Use `TypeIs` when a predicate should narrow both
the true and false branches.

### Tabular contracts living only in DataFrame code

Status: Observed.

Evidence: factors2 has strong Pandera usage at some pipeline boundaries, but
scripts, dashboards, and maintenance workflows also pass plain `DataFrame`
values whose columns, dtypes, indexes, ordering, and units live by convention.

Guideline candidate: validate tabular data at module and pipeline boundaries
with named schemas. Keep pandas internals flexible, but make external
DataFrame contracts explicit in domain vocabulary.

### Implicit coercion that hides data quality failures

Status: Observed.

Evidence: several Pandera fields use `coerce=True` for numeric or categorical
columns without making the expected source coercions explicit.

Guideline candidate: use coercion deliberately at ingestion boundaries.
Domain-stage schemas should reject unexpected types unless the stage owns the
normalization rule.

## Dynamic Dispatch And Extension Points

### Massive union type as a registry

Status: Observed.

Evidence: `src/pyfactors/indicators/functions.py` defines a large union of
indicator classes. Adding an indicator requires editing the central union, and
forgetting the edit can silently exclude the indicator.

Guideline candidate: extension catalogs need an explicit registration model.
Use a registry, entry point, decorator, or package-owned export list depending
on the deployment boundary and review needs.

### Reflection instead of an adapter contract

Status: Observed.

Evidence: `ComputedIndicator.compute_rolling` inspects method signatures to
decide whether to pass `reference_data`, while `deps` uses `hasattr` and type
ignores.

Guideline candidate: put dynamic dispatch behind a named adapter boundary.
Prefer a small `Protocol`, an explicit registry, or a normalized wrapper that
gives implementations one public call shape.

### Graph expansion without graph invariants

Status: Observed.

Evidence: computed indicator dependency expansion recurses through
dependencies and returns dependencies before dependents, but the exploration
found no visible cycle detection, deduplication, or stable node identity.

Guideline candidate: when a dependency list becomes a graph, give nodes stable
identities, deduplicate shared dependencies, reject cycles with a clear error,
and test ordering behavior.

### Dynamic `__all__` from type machinery

Status: Observed.

Evidence: one package builds `__all__` from `typing.get_args(...)` over a
strategy union and suppresses Pyright's unsupported dynamic export warning.

Guideline candidate: module exports should be statically visible. Generate
them as code if needed, but make the checked source contain literal exported
names.

### Over-engineered nominal interfaces

Status: Research-derived.

Evidence: the research pass calls out abstract base classes as a coupling
hazard when the interface is small and the implementations need not share
inheritance.

Guideline candidate: use `Protocol` for structural contracts, plain callables
for simple behavior, and abstract base classes when nominal membership carries
domain meaning or shared implementation.

### Text-to-code extension points

Status: Observed.

Evidence: `indicator_portfolios_sql_from_md.py` parses markdown code blocks
with regular expressions, walks module exports dynamically, evaluates extracted
Python expressions, and generates SQL text.

Guideline candidate: generated configuration uses structured input such as
TOML, YAML, JSON, or a deliberately trusted Python module. Code execution is an
explicit trust boundary, not a convenience parser.

## Mutability, Caches, And DataFrame Ownership

### Frozen objects with mutable internals

Status: Observed.

Evidence: `ReferenceData` is declared frozen but calls mutation methods on
nested fields. The research pass also notes that frozen dataclasses do not make
nested lists, dicts, or DataFrames immutable.

Guideline candidate: immutability claims must include nested state that
callers can observe. Use immutable containers, copy-on-write construction,
private caches, or explicit mutable aggregate types.

### Hidden mutation of supplied configuration

Status: Observed.

Evidence: `SimulationParams.model_post_init` appends a derived restriction to
`self.signal_strategy.restrictions`, mutating a nested configuration object
supplied by the caller.

Guideline candidate: treat user-supplied configuration as input. Build an
effective configuration by copying, deriving, or normalizing explicitly.

### Two-phase initialization with order dependency

Status: Observed.

Evidence: `PostgresSettings` can be constructed with `env=True`, `args=...`,
both, or neither, and the order of method calls determines which values win.

Guideline candidate: constructors return valid objects. Use classmethod
factories, settings models, or builder functions when different input sources
need different parsing paths.

### Stateful caches that drift from backing data

Status: Observed.

Evidence: `InterestRates`, `Benchmarks`, and `MarketPrices` expose both a
DataFrame and a derived lookup dictionary. `ReferenceData.build_dicts()` exists
to rebuild the caches after construction.

Guideline candidate: derived caches are either private and rebuilt on demand,
or immutable and built during construction. Avoid public mutable source data
beside public mutable derived data.

### Redundant truthiness checks on optional values

Status: Observed.

Evidence: lookup helpers call `dict.get(...)` and then return `data if data
else None`, converting legitimate falsy values such as `0.0` to missing.

Guideline candidate: preserve the difference between absence and falsy domain
values. Use sentinel checks or direct `dict.get` semantics when `None` is the
missing value.

### Defensive DataFrame copies around impure transforms

Status: Observed.

Evidence: the indicator orchestrator copies DataFrames before passing them to
indicator functions because some indicators add scratch columns to their
inputs.

Guideline candidate: dataframe transforms own their mutation contract. Pure
transforms read input columns and return new columns plus join keys. In-place
transforms make ownership explicit and keep scratch state out of shared inputs.

### Pandas row iteration in object construction

Status: Observed.

Evidence: position construction iterates over DataFrame rows with `iterrows`.

Guideline candidate: prefer vectorized operations or `to_dict(orient="records")`
for object construction from tabular data. Use row iteration only when the
operation is intrinsically row-local and not performance-sensitive.

### Deep copies in hot loops

Status: Observed.

Evidence: the simulation copies a position book deeply on each simulation
step.

Guideline candidate: snapshot only the state that changes. Use immutable value
objects, structural sharing, or targeted copies before reaching for deep copy
inside repeated work.

### Dataclass initialization abuse

Status: Research-derived.

Evidence: the research pass calls out `__post_init__` methods that fill in
partially initialized fields or perform complex runtime work.

Guideline candidate: use `__post_init__` for local invariant checks and simple
derived fields. Use classmethod factories or service functions for runtime
work, clock reads, I/O, and multi-step construction.

### Missing slots on value dataclasses

Status: Research-derived.

Evidence: the research pass notes that slotted dataclasses reduce memory and
prevent accidental dynamic attributes for value objects.

Guideline candidate: default internal value objects to
`@dataclass(frozen=True, slots=True, kw_only=True)` unless mutability or dynamic
attributes are part of the domain operation.

## SQL, Text-To-Code, And Generated Maintenance Workflows

### SQL built by f-string interpolation

Status: Observed.

Evidence: `pyfactors/database/database_queries.py` constructs SQL fragments
with f-strings and interpolated dates and symbols. The same codebase also has
examples of safer SQL composition through database APIs.

Guideline candidate: SQL values are bound parameters or typed SQL composition
objects. String formatting may assemble static clauses selected by code, but
user, data, and boundary values enter the query as parameters.

### Unsafe generated SQL maintenance scripts

Status: Observed.

Evidence: a maintenance script parses markdown, evaluates Python expressions,
escapes one JSON field manually, and writes SQL for downstream use.

Guideline candidate: maintenance generators use structured inputs and database
composition APIs. The accepted input language should be smaller than the full
Python language unless the file is intentionally trusted code.

### ORM overhead on read-heavy paths

Status: Research-derived.

Evidence: the research pass identifies full ORM hydration for read-only
dashboard paths as a common performance problem.

Guideline candidate: use ORM objects where persistence behavior, identity
management, and relationship navigation matter. For read-heavy projections,
prefer explicit SQL or query builders returning dataclasses, typed rows, or
DataFrames.

### N+1 query behavior

Status: Research-derived.

Evidence: the research pass highlights ORM lazy loading as a runtime failure
mode that static tools rarely catch.

Guideline candidate: database code should make relationship loading strategy
visible at query construction. Test or profile high-cardinality paths for query
counts.

### Standard-library replacements left unused

Status: Research-derived.

Evidence: the research pass notes that `tomllib` is available in the standard
library for Python 3.11 and later.

Guideline candidate: prefer standard-library modules when they meet the need
and avoid a runtime dependency. A third-party parser earns its place when it
adds required write support, validation, comments, or compatibility.

## Error Handling, Logging, And Operational Scripts

### Broad catch blocks away from boundaries

Status: Observed.

Evidence: Streamlit handlers, dataset fetching, and ETL scripts catch
`Exception` broadly. Some cases show only a toast, printed message, or generic
error.

Guideline candidate: catch specific exception classes in domain and adapter
code. Reserve broad `except Exception` for boundary layers that translate
failures to UI, HTTP, CLI, job, or queue protocols, and log the traceback
there.

### Over-broad try blocks

Status: Observed.

Evidence: `fetch_dataset` wraps proxy setup, HTTP fetch, schema validation,
and DuckDB load in one try block, making different failure classes converge on
one generic path.

Guideline candidate: wrap logical failure steps separately when they need
different context, retry policy, or user-facing translation.

### Inconsistent exception classes for one domain failure

Status: Observed.

Evidence: missing rates, benchmarks, and market prices are raised as a mix of
`ValueError` and `RuntimeError`.

Guideline candidate: use existing Python exceptions when they communicate the
failure clearly. Introduce custom domain exceptions when callers need to
distinguish a meaningful domain failure class, such as missing market data or
missing reference data.

### Logging exception strings instead of tracebacks

Status: Observed.

Evidence: some exception handlers log or toast `str(e)`, and some use
`logger.error(f"... {str(e)}")` inside `except`.

Guideline candidate: inside an exception handler, log with traceback context
through the project's logging stack. Use exception chaining when raising a new
failure with added context.

### Mixed logging stacks

Status: Observed.

Evidence: factors2 uses stdlib `logging` in one area and loguru in another.

Guideline candidate: choose one logging stack per project surface. Libraries
usually use stdlib logging; applications can configure loguru or structlog and
bridge library logs at startup.

### Variable data baked into log messages

Status: Observed.

Evidence: downloader code logs URLs assembled with query strings as part of
the message while passing some other values as structured fields.

Guideline candidate: log messages should be stable event names. Put request
URLs, identifiers, durations, counts, and domain values in structured fields.

### `print` for production events and timing

Status: Observed.

Evidence: backtest and pipeline modules print status, timing, and operational
events. Timing uses scattered `timeit.default_timer()` calls followed by
`print`.

Guideline candidate: production events use the project logger. Timed blocks
use a common context manager or decorator that emits structured duration data.
Profiling and microbenchmarks live in profiling commands or benchmark tests.

### Operational scripts that return success after failure

Status: Observed.

Evidence: scripts print invalid input or per-record failures and continue or
return from `main()` without a nonzero exit status.

Guideline candidate: command-line tools return nonzero on invalid user input
and unrecoverable failures. If partial success is valid, report counts, write
machine-readable failure details, and define the acceptable failure threshold.

### Log-and-reraise

Status: Enforcement-only.

Evidence: the source passes identify handlers that log an exception and then
raise again, which can duplicate logs at multiple layers.

Guideline candidate: either handle and log at the boundary, or raise with
added context and let the boundary log. Avoid logging the same exception at
every layer.

## Concurrency, Multiprocessing, Async Fan-Out, And Rate Limiting

### Fork-dependent multiprocessing contracts

Status: Observed.

Evidence: `generate_one_pagers.py` loads reference data and universe state
into module globals, then starts worker processes that rely on inherited
globals through fork behavior.

Guideline candidate: multiprocessing code names its start method, worker
initializer, shared resources, and input serialization contract. Test the path
under the start method used in production.

### Unstructured async fan-out

Status: Observed.

Evidence: `ProxyDownloader.fetch` creates tasks and awaits
`asyncio.gather(..., return_exceptions=True)`. The downloader has useful
semaphores and rate limiting, but sibling failure, cancellation, timeout, and
result typing are not all visible at the fan-out boundary.

Guideline candidate: use `asyncio.TaskGroup` for related concurrent work when
sibling failure should cancel the group. When partial failure is expected, make
the result type explicit and keep retry, timeout, rate-limit, and cancellation
policy in one place.

### `asyncio.gather(return_exceptions=True)` as data

Status: Research-derived.

Evidence: the research pass highlights that `return_exceptions=True` returns
exceptions in the result list, where callers can forget to inspect them.

Guideline candidate: when concurrent work can produce multiple failures, turn
the failure branch into an explicit result object or raise `ExceptionGroup` so
callers handle it intentionally.

### Blocking calls inside async functions

Status: Research-derived.

Evidence: the research pass calls out synchronous I/O and sleeps inside
`async def` as a common Python concurrency bug.

Guideline candidate: async functions use async-compatible I/O, async sleeps,
and async-aware clients. Blocking work moves to a bounded thread or process
pool with cancellation and timeout behavior documented at the boundary.

### Thread offload for CPU-bound Python work

Status: Research-derived.

Evidence: the research pass notes that `asyncio.to_thread` does not bypass the
GIL for CPU-bound Python code.

Guideline candidate: use thread offload for blocking I/O and process pools,
native extensions, vectorization, or external workers for CPU-bound Python
work.

### Duplicate rate limiting implementations

Status: Observed.

Evidence: one ETL source uses `time.sleep` for rate limiting while another
module uses `aiolimiter` for async rate limiting.

Guideline candidate: rate-limit policy belongs in a reusable adapter or client
layer. Reuse the project-approved primitive instead of adding a parallel local
implementation.

## Testing Numeric, Tabular, And Domain Behavior

### Session-scoped global side effects in tests

Status: Observed.

Evidence: `tests/conftest.py` sets `pd.options.mode.copy_on_write = True` for
the whole session without restoring the prior value.

Guideline candidate: tests that alter process-global state save and restore
it. If the library relies on a global mode, establish that mode at the library
or application boundary rather than only in the test suite.

### Brittle numeric and dataframe assertions

Status: Observed.

Evidence: factors2 has good domain-shaped tabular tests and approval tests,
but some tests assert long lists of floats or selected scalar cells where the
contract is a whole table, tolerance, dtype, index, sort order, or invariant.

Guideline candidate: for dataframe transforms, prefer `pandas.testing`
helpers, schema checks, and domain invariants. For floating-point results, use
explicit tolerances through pytest, NumPy, or pandas testing APIs.

### Casts to fake typed test data

Status: Observed.

Evidence: a test helper casts an empty DataFrame to `DataFrame[WeightedSignals]`.

Guideline candidate: typed test data should be constructed through schema-aware
factories, fixtures, or builders. A cast in a test hides schema drift from the
checker.

### Over-mocking behavior tests

Status: Research-derived.

Evidence: the research pass calls out tests that assert internal calls,
private method usage, or deep mock chains rather than observable behavior.
Factors2's stronger tests use realistic domain DataFrames, approval tests, and
public behavior.

Guideline candidate: mock external systems and expensive side effects. Test
core logic with real domain objects, fakes, fixtures, returned values,
persisted effects, emitted events, or user-visible errors.

### Mock signature drift

Status: Research-derived.

Evidence: regular `Mock()` objects accept any argument list, so tests can call
interfaces with signatures the real object would reject.

Guideline candidate: when mocking a typed collaborator, use autospecced mocks
or a small fake that implements the same protocol.

### Approval tests without review discipline

Status: Research-derived.

Evidence: the research pass identifies snapshot and approval tests that get
updated without reviewing the received diff.

Guideline candidate: approval updates require human review of the diff. Scrub
nondeterministic data before approval, and keep approved artifacts small enough
to review.

### Magic numbers for domain constants

Status: Observed.

Evidence: trading-day counts such as `252` appear directly in calculations
even though the codebase also has a period-to-annualization concept.

Guideline candidate: name domain constants at the concept boundary. Use the
named concept in calculations and tests so changes happen in one place.

## Documentation And Repository Hygiene

### Commented-out alternative behavior

Status: Observed.

Evidence: a backtest path contains a commented-out `raise RuntimeError(...)`
beside current missing-price behavior.

Guideline candidate: delete commented-out code. If a branch reflects a domain
rule, state the current rule in code, tests, or a present-tense comment at the
point where the rule matters.

### Negative and refactor-history comments

Status: Enforcement-only.

Evidence: the repository's agent instructions already define the negative
documentation anti-pattern, and commented-out branches in factors2 show the
same risk.

Guideline candidate: comments and docs describe present behavior and the
non-obvious reason behind it. They do not describe old behavior, moved
responsibilities, or absent behavior.

### Empty placeholder directories

Status: Observed.

Evidence: factors2 has an empty `one-pagers/` directory at the repository
root.

Guideline candidate: commit directories when they contain source, durable
documentation, configuration, tests, or intentionally versioned fixtures. Use a
short README when an otherwise empty directory has durable meaning.

### Documentation drift from generated or temporary inputs

Status: Enforcement-only.

Evidence: the porting plan treats `work-in-progress/` as ephemeral source
material and instructs durable guides not to depend on it.

Guideline candidate: durable guides cite stable concepts, durable project
files, or external sources. Temporary research artifacts feed writing but do
not become long-term documentation dependencies.

## Items To Keep Mostly As Tooling Policy

These findings are valid, but they should usually enter the guide set as
tooling defaults, review checks, or short side notes rather than full guide
sections:

- SQL interpolation: enforce with security lint rules where available, and
  teach parameterized SQL in the data-access guide.
- `print` in `src`: enforce with Ruff `T201`, with exceptions for CLI output
  paths that intentionally write user-facing text.
- broad `except Exception`: enforce with broad-exception rules, while the
  error-handling guide teaches boundary translation.
- `eval`: enforce with security rules, while the extension-point guidance
  teaches structured inputs and trusted-code boundaries.
- dynamic `__all__`: enforce through type-checker diagnostics and review.
- commented-out code and negative documentation: enforce through review and
  agent context.

## Durable Guide Placement

Use this mapping when writing the first guide set:

| Topic | Primary durable guide |
|---|---|
| Workspace boundaries, dependency groups, command surface, generated artifacts | `python-projects-and-tooling/` |
| Package layering, import boundaries, functional core, orchestration shells | `python-design-principles/architecture.md` |
| Domain values, `Any`, casts, protocols, enum exhaustiveness, modern syntax | `python-design-principles/types-and-correctness.md` and `python-design-principles/generics-and-protocols.md` |
| DataFrame schemas, mutation ownership, dataframe tests | `python-design-principles/data-pipeline-and-dataframes.md` if that conditional guide is written |
| SQL, configuration generation, text-to-code workflows | `python-design-principles/pipelines.md` and tooling guide sections |
| Exceptions, logging, operational scripts | `python-design-principles/error-handling.md` and `python-debugging-principles/logging.md` |
| Async fan-out, multiprocessing, rate limiting | `python-design-principles/runtime-and-concurrency.md` |
| Numeric tests, approval tests, mocks, fixtures | `python-testing-principles/` |
| Comments, commented-out code, negative documentation | `python-design-principles/comments-and-docstrings.md` and `agent/` |
