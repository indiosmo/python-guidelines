# C++ Source Python Port Consolidation

## Purpose And Source Coverage

This artifact consolidates the duplicated C++ source exploration into one
working source map for the Python guidelines port. It preserves the C++ guides'
engineering intent, identifies the Python mechanisms that replace C++-specific
machinery, and records the topics that need deliberate Python treatment during
guide writing.

Sources consolidated:

- Codex's narrative summary, per-guide analyses, and modern Python tooling
  notes.
- Claude's stricter file mapping, section-level dispositions, agent-context
  mapping, and Python-specific research notes.

The analysis covers every requested C++ source guide:

- 14 design guides.
- 8 testing guides.
- 4 debugging guides.
- 2 agent-context files.

Total source coverage is 28 guide and agent files. The Claude branch also
contains seven research notes on type checkers, strong types, PEP 695,
exhaustive matching, structured concurrency, Ruff rule sets, and observability.
The Codex branch contains a broader modern Python tooling note. Together they
support the Python defaults captured in `porting-plan.md`.

## Overall Finding

The C++ guides mostly transfer as design pressure, not as mechanisms. The
Python edition should preserve domain ownership, forward-only dependency
graphs, functional-core design, invariants, idempotency, deterministic tests,
condition-based waits, approval tests, root-cause debugging, and defensive
tooling. It should rewrite strong typing, error handling, concurrency,
templates, macros, logging, and sanitizer guidance into idiomatic Python.

The guide must not teach "C++ with Python syntax". Python types, validation,
exceptions, runtime checks, async task ownership, pytest, Ruff, uv, and Python
profilers carry the intent where C++ uses the compiler, RAII, templates,
macros, build presets, and sanitizers.

## Disposition Map

This table uses the narrower Claude file mapping because it best matches the
resolved guide structure in `porting-plan.md`. Mixed files are classified by
their dominant writing treatment.

