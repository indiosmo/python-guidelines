# Agent context rewrite analysis

Overall disposition: PORTS_WITH_ADAPTATION.

The C++ agent context is the most important source to port because it is the
always-loaded rule set. Most sections should survive, but the Python version
must change defaults around typing, errors, runtime, tests, and tooling. The
Python context should be shorter than the topical guides and sharp enough to
steer code edits.

## Opening and local vocabulary

Disposition: PORTS_WITH_ADAPTATION.

Replace `lib::` placeholder mapping with Python project mappings:

- package layout and import conventions;
- domain dataclasses and `types.py` modules;
- error base classes and code enums;
- pytest fixtures and factories;
- provider modules for clock/logger/timer;
- type checker command;
- `uv`, Ruff, and pytest commands.

Add a rule to search for existing local helpers before creating abstractions.

## Core Lens

Disposition: PORTS_AS_IS.

Keep almost unchanged. The questions port:

- Which package owns this concept?
- Which type or parser carries the proof this value is valid?
- Which boundary translates to another domain or external API?
- Which effect can move outward?
- Which invariant must remain true if a middle step fails?

Add "which dynamic escape hatch is justified?" so agents do not over-type
Python code.

## Layout

Disposition: PORTS_WITH_ADAPTATION.

Rewrite around Python package layout. A generic context should avoid one
hard-coded tree, but can express the shape:

```text
src/<package>/<domain>/
tests/<domain>/
```

Functional domain modules stay importable without runtime clients. Runtime
wiring lives in application factories, CLI entry points, ASGI startup, worker
entry points, or adapter packages.

## Tests

Disposition: PORTS_WITH_ADAPTATION.

Keep the behavior-first language. Rewrite mechanics to pytest:

- plain `assert`;
- `pytest.mark.parametrize`;
- fixtures for setup and teardown;
- `monkeypatch`, `tmp_path`, `caplog`;
- `pytest.raises`;
- approval or snapshot fixtures for large outputs.

Type-level expectations are checked by pyright or pyrefly, not by runtime
pytest alone.

## Debugging

Disposition: PORTS_AS_IS.

Keep almost unchanged. Python-specific additions:

- read full tracebacks including chained exceptions;
- inspect `__cause__` and `__context__`;
- use `faulthandler` for hangs;
- name async tasks and log request ids;
- treat cancellation as a control path, not a generic failure.

## Domain Ownership

Disposition: PORTS_AS_IS.

Keep. Replace `types.hpp` with `types.py` or domain model modules. Tell agents
not to pass raw dicts deep into the domain when a dataclass or parsed type is
available.

## Forward Dependencies

Disposition: PORTS_AS_IS.

Keep and add import cycles explicitly. Python agents should check imports when
adding a dependency and prefer extracting a lower-level module over circular
imports.

## Types Carry Proof

Disposition: PORTS_WITH_ADAPTATION.

Rewrite with restraint. Suggested always-loaded wording:

```text
Use type hints to make public contracts, domain ownership, and refactors safer.
Parse untyped input at boundaries into domain values. Use NewType, Literal,
Enum, frozen dataclasses, Annotated, Protocol, and assert_never where they make
the contract clearer. Escape typing at I/O boundaries, untyped third-party
libraries, dynamic dispatch that resists a clean Protocol, and prototypes.
Do not over-type local Python until the type layer becomes harder to read than
the code it protects.
```

## Strong-Type Ergonomics

Disposition: PORTS_WITH_ADAPTATION.

Replace C++ strong-type operator rules with Python guidance:

- use `NewType` for nominal IDs with no runtime behavior;
- use dataclasses for values with validation or behavior;
- do not unwrap to primitives except at serialization, database, wire, or UI
  boundaries;
- avoid wrapper classes that only create ceremony.

## Designated Initializers

Disposition: PORTS_WITH_ADAPTATION.

Map to keyword construction. Tell agents to avoid positional dataclass
construction for domain values with adjacent same-shaped fields.

## Functional Core, Imperative Shell

Disposition: PORTS_AS_IS.

Keep and make it central. Python-specific effects include import-time I/O,
global clients, framework dependency injection, task creation, environment
reads, ORM sessions, and logging.

## Pipelines

Disposition: PORTS_AS_IS.

Keep. Use callable fields, protocols, async callbacks, or queue sends. Warn
that an async callback must be awaited or scheduled at the wiring point.

## Runtime And Threads

Disposition: PORTS_WITH_ADAPTATION.

Rewrite around ownership by event loop, worker thread, or process. Add:

- `asyncio.TaskGroup` for sibling task ownership;
- cancellation discipline;
- `asyncio.Queue` and `queue.Queue`;
- `loop.call_soon_threadsafe`;
- GIL is not a data-invariant guarantee.

## Error Handling

Disposition: PYTHON_SPECIFIC_VARIANT.

Rewrite fully:

```text
Inside a trust boundary, use structured domain exceptions. At public
boundaries, catch and translate to the caller vocabulary: HTTP response, CLI
exit code, task log-and-swallow, retry, dead-letter event, or typed result
object. Domain exceptions carry attributes, not only message strings. Use
exception chaining when translating lower-level exceptions.
```

## Invariants And Rollback

Disposition: PORTS_AS_IS.

Keep. Replace scope guards with `ExitStack`, context managers, copied state,
transactions, and `try`/`except` rollback.

## Declarative Style

Disposition: PORTS_AS_IS.

Keep. Replace C++ algorithms/ranges with comprehensions, generator
expressions, `any`, `all`, `next`, `sorted(key=...)`, and `itertools`.

## Variants, Concepts, And Templates

Disposition: PORTS_WITH_ADAPTATION.

Rewrite as "unions, pattern matching, protocols, and generics". Use
`assert_never` for exhaustiveness pressure and PEP 695 syntax in new examples.

## State Machines

Disposition: PORTS_AS_IS.

Keep decision criteria and anti-patterns. Replace Boost.SML details with table
or library guidance.

## Cross-Cutting Services

Disposition: PORTS_WITH_ADAPTATION.

Replace variant-backed globals with explicit edge configuration, provider
modules, context managers, `ContextVar`, and pytest fixtures. Warn against
import-time configuration.

## Performance Discipline

Disposition: PORTS_WITH_ADAPTATION.

Keep clarity-first and measure-first. Replace bounded-storage details with
Python profiling, allocation, vectorization, caching, and native-boundary
guidance.

## Comments

Disposition: PORTS_AS_IS.

Keep. Add docstrings as the public API form and preserve the negative
documentation warning.

## Namespace Aliases

Disposition: PYTHON_SPECIFIC_VARIANT.

Rewrite as import alias rules. Prefer explicit imports. Use aliases only for
widely accepted conventions (`np`, `pd`) or repeated neighboring domain
vocabularies. Avoid `from module import *`.

## Const Placement

Disposition: DROP.

No Python analogue. Replace with formatting and naming rules from Ruff and the
project style.
