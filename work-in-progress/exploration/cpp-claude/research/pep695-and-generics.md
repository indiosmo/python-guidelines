# PEP 695 and generics in 2026

## Status

PEP 695 (type parameter syntax) is part of Python 3.12. Pyright supports it
fully. Pyrefly supports it. mypy: partial (TypeAliasType in 1.11+, generic
class/function syntax landing later). For guidelines targeting 3.13+/3.14, use
PEP 695 syntax as the default.

## Recommended syntax for the Python guidelines

```python
# Generic type alias - PEP 695
type Result[T] = T | Exception          # not quite the right shape; see below
type Maybe[T] = T | None

# Generic class - PEP 695
class Stack[T]:
    def push(self, item: T) -> None: ...
    def pop(self) -> T: ...

# Generic function - PEP 695
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

# Bound type parameters
class Sorted[T: SupportsLessThan]: ...

# Constrained type parameters
def parse[T: (int, float, str)](s: str) -> T: ...
```

## Equivalents to drop

- `TypeVar("T")` + `Generic[T]` -- legacy; use `class Foo[T]:` instead.
- `TypeAlias` (PEP 613) -- replaced by `type X = ...`.
- `Generic[T, S]` base class -- replaced by `class Foo[T, S]:`.

`TypeVar` and `Generic` are not removed; PEP 695 just gives a cleaner spelling.
For runtime introspection, `type X = ...` produces a `TypeAliasType` object,
which keeps the alias name visible in error messages and runtime tools (a
small but real ergonomic win over the old `X: TypeAlias = ...`).

## Sources

- https://peps.python.org/pep-0695/
- https://jellezijlstra.github.io/pep695.html
- https://github.com/microsoft/pyright/issues/5108
