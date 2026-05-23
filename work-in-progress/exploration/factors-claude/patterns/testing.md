# Testing

factors2's test suite is the strongest single piece of the codebase
in terms of guideline-readiness. The shape is:

- One file per source file (`tests/pyfactors/indicators/technical/test_amihud_illiquidity.py`
  for `src/pyfactors/indicators/technical/amihud_illiquidity.py`).
- Per-test factory helpers in `tests/pyfactors/util.py` and per-subpackage
  `util.py` (e.g. `tests/pyfactors/backtest/util.py`).
- One failure mode per test, with the exact error message asserted via
  `re.escape(...)`.
- approvaltests `verify(...)` for query-builder text (SQL, etc.).
- pytest-benchmark for hot-path indicators in `benchmarks/`.
- pyfakefs declared as a dev dep but currently unused.

## What factors2 does well

### Per-rule error-path tests

```python
# tests/pyfactors/backtest/test_validate_step_result.py:64-72
def test_validate_step_result_non_finite_mark_to_market_nav_raises() -> None:
    """Reject steps whose mark-to-market NAV is not finite."""
    step = replace(_make_valid_step_result(), mark_to_market_nav=float("inf"))

    with pytest.raises(
        ValueError,
        match=re.escape("StepResult validation failed: mark_to_market_nav must be finite"),
    ):
        validate_step_result(step)
```

30+ tests in that file, each:

- starts from a `_make_valid_step_result()` factory,
- mutates exactly one field to break exactly one invariant,
- asserts `pytest.raises(ValueError, match=re.escape(...))` with the
  exact message text.

The result is a test file that reads as documentation of every
invariant `validate_step_result` enforces, and that breaks immediately
if either the rule or the message text drifts.

**PROMOTE.** Guideline phrasing:
- "One test per validation rule. Each test mutates exactly one field
  off a valid factory, expects exactly one exception with an exact
  message."
- "Validation messages are part of the contract. Test them with
  `re.escape(...)`, not `in str(e)`."

### Factory functions for "valid" objects

```python
# tests/pyfactors/backtest/test_validate_step_result.py:15-61
def _make_valid_trade() -> Trade: ...
def _make_valid_position() -> Position: ...
def _make_valid_step_result() -> StepResult: ...
```

Plus the more elaborate `tests/pyfactors/backtest/util.py`
(`make_position_book`, `make_reference_data`, `make_simulation`) and
`tests/pyfactors/util.py` (`make_market_price_row`, `make_security_list`,
`make_blacklist`, `make_reference_data`, `make_quarterly_indicators`).

The factories accept kwargs for the fields that vary, default the
rest, and return a fully-validated object. Tests then mutate one
field and assert one outcome.

**PROMOTE.** *"Each test file has a `_make_valid_X()` factory; tests
mutate one field; one assertion per test."*

### Tabular test data with a comment block describing the table

```python
# tests/pyfactors/backtest/test_simulation.py:234-264
def test_accrue_daily_borrow_fees():
    """Test daily borrow fee accrual for short positions.

    Test Setup:
    - default_rate_pct = 5.0
    - daily_rate formula: (1 + rate_pct / 100) ** (1/252) - 1
    - accrued_fee = notional * daily_rate

    | Symbol | Position | Avg Price | Mkt Close | Mkt Rate | Scenario             |
    |--------|----------|-----------|-----------|----------|----------------------|
    | PETR4  |     +100 |      25.0 |      30.0 |      0.0 | Long, no accrual     |
    | VALE3  |      -50 |      75.0 |      70.0 |     10.0 | Short with mkt data  |
    | ITUB4  |      -30 |      45.0 |      None |     None | Short, use fallbacks |
    """
    # fmt: off
    symbols       = ["PETR4", "VALE3", "ITUB4"]
    positions     = [100,     -50,     -30    ]
    prices        = [25.0,    75.0,    45.0   ]
    mkt_symbols   = ["PETR4", "VALE3"]
    mkt_closes    = [30.0,    70.0   ]
    mkt_rates     = [0.0,     10.0   ]
    expected_fees = [0.0, 1.3240, 0.2614]
    # fmt: on
```

The docstring is the test plan; the `# fmt: off` block is the data.
Reading the test is reading the spec. Same shape in
`test_simulation.py:330-417`.

**PROMOTE.** *"For tests with multiple cases, embed a markdown-style
table in the docstring; align the test data with `# fmt: off`."*

### Property-based scenarios as named dictionaries

```python
# tests/pyfactors/indicators/technical/test_annualized_volatility.py:9-35
def test_annualized_volatility():
    returns_data = {
        "ASCENDING": [-0.01, 0, 0.02, 0.03],
        "CONSTANT": [0.01, 0.01, 0.01, 0.01],
        "DESCENDING": [0.05, 0.05, 0.04, 0.02],
        "MIXED": [0.01, 0.02, 0, 0.03],
        "MIXED_DIFFERENT_SIZE": [0.01, 0.02, 0, 0.03, 0.01, 0.01, 0.02],
        "ZERO": [0, 0, 0, 0],
    }
    expected_data = {
        "ASCENDING": [np.nan, np.nan, 0.242487, 0.242487],
        "CONSTANT": [np.nan, np.nan, 0.0, 0.0],
        ...
    }

    annualized_volatility = AnnualizedVolatility(window=3, series_period=SeriesPeriod.DAILY)
    input = create_test_dataframe(returns_data, "returns")
    expected = create_test_dataframe(expected_data, annualized_volatility.indicator_column_name)
    result = annualized_volatility.compute_rolling(input)
    pd.testing.assert_frame_equal(result, expected, atol=1e-6)
```

