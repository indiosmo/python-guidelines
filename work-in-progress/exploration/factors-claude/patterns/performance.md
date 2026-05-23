# Performance

C++ guidance: *clarity first, measure don't guess, hot-path
allocations, flat vs node hash maps, type-erased callables*. Python
analogues factors2 reaches for: vectorised pandas via `groupby +
transform`, `np.where` for branch-free per-row math, dict caches over
DataFrames for O(1) lookup, `pytest-benchmark` for the indicator hot
paths, `pyinstrument` in deps (but unused in CI).

## What factors2 does well

### `groupby(...).transform(...)` for per-symbol rolling math

```python
# indicators/technical/annualized_volatility.py:27-35
def compute_rolling(self, df):
    df[self.indicator_column_name] = (
        df.groupby(SYMBOL_KEY, sort=False, observed=True)[self.column]
        .transform(lambda x: x.rolling(window=self.window, min_periods=self.window).std(ddof=1))
        .mul(np.sqrt(self.series_period.annualization_factor))
    )
    return df[SYMBOL_JOIN_KEY + [self.indicator_column_name]]
```

`sort=False, observed=True` is correct for categorical group keys -
`observed=True` skips empty categorical buckets, `sort=False` saves
the sort cost. The pattern is consistent across every indicator.

**PROMOTE.** Guideline phrasing: *"per-symbol rolling math uses
`groupby(..., sort=False, observed=True)`. Always set these on
categorical groupers."*

### Pre-aggregating sums to avoid recomputing inside the rolling loop

```python
# indicators/technical/linear_regression_slope.py:66-100
def _calculate_rolling_slope(self, df, window_size, pos_col, y_col, pos_y_col):
    time_idx = np.arange(window_size)
    mean_t = time_idx.mean()
    denominator = np.sum((time_idx - mean_t) ** 2)
    if np.isclose(denominator, 0, atol=1e-10):
        return pd.Series(np.nan, index=df.index)

    sum_y = df.groupby(SYMBOL_KEY, sort=False, observed=True)[y_col].transform(
        lambda s: s.rolling(window=window_size, min_periods=window_size).sum()
    )
    sum_pos_y = df.groupby(SYMBOL_KEY, sort=False, observed=True)[pos_y_col].transform(
        lambda s: s.rolling(window=window_size, min_periods=window_size).sum()
    )
    numerator = sum_pos_y - (df[pos_col] - mean_t) * sum_y
    return numerator / denominator
```

This is the closed-form OLS slope rewritten so the rolling window
only needs to compute two sums (`sum_y` and `sum_pos_y`); the
denominator is computed once and held constant. The math comment
above the code explains why each pre-aggregation is the right thing.

**PROMOTE.** *"For rolling math, derive the closed form, identify
which sums are window-independent, hoist them out of the loop."*

### `np.where` over `if/else` for vector-shaped branches

```python
# indicators/technical/sharpe_ratio.py:58-64
return np.where(
    np.isclose(rolling_std, 0.0, atol=1e-10),
    np.nan,
    (rolling_mean / rolling_std) * np.sqrt(SeriesPeriod.DAILY.annualization_factor),
)
```

Branchless, vectorised; the comment on line 61-62 calls out the
"don't compare floats with `==`" rule. **PROMOTE.**

### `pytest-benchmark` covers every indicator

```
benchmarks/test_benchmark_annualized_volatility.py
benchmarks/test_benchmark_sharpe_ratio.py
benchmarks/test_benchmark_linear_regression_slope.py
... 30+ files
```

One benchmark per indicator, all grouped, all parameterised by the
shared `benchmarks/__init__.py` constants (`N=1_000_000`,
`SYMBOLS=[...1000...]`). `make benchmark` runs them all.

**PROMOTE.** *"Every hot-path function has a benchmark in
`benchmarks/`, grouped by domain, parameterised by a shared
representative input size."*

### Dict caches alongside DataFrames for O(1) lookups in the sim loop

```python
# pipeline/models.py:139-188 (and Benchmarks, MarketPrices)
class InterestRates:
    def get_rate(self, trade_date, symbol):
        return self.dict.get((trade_date, symbol))
```

The simulation loop calls `get_rate`/`get_close`/`get_vwap`/`get_borrow_rate`
inside `step()` (`simulation.py:200,220,223,312,313,367,368`) - tens
of thousands of times per backtest. Dict lookup is the right
optimisation. (The implementation is sloppy - see
`data-pipeline.md` - but the *intent* is right.)

**ADAPT.** The pattern is fine; the implementation needs the
generalisation discussed in `pydantic-pandera-dataclasses.md`.

### Ray actors carry the deserialisation cost once per worker

```python
# backtest/server.py:35-45
@ray.remote
class SimulationWorker:
    def __init__(self, refdata: ReferenceData) -> None:
        t0 = timeit.default_timer()
        self.refdata = refdata
        self.refdata.market_prices.build_dict()  # cache dict for O(1) lookup
        print(f"Worker initialized in {timeit.default_timer() - t0}")
        self.ready = True
```

The big `ReferenceData` is deserialised once per worker; the
`build_dict()` cache is built once per worker; the per-job
`run_simulation` work is pure compute over the cached state. This is
the right way to amortise per-process setup. **PROMOTE.**

