# Python Version And Typing Stack

Python guidelines should choose a modern baseline without turning typing into
a second implementation. Use Python 3.14 as the default baseline for new guide
examples and new projects. Treat Python 3.13 compatibility as a migration
constraint for existing projects.

Static typing is most valuable where it names stable contracts: public package
APIs, domain values, adapters, settings, message shapes, reusable helpers, and
test utilities. Dynamic Python remains appropriate in validated dataframe
work, exploratory code, plugin discovery, and narrow adapter points where the
type system would add ceremony without useful proof.

## Version Baseline

Target Python 3.14 for new projects unless deployment constraints require an
older interpreter. A project that supports Python 3.13 should state that
support in package metadata, CI, and examples.

Prefer modern syntax in Python 3.14 code:

```python
from collections.abc import Iterable, Mapping

type JsonScalar = str | int | float | bool | None


def first_item[T](items: Iterable[T]) -> T | None:
    for item in items:
        return item
    return None
```

Use built-in collection generics, `X | None`, `type` aliases, and PEP 695 type
parameters. Use type parameter defaults when they simplify a real public API,
not to make local code generic by habit.

Python 3.14 uses PEP 649 deferred annotation evaluation. New Python 3.14 code
can use direct annotations without quoting names solely to avoid eager
annotation evaluation. `TYPE_CHECKING` blocks still matter for import cycles
and runtime import cost. Runtime annotation consumers should use supported
inspection APIs instead of assuming raw `__annotations__` contains evaluated
runtime objects.

## Type Checker Policy

Use Pyright as the default checker because it has strong standards alignment,
editor support, and useful incremental behavior. Configure it in
`pyproject.toml` or `pyrightconfig.json`, and keep the chosen surface clear.

Start from a useful middle ground:

- Type public package APIs.
- Type domain values, command objects, result objects, and settings.
- Type adapters after boundary parsing.
- Type reusable helpers, fixtures, fakes, and assertion utilities.
- Move stable packages toward stricter checking by directory.

Use mypy alongside Pyright when the project already depends on mypy, when a
plugin provides important coverage, or when existing CI and contributor habits
make it worthwhile. Define the response policy for disagreements so contributors
know which diagnostic blocks a change.

Evaluate ty or pyrefly when fast whole-project feedback matters. Treat a
checker change as a project policy decision: diagnostics, inference,
configuration, editor support, and ignore comments all affect the team.

## Gradual Typing

Python's type system is gradual. `Any` is the explicit dynamic escape hatch,
and `object` is the type for values that allow only universally valid
operations. Use the distinction deliberately:

```python
def normalize_payload(payload: object) -> OrderRequest:
    if not isinstance(payload, Mapping):
        raise TypeError("payload must be a mapping")
    return OrderRequest.model_validate(payload)
```

Let untyped data enter at hostile boundaries, then normalize it quickly with a
Pydantic model, dataclass constructor, `TypedDict`, protocol adapter, Pandera
schema, or a small parser. Avoid letting `Any` flow into the domain core.

Use `cast` only after a nearby runtime check, a trusted library contract, or a
named invariant. Repeated casts around the same third-party library point to a
typed adapter or local stub.

Keep ignore comments narrow and checker-specific. An ignore around project
model access usually means the model contract needs work.

## Boundary Models

Choose the data carrier by boundary:

- Frozen slotted dataclasses for internal value objects built from already
  validated parts.
- Mutable dataclasses when mutation is the domain operation and the owner
  protects the invariant.
- Pydantic v2 models for HTTP, CLI, JSON, environment, config, queue, and
  cross-process boundaries.
- Pandera schemas for dataframe boundaries.
- `TypedDict` for dictionary-shaped data when a class would add ceremony
  without behavior.
- `NewType` for identity-only primitive distinctions such as account IDs or
  security IDs.
- A small validated class or dataclass when a primitive carries proof.

Prefer `Protocol` when the caller depends on behavior rather than concrete
class identity. Protocols fit repositories, clocks, serializers, storage
clients, plugin capabilities, and fakes.

## Dataframe-Heavy Code

Dataframe-heavy code benefits from scalar typing around the pipeline and
runtime validation at dataframe boundaries. Type file paths, settings, domain
identifiers, strategy objects, and scalar return values. Use Pandera, tests,
and named invariant checks for dataframe shape, dtypes, nullability, ordering,
and cross-column relationships.

Do not contort pandas, NumPy, Polars, SQL, or ibis code into elaborate type
aliases the checker cannot prove. Keep transformations native to the tool and
validate the boundary where the shape becomes a contract.

## Tests

Type test helpers when they are reused, non-obvious, or domain-shaped. Fixture
factories, fake implementations, assertion helpers, and dataframe builders
often benefit from annotations. Short test bodies can stay concise when
literals, assertions, and pytest names make the behavior clear.

Invalid-input tests may need local ignores or dynamic inputs to exercise
runtime validation. Keep those escapes inside the test that needs them.

## Migration

For existing Python 3.13 projects, adopt the stack in layers:

1. Add or align `.python-version`, `requires-python`, and CI interpreters.
2. Make `uv run` the command surface.
3. Run Ruff format and Ruff lint over an agreed scope.
4. Add Pyright in standard mode.
5. Type public boundaries and reusable helpers first.
6. Ratchet stricter checks by package as defects disappear.
7. Use PEP 695 syntax where the supported Python version allows it.

Migration should improve review and defect detection without freezing dynamic
areas that are clearer as Python.
