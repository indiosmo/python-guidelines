# Functional programming

Overall disposition: PORTS_WITH_ADAPTATION.

The functional framing ports, but Python has different enforcement. The guide
should push pure value functions, immutable dataclasses where useful,
generators, and explicit side-effect boundaries. It should avoid turning
Python into a monad tutorial.

## Pure functions and value semantics

Disposition: PORTS_AS_IS.

Keep the principle. Python examples should prefer values in, values out, no
ambient globals, and no logging in the inner calculation.

```python
def apply_discount(total: Money, discount_percent: int) -> Money:
    return total - total * discount_percent // 100
```

Use frozen dataclasses for values that should not be mutated:

```python
@dataclass(frozen=True)
class OrderState:
    total: Money
    filled_quantity: Quantity
```

## Pattern matching on sum types

Disposition: PORTS_WITH_ADAPTATION.

`std::variant` becomes a union of dataclasses, often with structural pattern
matching and `typing.assert_never` for exhaustiveness pressure.

```python
from typing import assert_never


type OrderEvent = OrderPlaced | OrderFilled | OrderCanceled


def describe(event: OrderEvent) -> str:
    match event:
        case OrderPlaced(request_id=request_id):
            return f"placed {request_id}"
        case OrderFilled():
            return "filled"
        case OrderCanceled(request_id=request_id):
            return f"canceled {request_id}"
    assert_never(event)
```

This depends on type-checker support. The runtime will not enforce
exhaustiveness, so tests still matter.

## Higher-order functions

Disposition: PORTS_WITH_ADAPTATION.

Keep the idea. Python already treats callables as values, so the chapter
should focus on readability and type shape:

```python
type Predicate[T] = Callable[[T], bool]


def count_matching[T](items: Iterable[T], predicate: Predicate[T]) -> int:
    return sum(1 for item in items if predicate(item))
```

Prefer a named function over dense `functools.partial` or nested lambdas when
the behavior has domain meaning.

## A small in-house functional vocabulary

Disposition: PORTS_WITH_ADAPTATION.

Be more conservative in Python than C++. The standard library already has
`itertools`, `functools`, `operator`, comprehensions, and generators. Add a
helper only when it makes domain code read better across several call sites.

## Storing callables

Disposition: PORTS_WITH_ADAPTATION.

Use `Callable`, `Protocol`, or a small dataclass dependency. Python has no
`std::function` allocation choice, but it still has lifetime and capture
issues.

```python
class OrderSink(Protocol):
    def __call__(self, order: RoutedOrder) -> None: ...


@dataclass
class Router:
    on_routed: OrderSink
```

For async callbacks, spell the awaitable:

```python
type AsyncSink[T] = Callable[[T], Awaitable[None]]
```

## Compile-time predicates and folds

Disposition: PYTHON_SPECIFIC_VARIANT.

Rewrite this as "type-level tools when they clarify contracts": `Protocol`,
`TypeGuard`/`TypeIs`, overloads, constrained `TypeVar`, and PEP 695 type
parameters. Python does not have compile-time folds over parameter packs in
the C++ sense.

## When functional style costs you

Disposition: PORTS_WITH_ADAPTATION.

Keep the section with Python-specific costs:

- Generator pipelines are lazy and single-use.
- Python function calls are not free on tight loops.
- `map`/`filter` chains are not automatically faster than comprehensions.
- Vectorized libraries, native extensions, or specialized algorithms may be
  needed for measured hot paths.
- Closures still capture references to objects, so mutable captured state can
  surprise readers.

```python
def make_predicate(prefix: str) -> Callable[[str], bool]:
    return lambda value: value.startswith(prefix)
```

This captures the string object safely because strings are immutable. Capturing
a mutable list or dict deserves more scrutiny.
