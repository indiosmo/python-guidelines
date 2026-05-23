# Generalizable C++ Design Guidance for Python

Research notes for a later Python-guidelines writing phase. These notes extract
principles that transfer cleanly from the C++ design guide and translate them
into idiomatic modern Python. They are not a full guide draft.

## Source files reviewed

- `/home/msi/repos/cpp-guidelines/cpp-design-principles/README.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/architecture.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/invariants.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/declarative-style.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/functional-programming.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/pipelines.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/compile-time-correctness.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/error-handling.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/state-machines.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/runtime.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/performance.md`
- `/home/msi/repos/cpp-guidelines/cpp-design-principles/comments.md`
- `/home/msi/repos/python-guidelines/README.md` (empty)
- `/home/msi/repos/python-guidelines/MONOREPO.md`

Local documentation convention notes:

- Existing Python durable guidance is prose-first, with concrete directory or
  code examples where they clarify a rule.
- `README.md` is empty, so `MONOREPO.md` is the only local durable convention
  source reviewed.
- `MONOREPO.md` favors strict boundaries, explicit ownership, and practical
  examples over abstract taxonomy. The Python design notes should follow that
  shape.

## Candidate Python guide topics

### Domain theory and shared vocabulary

Transferable C++ principle: a program encodes the team's theory of the domain.
Names, types, tests, and module boundaries should use the same domain language
the practitioners use.

Python translation: name modules, classes, dataclasses, functions, parameters,
and tests with domain vocabulary. Prefer `order_id`, `triage`, `shipment`,
`sample_rate`, or similarly precise terms over generic placeholders like
`data`, `item`, `obj`, or `record` when the domain supplies better language.
Represent domain concepts as values with named fields rather than passing loose
dicts and primitive tuples through the core.

Caveats and when not to apply: do not invent jargon when the domain is not yet
clear. Use plain names for technical glue and ask for practitioner vocabulary
before naming core concepts. Small local variables in obvious algorithms can be
short if they are conventional.

Likely guide placement: naming, domain modeling, architecture.

### Domain boundaries with adapters

Transferable C++ principle: domains own their vocabulary and communicate through
adapter modules. Vendor SDKs, wire protocols, and sibling domains should not
leak their shapes into the domain core.

Python translation: keep integration code in adapter packages or modules that
translate foreign data into domain dataclasses, Pydantic models, attrs classes,
or typed dictionaries owned by the receiving domain. A domain module should not
import a vendor SDK just to inspect vendor objects. Application composition can
depend on multiple domains and wire them together.

Caveats and when not to apply: small scripts and one-off automation may not need
formal adapter packages. Do not create a shared `types` package as a reflex; use
shared packages only for concepts that are genuinely shared, stable, and owned
at that level.

Likely guide placement: package architecture, integrations, monorepo source
boundaries.

### Forward-only dependencies and explicit data flow

Transferable C++ principle: dependency arrows should form a DAG. Functions
should take what they need and return what they produce instead of relying on
state left behind by earlier calls.

Python translation: avoid circular imports, module-level hidden state, and
helper functions that mutate a context object as an implicit output channel.
Pass required values explicitly. Return a dataclass, tuple with named binding,
or result object when a step produces multiple values. Mutating methods are fine
when the receiver owns the state and the method name makes the mutation clear,
such as `reset_to`, `add_order`, or `normalize_in_place`.

Caveats and when not to apply: Python frameworks often impose callbacks,
registries, dependency injection containers, or ORM sessions. Keep those at the
edge and avoid letting framework-global state become the domain API.

Likely guide placement: architecture, function design, module dependencies.

### Functional core, imperative shell

Transferable C++ principle: push effects toward the edge. Inner layers should
be pure or close to pure; runtime concerns live in the shell.

Python translation: write domain functions that accept plain Python values and
return plain Python values. Put file I/O, network calls, database sessions,
environment reads, logging backends, schedulers, threads, and event loops in
application or adapter layers. Unit tests for core logic should not need mocks
for sockets, clocks, thread pools, or vendor SDKs.

Caveats and when not to apply: Django, FastAPI, Dagster, Celery, and similar
frameworks naturally define entry points. Treat those entry points as shell
code and move business decisions behind plain functions or service objects.

Likely guide placement: architecture, testing, framework integration.

### Parse, do not repeatedly validate

Transferable C++ principle: parse untrusted input into refined domain values at
the boundary so downstream code receives trustworthy inputs.

Python translation: parse JSON, CLI arguments, environment variables, database
rows, and external messages into typed boundary models before they enter core
logic. Use Pydantic, attrs validators, dataclass factory functions,
`typing.NewType`, enums, constrained value objects, or small classes with
validated constructors where they clarify the domain. Downstream functions
should accept those refined types, not raw dicts or strings that each caller
must re-check.

