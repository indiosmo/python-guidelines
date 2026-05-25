# Error Handling

Python error handling is exception-first. Use exceptions for failures inside a
domain, and translate them deliberately at HTTP, CLI, job, queue, worker,
event-loop, and UI boundaries. Use result-shaped values only where a Python API
is clearer when the caller must inspect a typed success-or-failure branch.

Choose the smallest failure mechanism that communicates the contract.

## Built-In Exceptions

Use existing exception types when the name communicates the failure:

- `ValueError` for invalid values.
- `TypeError` for values of the wrong kind.
- `KeyError` for missing mapping keys when mapping access is the contract.
- `FileNotFoundError` for missing files.
- `TimeoutError` for elapsed timeouts.
- Library exceptions when the dependency exposes a precise contract, such as
  `pydantic.ValidationError`, `pandera.errors.SchemaError`, or
  `httpx.HTTPStatusError`.

```python
def parse_quantity(raw_quantity: str) -> int:
    try:
        quantity = int(raw_quantity)
    except ValueError as exc:
        raise ValueError(f"invalid quantity {raw_quantity!r}") from exc

    if quantity <= 0:
        raise ValueError("quantity must be positive")
    return quantity
```

The failure class and message tell the caller what is wrong.

## Domain Exceptions

Introduce a custom exception when it earns its place:

- The failure is a meaningful domain class, such as missing market data,
  rejected order, unavailable specimen, failed shipment pickup, or clipped
  audio buffer.
- Callers need to distinguish the failure by class.
- The exception carries structured attributes for logging, retries,
  translation, or tests.
- A boundary needs a stable translation target for HTTP status, CLI exit code,
  queue behavior, job state, or UI response.

Keep hierarchies small:

```python
from dataclasses import dataclass
from datetime import date


class MarketDataError(Exception):
    """Base class for market-data failures."""


@dataclass(frozen=True, slots=True)
class MissingClosePrice(MarketDataError):
    trade_date: date
    market: str
    symbol: str

    def __str__(self) -> str:
        return (
            "missing close price for "
            f"{self.symbol} on {self.trade_date} in {self.market}"
        )
```

Structured attributes let callers assert facts without parsing the message.

## Validation

Use one named rule per validation helper. Compose helpers into a public
validator for the type or boundary:

```python
def _validate_weighted_signals_non_empty(weighted_signals: DataFrame[WeightedSignals]) -> None:
    if weighted_signals.empty:
        raise ValueError("WeightedSignals validation failed: dataframe is empty")


def _validate_weighted_signals_unique_keys(weighted_signals: DataFrame[WeightedSignals]) -> None:
    duplicated = weighted_signals.duplicated(
        subset=["trade_date", "market", "symbol"],
        keep=False,
    )
    _raise_validation_error(
        weighted_signals,
        duplicated,
        context="WeightedSignals",
        columns=["trade_date", "market", "symbol"],
        message="duplicate symbol keys",
    )


def validate_weighted_signals(
    weighted_signals: DataFrame[WeightedSignals],
) -> DataFrame[WeightedSignals]:
    _validate_weighted_signals_non_empty(weighted_signals)
    _validate_weighted_signals_unique_keys(weighted_signals)
    return WeightedSignals.validate(weighted_signals)
```

For batch and dataframe validation, messages should include context, rule,
count, and a small sample of offending rows when that helps the operator or
test identify the problem.

## Exception Chaining

Use exception chaining when translating a low-level failure into the current
boundary's vocabulary:

```python
def load_portfolio_definition(path: Path) -> PortfolioDefinition:
    try:
        payload = json.loads(path.read_text())
        return PortfolioDefinition.model_validate(payload)
    except FileNotFoundError:
        raise
    except json.JSONDecodeError as exc:
        raise ValueError(f"invalid portfolio JSON in {path}") from exc
    except ValidationError as exc:
        raise ValueError(f"invalid portfolio definition in {path}") from exc
```