| C++ source | Disposition | Python guide direction |
|---|---|---|
| `cpp-design-principles/architecture.md` | PORTS_AS_IS | Keep domain ownership, adapters, forward dependency DAGs, explicit inputs and outputs, and functional core / imperative shell. Render the units as packages, modules, imports, protocols, and runtime composition. |
| `cpp-design-principles/compile-time-correctness.md` | PYTHON_SPECIFIC_VARIANT | Rename as types and correctness. Replace compiler proof with pyright or pyrefly strict mode, `NewType`, frozen dataclasses, `Annotated`, Pydantic at boundaries, discriminated unions, and `assert_never`. |
| `cpp-design-principles/declarative-style.md` | PORTS_AS_IS | Keep decomposition, simpler inputs, staged values, named predicates, and lazy composition. Use built-ins, comprehensions, generator expressions, `itertools`, and dataframe `pipe` chains. |
| `cpp-design-principles/functional-programming.md` | PORTS_WITH_ADAPTATION | Keep pure functions, value returns, sum-type dispatch, and higher-order functions. Replace variant matching with `match`, discriminated unions, `Protocol`, `TypeIs`, and Python closure guidance. |
| `cpp-design-principles/templates.md` | PYTHON_SPECIFIC_VARIANT | Rewrite as generics and protocols. Use PEP 695 syntax, `Protocol` bounds, `ParamSpec`, `TypeVarTuple`, `TypeIs`, and `assert_never`. |
| `cpp-design-principles/preprocessor-macros.md` | PYTHON_SPECIFIC_VARIANT | Rewrite as decorators and metaprogramming. Prefer plain functions, then decorators, then `__init_subclass__`, with metaclasses as the last resort. |
| `cpp-design-principles/error-handling.md` | PYTHON_SPECIFIC_VARIANT | Rewrite as exception-first Python. Use domain exception hierarchies inside a domain, explicit translation at boundaries, exception chaining, ordered `except` clauses, and top-level catch-all handling. |
| `cpp-design-principles/invariants.md` | PORTS_WITH_ADAPTATION | Keep commit-at-end, rollback, idempotency, and invariant ownership. Replace scope guards with `contextlib.ExitStack`, copied dataclasses, context managers, and explicit rollback tests. |
| `cpp-design-principles/state-machines.md` | PORTS_WITH_ADAPTATION | Keep the FSM decision criteria, diagrams, edge tests, and deferred-event discipline. Prefer `match` over dataclass state unions for small machines and `transitions` or a similar library for larger graphs. |
| `cpp-design-principles/pipelines.md` | PORTS_AS_IS | Keep stages with inbound methods, outbound callbacks, and wiring near the runtime entry point. Replace `inplace_function` with `Callable` fields and async/thread marshalling primitives. |
| `cpp-design-principles/runtime.md` | PORTS_WITH_ADAPTATION | Keep "threading is an edge effect" and "marshalling beats sharing". Use `asyncio.TaskGroup`, anyio task groups, `loop.call_soon_threadsafe`, queues, cancellation, and free-threaded Python caveats. |
| `cpp-design-principles/cross-cutting.md` | PYTHON_SPECIFIC_VARIANT | Rewrite around module-level services, `ContextVar` overrides, protocols, and pytest fixtures. Drop the C++ `std::variant` global argument. |
| `cpp-design-principles/performance.md` | PORTS_WITH_ADAPTATION | Keep clarity-first and measure-first. Replace C++ allocation primitives with profiling, vectorization, `__slots__`, caching, native extensions, and "leave Python" thresholds. |
| `cpp-design-principles/comments.md` | PORTS_AS_IS | Preserve present-tense, positive comments. Translate to Python comments and docstrings. Keep the negative-documentation ban. |
| `cpp-testing-principles/philosophy.md` | PORTS_AS_IS | Keep independent-oracle testing, behavior focus, and component-vs-integration distinction. |
| `cpp-testing-principles/catch2-conventions.md` | PYTHON_SPECIFIC_VARIANT | Rewrite as pytest conventions: collection, node IDs, marks, fixtures, parametrization, assertion rewriting, `pytest.raises`, xdist, and plugin policy. |
| `cpp-testing-principles/test-patterns.md` | PORTS_WITH_ADAPTATION | Map Catch2 `TEST_CASE`, `SECTION`, generators, and typed tests to pytest functions, fixtures, parametrization, `ids=`, subtests, and type-checker-only tests. |
| `cpp-testing-principles/test-helpers.md` | PORTS_WITH_ADAPTATION | Keep factories, presets, fixtures, probes, and provider guards. Use keyword-only dataclasses, factory fixtures, yield fixtures, contextvars tokens, and probe modules. |
| `cpp-testing-principles/error-path-testing.md` | PORTS_WITH_ADAPTATION | Default to `pytest.raises` and structured exception assertions. Keep rollback-after-failure tests. Mention Result-library helpers only for projects that use that style. |
| `cpp-testing-principles/condition-based-waiting.md` | PORTS_AS_IS | Keep predicate-based waits. Provide sync and async helpers using monotonic clocks, `threading.Event`, `threading.Condition`, `asyncio.Event`, and anyio cancellation scopes. |
| `cpp-testing-principles/approval-tests.md` | PORTS_AS_IS | Keep approval-test fit, determinism, scrubbers, and reviewable diffs. Use `approvaltests`, `pytest-approvaltests`, stable serializers, and dataframe stabilization. |
| `cpp-testing-principles/qt-gui.md` | DROP_CONDITIONAL | Drop Qt by default. Add `web-ui.md` only when the target repo has web UI tests; add Python Qt guidance only when a real PySide or PyQt surface exists. |
| `cpp-debugging-principles/root-cause-tracing.md` | PORTS_AS_IS | Keep backward tracing from bad value to root cause. Use tracebacks, `traceback.format_stack`, `pdb`, log capture, `pytest-randomly`, and low-perturbation probes. |
| `cpp-debugging-principles/defense-in-depth.md` | PORTS_AS_IS | Keep layered validation and "make the bug class impossible". Use boundary parsing, domain checks, environment guards, debug logging, and Python assertions carefully. |
| `cpp-debugging-principles/logging.md` | PORTS_WITH_ADAPTATION | Preserve severity discipline, throttling, domain formatting, and logger ownership. Use stdlib logging, loguru, or structlog by project policy; document disabled-level costs. |
| `cpp-debugging-principles/sanitizers.md` | PYTHON_SPECIFIC_VARIANT | Rename as static analysis and runtime checks. Use pyright or pyrefly, Ruff, `python -X dev`, warning-as-error tests, asyncio debug mode, tracemalloc, memray, py-spy, and suppression policy. |
| `agent/cpp-agent-context.md` | PORTS_WITH_ADAPTATION | Preserve the condensed always-loaded rule sheet shape. Swap C++ mechanics for Python package, typing, pytest, exception, async, and tooling defaults. |
| `agent/cpp-agent-examples.md` | PORTS_WITH_ADAPTATION | Preserve good/bad pairs keyed to the context sections. Re-render examples in Python with dataclasses, pytest, protocols, exceptions, pipelines, and async. |

