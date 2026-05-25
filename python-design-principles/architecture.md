# Architecture

Python architecture should make domain ownership, dependency direction, and
runtime effects visible. A reviewer should be able to tell which package owns
the vocabulary, which package translates foreign data, and which package wires
the program to files, sockets, workers, schedulers, and frameworks.

Keep the domain core small enough to test with plain values. Put integration
code at adapters, and put runtime wiring at application or orchestration entry
points.

## Domain Vocabulary

Source code is the working theory of the domain. Use the words practitioners
use for packages, modules, classes, functions, parameters, tests, and errors.
Names such as `trade_date`, `rebalance_frequency`, `specimen`, `waybill`,
`buffer_size`, and `latency` carry more design information than `data`,
`item`, `record`, or `helper`.

Represent important domain concepts as named values instead of loose tuples,
generic dictionaries, or primitive parameters with ambiguous roles:

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

Do not invent vocabulary before the domain supplies it. Use plain technical
names for glue code, and ask for practitioner language before naming core
concepts.

## Package Layers

Use packages and modules as architectural boundaries:

- Domain packages own vocabulary, values, invariants, errors, and public
  contracts.
- Adapter packages translate vendor SDKs, wire formats, database rows, CLI
  inputs, queue messages, UI inputs, and dataframe schemas into domain values.
- Application and orchestration packages compose domains and own runtime
  concerns: settings, logging configuration, event loops, worker pools,
  database sessions, schedulers, and framework entry points.

A useful import shape is:

```text
application or orchestration
  imports adapters and domain packages

adapters
  import domain packages and vendor libraries

domain packages
  import the standard library, local domain modules, and narrow support libraries
```

Keep import arrows acyclic. When two packages need the same concept, move that
concept to a package both can import. Shared values belong in a named domain
package, not in a bucket called `common`, `shared`, or `utils`.

## Adapters

Adapters translate between neighboring domains. They are the place where
foreign shapes become local shapes:

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

The adapter owns aliases, codecs, vendor exceptions, and wire-format quirks.
The domain receives the domain value and uses domain names.

When an adapter supports several vendors, keep three pieces separate:

- A domain-facing protocol or function surface named for the role.
- One implementation module per vendor, counterparty, source, or codec.
- A runtime composition layer that creates clients, sessions, resources, and
  retries.

## Functional Core

Push effects to the edge. Domain functions should accept values and return
values. Application code should read the environment, open files, configure
logging, create clients, start workers, and call framework APIs.

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

This function needs no database, logger, clock, framework request, or worker
pool. The shell decides where input comes from and where output goes.

Framework entry points are shell code. A FastAPI route, Streamlit handler,
Dagster asset, Celery task, scheduled job, or CLI command can translate
framework inputs into domain values, call the core, then translate the result
back to the framework.

## Explicit Data Flow

Functions should take the values they need and return the values they produce.
Avoid hidden dataflow through module globals, framework session state, mutated
context bags, and import-time setup.

Use mutation when the receiver owns the state and the method name makes the
operation visible:

```python
class PositionBook:
    def add_fill(self, fill: Fill) -> None: ...

    def reset_to_cash(self, cash_balance: float) -> None: ...
```

Use returned values when a step derives new data:

```python
def rank_universe(universe: Universe, scores: Scores) -> Rankings: ...


def compute_weighted_signals(universe: Universe) -> WeightedSignals:
    scores = score_universe(universe)
    rankings = rank_universe(universe, scores)
    signals = select_signals(rankings)
    return weight_signals(signals)
```

Each name has one meaning. The sequence reads as the pipeline it implements.

## Runtime Boundaries

Runtime ownership belongs near the entry point. Threads, event loops, process
pools, schedulers, database sessions, HTTP clients, and logging configuration
should be created by the application or orchestration layer and passed inward
through explicit objects.

Use context-local facades for truly cross-cutting services such as a logger,
clock, settings object, tracing context, or timer when the runtime needs scoped
overrides. Use explicit parameters for domain-specific collaborators such as a
portfolio repository, pricing client, or shipment carrier.

For data platforms, keep in-process pipelines explicit until graph behavior is
an operational concern. Use orchestration tools when the graph owns schedules,
partitions, materialization, lineage, retries, resources, and recovery.

## Testability

Unit tests for the core should use domain values and inspect returned values,
raised exceptions, emitted events, or persisted effects owned by the object
under test. If a core test needs a real network client, event loop scheduler,
database container, or framework request object, an effect has probably moved
too far inward.

Integration tests exercise the shell and adapters. They legitimately use real
clients, framework fixtures, service resources, and realistic data.