Use `raise NewError(...) from exc` when adding context. Use `raise` when the
same exception should continue unchanged. Suppress a cause only when the public
contract intentionally hides lower-level details.

## Boundary Translation

Catch failures at the layer that can translate them into the next protocol.
Examples include HTTP handlers, CLI commands, worker callbacks, queue
consumers, scheduled jobs, UI handlers, and event-loop task roots.

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

Inside libraries and domain code, catch specific exception classes only when
the code can add context, retry, recover, or translate to the public contract.
At a stopping boundary, log with traceback context through the project's
logging stack:

```python
def handle_job(job: Job) -> None:
    try:
        run_job(job)
    except Exception:
        logger.exception("job failed", extra={"job_id": job.job_id})
        mark_job_failed(job.job_id)
```

Avoid logging the same exception at every layer. Propagate it, or log and
convert it at the boundary.

## Grouped Failures

Use `ExceptionGroup` and `except*` when grouped concurrent work can fail in
several places. `asyncio.TaskGroup` uses this model.

```python
async def refresh_sources(sources: Iterable[Source]) -> None:
    try:
        async with asyncio.TaskGroup() as task_group:
            for source in sources:
                task_group.create_task(refresh_source(source))
    except* httpx.TimeoutException as group:
        raise SourceRefreshError("one or more sources timed out") from group
```

When partial failure is expected and useful to the caller, return an explicit
per-item status object so failures cannot be forgotten in a list of mixed
values.

## Cleanup And Rollback

Use Python's resource-management tools for cleanup:

- Context managers for resources.
- `contextlib.ExitStack` when cleanup callbacks are registered dynamically.
- `try` / `finally` when cleanup is local and obvious.
- Database transactions for multi-statement writes.
- Temporary files followed by atomic rename for file replacement.
- Idempotency keys, durable markers, and compensating actions for external
  effects that cannot be rolled back.

```python
from contextlib import ExitStack


def write_reports(reports: Iterable[Report], output_dir: Path) -> None:
    with ExitStack() as stack:
        temporary_paths: list[Path] = []

        for report in reports:
            temporary_path = output_dir / f"{report.name}.tmp"
            final_path = output_dir / f"{report.name}.json"
            temporary_paths.append(temporary_path)
            stack.callback(temporary_path.unlink, missing_ok=True)
            temporary_path.write_text(report.to_json())

        for temporary_path in temporary_paths:
            final_path = temporary_path.with_suffix(".json")
            temporary_path.replace(final_path)

        stack.pop_all()
```

Build new state in locals and commit at the end when possible.

## Warnings And Sentinels

Use warnings for recoverable conditions that callers may choose to escalate:
deprecations, questionable but accepted input, and runtime conditions where
continuing is safe. Tests and CI can turn project warnings into errors.

Use sentinel values when absence is the normal API contract. `None` is fine for
lookups, optional configuration, caches, and "not found" queries when the type
signature makes absence explicit:

```python
def close_price(trade_date: date, market: str, symbol: str) -> float | None: ...
```

Use a private sentinel when `None` is a valid domain value:

```python
_MISSING = object()


def get_config_value(name: str, default: object = _MISSING) -> object:
    if name in _CONFIG:
        return _CONFIG[name]
    if default is not _MISSING:
        return default
    raise KeyError(name)
```

Preserve the distinction between absence and falsy domain values such as `0`,
`0.0`, `False`, and an empty string when those values are valid.

## Result-Shaped Values

Exceptions are the default Python failure channel. A result-shaped value can be
useful for a boundary where explicit success and failure branches make the
consumer clearer than exceptions, such as bulk import row statuses, best-effort
partial processing, or APIs that must serialize every item outcome.

```python
@dataclass(frozen=True, slots=True, kw_only=True)
class ImportRowStatus:
    row_number: int
    imported: bool
    error_message: str | None = None
```

Use this style deliberately at the boundary that benefits from it. Do not make
Result, LEAF, or `std::expected` patterns the default shape of Python domain
code.