## What factors2 does poorly

### `pyinstrument` declared but unused

`pyinstrument>=5.0.3` is in `[dependency-groups].dev`, but I could
not find any `with Profiler():` block, any `pyinstrument` CLI usage
in the Makefile, or any "profiling guide" in `doc/`. It's a tool
nobody picks up.

**ADAPT.** Either add `make profile <args>` that wraps
`pyinstrument python -m src.pyfactors.backtest.example <args>` and a
short `doc/profiling.md`, or remove the dep. Carrying dev deps that
nobody invokes erodes the signal of `pyproject.toml`.

### `inputs.copy()` per indicator in the orchestrator

```python
# pipeline/stages.py:243-260
def compute_indicators(universe, indicators, reference_data):
    inputs = universe.copy()
    columns = ["trade_date", "market", "symbol"]
    for indicator in indicators:
        if indicator.indicator_column_name not in universe:
            columns.append(indicator.indicator_column_name)
            indicator_df = indicator.compute_rolling(inputs.copy(), reference_data)
            inputs = inputs.merge(indicator_df, on=["trade_date", "market", "symbol"], how="inner")
```

Two `.copy()` calls in a loop. Cost: for a 1M-row universe with 40
indicators, each `.copy()` is roughly the memory size of the frame,
*per indicator*. The "fix" is to enforce that indicators are pure
(read inputs, return only the new column with the join keys), and
the orchestrator joins. Several indicators (`AnnualizedVolatility`,
`SharpeRatio`) already do this; some (`LinearRegressionSlope`) write
scratch columns (`_log_close`, `_pos`, `_y`, `_pos_y`, `_slope_long`,
`_slope_short`) into the input.

**AVOID.** Make indicators pure. Drop the `.copy()`.

### `print("X time: ", timeit.default_timer() - t1)` instead of a proper timing primitive

```python
# pipeline/stages.py:298-329, backtest/server.py:38-132
t1 = timeit.default_timer()
universe = dataset_queries.fetch_universe(...)
print("universe time: ", timeit.default_timer() - t1)
```

If timing matters in production, it deserves a context manager
(`with timed_block("fetch_universe"): ...`) that emits one structured
log line on exit. If it only matters in development, it belongs in
`benchmarks/` or under `pyinstrument`. The current form is neither -
it writes to stdout, isn't filterable, and clutters the production
log.

**AVOID.** Either:
- a `@timed` decorator on the function that emits one
  `logger.info("X", elapsed_s=...)`, or
- move to `benchmarks/`, or
- use `pyinstrument` for the development case.

### `LinearRegressionSlope` scratch columns are not cleaned up

```python
# indicators/technical/linear_regression_slope.py:102-136
def compute_rolling(self, df):
    df["_log_close"] = np.log(df["close"])
    df["_pos"] = df.groupby(SYMBOL_KEY, ...).cumcount()
    df["_y"] = df.groupby(SYMBOL_KEY, ...)["_log_close"].shift(self.offset)
    df["_pos_y"] = df["_pos"] * df["_y"]
    df["_slope_long"] = self._calculate_rolling_slope(...)
    if self.short_term_reversal_period != 0:
        s = self.short_term_reversal_period
        df["_slope_short"] = self._calculate_rolling_slope(...)
        df[self.indicator_column_name] = df["_slope_long"] - df["_slope_short"]
    else:
        df[self.indicator_column_name] = df["_slope_long"]
    return df[SYMBOL_JOIN_KEY + [self.indicator_column_name]]
```

The function mutates `df` with six scratch columns prefixed `_`. The
return statement projects only the join keys + the indicator, so the
scratch columns *don't leak in the return* - but they do leak in the
input (the caller's DataFrame). That's why the orchestrator has to
`.copy()` defensively (see above).

**AVOID.** Operate on a local frame; mutate the caller's frame is a
contract violation. The minimal fix is `df = df.copy()` at the top
of `compute_rolling`; the better fix is to compute the scratch
columns as local series and never assign them to `df`.

## Recommendations for the new guideline

1. **Vectorise first; profile before parallelising.** `groupby +
   transform`, `np.where`, closed-form math. If the hot path is in
   pandas, the first move is to write more pandas, not to add threads.
2. **Categorical groupers use `sort=False, observed=True`.** Both
   are non-default and both matter on real data sizes.
3. **Indicators are pure.** Read input columns; return new column
   with the join keys; never mutate the input. The orchestrator
   joins.
4. **One benchmark per hot-path function**, in `benchmarks/`,
   parameterised by a shared `N` / `SYMBOLS` from `benchmarks/__init__.py`.
5. **Timing primitives, not `print` and `timeit.default_timer()`.**
   A `with timed_block("..."): ...` context manager that emits a
   structured `logger.info(...)` on exit, or `pyinstrument` for
   profiling.
6. **Ray actors carry per-process setup cost once.** Heavy
   deserialisation, dict-cache building, model loading - all in the
   actor's `__init__`. The actor methods are pure compute.
7. **If `pyinstrument` is in deps, document how to use it.** If not,
   prune.
