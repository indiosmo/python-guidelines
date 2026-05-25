# Types And Correctness

Python types make contracts visible early. They help reviewers see domain
roles, help tools catch refactor mistakes, and give tests clearer inputs.
Runtime validation still matters because Python receives untyped values from
JSON, databases, files, queues, dataframes, plugins, and users.

Use static typing where it clarifies a stable contract. Keep dynamic Python
where the language or library is clearer with runtime validation, especially
inside dataframe transformations, plugin adapters, and exploratory boundaries.

## Public Contracts

Type the surfaces other code depends on:

- Public package APIs.
- Domain values and command objects.
- Settings, messages, events, and request models.
- Adapter outputs after boundary parsing.
- Reusable helpers, fixtures, fakes, and assertion utilities.
- Dataframe boundaries where a named schema carries meaning.

Short local code can rely on inference when the names and literals are clear.
Avoid turning typing into a second implementation of obvious control flow.

## Parse At Boundaries

Untyped input should become a typed value before it reaches the domain core.
Use the tool that fits the boundary:

- Pydantic v2 for HTTP, CLI, JSON, environment, config, queue, and
  cross-process records.
- Pandera for dataframe shape, dtype, nullability, and simple value checks.
- Dataclasses for internal values assembled from already validated parts.
- `TypedDict` for dictionary-shaped data with a stable key set and no behavior.
- `NewType` for identity-only primitive distinctions.
- Small value classes when construction proves a domain rule.

```python
from dataclasses import dataclass
from typing import NewType

from pydantic import BaseModel, Field

AccountId = NewType("AccountId", str)
Symbol = NewType("Symbol", str)


class OrderRequestModel(BaseModel):
    account_id: str = Field(min_length=1)
    symbol: str = Field(min_length=1)
    quantity: int = Field(gt=0)


@dataclass(frozen=True, slots=True, kw_only=True)
class OrderRequest:
    account_id: AccountId
    symbol: Symbol
    quantity: int


def parse_order_request(payload: object) -> OrderRequest:
    model = OrderRequestModel.model_validate(payload)
    return OrderRequest(
        account_id=AccountId(model.account_id),
        symbol=Symbol(model.symbol),
        quantity=model.quantity,
    )
```

The parser checks hostile input once. The domain signature then carries the
roles forward.

## Data Carriers

Choose one record style for the boundary:

- Frozen slotted dataclasses for internal value objects and passive aggregates.
- Mutable dataclasses when mutation is the domain operation and one owner
  protects the invariant.
- Pydantic models when the object parses external input or serializes across a
  process boundary.
- attrs classes when the project already uses attrs or needs its validators
  and converters without the full Pydantic boundary model.

Prefer keyword construction for multi-field objects:

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True, kw_only=True)
class Trade:
    trade_date: date
    market: str
    symbol: str
    quantity: int
    price: float
```

Keyword-only construction protects adjacent same-type fields from swaps and
makes later field additions safer.

Frozen objects freeze the top-level attributes. They do not freeze nested
lists, dictionaries, dataframes, or mutable model objects. When callers can
observe nested state, use immutable containers, private caches, copied inputs,
or an explicit mutable aggregate type.

## Domain Values

Wrap primitives when the role matters:

- Use `NewType` when the runtime value is already valid and the distinction is
  identity only.
- Use a dataclass, Pydantic model, or validated constructor when the value
  carries proof, such as non-empty text, positive quantity, bounded percentage,
  normalized path, or parsed identifier.
- Use `Annotated` with a validator when a framework understands it and the
  validation belongs at that boundary.

Do not wrap every primitive. Wrap values with ambiguous roles, meaningful
constraints, external visibility, or high swap risk.

## Closed Sets

Use `Enum`, `Literal`, or tagged unions for closed domain vocabularies.
Dispatch on closed sets with `match` and `assert_never` so adding a case makes
missing behavior visible to the type checker:

```python
from enum import StrEnum
from typing import assert_never


class Side(StrEnum):
    BUY = "buy"
    SELL = "sell"
    SELL_SHORT = "sell_short"


def side_to_fix_value(side: Side) -> str:
    match side:
        case Side.BUY:
            return "1"
        case Side.SELL:
            return "2"
        case Side.SELL_SHORT:
            return "5"
        case _ as unreachable:
            assert_never(unreachable)
```

Boundary parsers still need explicit unknown-value handling because external
data is open:

```python
def parse_side(raw: str) -> Side:
    try:
        return Side(raw)
    except ValueError as exc:
        raise ValueError(f"invalid side {raw!r}") from exc
```

## Discriminated Unions

When configuration selects one strategy from a supported set, use an explicit
discriminated union. The registry is the public configuration surface.

```python
from typing import Annotated, Literal

from pydantic import BaseModel, Field


class TopN(BaseModel):
    strategy: Literal["top_n"] = Field(default="top_n", frozen=True)
    count: int = Field(gt=0)

    def select(self, rankings: Rankings) -> Signals: ...


class Threshold(BaseModel):
    strategy: Literal["threshold"] = Field(default="threshold", frozen=True)
    minimum_score: float

    def select(self, rankings: Rankings) -> Signals: ...


type SignalStrategyVariant = TopN | Threshold


class SignalStrategy(BaseModel):
    strategy: Annotated[
        SignalStrategyVariant,
        Field(discriminator="strategy"),
    ]
```

Dynamic discovery fits open plugin systems. A reviewed union fits supported
variants that define the project's stable configuration contract.

## Protocols And Callables

Use `Protocol` when callers depend on behavior rather than class identity:

```python
from typing import Protocol


class PriceRepository(Protocol):
    def close_price(self, trade_date: date, market: str, symbol: str) -> float | None: ...
```

Use `Callable` for small policies, predicates, factories, clocks, and test
probes. Prefer a protocol when the behavior has several methods, named
attributes, or domain meaning.

## Any, Object, Cast, And Ignores

`Any` is the dynamic escape hatch. Let it appear at hostile or weakly typed
boundaries, then normalize it quickly with validation, a typed mapping, a
protocol, or an adapter. Use `object` when code may hold any value but must
prove its shape before using it.

Use `cast` after a nearby runtime check, trusted library contract, or named
invariant. Repeated casts around the same dependency point to a typed adapter
or local stub.

Keep type ignores narrow and checker-specific. An ignore around project-owned
model access usually means the model contract needs work. Tests may need local
escapes for deliberately invalid input, but reusable test data should come from
typed factories or schema-aware builders.

## Dataframes

Static typing should frame dataframe-heavy code, not fight the dataframe
library. Type paths, settings, strategy objects, scalar results, and boundary
schemas. Use Pandera and tests for dataframe columns, dtypes, nullability,
ordering, and cross-column relationships.

Inside pandas, NumPy, Polars, SQL, or ibis transformations, prefer native
library expressions over elaborate aliases the checker cannot prove. Validate
at the boundary where the dataframe shape becomes a contract.
