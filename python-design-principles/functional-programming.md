# Functional Programming

Python design benefits from a functional core: values enter, values leave, and
effects stay near the runtime boundary. Pure functions make domain rules easy
to test, reuse, parallelize, and review because the reader can understand the
result from the arguments.

Use functional style for business decisions, calculations, parsing results,
strategy selection, and data transformations. Use mutation where mutation is
the domain operation, and keep that mutation inside the object or service that
owns the invariant.

## Pure Core

A pure function receives the data it needs and returns the data it produces.
Keep environment reads, file loading, database sessions, operational logging,
module-global mutation, and clock access in the application shell.

```python
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True, slots=True, kw_only=True)
class OrderQuote:
    quantity: int
    limit_price: Decimal
    fee: Decimal


def quote_order(quantity: int, limit_price: Decimal, fee_rate: Decimal) -> OrderQuote:
    fee = quantity * limit_price * fee_rate
    return OrderQuote(quantity=quantity, limit_price=limit_price, fee=fee)
```

The application shell reads settings, chooses the fee rate, handles logging,
and persists the result. The core function only computes the quote.

When a calculation needs derived values, stage them in named locals before the
decision point:

```python
def can_fill_order(*, bid: Decimal, ask: Decimal, limit_price: Decimal) -> bool:
    spread = ask - bid
    crosses_top_of_book = limit_price >= ask
    spread_is_valid = spread >= Decimal("0")
    return spread_is_valid and crosses_top_of_book
```

Names such as `spread` and `crosses_top_of_book` carry domain meaning that a
compound condition hides.

## Sum Types And Match

Closed sets of domain events should make every case visible. Use enums,
literal-tagged dataclasses, or Pydantic discriminated unions, then consume them
with `match`. Put `assert_never` in the final arm so the type checker flags a
new case that still needs a branch.

```python
from dataclasses import dataclass
from typing import Literal, assert_never


@dataclass(frozen=True, slots=True)
class OrderAccepted:
    kind: Literal["accepted"]
    order_id: str


@dataclass(frozen=True, slots=True)
class OrderRejected:
    kind: Literal["rejected"]
    order_id: str
    reason: str


@dataclass(frozen=True, slots=True)
class OrderFilled:
    kind: Literal["filled"]
    order_id: str
    fill_quantity: int


type OrderEvent = OrderAccepted | OrderRejected | OrderFilled


def describe_event(event: OrderEvent) -> str:
    match event:
        case OrderAccepted(order_id=order_id):
            return f"accepted {order_id}"
        case OrderRejected(order_id=order_id, reason=reason):
            return f"rejected {order_id}: {reason}"
        case OrderFilled(order_id=order_id, fill_quantity=fill_quantity):
            return f"filled {order_id}: {fill_quantity}"
        case _ as unreachable:
            assert_never(unreachable)
```

Use this shape when the operation's contract covers every supported variant.
For open extension points, use registration, protocols, or plugin discovery
instead of pretending the set is closed.

## Higher-Order Functions

Pass behavior as a value when the behavior is a small policy, predicate,
projection, clock, identifier factory, serializer, or test probe. Prefer named
functions for domain rules and short lambdas for local projections.

```python
from collections.abc import Callable, Iterable


def active_orders(
    orders: Iterable[OrderQuote],
    predicate: Callable[[OrderQuote], bool],
) -> list[OrderQuote]:
    return [order for order in orders if predicate(order)]


def quantity_at_least(minimum_quantity: int) -> Callable[[OrderQuote], bool]:
    def predicate(order: OrderQuote) -> bool:
        return order.quantity >= minimum_quantity

    return predicate
```

Storing callables on long-lived objects is useful for pipeline callbacks,
scheduled work, test probes, and narrow policy hooks. When the callback is a
full dependency with several operations or its own lifecycle, use a `Protocol`
or a concrete service object.

## Closures And Late Binding

Python closures capture variable bindings rather than value snapshots. A
closure created inside a loop should bind the current value through a default
argument or a small factory.

```python
from collections.abc import Callable


def make_symbol_filters(symbols: list[str]) -> list[Callable[[str], bool]]:
    filters: list[Callable[[str], bool]] = []
    for symbol in symbols:
        filters.append(lambda candidate, symbol=symbol: candidate == symbol)
    return filters
```

Capture mutable state only when the state belongs to the closure's purpose.
For counters, caches, or deferred work queues, a small class with named fields
often reads better than a closure with hidden mutation.

## Dynamic Boundaries

Functional code can still sit behind dynamic Python. Reflection, `hasattr`,
dynamic imports, or dataframe operations are acceptable at adapter boundaries
when they keep the public contract smaller. Normalize the dynamic shape before
the value reaches the core.

Use a small adapter when implementations have several call shapes:

```python
from typing import Protocol


class Indicator(Protocol):
    def compute(self, prices: PriceFrame, reference_data: ReferenceData) -> PriceFrame:
        ...


@dataclass(frozen=True, slots=True, kw_only=True)
class CloseOnlyIndicator:
    name: str

    def compute(self, prices: PriceFrame, reference_data: ReferenceData) -> PriceFrame:
        return compute_close_signal(prices, self.name)
```

The adapter gives callers one typed operation. The dynamic detail stays close
to the extension boundary and has focused tests.

## Review Checks

- Core functions receive explicit inputs and return explicit outputs.
- Domain decisions use named predicates and staged values.
- Closed sets use `match` plus `assert_never` where the type checker can help.
- Callables represent small behavior; richer dependencies use protocols or
  service objects.
- Closures created in loops bind the current value intentionally.
- Dynamic dispatch has a named boundary and tests for each supported shape.
