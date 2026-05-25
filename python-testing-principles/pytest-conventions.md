# Pytest Conventions

Pytest is the default correctness runner for Python projects. Configure one
command surface that developers, CI, and automation all use, and keep tests
importing through the same package names production code uses.

Use pytest's native features before building a custom harness: fixtures,
parametrization, assertion rewriting, marks, node IDs, `tmp_path`,
`monkeypatch`, `caplog`, `capsys`, and `pytest.raises`.

## Discovery And Layout

Put correctness tests under `tests/`, and mirror the source package path when
that helps readers find coverage.

```text
src/market_data/indicators/annualized_volatility.py
tests/market_data/indicators/test_annualized_volatility.py
```

Configure discovery and import roots explicitly:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = ["--import-mode=importlib"]
```

For projects with a build system, `uv sync` installs the project and workspace
members as editable packages by default. `--import-mode=importlib` keeps pytest
from changing `sys.path` for test module imports and works well for new
projects. Use `pythonpath = ["src"]` only when the project intentionally tests
the local source tree directly instead of the editable install. When packaging
correctness matters, add a release check that tests the built or installed
artifact.

Keep long-running benchmarks under `benchmarks/` and run them through an
explicit command:

```sh
uv run pytest benchmarks/
```

## Test Names

Name tests for behavior, not mechanics. A useful name tells the reviewer the
condition and the expected outcome.

```python
def test_rejects_duplicate_security_ids() -> None:
    ...


def test_rolls_back_file_when_upload_fails() -> None:
    ...
```

Use one behavior per test by default. Separate tests are clearer than a table
when setup or assertions differ meaningfully.

## Assertions

Lean on pytest assertion rewriting. Direct assertions usually produce the
best failure message:

```python
assert portfolio.cash == Decimal("1000.00")
assert result.trade_dates == [date(2026, 1, 2), date(2026, 1, 5)]
```

Use a precondition assertion when later checks need a value to exist:

```python
trade = blotter.last_trade()
assert trade is not None
assert trade.symbol == "VALE3"
```

Use multiple focused assertions when each one reports an independent fact.
Use helper assertions when the same domain comparison appears in several
tests.

## Parametrization

Use `pytest.mark.parametrize` when one test body covers several rows. Add
`ids=` or a labeled row object so a failing case is easy to identify.

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class FeeCase:
    label: str
    quantity: int
    price: Decimal
    expected_fee: Decimal


CASES = [
    FeeCase("zero quantity", 0, Decimal("10.00"), Decimal("0.00")),
    FeeCase("standard lot", 100, Decimal("10.00"), Decimal("1.50")),
]


@pytest.mark.parametrize("case", CASES, ids=lambda case: case.label)
def test_calculates_execution_fee(case: FeeCase) -> None:
    assert execution_fee(case.quantity, case.price) == case.expected_fee
```

Split wide tables into input and expectation objects when positional tuples
become hard to read. Keep scenario labels stable because they become part of
the failure signal.

## Fixtures

Use fixtures for shared setup, teardown, expensive resources, and process
state that needs restoration. Prefer explicit fixture names that reveal the
state they provide.

```python
@pytest.fixture
def frozen_clock() -> Clock:
    return FrozenClock(datetime(2026, 1, 2, 10, 0, tzinfo=UTC))


def test_uses_valuation_date(frozen_clock: Clock) -> None:
    report = build_daily_report(clock=frozen_clock)

    assert report.valuation_date == date(2026, 1, 2)
```

Use `yield` fixtures when setup and cleanup form one resource lifetime:

```python
@pytest.fixture
def configured_warning_policy() -> Iterator[None]:
    with warnings.catch_warnings():
        warnings.simplefilter("error", ProjectWarning)
        yield
```

Keep `autouse=True` rare. It fits a broad invariant for a module or suite,
such as a warning policy or a test-only settings mode. Process-global changes
should save and restore their previous value.

## Marks And Selection

Use marks for durable selection axes:

- `integration` for tests that cross process, service, database, queue, or
  filesystem boundaries.
- `slow` for tests that exceed the normal local loop budget.
- Domain marks when a project repeatedly runs a named subset.

Register marks in pytest configuration so typos fail loudly:

```toml
[tool.pytest.ini_options]
strict_config = true
strict_markers = true
strict_xfail = true
filterwarnings = [
    "error",
]
markers = [
    "integration: crosses a real boundary",
    "slow: outside the normal fast correctness loop",
]
```

For pytest 9 and locked pytest versions, `strict = true` can carry the same
strictness policy in one option. Add targeted warning ignores only after the
warning source is triaged.

Use file paths, node IDs, marks, and `-k` for focused loops:

```sh
uv run pytest tests/pipeline/test_portfolio_definition.py -q
uv run pytest -m "not slow"
uv run pytest -k "portfolio and validation"
```

## Plugin Policy

Add a pytest plugin when it carries a named testing policy. Useful examples
include approval testing, benchmarking, async tests, subtests, coverage, and
filesystem fakes for code that truly works at the filesystem boundary.

Development dependencies should earn their place through tests, CI, or a
documented command. Remove unused plugins so the test environment stays
understandable.

Use `pytest-cov` when coverage is a project policy or release gate. Prefer
branch coverage for mature packages, and treat `fail_under` as a ratchet for
stable code rather than a substitute for meaningful assertions.

## Parallel Runs

Use `pytest-xdist` after the suite isolates process-global state, temporary
paths, ports, databases, random seeds, caches, and environment variables.
Mark or separate tests that share a real resource so parallel execution does
not create order dependence.

Prefer deterministic per-test resources over serializing the whole suite. Use
serialization only for tests whose behavior inherently depends on one
process-wide resource.

Use `pytest-randomly` or a similar plugin as a diagnostic tool for hidden
inter-test dependencies. Record the seed in failure output so order-dependent
failures can be reproduced. Enable randomized order after process-global
state, temporary paths, ports, databases, and random seeds are isolated well
enough for useful failures.

## Command Surface

Keep common commands stable:

```sh
uv run pytest
uv run pytest tests/
uv run pytest tests/path/to/test_file.py::test_name -q
uv run pytest --maxfail=1 -x
```

CI should run the same command family developers run locally, with extra
coverage, reporting, or selection flags only where the project needs them.