Caveats and when not to apply: Python cannot make invalid states impossible as
strongly as C++. Runtime validation remains necessary at untyped boundaries,
deserialization points, and places where values may have changed externally.
Avoid wrapping every primitive; wrap only values with real domain constraints,
ambiguous roles, or high swap risk.

Likely guide placement: typing, data modeling, boundary validation.

### Plain data carriers with named fields

Transferable C++ principle: data structs should be easy to construct, inspect,
serialize, and refactor. Named initialization is safer than positional
construction.

Python translation: use dataclasses, attrs classes, Pydantic models, enums, and
typed dictionaries for structured data. Prefer keyword construction for
multi-field objects. Use `kw_only=True` for dataclasses or attrs classes when
positional construction would invite swaps. Keep data carriers free of
surprising side effects.

Caveats and when not to apply: positional tuples are fine for local, conventional
pairs such as `(row, column)` or return values immediately unpacked at the call
site. Do not force dataclasses onto behavior-rich objects whose invariant is
better protected by methods and private fields.

Likely guide placement: data modeling, type hints, API design.

### Exhaustive handling of closed sets

Transferable C++ principle: closed sets should be handled in one visible table
of cases, and adding a case should make missing behavior easy to find.

Python translation: model closed vocabularies with `Enum`, `Literal`, tagged
unions, or explicit subclasses. Prefer `match` or a dispatch table when it makes
the cases visible. Use type checkers, `typing.assert_never`, and tests to catch
missing cases where practical.

Caveats and when not to apply: Python exhaustiveness is weaker and depends on
tooling. Do not claim runtime safety from type hints alone. For open plugin
systems, use registration or polymorphism instead of pretending the set is
closed.

Likely guide placement: typing, control flow, error handling.

### Declarative decomposition

Transferable C++ principle: decompose complex functions, work on simpler
inputs, stage derived variables before logic, and name predicates.

Python translation: split parse, validation, policy, and dispatch steps into
named helpers. Pass helpers the fields they use rather than whole framework
objects. Compute derived values before building a response. Name predicates
such as `is_active_user`, `is_missing_required_field`, or
`is_crossing_order` instead of burying multi-clause conditions inside loops.

Caveats and when not to apply: do not fragment a short, linear function into
helpers that are only names for individual lines. Inline lambdas are fine for
simple projections like `key=lambda order: order.price`.

Likely guide placement: function design, readability.

### Pythonic pipelines and lazy composition

Transferable C++ principle: compose transformations lazily and materialize once
when the result must be stored, returned, or passed to an API that expects a
container.

Python translation: use generator expressions, iterators, `itertools`, and
small generator functions for streaming transformations. Materialize with
`list`, `tuple`, `set`, or a dataframe only at the boundary where the result
must outlive the stream or be consumed more than once. Keep pipeline stages
named when the transformation carries domain meaning.

Caveats and when not to apply: Python iterators are single-use. If the caller
will traverse the data twice, materialize intentionally. For pandas, Polars,
NumPy, and SQL workloads, prefer the library's native vectorized or query API
over Python-level generator chains.

Likely guide placement: collection processing, performance, data engineering.

### Pure functions and value-oriented core logic

Transferable C++ principle: values in, values out, no ambient state, no hidden
logging or mutation in the core.

Python translation: keep business calculations deterministic where possible.
Pass configuration, clock values, and inputs explicitly. Return new values or
well-named updates instead of mutating shared module state. Use immutable or
frozen data models selectively for values whose identity should not change.

Caveats and when not to apply: Python object mutation is normal and sometimes
clearer. Mutable accumulators and in-place updates are acceptable inside a
well-scoped function or owner class when the mutation is visible and tested.

Likely guide placement: function design, testing, state management.

### Callable injection without framework coupling

Transferable C++ principle: pass behavior as a value when it avoids inheritance
or framework coupling. Store callbacks intentionally and understand lifetime
and cost.

Python translation: accept `Callable` parameters for small policies,
predicates, clocks, ID generation, or test probes. Use named functions or
closures when they make a call site read clearly. Store callbacks on objects
only when the object's role is to emit events or delegate policy.

Caveats and when not to apply: avoid callback-heavy designs when ordinary
method calls, protocols, or dependency objects are easier to understand. Watch
late binding in closures inside loops, and avoid capturing mutable state whose
lifetime is unclear.

Likely guide placement: dependency injection, testing, pipelines.

### Components as explicit pipelines

Transferable C++ principle: pipeline stages have inbound methods and outbound
callbacks; wiring near `main` owns the topology and cross-cutting behavior.