Disposition counts:

- PORTS_AS_IS: 10.
- PORTS_WITH_ADAPTATION: 11.
- PYTHON_SPECIFIC_VARIANT: 6.
- DROP_CONDITIONAL: 1.

The counts are by dominant file treatment. Several files have section-level
mixtures; for example, `preprocessor-macros.md` preserves the "last resort"
principle but drops the preprocessor mechanics.

## Common Topic Map

### Design

Design guidance should preserve these C++ principles:

- Source code is the team's working theory of the domain.
- Domains own their vocabulary, types, errors, invariants, and adapters.
- Dependency arrows point forward and form a DAG.
- Functions take the data they need and return the data they produce.
- The functional core stays free of I/O, sockets, timers, logging, and thread
  ownership.
- Pipelines wire stages near the runtime entry point through explicit callbacks.
- Mutations with meaningful invariants commit at the end or register rollback.
- State machines earn their place when legal behavior is a graph of states and
  events.
- Comments explain present intent and non-obvious flow, not old behavior or
  absent behavior.

Python guide anchors:

- Packages and modules are the dependency units.
- `types.py`, `errors.py`, and adapter modules carry domain vocabulary.
- `Protocol` expresses structural contracts without forcing inheritance.
- Frozen slotted dataclasses are the default internal value object.
- Pydantic v2 parses trust-boundary data; Pandera belongs at dataframe
  boundaries.
- `match`, discriminated unions, and `assert_never` replace C++ variant
  exhaustiveness.
- `contextlib.ExitStack`, context managers, and copied dataclasses replace RAII
  scope guards.
- `ContextVar` backed services replace variant-backed globals for clocks,
  timers, settings, and test-overridable providers.

### Testing

Testing guidance should keep the C++ repository's independent-oracle
discipline, but use pytest vocabulary throughout:

- One scenario per test function is the default.
- Shared setup belongs in fixtures.
- Tables belong in `pytest.mark.parametrize`, with `ids=` for readable failure
  names.
- Expected failures use `pytest.raises`, bound exception payloads, and message
  matching only when the message is part of the contract.
- Rollback tests assert the post-failure state directly.
- Condition-based waits poll a predicate or use event primitives; arbitrary
  sleeps are only for documented time windows after a trigger.
- Approval tests verify complex rendered output, not simple scalar facts.
- Type-checker tests use `typing.assert_type` or pyright/pyrefly checked files
  where the behavior is static typing itself.

### Debugging

Debugging guidance transfers almost directly:

- Trace the wrong value backward to the point it first became wrong.
- Add instrumentation before the failure point when manual tracing stops.
- Prefer low-perturbation observability for timing-sensitive bugs.
- Make bug classes structurally impossible through types, parsing, validation,
  runtime guards, and diagnostics.
- Treat suppressions as triaged debt with comments and narrow scope.

Python guide anchors:

- Tracebacks, `traceback.format_stack`, `pdb`, `breakpoint`, `caplog`,
  `pytest-randomly`, `faulthandler`, rich tracebacks, and loguru backtraces
  replace C++ stack-trace helpers.
