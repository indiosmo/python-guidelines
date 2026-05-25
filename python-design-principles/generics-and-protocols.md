# Generics And Protocols

Generics and protocols describe reusable contracts without forcing concrete
inheritance. Use them where they make a public API, helper, fake, decorator, or
extension point clearer to the reader and to the type checker.

Python 3.14 examples should use current typing syntax: PEP 695 type
parameters, `type` aliases, `Protocol`, `ParamSpec`, `TypeIs`, and
`assert_never` where they fit. Keep generic code small and purposeful. A
concrete function is clearer than a generic abstraction whose only caller has
one type.

## Generic Functions And Classes

Use PEP 695 syntax for new generic functions, classes, and type aliases.

```python
from collections.abc import Iterable


type NonEmptyBatch[T] = tuple[T, ...]


def first[T](items: Iterable[T]) -> T | None:
    for item in items:
        return item
    return None


class InMemoryRepository[T]:
    def __init__(self) -> None:
        self._items: dict[str, T] = {}

    def put(self, item_id: str, item: T) -> None:
        self._items[item_id] = item

    def get(self, item_id: str) -> T | None:
        return self._items.get(item_id)
```

Use PEP 696 type parameter defaults when they reduce public API boilerplate:

```python
class Page[T = str]:
    items: tuple[T, ...]
```

Avoid default type parameters in local code that has one concrete caller.

Add bounds when the body relies on a capability:

```python
from typing import Protocol


class HasTradeDate(Protocol):
    trade_date: date


def latest_by_trade_date[T: HasTradeDate](items: Iterable[T]) -> T:
    return max(items, key=lambda item: item.trade_date)
```

The bound documents the operation the function needs. It also lets callers use
dataclasses, Pydantic models, named tuples, or test fakes that expose the same
field.

## Protocols

Use `Protocol` when the caller depends on behavior rather than class identity.
Protocols fit clocks, repositories, serializers, storage clients, dataframe
loaders, strategy objects, and fakes.

```python
from datetime import datetime
from typing import Protocol


class Clock(Protocol):
    def now(self) -> datetime:
        ...


class OrderStore(Protocol):
    def save(self, order: Order) -> None:
        ...

    def get(self, order_id: str) -> Order | None:
        ...
```

A protocol should be as small as the caller's need. Split read and write
capabilities when tests or production code commonly use one side alone.

Use nominal abstract base classes when membership itself is domain meaning, a
shared implementation matters, or a framework requires inheritance. Use a
protocol for structural capability.

## TypeIs And TypeGuard

Use `TypeIs` for predicates that narrow both the true and false branches. Use
`TypeGuard` when a predicate narrows only the true branch or when compatibility
with older typing behavior matters.

```python
from typing import TypeIs


def is_filled_event(event: OrderEvent) -> TypeIs[OrderFilled]:
    return isinstance(event, OrderFilled)


def filled_quantity(event: OrderEvent) -> int:
    if is_filled_event(event):
        return event.fill_quantity
    return 0
```

Prefer predicates that name domain meaning. A helper named
`is_tradeable_symbol` tells more than an inline `isinstance` and length check.

Runtime structural checks with `isinstance` require `@runtime_checkable` on the
protocol. Use that form for narrow boundary checks and keep hot-path capability
dispatch simple.

## ParamSpec And Decorators

Decorators that wrap callables should preserve the wrapped signature unless
the decorator intentionally changes it. Use `ParamSpec` and `functools.wraps`
for the common case.

```python
from collections.abc import Callable
from functools import wraps
from time import perf_counter


def timed[**P, R](
    record: Callable[[str, float], None],
) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorate(function: Callable[P, R]) -> Callable[P, R]:
        @wraps(function)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            started_at = perf_counter()
            try:
                return function(*args, **kwargs)
            finally:
                record(function.__qualname__, perf_counter() - started_at)

        return wrapper

    return decorate
```

Use `Concatenate` when a decorator adds or removes a leading argument, such as
a request context or lock. Keep such decorators rare; a plain function call is
often easier to read.

## Callable Protocols

Use `Callable` for simple function-shaped parameters. Use a callable protocol
when the callable has attributes, overloads, keyword-only parameters, or a
meaningful name in the domain.

```python
from typing import Protocol


class ScoreFunction(Protocol):
    name: str

    def __call__(self, market_data: MarketDataFrame) -> ScoreFrame:
        ...
```

This shape gives a strategy a domain name and still lets tests pass a small
fake object.

## Variadic Generics

Use `TypeVarTuple` and unpacked type parameters for array-like or tuple-like
APIs where the shape is part of the contract. Keep ordinary application code
with simple `tuple[...]`, dataclasses, or dataframe schemas unless the extra
typing proves a real relationship.

```python
from typing import TypeVarTuple


Shape = TypeVarTuple("Shape")


class Tensor[*Shape]:
    ...
```

Most business code gains more from clear domain models than from highly
generic shape typing.

## Review Checks

- Generic parameters appear because more than one concrete type is expected.
- Bounds and protocols name the operations the body actually uses.
- Protocols stay small and capability-focused.
- Decorators preserve signatures with `ParamSpec` and `wraps`.
- Type narrowing predicates carry domain names and use `TypeIs` when both
  branches benefit.
- Advanced typing reduces ambiguity for callers instead of creating a second
  implementation in annotations.