Python translation: for event-driven systems, model sources, stages, and sinks
with explicit methods and typed callbacks, queues, or async functions. Wire them
in application startup code. Let the wiring layer add logging, metrics,
filtering, thread or event-loop marshalling, and test probes. External wrappers
should translate vendor events into domain events before emitting them.

Caveats and when not to apply: Python projects may be better served by
framework routing, async streams, actors, or message queues. Avoid a global
event bus of generic dict messages unless the system genuinely needs dynamic
pub-sub; explicit edges are easier to read and test.

Likely guide placement: architecture, async and event-driven design,
integrations.

### Top-level error containment

Transferable C++ principle: top-level functions catch and translate failures so
one bad event does not take down the runtime. Component boundaries define how
failures cross.

Python translation: entry points such as HTTP handlers, queue consumers, CLI
commands, scheduled jobs, worker callbacks, and event-loop tasks should catch
expected domain exceptions and log or translate unexpected exceptions into safe
framework responses. Inside a domain, use a consistent exception or result
style; at boundaries, convert into the caller's vocabulary.

Caveats and when not to apply: do not swallow exceptions silently. Tests should
still be able to assert failures. Library code should not catch broadly unless
it can add context or translate to its public contract.

Likely guide placement: error handling, framework integration, operations.

### Domain-owned errors

Transferable C++ principle: a domain owns its failure vocabulary. Structured
errors should carry machine-readable context, not only formatted strings.

Python translation: define domain exception classes or result error objects in
the package that produces them. Include fields such as `field_name`, `order_id`,
`code`, or `reason` as attributes. Adapters translate errors between domains;
unrelated domains should not import each other's private error vocabulary.

Caveats and when not to apply: Python exception hierarchies can become too
large. Use a small number of meaningful exception classes with structured
attributes rather than one class per log message. For simple absence, `None` or
an empty collection may be clearer than an exception.

Likely guide placement: error handling, package boundaries.

### Transactional invariants and rollback

Transferable C++ principle: operations that update several pieces of state
should either fully apply or leave the object as it was. Prefer commit-at-end;
use rollback guards when intermediate mutation must be visible.

Python translation: build new state in local variables first, then assign it to
object fields or persist it once all fallible work succeeds. Use database
transactions, context managers, `ExitStack`, temporary files followed by atomic
rename, or explicit compensating actions when work must be applied before later
steps can run. Keep invariant-maintaining methods on the type that owns the
state.

Caveats and when not to apply: external side effects cannot always be rolled
back. In those cases, design idempotent compensating actions and durable
reconciliation instead of pretending the operation is atomic.

Likely guide placement: state management, error handling, persistence.

### Idempotent handlers

Transferable C++ principle: retried operations should produce the same final
state whether they run once or many times. Record rollback facts at admission
time and use atomic check-then-act operations.

Python translation: make queue consumers, webhook handlers, scheduled jobs, and
message processors safe under duplicate delivery. Store idempotency keys,
processed-event markers, or exact reservation amounts. Use database uniqueness
constraints, `INSERT ... ON CONFLICT`, transactions, or atomic cache operations
instead of check-then-insert races in Python code.

Caveats and when not to apply: pure calculations do not need idempotency
machinery. Idempotency requires storage and clear semantics; do not add keys
without defining what repeat processing should return or emit.

Likely guide placement: reliability, persistence, distributed systems.

### State machines for lifecycle-heavy behavior

Transferable C++ principle: when behavior is a graph of legal states and events,
an explicit state machine beats scattered booleans and mode flags.

Python translation: use explicit enums, transition tables, state pattern
objects, or a mature state-machine library for protocols, workflows, connection
lifecycle, retries, approvals, and order flows. Keep side effects in actions
owned by the surrounding class or service. Test transitions edge by edge and
test the owner separately.

Caveats and when not to apply: do not use a state machine for a single boolean,
a linear parse-validate-write pipeline, or a pure transformation. Python lacks
Boost.SML-style compile-time checking; the transition table must be backed by
tests.

Likely guide placement: state management, workflow modeling, testing.

### Deferred work instead of re-entrant recursion

Transferable C++ principle: callbacks or FSM events that would re-enter the
current call stack should be queued and processed after the current operation
unwinds.

Python translation: use `asyncio.create_task`, event-loop callbacks, queues, or
a local pending-work deque to defer follow-up work that would otherwise recurse
through the same stage. In synchronous code, drain a queue at the outermost call
instead of calling back into the owner from the middle of a mutation.

Caveats and when not to apply: deferral changes ordering and error propagation.
Use it for real re-entrance hazards, not to hide missing state transitions or
avoid writing a clear sequence.

Likely guide placement: async design, event-driven systems, state machines.

### Threading and async as edge effects