- `pyright` or `pyrefly`, Ruff, warning-as-error test runs, asyncio debug mode,
  memray, tracemalloc, py-spy, pyinstrument, and scalene replace sanitizer
  presets.
- stdlib logging, loguru, and structlog are policy choices, not utilities to
  mix freely.

### Agent Context

The Python agent context should stay condensed and prescriptive. It should not
repeat every guide, but it should give coding agents the default vocabulary:

- Python package layout, tests outside `src`, and runtime wrappers near the
  entry point.
- Domain ownership through `types.py`, `errors.py`, adapters, and import DAGs.
- Static typing that proves contracts without modeling every dynamic fact.
- Exceptions inside domains and explicit translation at boundaries.
- pytest-first testing.
- Root-cause debugging and checker-backed feedback.
- Comments in present tense with no negative documentation.

The examples file should remain an on-demand companion with good and bad Python
pairs for each rule.

## Python Replacements For C++ Mechanisms

| C++ mechanism | Python replacement |
|---|---|
| Header and library dependency graph | Package and module import graph, enforced with import-linter, pydeps, and Ruff import rules. |
| Domain `types` namespace | Per-domain `types.py` or `_types.py`, imported as `domain.types`. |
| `lib::strong_type<T, Tag>` | `NewType` for identity-only distinctions; frozen dataclass or `Annotated` plus validation when the value carries proof. |
| Parse factories returning `expected` | Parse factories that raise domain exceptions by default; Result-shaped values only where a Python boundary benefits from typed failure branches. |
| Plain aggregate structs | `@dataclass(frozen=True, kw_only=True, slots=True)` for internal values; mutable dataclasses when mutation is the domain operation. |
| `static constexpr` discriminator | `Literal[...]` discriminator field on dataclasses and Pydantic models. |
| `std::variant` and `lib::match` | Union types, `match` / `case`, and `assert_never`. |
| Concepts and constrained templates | `Protocol`, PEP 695 type parameters, bounds, `ParamSpec`, `TypeVarTuple`, and user-defined narrowing with `TypeIs`. |
| `if constexpr (requires ...)` | Runtime capability checks with `hasattr`, or type-checker friendly `Protocol` and `TypeIs` narrowing. |
| `dependent_false_v` | Drop. Use `assert_never` for exhaustiveness. |
| Preprocessor macros | Plain functions, decorators, `functools.singledispatch`, `__init_subclass__`, code generation where justified, and metaclasses last. |
| LEAF result types and `BOOST_LEAF_CHECK` | Python exceptions, exception chaining, ordered `except` clauses, and boundary translation. |
| Scope guards and RAII rollback | `contextlib.ExitStack`, context managers, `try` / `finally`, and `dataclasses.replace`. |
| `inplace_function` callback storage | `Callable` annotations, `Protocol` with `__call__`, `functools.partial`, and explicit `None` or list defaults for outbound callbacks. |
| Event loop `post` | `loop.call_soon_threadsafe`, `asyncio.create_task`, anyio task groups, and queue bridges. |
| SPSC queues and atomics | `asyncio.Queue`, `queue.Queue`, `multiprocessing.Queue`, `janus`, `threading.Event`, `asyncio.Event`, and explicit locks when truly shared. |
| `std::mutex` / `shared_mutex` | `asyncio.Lock`, anyio locks, `threading.Lock`, and `threading.RLock`, used near the runtime layer. |
| Variant-backed cross-cutting service | Module-level service facade with a `ContextVar` override or standard logging configuration at startup. |
| Catch2 `TEST_CASE`, `SECTION`, `GENERATE` | pytest test functions, fixtures, parametrization, `ids=`, marks, node IDs, and optional subtests. |
| `REQUIRE_THROWS_AS` / matchers | `pytest.raises`, `match=`, and assertions on `exc_info.value`. |
| ApprovalTests C++ API | `approvaltests.verify`, `pytest-approvaltests`, deterministic serializers, scrubbers, and reviewable received files. |
| Qt Catch2 test chapter | Drop by default; use pytest-qt for real Qt projects or Playwright for web UI projects. |
| Sanitizer presets | Static analysis, runtime checks, profilers, dev mode, warning policy, and suppression policy. |
| `cpptrace` stack helpers | Python tracebacks, `traceback.format_stack`, `faulthandler`, loguru backtraces, rich tracebacks, `pdb`, and `breakpoint`. |

