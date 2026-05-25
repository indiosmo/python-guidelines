# Cross-Cutting Services

Cross-cutting services are runtime facilities many layers may need: logging,
clock access, timers, settings, metrics, tracing, feature flags, and request
context. Treat them as runtime services rather than domain dependencies.

Use standard project configuration for logging. Use small facades backed by
`ContextVar` when tests, tasks, or request scopes need local overrides. Pass
domain-specific dependencies explicitly.

## Service Categories

Use different shapes for different dependency kinds:

- Logging: one project logging stack, configured at startup.
- Clock and timer: a small protocol or facade, with test overrides.
- Settings: a validated object built at the process boundary.
- Metrics and tracing: startup-configured providers with context-local span or
  request data.
- Domain dependencies: explicit parameters, constructor arguments, or service
  objects.

A tax calculator, market-data repository, payment client, or model runner is a
domain dependency. Pass it explicitly so the caller sees the business contract.

## Logging

Libraries should use stdlib logging unless the project policy says otherwise.
Applications can configure stdlib logging, loguru, or structlog at startup and
bridge library logs there.

Keep log calls stable and structured:

```python
logger.info(
    "orders routed",
    extra={
        "strategy": strategy_name,
        "order_count": len(orders),
        "trade_date": trade_date.isoformat(),
    },
)
```

Log and stop or translate at boundaries such as CLI commands, HTTP handlers,
worker loops, and scheduled jobs. Inside a domain function, raise the domain
exception with useful context and let the boundary decide how to report it.

For structlog, `structlog.contextvars` fits request-scoped fields in web and
async applications. Bind validated request context at the boundary so later
log calls carry the same correlation identifiers.

## Context-Local Facades

`ContextVar` fits services that need scoped override without threading a handle
through every function. Clocks, timers, request context, trace identifiers, and
test probes are common examples.

```python
from contextvars import ContextVar
from dataclasses import dataclass
from datetime import UTC, datetime
from typing import Protocol


class Clock(Protocol):
    def now(self) -> datetime:
        ...


@dataclass(frozen=True, slots=True)
class SystemClock:
    def now(self) -> datetime:
        return datetime.now(UTC)


_clock: ContextVar[Clock] = ContextVar("clock", default=SystemClock())


def now() -> datetime:
    return _clock.get().now()


class clock_override:
    def __init__(self, clock: Clock) -> None:
        self._clock = clock

    def __enter__(self) -> None:
        self._token = _clock.set(self._clock)

    def __exit__(self, exc_type: object, exc: object, traceback: object) -> None:
        _clock.reset(self._token)
```

The facade gives production code a simple call and gives tests a scoped
override. Use this sparingly; a facade that becomes a grab bag of application
dependencies hides the design.

## Settings

Build settings once at the process boundary from environment variables, CLI
arguments, files, or secret stores. Pass the validated settings object inward.

```python
def main(argv: list[str]) -> int:
    settings = AppSettings.from_cli_and_environment(argv)
    configure_logging(settings.logging)
    services = build_services(settings)
    return run_job(settings.job, services)
```

The precedence rules belong to the settings builder. Domain functions should
receive concrete values or a narrow settings object for the feature they own.

## Timers And Metrics

Use context managers for timing blocks and emit metrics through a project
provider configured at startup.

```python
from contextlib import contextmanager
from time import perf_counter


@contextmanager
def timed_operation(metrics: Metrics, name: str, **fields: object):
    started_at = perf_counter()
    try:
        yield
    finally:
        metrics.observe(name, perf_counter() - started_at, fields)
```

Keep high-cardinality fields out of metric labels. Put detailed identifiers in
logs or traces where the backend can handle them.

## Tracing And Request Context

Tracing libraries already provide context propagation. Wrap them only to
standardize project vocabulary, span names, and attributes.

Use context-local request data for correlation identifiers, tenant identifiers,
or actor identifiers that belong to a request or task. Parse and validate that
context at the boundary, then make it available to logging and tracing.

## Explicit Domain Dependencies

Pass domain dependencies as explicit arguments or constructor parameters:

```python
@dataclass(frozen=True, slots=True)
class RebalanceServices:
    price_source: PriceSource
    order_sink: OrderSink
    clock: Clock


def rebalance_portfolio(request: RebalanceRequest, services: RebalanceServices) -> RebalanceResult:
    trade_date = request.trade_date or services.clock.now().date()
    prices = services.price_source.fetch(trade_date)
    orders = build_orders(request, prices)
    services.order_sink.submit(orders)
    return RebalanceResult(trade_date=trade_date, orders=orders)
```

This makes the business dependencies visible and gives tests straightforward
substitution points.

## Test Substitution

Use pytest fixtures to override context-local services and to pass explicit
fake dependencies.

```python
@pytest.fixture
def frozen_clock() -> Iterator[FrozenClock]:
    clock = FrozenClock(datetime(2026, 1, 15, tzinfo=UTC))
    with clock_override(clock):
        yield clock
```

Fixtures should reset process-global or context-local state they change. A
test that mutates logging configuration, pandas options, environment variables,
or service facades should restore the previous value.

## Async And Task Context

`ContextVar` values flow naturally through asyncio task creation. Thread pools,
process pools, and external workers need explicit context transfer when the
context matters. Prefer passing the required values into worker input records
over relying on ambient state across process boundaries.

## Review Checks

- Logging uses the project stack and stable messages with structured values.
- Settings are built and validated at startup.
- Context-local facades are limited to truly cross-cutting services.
- Domain-specific dependencies are explicit in function or service signatures.
- Tests override services through fixtures and reset the state they change.
- Runtime context crossing threads or processes is passed deliberately.
