# Architecture

Overall disposition: PORTS_AS_IS.

The chapter's architecture rules are language-agnostic. Python changes the
syntax and packaging mechanics, but not the shape: the domain model belongs in
the code, dependencies form a DAG, adapters translate between neighboring
domains, and effects move outward.

## Theory of the domain

Disposition: PORTS_AS_IS.

Keep the section. Python needs the same "code as domain theory" framing
because dynamic language code decays quickly when names become generic and
domain concepts are passed as untyped dictionaries.

Python rendering:

```python
from dataclasses import dataclass
from typing import NewType

AccountId = NewType("AccountId", str)
Symbol = NewType("Symbol", str)


@dataclass(frozen=True)
class NewOrder:
    account_id: AccountId
    symbol: Symbol
    quantity: int
```

This is not C++ strong typing, but it preserves the domain vocabulary in the
interface and lets a type checker catch common swaps.

## Domain separation with adapters

Disposition: PORTS_AS_IS.

Keep the dependency shape exactly. In Python, translate C++ libraries and
namespaces into packages and modules:

```text
routing/                 protocol-agnostic domain
routing_fix/             FIX expression of routing concepts
routing_fix_vendor/      adapter over the vendor SDK
vendor_sdk_wrapper/      thin wrapper around the foreign package
```

Adapters should convert foreign dictionaries, Pydantic models, ORM rows, or
SDK objects into domain dataclasses at the edge. Domain packages should not
import vendor SDKs.

## Domain-owned types

Disposition: PORTS_AS_IS.

Keep the principle. Python has less nominal typing, so the guide should use a
mix of `NewType`, frozen dataclasses, enums, and parser functions.

```python
from typing import NewType

RoutingOrderId = NewType("RoutingOrderId", str)
RiskOrderId = NewType("RiskOrderId", str)


def to_risk_order_id(order_id: RoutingOrderId) -> RiskOrderId:
    return RiskOrderId(order_id)
```

Use a named conversion only when the mapping has domain meaning. Direct
construction is fine for same-representation ownership changes.

## Internal shape of an adapter module

Disposition: PORTS_WITH_ADAPTATION.

The three-piece shape ports, but Python should use `Protocol` or ABCs only
when they buy a testable boundary. Avoid Java-style interface proliferation.

```python
from typing import Protocol


class CounterpartySession(Protocol):
    def send(self, order: RoutedOrder) -> None: ...


class OnixsSession:
    def send(self, order: RoutedOrder) -> None:
        ...
```

Per-vendor implementations remain sibling modules. The runtime module composes
the chosen implementation and owns foreign callbacks.

## Forward-only dependencies

Disposition: PORTS_AS_IS.

Keep this unchanged. Python import cycles, circular service dependencies, and
call graphs where `A` leaves residue for `B` are the same design bug.

```python
def untie(data: Table) -> Column:
    return build_tiebreaker(data)


def rank(data: Table, tiebreaker: Column) -> Table:
    return apply_monotonic(data, tiebreaker)


def score(data: Table) -> Table:
    return rank(data, untie(data))
```

The Python guide should explicitly call out import cycles. If two modules want
to import each other, extract a third module or move the composition upward.

## No implicit contracts

Disposition: PORTS_AS_IS.

Keep as a subsection under forward dependencies. The Python version should
warn against hidden writes to `self`, mutable context dictionaries, and pandas
columns left for later functions to consume.

Bad Python shape:

```python
def untie(working_set: WorkingSet) -> None:
    working_set.tiebreaker_column = build_tiebreaker(working_set.data)


def rank(working_set: WorkingSet) -> None:
    working_set.data = apply_monotonic(
        working_set.data,
        working_set.tiebreaker_column,
    )
```

Prefer explicit inputs and outputs.

## Functional core, imperative shell

Disposition: PORTS_AS_IS.

Keep it and make it one of the central Python rules. Python code often hides
effects in module imports, global clients, decorators, and framework callbacks.
The guide should say that files, network clients, environment variables,
database sessions, clocks, and task creation belong in the shell.

```python
def decide_order(order: NewOrder, limits: Limits) -> Decision:
    return accept(order) if order.notional <= limits.available else reject(order)


async def handle_order(raw: bytes, limits_client: LimitsClient) -> None:
    order = parse_order(raw)
    limits = await limits_client.fetch(order.account_id)
    await publish(decide_order(order, limits))
```

## Testability without mocks

Disposition: PORTS_AS_IS.

Keep the same litmus test. A unit test should feed values into the functional
core and inspect return values or captured callbacks. If a simple domain test
needs a database, event loop, HTTP server, or monkeypatch forest, an effect
probably leaked inward.
