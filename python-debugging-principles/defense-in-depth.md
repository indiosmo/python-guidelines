# Defense In Depth

A root-cause fix removes the original defect. Defense in depth removes the
class of defect from the surrounding paths. Each layer enforces the invariant
it owns: boundary parsing handles untrusted input, domain validation handles
business rules, environment guards handle runtime context, assertions mark
internal impossibility, and diagnostics preserve evidence.

The layers should reinforce each other without repeating the same check in
every function. Once a boundary has parsed a value into a domain type, internal
code can rely on that type and focus on the next layer's rule.

## Layer 1: Parse At The Boundary

Raw input enters as strings, bytes, JSON objects, CLI arguments, environment
variables, database rows, queue messages, or dataframes. Convert that input
into typed Python values near the boundary, then pass those values inward.

```python
from typing import Literal

from pydantic import BaseModel, Field


class OrderPayload(BaseModel, frozen=True):
    account_id: str = Field(min_length=1)
    symbol: str = Field(min_length=1)
    quantity: int = Field(gt=0)
    side: Literal["buy", "sell"]


def parse_order_payload(payload: object) -> OrderPayload:
    return OrderPayload.model_validate(payload)
```

Use the parser that fits the boundary:

- Pydantic models for JSON, HTTP, CLI, environment, config, and queue payloads.
- Dataclass constructors or small validated classes for internal values made
  from already checked parts.
- Pandera schemas for dataframe boundaries.
- `TypedDict`, `NewType`, enums, and `Literal` where they make contracts clear.

Boundary parsing gives later code a real contract. It also gives invalid input
a stable failure point.

## Layer 2: Validate Domain Rules

Some rules depend on relationships, state, or a specific operation. Put those
checks in the type, service, or function that owns enough context to decide.

```python
def validate_rebalance_request(
    request: OrderPayload,
    account: Account,
    universe: SecurityUniverse,
) -> None:
    if request.symbol not in universe.symbols:
        raise UnknownSecurityError(symbol=request.symbol)

    required_cash = estimate_required_cash(request)
    if account.available_cash < required_cash:
        raise InsufficientCashError(
            account_id=request.account_id,
            required_cash=required_cash,
            available_cash=account.available_cash,
        )
```

Keep each rule visible. One named predicate or short block per rule makes
failures easier to test and diagnose. Error payloads should carry the domain
context a boundary, log line, retry policy, or test needs.

## Layer 3: Guard The Environment

Some operations are valid only in the right runtime context. Startup code
should validate settings before domain work starts, and dangerous operations
should check the configured mode before performing side effects.

```python
from dataclasses import dataclass
from typing import Literal


@dataclass(frozen=True, slots=True, kw_only=True)
class RuntimeSettings:
    environment: Literal["local", "staging", "production"]
    dry_run: bool
    order_gateway_url: str


def submit_order(order: Order, settings: RuntimeSettings, gateway: OrderGateway) -> None:
    if settings.dry_run:
        logger.info(
            "order submission skipped for dry run",
            extra={"order_id": order.order_id},
        )
        return

    if settings.environment != "production" and order.route == "production":
        raise RuntimeError(
            "non-production environment cannot use production order route"
        )

    gateway.submit(order)
```

Environment guards fit CLI entry points, web app startup, worker initializers,
Dagster resources, scheduled jobs, and integration-test fixtures. They protect
resources that a valid domain value can still affect incorrectly.

## Layer 4: Assert Internal Impossibility

Use assertions for programmer errors inside trusted code paths. Use explicit
exceptions for invalid external input and expected domain failures.

```python
from typing import assert_never


def annualization_factor(period: SeriesPeriod) -> int:
    match period:
        case SeriesPeriod.DAILY:
            return 252
        case SeriesPeriod.MONTHLY:
            return 12
        case SeriesPeriod.QUARTERLY:
            return 4
        case _ as unreachable:
            assert_never(unreachable)
```

Assertions and `assert_never` are useful when the type checker and the runtime
contract agree that a path is unreachable. They are not a boundary validation
strategy because optimized Python can remove `assert` statements.

## Layer 5: Preserve Evidence

Diagnostics make the next investigation short. Log at the boundary where the
operation is accepted, translated, retried, or stopped. Include stable event
names and structured context.

```python
logger.info(
    "rebalance order accepted",
    extra={
        "account_id": request.account_id,
        "symbol": request.symbol,
        "quantity": request.quantity,
    },
)
```

Use warnings for recoverable conditions callers can choose to escalate, such
as deprecated inputs or questionable but accepted configuration. In tests and
CI, run warning-sensitive checks so accepted warnings stay intentional.

## Test The Layers

Tests should exercise each layer directly:

- Boundary parsing rejects malformed external input.
- Domain validation rejects a typed value that violates a business rule.
- Environment guards refuse dangerous runtime combinations.
- Assertions and exhaustive checks cover closed internal states.
- Logs or warnings carry the stable context operators and tests need.

A good regression test follows the path that originally failed. Layer tests
then protect alternate paths that could reintroduce the same bug class.
