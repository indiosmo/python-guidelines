# Python Design Principles

Python design should keep domain ownership, dependency direction, invariants,
and runtime effects visible. Domain code works with values and domain errors.
Adapters translate external shapes. Application and orchestration layers own
runtime wiring.

## Navigation

- [architecture.md](architecture.md) covers domain ownership, package layers,
  adapters, functional core, explicit dataflow, runtime boundaries, and
  testability.
- [types-and-correctness.md](types-and-correctness.md) covers static typing,
  boundary parsing, dataclasses, Pydantic, Pandera, domain values, unions,
  protocols, `Any`, `cast`, and ignores.
- [comments-and-docstrings.md](comments-and-docstrings.md) covers
  present-tense comments, precondition comments, docstrings, rationale,
  validation comments, inline comments, and commented-out code.
- [declarative-style.md](declarative-style.md) covers decomposition, staged
  values, named predicates, built-ins, lazy composition, dataframe-native style,
  and explicit pipelines.
- [functional-programming.md](functional-programming.md) covers pure core, sum
  types, `match`, `assert_never`, higher-order functions, closures, and dynamic
  boundaries.
- [generics-and-protocols.md](generics-and-protocols.md) covers PEP 695 syntax,
  `Protocol`, `TypeIs`, `TypeGuard`, `ParamSpec`, callable protocols, and
  variadic generics.
- [decorators-and-metaprogramming.md](decorators-and-metaprogramming.md) covers
  plain functions first, decorators, registration, `singledispatch`,
  `__init_subclass__`, descriptors, code generation, and metaclasses.
- [error-handling.md](error-handling.md) covers built-in exceptions, domain
  exceptions, validation, chaining, boundary translation, grouped failures,
  cleanup, warnings, sentinels, and narrow result-shaped values.
- [invariants-and-rollback.md](invariants-and-rollback.md) covers invariant
  ownership, commit-at-end mutation, transactions, context managers, atomic
  writes, idempotency, caches, free-threaded caveats, and rollback tests.
- [state-machines.md](state-machines.md) covers when state machines fit,
  enum-and-match models, state classes, transition tables, owner boundaries,
  deferred events, libraries, and tests.
- [pipelines.md](pipelines.md) covers explicit stages, sources, sinks,
  callbacks, queues, async boundaries, data pipelines, graph tools, runtime
  wiring, and tests.
- [runtime-and-concurrency.md](runtime-and-concurrency.md) covers runtime
  ownership, `TaskGroup`, anyio triggers, clients, timeouts, queues,
  marshalling, shared state, re-entrant work, and multiprocessing.
- [cross-cutting-services.md](cross-cutting-services.md) covers logger, clock,
  settings, timer, metrics, tracing, `ContextVar` facades, explicit domain
  dependencies, and test substitution.
- [performance.md](performance.md) covers budgets, measurement, profilers,
  algorithmic wins, vectorization, copies, caching, slots, parallelism, logging
  cost, and leaving Python.
- [data-pipeline-and-dataframes.md](data-pipeline-and-dataframes.md) covers
  pipeline layer choice, Pandera schemas, dataframe mutation ownership, SQL
  composition, Dagster, dbt, pandas, Polars, ibis, and dataframe tests.

Start with `architecture.md`, then read the narrower guide that matches the
code under review.
