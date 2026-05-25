# Python Agent Examples

Good and bad examples for the rules in
[python-agent-context.md](python-agent-context.md). Load this file when the
rule alone is not enough to shape an edit or review a concrete patch.

## Domain Ownership

Good: the domain owns named values.

```python
from dataclasses import dataclass
from datetime import date


@dataclass(frozen=True, slots=True, kw_only=True)
class RebalanceRequest:
    portfolio_id: str
    trade_date: date
    gross_exposure: float
    net_exposure: float
```

Bad: loose primitives hide roles and ownership.

```python
def rebalance(id: str, day: str, a: float, b: float) -> None:
    ...
```

## Boundary Parsing

Good: parse foreign input once and return a domain value.

```python
from dataclasses import dataclass
from datetime import date
from typing import Any

from pydantic import BaseModel, Field


class VendorPriceRow(BaseModel):
    trade_date: date = Field(alias="data")
    symbol: str = Field(alias="ticker")
    close_price: float = Field(alias="fechamento")


@dataclass(frozen=True, slots=True, kw_only=True)
class MarketClose:
    trade_date: date
    symbol: str
    close_price: float


def parse_market_close(payload: dict[str, Any]) -> MarketClose:
    row = VendorPriceRow.model_validate(payload)
    return MarketClose(
        trade_date=row.trade_date,
        symbol=row.symbol,
        close_price=row.close_price,
    )
```

Bad: vendor aliases leak into the domain.

```python
def close_price(payload: dict[str, object]) -> float:
    return float(payload["fechamento"])
```

## Functional Core

Good: pure logic accepts and returns values.

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True, kw_only=True)
class WeightedSignal:
    symbol: str
    score: float
    weight: float


def normalize_weights(signals: list[WeightedSignal]) -> list[WeightedSignal]:
    total = sum(signal.score for signal in signals)
    if total <= 0:
        raise ValueError("total score must be positive")

    return [
        WeightedSignal(
            symbol=signal.symbol,
            score=signal.score,
            weight=signal.score / total,
        )
        for signal in signals
    ]
```

Bad: core logic reaches into runtime state.

```python
def normalize_weights() -> list[WeightedSignal]:
    signals = database.fetch_signals(settings.trade_date)
    logger.info("normalizing weights")
    ...
```

## Error Handling

Good: a domain exception carries structured facts.

```python
from dataclasses import dataclass
from datetime import date


class MarketDataError(Exception):
    pass


@dataclass(frozen=True, slots=True)
class MissingClosePrice(MarketDataError):
    trade_date: date
    market: str
    symbol: str

    def __str__(self) -> str:
        return f"missing close price for {self.symbol} on {self.trade_date}"
```

Bad: the caller must parse a generic string.

```python
raise Exception(f"bad data: {market}:{symbol}:{trade_date}")
```

## Boundary Translation

Good: translate at the CLI boundary.

```python
def run_rebalance_command(args: RebalanceArgs) -> int:
    try:
        request = parse_rebalance_args(args)
        rebalance_portfolio(request)
    except ValueError as exc:
        print(f"invalid input: {exc}", file=sys.stderr)
        return 2
    except MarketDataError as exc:
        print(f"market data error: {exc}", file=sys.stderr)
        return 3
    return 0
```

Bad: log and re-raise the same exception at an intermediate layer.

```python
try:
    rebalance_portfolio(request)
except MarketDataError:
    logger.exception("rebalance failed")
    raise
```

## Tests

Good: expected values come from the behavior being specified.

```python
def test_rectangle_area() -> None:
    rectangle = Rectangle(width=6, height=4)

    assert rectangle.area() == 24
```

Bad: the expected value repeats the implementation.

```python
def test_rectangle_area() -> None:
    rectangle = Rectangle(width=6, height=4)

    assert rectangle.area() == rectangle.width * rectangle.height
```

## Parametrization

Good: case IDs describe the behavior.

```python
@pytest.mark.parametrize(
    ("raw_quantity", "expected"),
    [
        pytest.param("1", 1, id="minimum"),
        pytest.param("500", 500, id="large"),
    ],
)
def test_parse_quantity(raw_quantity: str, expected: int) -> None:
    assert parse_quantity(raw_quantity) == expected
```

Bad: anonymous cases make failures hard to read.

```python
@pytest.mark.parametrize(("value", "expected"), [("1", 1), ("500", 500)])
def test_parse_quantity(value: str, expected: int) -> None:
    assert parse_quantity(value) == expected
```

## Condition-Based Waiting

Good: wait for the observable condition.

```python
def wait_until(predicate: Callable[[], bool], timeout: float) -> None:
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        if predicate():
            return
        time.sleep(0.01)
    raise TimeoutError("condition was not reached")
```

Bad: fixed sleeps make tests slow and flaky.

```python
time.sleep(2)
assert worker.completed
```

## Comments

Good: the comment explains a current precondition.

```python
# Precondition: the session lock is held while next_sequence is read.
sequence = session.next_sequence
```

Bad: the comment narrates refactor history.

```python
# Sequence locking used to happen here, but it moved upstream.
sequence = session.next_sequence
```
