# Python Agent Context

Always-loaded reference for agents working in Python codebases. It carries the
rules that shape code an agent would otherwise write: project layout, domain
ownership, dependency direction, typing, exceptions, runtime boundaries,
pytest, debugging, tooling, and comments. Concrete good and bad code pairs
live in [python-agent-examples.md](python-agent-examples.md); load it when a
specific edit needs examples.

Before writing code, search for the local abstraction that already expresses
the idea. Prefer project helpers, package conventions, and command surfaces
over new utilities unless the same missing shape recurs.

## Core Lens

Code is the working theory of the domain. Preserve that theory in names,
package boundaries, types, validation, errors, and tests. Before adding
behavior, ask:

- Which domain owns this concept?
- Which boundary parses untrusted input into domain values?
- Which type or validator carries the proof that the value is valid?
- Which adapter translates between external and local vocabulary?
- Which runtime effect can move outward toward the shell?
- Which invariant must remain true if a middle step fails?

If the answer is unclear, make ownership, inputs, outputs, and failure modes
visible in the signature before adding more behavior.

## Project Layout

Use the project's existing layout. The common shape is a root `pyproject.toml`
and `uv.lock`, package code under `src/`, tests under `tests/`, benchmarks
under `benchmarks/`, and infrastructure outside importable package trees.
Reusable libraries, applications, orchestration code, and data pipelines should
live in packages whose names describe the domain or layer they own.

Avoid generic package names such as `common`, `shared`, `helpers`, and `utils`
for core concepts. Use practitioner vocabulary such as `market_data`,
`portfolio_rebalance`, `specimen_processing`, `shipment_routing`, or
`audio_rendering` when those are the real domains.

## Domain Ownership

Domain packages own vocabulary, values, invariants, errors, and public
contracts. Adapter packages translate vendor SDKs, wire formats, database
rows, CLI inputs, queue messages, UI inputs, and dataframe schemas into domain
values. Application and orchestration packages own settings, logging
configuration, event loops, worker pools, database sessions, schedulers, and
framework entry points.

Keep import arrows acyclic. When two packages need the same concept, move that
concept to a named package both can import.

## Types And Validation

Use type annotations where they clarify stable contracts: public APIs, domain
values, adapters after parsing, settings, reusable helpers, fixtures, fakes,
and dataframe boundary objects. Keep dynamic Python where runtime validation
or native dataframe operations are clearer than elaborate static types.

Parse hostile input at boundaries. Use Pydantic v2 for JSON, CLI, environment,
config, queue, and cross-process shapes. Use Pandera for dataframe schemas.
Use frozen slotted dataclasses for internal values built from validated parts.
Use `NewType` or small validated classes when primitive identity matters.

Let `Any` stop at adapters. Use `object` when a value is unknown and only
runtime checks permit operations. Use `cast` only after a nearby runtime check,
a trusted library contract, or a named invariant.

## Functional Core And Runtime Shell

Domain functions accept values and return values. Runtime code reads
environment variables, opens files, configures logging, creates clients,
starts workers, and calls framework APIs. Framework routes, CLI commands,
Dagster assets, scheduled jobs, and UI handlers translate framework inputs to
domain values, call the core, and translate the result back.

Use explicit parameters for domain-specific collaborators such as repositories,
pricing clients, carriers, or codecs. Use context-local facades for truly
cross-cutting services such as logger, clock, settings, tracing context, and
timers when scoped overrides are useful.

## Error Handling

Python code is exception-first. Use built-in exceptions when the name
communicates the failure. Add small custom domain exceptions when callers need
to distinguish a meaningful failure class, when structured attributes support
logging or tests, or when a boundary needs a stable translation target.

Translate failures at HTTP, CLI, job, queue, worker, event-loop, and UI
boundaries. Use exception chaining when adding context. Use `ExceptionGroup`
and `except*` for grouped concurrent failures. Use warnings for recoverable
conditions that callers may escalate. Use sentinel values when absence is the
normal API contract.

Result-shaped values fit bulk item status, best-effort partial processing, and
serialized per-item outcomes. Exceptions remain the default failure channel.

## Invariants And Rollback

Put invariant ownership with the object, package, transaction, or pipeline
stage that can maintain it. Build new state in locals and commit at the end
when possible. Use context managers, `ExitStack`, database transactions,
atomic file replacement, idempotency keys, durable markers, and compensating
actions for effects that must remain coherent.

Tests for rollback should assert the observable state after failure, not only
the raised exception.

## Concurrency And Pipelines

Runtime ownership belongs near the entry point. Use `asyncio.TaskGroup` for
structured async work. Use anyio when cancel scopes, timeout composition, or
trio compatibility earn the dependency. Use queues to cross ownership
boundaries. Keep shared mutable state owned by one task, thread, process, or
object.

Pipeline stages should expose explicit inputs and outputs. Use callbacks,
queues, async channels, Dagster assets, dbt models, or graph tools when the
graph shape has operational meaning. Keep runtime wiring close to the shell.

## Testing

Tests assert behavior through independent evidence from the domain, contract,
specification, bug report, or public example. They should fail for plausible
defects and stay readable as examples of the intended behavior.

Use pytest. Prefer clear test names, direct assertions, parametrization with
meaningful `ids`, domain factories, fixture factories, dataframe builders,
fakes that implement protocols, and condition-based waits. Keep benchmarks out
of the normal correctness loop.

For bug fixes, reproduce the failure, write or tighten the regression test,
then fix the production cause.

## Debugging

Do not patch symptoms. Reproduce the failure, read the whole traceback or
diagnostic output, trace the bad value or state backward, state one hypothesis,
run one focused experiment, write the regression test, and fix the origin.

Use logs, probes, `pytest -k`, warning escalation, Pyright, Ruff, Python
development mode, asyncio debug mode, `faulthandler`, `tracemalloc`, memray,
and py-spy as evidence sources. When a fix attempt fails, treat that as new
evidence and return to investigation.

## Comments And Docs

Write comments and docstrings in present tense. Explain what the code does and
why the why is non-obvious. Prefer positive statements about current behavior.
Delete refactor notes, history notes, commented-out code, and lists of absent
responsibilities.

Use comments for non-obvious grouping, invariants, preconditions, algorithmic
rationale, or domain rules. Keep field-level and variable documentation close
to the code that defines the field or variable.

## Tooling

Use the repository command surface. The default shape is:

```sh
uv sync
uv run ruff format .
uv run ruff check .
uv run pyright
uv run pytest
```

Run the narrowest useful command while editing, then run the broader command
that verifies the changed surface before calling the work complete.
