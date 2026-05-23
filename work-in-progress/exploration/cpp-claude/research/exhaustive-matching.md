# Exhaustive matching in Python (2026)

## The pattern

```python
from typing import assert_never
from enum import Enum

class Side(Enum):
    BUY = "buy"
    SELL = "sell"
    SELL_SHORT = "sell_short"

def to_wire(side: Side) -> str:
    match side:
        case Side.BUY: return "1"
        case Side.SELL: return "2"
        case Side.SELL_SHORT: return "5"
        case _ as unreachable:
            assert_never(unreachable)
```

If a new `Side` member is added, pyright/pyrefly flag the `assert_never` call
because the discarded `_` is no longer typed as `Never`. mypy with
`--enable-error-code exhaustive-match` does the same.

## Discriminated unions (the variant equivalent)

```python
from dataclasses import dataclass
from typing import Literal, assert_never

@dataclass(frozen=True, slots=True)
class OrderPlaced:
    kind: Literal["placed"] = "placed"
    order_id: str = ""
    request_id: str = ""

@dataclass(frozen=True, slots=True)
class OrderFilled:
    kind: Literal["filled"] = "filled"
    order_id: str = ""

@dataclass(frozen=True, slots=True)
class OrderCanceled:
    kind: Literal["canceled"] = "canceled"
    order_id: str = ""
    request_id: str = ""

type OrderEvent = OrderPlaced | OrderFilled | OrderCanceled

def describe(event: OrderEvent) -> str:
    match event:
        case OrderPlaced(order_id=oid, request_id=rid):
            return f"placed {oid} (req {rid})"
        case OrderFilled(order_id=oid):
            return f"filled {oid}"
        case OrderCanceled(order_id=oid):
            return f"canceled {oid}"
        case _ as unreachable:
            assert_never(unreachable)
```

The `Literal[...]` `kind` field is the Python equivalent of the C++
`static constexpr` discriminator pattern -- it lets serializers and
deserializers route on a single field without runtime `isinstance` chains.

## Capability dispatch (C++ `if constexpr (requires { ... })`)

Python equivalent is a runtime `hasattr` check inside the match arm, or a
union-narrowing structural check:

```python
def request_id_of(event: OrderEvent) -> str | None:
    # capability dispatch
    if hasattr(event, "request_id"):
        return event.request_id
    return None
```

For type-checker-friendly capability dispatch, define a `Protocol`:

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class HasRequestId(Protocol):
    request_id: str

def request_id_of(event: OrderEvent) -> str | None:
    if isinstance(event, HasRequestId):
        return event.request_id
    return None
```

The protocol form gives the type checker enough information to narrow inside
the branch. `runtime_checkable` adds a small overhead but is necessary for
the `isinstance` use.

## Sources

- https://typing.python.org/en/latest/guides/unreachable.html
- https://adamj.eu/tech/2022/10/14/python-type-hints-exhuastiveness-checking/
- https://github.com/python/mypy/issues/13597
