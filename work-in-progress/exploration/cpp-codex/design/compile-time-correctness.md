# Compile-time correctness

Overall disposition: PORTS_WITH_ADAPTATION.

Rename for Python to `types-and-static-analysis.md`. Python does not have C++
compile-time enforcement, but static analysis can still carry useful proof.
The chapter should teach where to use types and where to stop.

## Strong typing

Disposition: PORTS_WITH_ADAPTATION.

Use `NewType` for cheap nominal distinctions, frozen dataclasses for values
with behavior or validation, `Annotated` for metadata consumed by validators,
and `Literal` for small closed sets.

```python
AccountId = NewType("AccountId", str)
Symbol = NewType("Symbol", str)
Quantity = NewType("Quantity", int)


def place_order(account_id: AccountId, symbol: Symbol, quantity: Quantity) -> None:
    ...
```

For runtime validation, prefer a constructor or parser:

```python
@dataclass(frozen=True)
class Symbol:
    value: str

    @classmethod
    def parse(cls, raw: str) -> "Symbol":
        if not raw:
            raise InvalidSymbol(raw)
        return cls(raw)
```

## Per-domain `types` namespace

Disposition: PORTS_WITH_ADAPTATION.

Replace with per-domain `types.py` or `models.py` modules. Keep ownership
visible in imports.

```python
from routing import types as routing_types


@dataclass(frozen=True)
class Message:
    symbol: routing_types.Symbol
```

Avoid pulling many type names into shared utility modules.

## Parse, don't validate

Disposition: PORTS_AS_IS.

Keep the section. Python should parse untrusted input into domain objects at
the boundary and let inner code accept refined types.

```python
def parse_symbol(raw: str) -> Symbol:
    if not SYMBOL_PATTERN.fullmatch(raw):
        raise InvalidSymbol(raw)
    return Symbol(raw)
```

Escape typing at I/O boundaries, untyped third-party APIs, dynamic dispatch
that resists a clean `Protocol`, and prototypes. Do not over-type local
plumbing just to imitate C++.

## Data structs as plain aggregates

Disposition: PORTS_WITH_ADAPTATION.

Use dataclasses, `attrs`, Pydantic, msgspec, or plain dictionaries based on the
boundary. For domain data, prefer dataclasses with clear mutability:

```python
@dataclass(frozen=True, slots=True)
class OrderPlaced:
    order_id: OrderId
    request_id: RequestId
```

`frozen=True` is useful for value objects, but mutable domain state should be
plainly mutable behind methods. Do not freeze everything by reflex.

## Exhaustiveness checking

Disposition: PORTS_WITH_ADAPTATION.

Use `Enum`, `Literal`, unions, `match`, and `assert_never` where type checkers
can help.

```python
def to_wire(side: Side) -> str:
    match side:
        case Side.BUY:
            return "1"
        case Side.SELL:
            return "2"
    assert_never(side)
```

Boundary parsers still need an explicit reject for unknown primitive values.

## Return a structured error on fallthrough

Disposition: PYTHON_SPECIFIC_VARIANT.

Raise a structured domain exception at runtime boundaries. `assert_never`
is for type-checker exhaustiveness, not user input.

## When `std::unreachable()` is still appropriate

Disposition: PYTHON_SPECIFIC_VARIANT.

Use `assert_never` for type-checker unreachable paths and `assert False` only
for programmer errors in code that should be impossible after static checking.
Never use assertions for caller-visible invalid input.

## `if/else` chains need an explicit `else`

Disposition: PORTS_AS_IS.

Keep it. If a chain is intended to cover all cases, finish with an explicit
error or `assert_never`.

## Designated initializers

Disposition: PORTS_WITH_ADAPTATION.

Python has keyword arguments. Keep the intent: named construction, defaults in
one place, and swap resistance.

```python
config = ServerConfig(
    listen_address="0.0.0.0",
    listen_port=8080,
    reuse_address=True,
    backlog=1024,
)
```

Omit fields with defaults. Avoid positional construction for dataclasses with
more than one or two same-shaped fields.
