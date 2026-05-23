# Agent examples rewrite analysis

Overall disposition: PORTS_WITH_ADAPTATION.

The examples file should remain an on-demand companion. Re-render examples in
Python and keep each pair short enough for an agent to load when shaping a
specific edit. Do not keep C++ examples in the Python edition except where the
topic is explicitly a native-extension boundary.

## Tests

Disposition: PORTS_WITH_ADAPTATION.

Use pytest:

```python
def test_rectangle_area() -> None:
    rectangle = Rectangle(width=6, height=4)
    assert rectangle.area() == 24
```

Bad remains expected-value rederivation.

## Debugging

Disposition: PORTS_AS_IS.

Keep the text loop exactly in Python terms:

```text
observe -> trace source -> state one hypothesis -> run one experiment
-> write regression test -> fix root cause -> run relevant tests
```

## Domain Ownership

Disposition: PORTS_WITH_ADAPTATION.

Render with `NewType`, enums, and dataclasses.

```python
UserId = NewType("UserId", int)
UserOrderId = NewType("UserOrderId", int)


class Side(Enum):
    BUY = "buy"
    SELL = "sell"


@dataclass(frozen=True)
class NewOrder:
    user_id: UserId
    order_id: UserOrderId
    symbol: Symbol
    side: Side
    quantity: Quantity
```

Bad: raw `str` and `int` everywhere, or a shared `common_types.py` that erases
domain ownership.

## Forward Dependencies

Disposition: PORTS_AS_IS.

Keep the explicit input/output example and add an import-cycle bad example.

## Types Carry Proof

Disposition: PORTS_WITH_ADAPTATION.

Show parse once, trust refined values:

```python
def parse_symbol(raw: str) -> Symbol: ...


def place_order(account_id: AccountId, symbol: Symbol, quantity: Quantity) -> None:
    ...
```

Bad: `validate_symbol(raw) -> bool` followed by raw strings downstream.

## Strong-Type Ergonomics

Disposition: PORTS_WITH_ADAPTATION.

Examples should show no primitive unwrap until serialization:

```python
payload = {"order_id": str(order_id)}
```

Do not mimic C++ `.get()` examples.

## Designated Initializers

Disposition: PORTS_WITH_ADAPTATION.

Use keyword dataclass construction.

```python
config = ServerConfig(
    listen_address="0.0.0.0",
    listen_port=8080,
    reuse_address=True,
    backlog=1024,
)
```

Bad: positional construction with many same-shaped fields.

## Functional Core, Imperative Shell

Disposition: PORTS_AS_IS.

Good:

```python
def apply_fill(order: OrderState, fill: Fill) -> OrderState:
    return replace(order, filled_quantity=order.filled_quantity + fill.quantity)
```

Bad: domain function opens a database session, logs, and starts a task.

## Pipelines

Disposition: PORTS_AS_IS.

Good:

```python
@dataclass
class Session:
    on_request: Callable[[Request], None]
    on_rejected: Callable[[Rejection], None]

    def send(self, datagram: bytes) -> None:
        ...
```

Bad: `Session` imports and directly calls `Engine`.

## Runtime And Threads

Disposition: PORTS_WITH_ADAPTATION.

Examples should cover:

```python
loop.call_soon_threadsafe(consumer.send, event)
```

and async queues:

```python
await queue.put(event)
```

Bad: direct cross-thread mutation of a component's internal dict.

## Error Handling

Disposition: PYTHON_SPECIFIC_VARIANT.

Good:

```python
@dataclass(frozen=True)
class UnknownOrder(RoutingError):
    order_id: OrderId


def handle(request: Request) -> Response:
    try:
        return build_response(route(request))
    except UnknownOrder as error:
        return reject("routing.unknown_order", order_id=error.order_id)
```

Bad: return `False` and log, or raise `Exception("failed")` with no typed
context.

## Invariants And Rollback

Disposition: PORTS_WITH_ADAPTATION.

Use `ExitStack` examples.

```python
with ExitStack() as rollback:
    orders[request.order_id] = order_state
    rollback.callback(orders.pop, request.order_id, None)
    limits.reserve(request)
    rollback.pop_all()
```

## Declarative Style

Disposition: PORTS_AS_IS.

Use staged variables, comprehensions, and named predicates.

```python
has_residual = order.remaining_quantity > 0
is_market = order.limit_price is None
should_rest = has_residual and not is_market
```

## Variants, Concepts, And Templates

Disposition: PORTS_WITH_ADAPTATION.

Use union pattern matching and protocols.

```python
def send(request: NewOrder | CancelOrder) -> None:
    match request:
        case NewOrder():
            handle_new_order(request)
        case CancelOrder():
            handle_cancel_order(request)
        case _:
            assert_never(request)
```

## State Machines

Disposition: PORTS_AS_IS.

Good: explicit transition table. Bad: correlated booleans.

## Cross-Cutting Services

Disposition: PORTS_WITH_ADAPTATION.

Good: application edge configures provider and tests restore with
`monkeypatch`. Bad: domain code constructs its own real clock or global SDK
client.

## Performance Discipline

Disposition: PORTS_WITH_ADAPTATION.

Good examples should cite a benchmark or profile before micro-optimizing.
Bad examples should show premature cleverness in cold code.

## Comments

Disposition: PORTS_AS_IS.

Keep good and bad comment categories. Add docstring examples that state API
contract instead of restating parameters.

## Namespace Aliases

Disposition: PYTHON_SPECIFIC_VARIANT.

Good:

```python
from market_data import types as market_data_types
```

Accept standard aliases such as `np` and `pd` when the project uses them.
Bad: wildcard imports or ad hoc aliases that hide domain ownership.

## Const Placement

Disposition: DROP.

Do not port. Replace with formatter/linter examples if the Python agent context
needs a style section.
