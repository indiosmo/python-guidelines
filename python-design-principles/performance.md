# Performance

Performance work starts with a budget and a measurement. Python gives many
ways to make code faster: better algorithms, vectorized libraries, database
work, batching, caching, worker processes, native extensions, and lower-overhead
data layouts. Choose among them after a profiler or benchmark identifies the
hot path.

Default to clear Python until the code misses a latency, throughput, memory, or
cost target. Optimize the path that moves the end-to-end budget, not the line
that merely looks expensive.

## State The Budget

Optimization needs a target. Name the budget before changing the code:

- Maximum response latency for an API route.
- Throughput for a queue consumer or ingest job.
- Wall-clock target for a scheduled pipeline.
- Memory ceiling for a dataframe transform.
- Cost target for a warehouse query or external API call.

A local speedup matters when it changes the user, operator, or infrastructure
budget. If a function consumes a small fraction of the total runtime, a clever
rewrite usually leaves maintenance cost without visible benefit.

## Measure First

Use the measurement tool that answers the current question:

- `pyinstrument` for everyday CPU profiling and readable call-tree reports.
- `py-spy` for low-overhead sampling of a running process.
- `cProfile` for deterministic function-level comparisons in a small scope.
- `pytest-benchmark` for reviewable benchmark tests around public hot paths.
- `memray` for allocation-heavy memory investigations.
- `tracemalloc` for standard-library allocation snapshots.
- `scalene` for line-level CPU and memory attribution.
- Database query plans and warehouse metrics for SQL-heavy paths.

Benchmarks belong outside the normal correctness loop, usually under
`benchmarks/`, and should use representative input sizes. Treat benchmark
movement as review evidence. Keep correctness tests separate.

```python
def test_compute_signal_scores(benchmark, score_input_frame):
    strategy = ZScore(column="value")

    result = benchmark(strategy.compute_scores, score_input_frame)

    assert set(result.columns) >= {"trade_date", "symbol", "score"}
```

Use production metrics when the bottleneck is operational: queue lag, query
duration, HTTP retries, worker cold start, memory growth, or import time.

## Prefer Algorithmic And Data-Shape Wins

The largest Python performance gains usually come from doing less work or
asking the right engine to do it:

- Move filters, joins, aggregations, and projections into SQL when the database
  already owns the data.
- Use pandas, NumPy, Polars, ibis, DuckDB, or the warehouse query engine for
  tabular operations.
- Batch network, database, and filesystem calls.
- Cache expensive reference data under one owner.
- Precompute invariant values outside repeated loops.
- Replace repeated scans with indexed lookups when the access pattern is
  stable.

```python
import numpy as np
import pandas as pd


def compute_sharpe_ratio(
    returns: pd.Series,
    rolling_mean: pd.Series,
    rolling_std: pd.Series,
    annualization_factor: float,
) -> np.ndarray:
    return np.where(
        np.isclose(rolling_std, 0.0, atol=1e-10),
        np.nan,
        (rolling_mean / rolling_std) * np.sqrt(annualization_factor),
    )
```

For per-symbol rolling calculations, keep the work in the dataframe library.
When the grouping key is categorical, set `sort=False` and `observed=True`
unless the result contract needs sorted output or empty categories.

```python
def rolling_volatility(
    prices: pd.DataFrame,
    *,
    window: int,
    annualization_factor: float,
) -> pd.DataFrame:
    volatility = (
        prices.groupby("symbol", sort=False, observed=True)["returns"]
        .transform(lambda series: series.rolling(window, min_periods=window).std())
        .mul(np.sqrt(annualization_factor))
    )
    return prices[["trade_date", "symbol"]].assign(volatility=volatility)
```

For rolling math, derive the formula before parallelizing. Hoist constants,
precompute repeated sums, and keep scratch values as local series when the
caller owns the input frame.

## Mutation And Copies

Copies are expensive when dataframes are large. A defensive `.copy()` inside a
loop often signals an unclear mutation contract.

Prefer pure dataframe transforms for pipeline stages: read input columns,
compute a result, and return the join keys plus new columns. The orchestrator
joins stage outputs. In-place mutation is fine when the function clearly owns
the frame and the caller does not reuse it.