## Deliberate Python Deviations

### Error Handling Is Exception-First

The C++ split between LEAF results inside a domain and `std::expected` at a
boundary should not be ported directly. Python's default in-domain error
channel is exceptions. The Python guide should teach:

- Domain base exceptions and concrete structured exception types.
- Existing built-in exceptions when they communicate the failure clearly.
- Exception chaining with `raise ... from exc` at translation points.
- `ExceptionGroup` and `except*` where concurrent work can fail in several
  places.
- `contextlib`, warnings, and sentinel values where they are idiomatic.
- Result-shaped values only for boundary APIs where the consuming code benefits
  from type-checker-visible branches.

### Static Typing Is Useful Proof, Not A Second Program

Python cannot reproduce C++ compile-time enforcement. The replacement is
strict static analysis plus parse-once validation. The guide should recommend
typing for public contracts, domain values, discriminated unions, protocols,
and refactor safety, while keeping dynamic Python where the type layer adds
ceremony without useful proof.

This matters most for dataframe-heavy code, untyped I/O boundaries, plugin
dispatch, and deliberate adapter points.

### Async And Cancellation Are First-Class

The C++ runtime guide centers on thread ownership and posting work across
loops. Python needs an async-aware version:

- `asyncio.TaskGroup` is the standard fan-out-and-join primitive.
- anyio adds cancel scopes and a richer structured-concurrency model where the
  dependency is justified.
- Cancellation must propagate. Code that catches broad exceptions must avoid
  swallowing cancellation.
- `ExceptionGroup` is part of the failure model for grouped tasks.
- Runtime wrappers own threads, executors, event loops, and process pools.

### Performance Starts With Vectorization And Profiling

C++ hot-path allocation patterns do not transfer directly. Python performance
guidance should start with measurements and then choose among vectorization,
better data layout, `__slots__`, caching, profilers, native extensions, and
process boundaries. Micro-optimizing Python loops is rarely the right first
move.

### Metaprogramming Means Decorators First, Metaclasses Last

Python has no preprocessor. The replacement chapter should help readers choose
between a plain function, decorator, `__init_subclass__`, descriptor,
singledispatch, code generation, and metaclass. The rule "last resort" still
applies, but the default escape hatch is a typed decorator with
`functools.wraps` and `ParamSpec`, not a global macro-like utility.

### Logging Has No True Zero-Cost Disabled Level

The C++ logging guide's compile-time cutoff has no Python equivalent. The
Python guide should teach lazy formatting, `isEnabledFor` around expensive
payloads, throttled call sites, and one logging stack per project. It should
not promise zero runtime cost.

### Free-Threaded Python Needs Explicit Caveats

CPython's GIL-derived atomicity assumptions are changing for free-threaded
builds. The Python guide should flag any advice that relies on CPython under
the GIL, such as incidental dict operation atomicity. Shared state in
free-threaded Python needs explicit locks or the same owning-thread discipline
the runtime guide already recommends.

## Research Notes To Preserve

### Python Version And Typing Stack

- Target Python 3.13 as the baseline and Python 3.14 as the forward target.
- Use PEP 695 syntax for new generic aliases, functions, and classes.
- Prefer pyright as the default type checker because it has the strongest
  conformance and IDE story.
- Mention pyrefly as the fast large-codebase alternative once its typing-spec
  conformance is acceptable for the project.
- Treat mypy as a legacy or existing-project fallback.
- Do not rely on checker-specific tricks unless they support an explicit
  project policy.

### Data Carriers And Boundary Models

- Use `NewType` when a primitive already has valid domain meaning and needs a
  static identity distinction.
