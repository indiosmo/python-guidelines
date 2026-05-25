# Test Patterns

Choose the simplest pytest structure that states the behavior clearly. The
shape of the test should match the shape of the contract: a single example, a
table of equivalent cases, a shared setup with separate outcomes, or a
type-checker expectation.

## Separate Tests

Use separate tests when setup, action, or assertions differ meaningfully.

```python
def test_accepts_well_formed_order_request() -> None:
    request = make_order_request(quantity=100, limit_price=Decimal("10.25"))

    assert validate_order_request(request) == request


def test_rejects_zero_quantity_order_request() -> None:
    request = make_order_request(quantity=0)

    with pytest.raises(ValueError, match="quantity must be positive"):
        validate_order_request(request)
```

Separate tests give the clearest failure signal when each scenario has its
own rule.

## Parametrized Cases

Use parametrization when the same logic applies to many rows. Include labels
through `ids=` or row objects.

```python
@pytest.mark.parametrize(
    ("raw_value", "expected"),
    [
        pytest.param("0", 0, id="zero"),
        pytest.param("42", 42, id="positive"),
        pytest.param("-17", -17, id="negative"),
    ],
)
def test_parses_decimal_integer(raw_value: str, expected: int) -> None:
    assert parse_decimal_integer(raw_value) == expected
```

For wide tables, use dataclasses so fields carry names.

```python
@dataclass(frozen=True, slots=True)
class SessionCase:
    label: str
    status: SessionStatus
    event: SessionEvent
    expected: SessionStatus


@pytest.mark.parametrize("case", SESSION_CASES, ids=lambda case: case.label)
def test_session_transitions(case: SessionCase) -> None:
    assert transition(case.status, case.event) is case.expected
```

## Shared Setup With Branches

When several outcomes share a meaningful setup, keep the setup visible and
use separate tests or a fixture. Pytest has no built-in section construct; the
usual Python shape is a named fixture plus focused tests.

```python
@pytest.fixture
def password_policy() -> PasswordPolicy:
    return PasswordPolicy(min_length=8, max_length=64)


def test_accepts_password_within_bounds(password_policy: PasswordPolicy) -> None:
    assert validate_password("correct_horse", password_policy)


def test_rejects_short_password(password_policy: PasswordPolicy) -> None:
    assert not validate_password("short", password_policy)
```

Use `pytest-subtests` only when one test naturally checks many related
observations and reporting every failing observation matters.

## Dataframe Assertions

For dataframe transforms, compare the frame the caller receives. Use the
library's testing API so failures report dtype, index, column, ordering, and
tolerance differences.

```python
def test_computes_annualized_volatility_for_each_symbol() -> None:
    input_frame = make_price_returns(
        {
            "ASCENDING": [-0.01, 0.00, 0.02, 0.03],
            "CONSTANT": [0.01, 0.01, 0.01, 0.01],
            "ZERO": [0.00, 0.00, 0.00, 0.00],
        }
    )
    expected = make_indicator_frame(
        {
            "ASCENDING": [np.nan, np.nan, 0.242487, 0.242487],
            "CONSTANT": [np.nan, np.nan, 0.0, 0.0],
            "ZERO": [np.nan, np.nan, 0.0, 0.0],
        }
    )

    result = AnnualizedVolatility(window=3, series_period=SeriesPeriod.DAILY).compute(input_frame)

    pd.testing.assert_frame_equal(result, expected, atol=1e-6)
```

Enumerate meaningful input shapes: ascending, descending, constant, zero,
mixed length, missing values, duplicates, and unsorted rows when those matter
to the contract.

## One Rule Per Validation Test

Validation suites read well when each test starts from a valid object and
changes one field to violate one rule.

```python
def test_rejects_non_finite_mark_to_market_nav() -> None:
    step = replace(make_valid_step_result(), mark_to_market_nav=float("inf"))

    with pytest.raises(
        ValueError,
        match=re.escape("StepResult validation failed: mark_to_market_nav must be finite"),
    ):
        validate_step_result(step)
```

Exact message checks fit when the message is part of the public contract or a
support workflow depends on it. Prefer attribute or error-code assertions
when those are the stable contract.

## Type-Checker Tests

Some contracts are static: a protocol accepts one shape, a generic preserves
the input type, or a public function rejects a misuse during checking. Put
those expectations in checked files or small tests that use typing helpers.

```python
from typing import assert_type


def test_repository_get_type(repository: Repository[OrderId, Order]) -> None:
    order = repository.get(OrderId("abc"))

    assert_type(order, Order | None)
```

When a project uses checked example files for negative typing cases, run them
through the project's selected type checker in CI. Keep runtime tests for
runtime behavior.

## Benchmarks

Correctness tests prove behavior. Benchmarks measure movement in hot paths.
Put benchmarks under `benchmarks/`, group them by domain, and exercise public
APIs on representative input sizes.

```python
@pytest.mark.benchmark(group="technical_indicators")
def test_annualized_volatility_benchmark(
    benchmark: BenchmarkFixture,
    price_returns: pd.DataFrame,
) -> None:
    indicator = AnnualizedVolatility(window=20, series_period=SeriesPeriod.DAILY)

    benchmark(indicator.compute, price_returns.copy())
```

Review benchmark changes as performance signals. Keep correctness assertions
in normal tests.