```python
def compute_momentum(prices: pd.DataFrame, *, window: int) -> pd.DataFrame:
    close = prices["close"]
    prior_close = prices.groupby("symbol", sort=False, observed=True)[
        "close"
    ].shift(window)
    momentum = (close / prior_close) - 1.0
    return prices[["trade_date", "symbol"]].assign(momentum=momentum)
```

Keep scratch columns out of shared input frames. Local series, `assign`, and
small temporary frames make ownership visible without forcing every caller to
copy the whole dataset.

## Caching

Cache when repeated access dominates the cost and the cache has a clear owner.
A cache should have one of these shapes:

- Built once during construction from immutable input.
- Private and rebuilt on demand when the owner changes its source data.
- Backed by an explicit invalidation policy.
- Stored in a process, actor, or worker initializer to amortize setup.

```python
from dataclasses import dataclass
from datetime import date


@dataclass(frozen=True, slots=True)
class MarketPrice:
    close: float
    volume_weighted_average_price: float


class MarketPriceLookup:
    def __init__(self, prices: pd.DataFrame) -> None:
        self._by_key: dict[tuple[date, str], MarketPrice] = {
            (row.trade_date, row.symbol): MarketPrice(
                close=row.close,
                volume_weighted_average_price=row.volume_weighted_average_price,
            )
            for row in prices.itertuples(index=False)
        }

    def get(self, trade_date: date, symbol: str) -> MarketPrice | None:
        return self._by_key.get((trade_date, symbol))
```

Preserve the difference between absence and falsy domain values. A price,
rate, score, or count of `0` may be valid. Use `dict.get` directly when `None`
is the missing value, or use a private sentinel when `None` is a valid domain
value.

## Object Layout

Use slotted dataclasses for large numbers of small internal value objects.
Slots reduce per-instance memory and prevent accidental dynamic attributes.

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True, kw_only=True)
class Fill:
    order_id: str
    symbol: str
    quantity: int
    price: float
```

Use Pydantic where runtime validation and serialization earn their cost. Use
dataclasses for already-validated internal aggregates. Use tuples or
`NamedTuple` only when positional structure is part of the intended API.

## Parallelism

Profile before adding parallelism. Threads help blocking I/O and extension
code that releases the GIL. Process pools, Ray, job queues, native extensions,
vectorized libraries, or database-side execution fit CPU-bound Python work
better than adding threads around Python loops.

Move heavy setup into the worker lifetime. Load reference data, build lookup
caches, initialize models, and open expensive clients once per worker when the
workload has many independent jobs.

Keep worker inputs and outputs serializable. Large dataframes and model objects
can dominate runtime if each job ships them across a process boundary.

## Timing And Logging

Production timing should use one timing primitive and the project logger.
Development profiling should use a profiler or benchmark command. Scattered
`print` calls and ad hoc `timeit.default_timer()` blocks produce output that is
hard to filter, aggregate, or test.

```python
from collections.abc import Iterator
from contextlib import contextmanager
from time import perf_counter

from loguru import logger


@contextmanager
def timed_block(operation: str) -> Iterator[None]:
    started_at = perf_counter()
    try:
        yield
    finally:
        elapsed_seconds = perf_counter() - started_at
        logger.info(
            "operation completed",
            operation=operation,
            elapsed_seconds=elapsed_seconds,
        )
```

Use level checks, lazy formatting, sampling, counters, or throttling for
expensive high-volume diagnostics. Python logging has runtime cost even when a
message is disabled if the payload is expensive to build before the call.

## Leaving Python

Leave pure Python when measurement shows the bottleneck belongs below the
interpreter:

- Use SQL, dbt, DuckDB, or ibis for relational work.
- Use pandas, NumPy, Polars, or PyArrow for columnar work.
- Use Numba, Cython, Rust, C, or a domain library for tight numeric kernels.
- Use process pools, Ray, or external workers for independent CPU-bound jobs.
- Use a service or database feature when the work is already operational data
  movement.

The boundary matters. Keep the Python side typed, validated, and benchmarked
so the faster implementation remains a replaceable detail.

## Review Checklist

- The change names the performance budget it improves.
- A profiler, benchmark, query plan, or production metric identifies the hot
  path.
- The optimization moves end-to-end latency, throughput, memory, or cost.
- Dataframe and SQL work use native engines before Python loops.
- Repeated setup moves to construction, startup, or worker initialization.
- Caches have one owner and a clear invalidation or immutability story.
- Benchmarks live outside the normal correctness test loop.
