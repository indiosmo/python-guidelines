# Testing Philosophy

Python tests should verify behavior through the public contract the code
claims to provide. Expected values come from domain rules, specifications,
documented examples, invariants, and independently checked calculations.
The implementation is the thing under test, not the oracle.

A useful test answers one question: if this behavior breaks, does the test
fail for the right reason?

## Behavior Over Implementation

Test the result the caller can observe: returned values, raised exceptions,
persisted state, emitted events, generated output, or visible side effects at
a boundary. Internal call order, private helper names, and incidental
intermediate shapes make brittle tests unless they are the contract under
review.

```python
def test_position_value_uses_quantity_and_price() -> None:
    position = Position(symbol="AAPL", quantity=25, average_price=Decimal("12.50"))

    assert position.market_value == Decimal("312.50")
```

The expected value is a worked example of the domain rule. A weaker test
repeats the implementation formula through the same attributes and catches
less.

For dataframe and numeric code, test the semantic output. Use the dataframe
library's testing helpers, explicit tolerances, schema checks, and domain
invariants instead of isolated cells when the contract is a whole table.

```python
def test_weighted_signal_output_is_sorted_and_complete() -> None:
    input_frame = make_weighted_signals(symbols=["PETR4", "VALE3"])

    result = normalize_weighted_signals(input_frame)

    pd.testing.assert_frame_equal(
        result,
        expected_weighted_signals(symbols=["PETR4", "VALE3"]),
        check_like=False,
    )
```

## Independent Oracles

Read the domain before reading the implementation. Useful expected values
come from:

- A protocol, file format, mathematical definition, or product rule.
- A business invariant such as "a long position carries no borrow fee".
- A documented example.
- A separate trusted calculation.
- A stable fixture that represents a real boundary payload.

Tests become tautological when they compute the expected result by calling the
same helper, parser, or transformation that production code uses. Use small
literal examples, independent calculators, or pre-reviewed expected files
when the production logic is the target.

## Component Scope

Component tests exercise one function, class, module, or domain service. They
give precise failure signals because the setup, action, and assertion all sit
near the behavior under test.

Good component tests target:

- Boundary values such as empty input, one row, zero, limits, and missing keys.
- State transitions such as open to closed, pending to accepted, or draft to
  published.
- Validation rules and error messages.
- Idempotency, rollback, and cleanup.
- Ordering, grouping, and deduplication rules.
- Numeric tolerances and floating-point edge cases.

Keep the test body focused on one behavior. Shared construction belongs in
factories or fixtures when it lets the scenario-specific data stand out.

## Integration Scope

Integration tests prove behavior that a component test cannot prove alone.
They cross a real boundary or combine several components in the same shape
the application uses.

Good integration tests assert:

- Data moves through a boundary without losing shape, type, encoding, or
  ordering.
- A domain error becomes the expected HTTP response, CLI exit code, job
  state, queue behavior, or user-facing message.
- A transaction, file write, cache update, or emitted event is visible at the
  receiving boundary.
- Several packages agree on one domain concept, such as account identity or
  trading calendar dates.

Avoid broad integration tests that repeat component assertions through a
larger setup. Put the narrow rule in the component test, then use integration
coverage for wiring, translation, and side effects.

## Determinism

Deterministic tests build confidence because failures carry information. Keep
time, randomness, filesystem state, environment variables, process-global
configuration, network access, and unordered collections under test control.

Use pytest tools for isolation:

- `tmp_path` for filesystem writes.
- `monkeypatch` for environment and attribute replacement.
- `caplog` for log assertions.
- `capsys` or `capfd` for stdout and stderr.
- `pytest.raises` for exception paths.
- Fixtures and context managers for process-global state.

When randomness is part of the test strategy, control the seed and report it
on failure. When ordering is not semantically meaningful, sort before
comparing. When output includes paths, line endings, UUIDs, timestamps, or
host-specific values, canonicalize them before asserting.

## Public Contracts

Treat test names, fixture names, factory names, and assertion helpers as part
of the guide to the code. Name them in the domain vocabulary readers use:
`shipment`, `encounter`, `order_request`, `weighted_signals`,
`reference_data`, `market_calendar`, or `portfolio_definition`.

Typed test helpers earn their annotations because other tests rely on them.
Small test bodies can stay concise when literals and assertions make the
behavior clear, but reusable fakes, fixture factories, assertion helpers, and
dataframe builders should expose useful types.

## Focused Feedback

Use the smallest command that reproduces the failure while working:

```sh
uv run pytest tests/market_data/test_calendar.py::test_rejects_holiday_session -q
```

Broaden the command only after the local behavior is correct:

```sh
uv run pytest
uv run ruff check .
uv run pyright
```

Fast, focused loops make tests easier to write and failures easier to
understand. The full suite remains the release signal.
