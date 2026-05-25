# Invariants And Rollback

An invariant is a fact the program relies on after every public operation:
weights sum to one, a cache matches its source data, an order has one lifecycle
state, a file appears atomically, or a database write commits as a unit.

Put each invariant on the type, service, or boundary that has enough context to
protect it. Mutating operations should either fully apply or leave the
observable state unchanged. Where external effects make perfect rollback
impossible, define idempotent retry and reconciliation behavior.

## Invariant Ownership

The owner of related state owns the operations that keep it consistent. Two
fields that must change together usually want one small object with methods
that update both.

```python
from dataclasses import dataclass, field
from decimal import Decimal


@dataclass(slots=True)
class PositionBook:
    cash: Decimal
    positions: dict[str, Position] = field(default_factory=dict)

    def apply_trade(self, trade: Trade) -> None:
        new_cash = self.cash - trade.notional
        new_positions = self.positions.copy()
        new_positions[trade.symbol] = new_positions[trade.symbol].after_trade(trade)

        self.cash = new_cash
        self.positions = new_positions
```

The method builds the next state first, then commits the fields together. A
caller sees either the old book or the new book.

## Commit At The End

Commit-at-end is the simplest rollback strategy. Perform fallible work in
locals, validate the result, and assign to shared state only after every check
passes.

```python
from dataclasses import replace


def add_restriction(
    definition: PortfolioDefinition,
    restriction: Restriction,
) -> PortfolioDefinition:
    candidate = replace(
        definition,
        restrictions=(*definition.restrictions, restriction),
    )
    validate_portfolio_definition(candidate)
    return candidate
```

This pattern works well for dataclasses, Pydantic models through
`model_copy(update=...)`, immutable tuples, staged dictionaries, and dataframe
transforms that return a new frame.

For mutable dataframe stages, name the mutation contract. A pure transform
reads input columns and returns new columns or a new frame. An in-place
transform owns the frame and keeps scratch columns out of shared inputs.

## Transactions And Context Managers

Use the storage system's transaction primitive for multi-step persistent
writes. For Python resources and compensating actions, use context managers,
`try` / `finally`, or `contextlib.ExitStack`.

```python
from contextlib import ExitStack


def publish_report(report: Report, storage: ReportStorage) -> None:
    with ExitStack() as stack:
        draft_path = storage.write_draft(report)
        stack.callback(storage.discard_draft, draft_path)

        storage.validate_draft(draft_path)
        final_path = storage.promote_draft(draft_path)

        stack.pop_all()
        storage.record_publication(report.report_id, final_path)
```

`ExitStack` fits dynamic cleanup: the function registers rollback actions as it
goes and releases them when the operation reaches its commit point. Use a
dedicated context manager when the pattern has a domain name.

## Atomic File Writes

Write durable files through a temporary path in the same directory, flush the
contents, and replace the target path atomically.

```python
from pathlib import Path
from tempfile import NamedTemporaryFile


def write_json_atomically(path: Path, payload: str) -> None:
    with NamedTemporaryFile("w", encoding="utf-8", dir=path.parent, delete=False) as file:
        temporary_path = Path(file.name)
        file.write(payload)
        file.flush()

    temporary_path.replace(path)
```

Use the platform and filesystem guarantees that match the durability need. A
local cache, a reviewed generated file, and an operational ledger can require
different fsync and directory-sync policy.

## Idempotency

Retried handlers should define the same final state for one run or many runs.
Use idempotency keys, processed-event markers, uniqueness constraints, atomic
cache operations, write dispositions, and durable reservations.

```python
def handle_webhook(event: WebhookEvent, store: EventStore) -> None:
    with store.transaction():
        inserted = store.insert_processed_event(event.event_id)
        if not inserted:
            return
        store.apply_event(event)
```

The check and insert happen in one transaction. A duplicate delivery observes
the recorded event and exits before applying the event again.

Record rollback facts at the time of admission. A reservation release should
use the quantity originally reserved, not a recomputed value from state that
may have changed.

## Caches And Staleness

Derived caches should be private and rebuilt from immutable source state, or
they should be the owned source of truth. A public dataframe beside a public
derived dictionary creates two mutable truths.

```python
@dataclass(frozen=True)
class MarketPriceLookup:
    prices: tuple[MarketPrice, ...]

    @cached_property
    def by_key(self) -> dict[tuple[date, str, str], MarketPrice]:
        return {
            (price.trade_date, price.market, price.symbol): price
            for price in self.prices
        }
```

If a value can be stale by policy, carry the freshness in the recorded state:
for example, store `mark_trade_date` beside `mark_price`.

Preserve the difference between absence and falsy domain values. A rate of
`0.0`, an empty string from a vendor, and a missing value can mean different
things.

## Free-Threaded Caveats

Python code should use explicit ownership for shared mutable state. Async tasks
can interleave at `await` points, threads can race around I/O and mutable
containers, and free-threaded builds make ownership discipline more important.

Keep shared mutation behind one owner, a queue, a transaction, or an explicit
lock at the runtime boundary. Domain objects should state their intended
ownership through API shape: immutable values, owner methods, or runtime
wrappers that serialize access.

## Tests

Failure-path tests should assert the post-failure state directly:

- The database contains the old rows or the full new rows.
- The target file is old or fully replaced.
- The object has the old fields after a failed update.
- Duplicate messages produce the documented final state.
- Rollback actions run in the order required by the resource contract.

Use focused tests for each invariant. For dataframe invariants, pair schema
validation with named domain checks such as uniqueness, sortedness, group sums,
or cross-column relationships.

## Review Checks

- The invariant lives where all required context is available.
- Fallible work happens before shared state is committed where possible.
- Transactions, context managers, or `ExitStack` cover multi-step effects.
- File writes use an atomic replace shape when readers need all-or-nothing
  visibility.
- Retried handlers have explicit duplicate-delivery behavior.
- Derived caches stay synchronized with their source state.
- Tests assert state after failures, retries, and partial effects.