Transferable C++ principle: inner components should be single-threaded. Runtime
layers own threads, event loops, queues, and cross-thread marshalling.

Python translation: keep domain services unaware of whether they are called
from a thread, process, or async task. Use queues, executors, actor-like
wrappers, or event-loop scheduling to cross runtime boundaries. Document classes
that are deliberately thread-safe or tied to a specific loop. Avoid sharing
mutable domain state across threads; let one owner serialize access.

Caveats and when not to apply: Python has the GIL but still has races around
mutable state and I/O interleavings. Async code can race through interleaved
await points. Multiprocessing and distributed workers require persistence-level
coordination, not only in-process locks.

Likely guide placement: concurrency, async, runtime architecture.

### Performance requirements before optimization

Transferable C++ principle: default to clarity, know the budget, measure, and
optimize only where the saved time matters end to end.

Python translation: use profiling, tracing, benchmarks, query plans, and
production metrics before replacing clear code with clever code. State the
latency, throughput, memory, or cost target. Prefer algorithmic improvements,
vectorized libraries, database-side work, batching, caching, or fewer network
round trips before micro-optimizing Python loops.

Caveats and when not to apply: Python performance bottlenecks often live in I/O,
serialization, database access, dataframe operations, dependency import time, or
framework overhead. C++ advice about allocations, cache locality, and inlining
should be reframed around Python's runtime costs.

Likely guide placement: performance.

### Comments as present-tense guidance

Transferable C++ principle: comments should explain present intent near the
code, especially non-obvious ordering, domain rules, lifetimes, and performance
constraints. They should not narrate obvious statements or preserve migration
history.

Python translation: write short comments before non-obvious blocks. Use comments
to name protocol ordering, transaction reasoning, retry behavior, concurrency
preconditions, or domain rules that are not visible from names. Avoid comments
that say what the code used to do, what moved elsewhere, or what a function does
not handle.

Caveats and when not to apply: public API documentation can include contracts
and examples, but it should still describe current behavior. Historical
rationale belongs in ADRs or commit history, not inline comments.

Likely guide placement: comments, documentation, code review.

## Concepts to avoid or weaken for Python

- Do not port "compile-time correctness" as a literal promise. Python type
  hints and type checkers are valuable, but enforcement is optional and runtime
  values can still violate annotations. Phrase this as "make mistakes visible
  early" or "use types and validation to move checks earlier."
- Do not recommend a per-domain `types` namespace. In Python, use domain modules
  with clear names, such as `orders.models`, `orders.types`, or
  `orders.value_objects`, only when the package size justifies the split.
- Do not wrap every primitive in a class. Python object overhead and ergonomics
  are different. Prefer wrappers for constrained, ambiguous, externally visible,
  or high-risk values.
- Do not carry over C++ result-type machinery directly. Python's default failure
  channel is exceptions, with result objects useful in selected workflows.
  Recommend consistency and boundary translation rather than a universal
  `Result[T, E]` rule.
- Do not port LEAF, `std::expected`, scope guards, or `BOOST_LEAF_ASSIGN`
  mechanics. Translate them into domain exceptions or result objects,
  context managers, `try`/`except`, database transactions, and boundary handlers.
- Do not port `std::variant` plus exhaustive visitor mechanics directly. Python
  can use `Enum`, tagged unions, pattern matching, protocols, and
  `assert_never`, but exhaustiveness is a tooling aid rather than a language
  guarantee.
- Do not port callback storage performance advice around `std::function`,
  `function_ref`, or inplace functions. Python callables all have runtime
  dispatch costs; optimize callback-heavy hot paths only after profiling.
- Do not port C++ container-choice advice literally. Python's built-in `dict`,
  `set`, `list`, `deque`, heap queues, arrays, NumPy, pandas, Polars, and
  database indexes have different cost models.
- Do not port "no allocations on the hot path" as a broad Python rule. Python
  allocates frequently by design. Reframe as reducing avoidable object churn,
  avoiding unnecessary materialization, batching work, and moving hot loops to
  specialized libraries when measurement justifies it.
- Do not recommend Boost.SML, PlantUML-in-header conventions, or template-based
  FSM patterns. The transferable idea is explicit state transition modeling,
  documentation that matches the transition table, and edge-by-edge tests.
- Do not treat threads as the only runtime concern. Python guidance should cover
  `asyncio`, worker pools, multiprocessing, background task frameworks, and
  framework-managed request lifecycles.
- Do not overstate immutability. Frozen dataclasses and immutable values are
  useful, but Python code often benefits from simple mutable objects inside a
  clear owner.
- Do not import C++ examples or terminology into the Python guide. Replace
  "headers", "translation units", "RAII", "constexpr", "UB", "move semantics",
  and "type erasure" with Python-native concepts.