- Use frozen slotted dataclasses for internal domain value objects.
- Use Pydantic v2 at trust boundaries such as HTTP, CLI, config, JSON, and
  database adapter outputs that need parsing.
- Use attrs when a project already depends on attrs or needs converters and
  validators without the full Pydantic boundary model.
- Use Pandera at dataframe boundaries where column shape and value constraints
  matter.

### Exhaustiveness

- Use `match` over enums and discriminated unions.
- End closed dispatch with `case _ as unreachable: assert_never(unreachable)`.
- Boundary parsers still need explicit failure branches that raise structured
  errors for unknown external values.

### Ruff And Tooling

- Use Ruff for linting and formatting.
- Select rules that support the guideline themes: annotations on public
  surfaces, modern syntax, bugbear, import hygiene, pytest rules, async
  anti-patterns, exception rules, no production `print`, pathlib, timezone-aware
  datetime, and security checks where appropriate.
- Use uv as the package manager and command runner in examples.
- Keep `pyproject.toml` as the main project configuration surface.

### Structured Concurrency

- Prefer `asyncio.TaskGroup` for simple standard-library fan-out and join.
- Prefer anyio when cancel scopes, timeout composition, or trio compatibility
  earn the dependency.
- Use anyio `fail_after` or `move_on_after` for composable timeouts.
- Re-raise cancellation; do not hide it in broad catch handlers.

### Observability And Profiling

- Use `python -X dev`, `PYTHONASYNCIODEBUG=1`, warning-as-error test runs, and
  `faulthandler` for runtime checks.
- Use `pyinstrument` for everyday CPU profiling, `py-spy` for production-safe
  sampling, `memray` for memory attribution, `tracemalloc` for stdlib allocation
  tracking, `scalene` for line-level CPU and memory, and `cProfile` for
  deterministic unit-level comparisons.
- Keep checker and warning suppressions narrow, commented, and versioned.

## Dropped Or Conditional Topics

- Drop direct C++ macro mechanics: include discipline, token paste, variadic
  macro dispatch, umbrella headers, and preprocessor utility layering.
- Drop LEAF-specific macros and the two-result-types frame as the default
  Python error model.
- Drop `fixed_string`, `static_vector`, `inplace_function`, flat-vs-node map
  advice, and equivalent C++ allocation machinery.
- Drop sanitizer-specific build presets, ASan, MSan, UBSan, TSan, LSan, and
  Fil-C mechanics for pure Python guidance.
- Drop Qt GUI testing by default. Add a GUI chapter only if the target codebase
  has a real GUI surface. Use pytest-qt for PySide or PyQt, Playwright for web
  UI, and HTTP clients for API surfaces.
- Drop C++ const-placement guidance. Python replacements are `Final`, frozen
  dataclasses, read-only properties, and clear mutation ownership.
- Drop namespace-alias rules tied to C++ `using`. Python guidance should cover
  absolute imports, rare explicit `import ... as ...`, no wildcard imports, and
  import-linter contracts.

## Questions Resolved By The Active Plan

These questions shaped the reconciliation pass. `porting-plan.md` carries the
current decisions for guide writing:

1. The first pass includes a dedicated dataframe and pipeline guide because the
   reference codebase is dataframe-heavy.
2. Structured concurrency defaults to stdlib `asyncio.TaskGroup`, with anyio as
   the option when cancel scopes, timeout composition, or trio compatibility
   earn the dependency.
3. Logging policy is selected by project surface: stdlib logging for libraries,
   loguru for applications, and structlog for strict key/value logs.
4. Strict typing is the target state; existing code may ratchet by package or
   directory.
5. Agent examples should use factors2 evidence where it illustrates a
   domain-neutral Python rule.
6. Free-threaded Python belongs as caveats in runtime, invariants, performance,
   and static-analysis guidance.

## Writing Consequences

Use this artifact as the C++ source consolidation behind `porting-plan.md`.
Durable guides should not link to this file; they should absorb the stable
guidance and cite durable sources or final guide structure instead.