Each indicator test enumerates the meaningful input shapes (ascending,
descending, constant, zero, mixed of different sizes), then asserts
the whole result frame against an expected frame. This is the
"property/scenario" coverage style from `doc/guidelines.md:165-184`.

**PROMOTE.** *"For pure functions over a series, enumerate the
meaningful input shapes (ascending, descending, constant, zero,
mixed) and assert the full output frame."*

### approvaltests for query-text snapshots

```python
# tests/pyfactors/database/test_universe_query.py:1-34
from approvaltests import verify

def test_query_builder() -> None:
    filters = [...]
    query = make_universe_query(
        filters,
        start_date=date(2023, 1, 1),
        end_date=date(2023, 4, 1),
        benchmark_rate="CDI",
        benchmark_index="IBOV",
    )
    assert query.params == ["CDI", "IBOV", "CDI", "IBOV", "NYSE", ["Technology", "Healthcare"]]
    verify(query.query)
```

`assert query.params == [...]` for the parameter list (because it's
small and the value of testing the literal text directly outweighs
the diff noise), `verify(query.query)` for the multi-line SQL
(because asserting it inline would be unreadable).

**PROMOTE.** *"For multi-line text artifacts (SQL, JSON, generated
config), use approvaltests `verify(...)`; the .approved.txt file is
part of the source tree."*

### `pytest-benchmark` for hot paths, scoped to the public API

```python
# benchmarks/test_benchmark_annualized_downside_volatility.py:25-28
@pytest.mark.benchmark(group="technical_indicators")
def test_annualized_volatility_benchmark(benchmark, test_df):
    annualized_downside_vol = AnnualizedDownsideVolatility(window=20, series_period=SeriesPeriod.DAILY)
    benchmark(annualized_downside_vol.compute_rolling, test_df.copy())
```

One benchmark per indicator, all grouped (`group="technical_indicators"`),
fixture-built input from `benchmarks/__init__.py:N=1_000_000, SYMBOLS=1000`,
`Makefile` target `make benchmark`.

**PROMOTE.** *"Benchmarks live in `benchmarks/`, are run by a Makefile
target, and exercise the public API on a representative data size."*

## What factors2 does poorly

### `cast(...)` to fake a typed empty DataFrame in tests

```python
# tests/pyfactors/backtest/util.py:78-81
signals = cast(
    DataFrame[WeightedSignals],
    pd.DataFrame(columns=["trade_date", "market", "symbol", "close", "vwap", "score", "signal", "weight"]),
)
```

The cast lies to the type checker; if `WeightedSignals` ever adds a
column, no one notices. Better: a real
`WeightedSignals.empty() -> DataFrame[WeightedSignals]` classmethod
that builds an empty frame from the schema and validates it.

**AVOID** `cast(...)` in tests as a substitute for a constructor.

### `try/except` swallowing assertions inside test helpers - not actually present

I looked for the pattern and didn't find it. Worth noting as a
**non**-finding: the test code does not catch broad exceptions.

### `pyfakefs` declared but unused

`pyfakefs` is in `[dependency-groups].dev` but I could not find any
test using it. Either prune the dep or document where it should be
used (the obvious candidates are
`etl_factors/sources/economatica/downloads.py` for the file-write
path and `etl_factors/sources/directory/csv_to_postgres.py` for the
COPY path).

**ADAPT.** Either remove or use.

### `conftest.py` autouse for global pandas mode

```python
# tests/conftest.py:6-10
@pytest.fixture(scope="session", autouse=True)
def enable_copy_on_write():
    pd.options.mode.copy_on_write = True
    yield
```

This sets a global pandas option for the whole test session.
Production code in `portfolio_management/app.py:239` *also* sets the
same option. The tests run with CoW, the app runs with CoW, but
nothing else does. If an importer of `pyfactors` runs without CoW,
behaviour diverges from what the test suite covers.

**ADAPT.** Pick one: (a) require CoW at the top of `pyfactors/__init__.py`
(with a comment about the contract), or (b) write the library to not
depend on CoW. The current state (set in tests, set in one app,
implicit elsewhere) is fragile.

## Recommendations for the new guideline

1. **One file per source file, mirroring the package layout.**
2. **Per-file `_make_valid_X()` factories; per-package `util.py`
   factories.** Tests vary one field off the factory.
3. **One assertion per test for error paths.** Each rule has its own
   test, asserts the *exact* message via `re.escape(...)`. The
   validation message is part of the contract.
4. **For pure-function tests, enumerate the meaningful input shapes**
   (ascending, descending, constant, zero, mixed) and assert the
   full output, not just a scalar.
5. **For tabular tests, embed a markdown table in the docstring and
   align data with `# fmt: off`.** Reading the test is reading the
   spec.
6. **approvaltests `verify(...)` for multi-line text artifacts;
   `.approved.txt` lives in the source tree.**
7. **pytest-benchmark in `benchmarks/`, one per public function;
   group by domain; representative input size in a shared fixture.**
8. **No `cast(...)` in tests; write a constructor instead.**
9. **No `except Exception` in tests.** Anything that needs catching
   needs to be caught by class.
10. **Global pandas/numpy modes go in `__init__.py`, not in
   `conftest.py`,** so the production behaviour matches the test
   behaviour for every importer.
