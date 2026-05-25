# Error-Path Testing

Fallible Python code needs tests for successful behavior, failure behavior,
and post-failure state. Most Python APIs communicate failure with exceptions.
Some APIs use `None`, sentinel values, warnings, or explicit result objects
when that shape is the clearest contract.

Test the contract the caller observes: exception type, stable message,
structured attributes, warning class, returned status, rollback, cleanup, and
boundary translation.

## Exceptions

Use `pytest.raises` for exception paths. Assert the exception class first,
then the stable contract.

```python
def test_rejects_unknown_symbol() -> None:
    request = make_order_request(symbol="UNKNOWN")

    with pytest.raises(UnknownSymbolError) as exc_info:
        validate_order_request(request)

    assert exc_info.value.symbol == "UNKNOWN"
```

Use `match=` when the message is part of the public contract:

```python
def test_rejects_empty_weighted_signals() -> None:
    weighted_signals = WeightedSignals.empty()

    with pytest.raises(
        ValueError,
        match=re.escape("WeightedSignals validation failed: dataframe is empty"),
    ):
        validate_weighted_signals(weighted_signals)
```

Match stable message text, error codes, or structured attributes. Avoid
depending on incidental punctuation or a third-party traceback format.

## One Rule Per Test

Validation tests read as executable rules when each test changes exactly one
fact from a valid baseline.

```python
def test_rejects_position_key_that_does_not_match_position() -> None:
    position_book = make_position_book()
    mismatched_key = PositionKey(market="B3", symbol="VALE3")
    position_book[mismatched_key] = make_position(symbol="PETR4")

    with pytest.raises(ValueError, match="position key .* does not match contained position"):
        validate_position_book(position_book)
```

The factory owns the valid baseline. The test owns the violated rule.

## Predicate Tests

When production validation is a list of named predicates, test the public
validator and let the predicate name guide the test name. This keeps tests
aligned with behavior while avoiding a separate test surface for every
private helper.

```python
def test_rejects_duplicate_symbol_join_keys() -> None:
    weighted_signals = make_weighted_signals(
        [
            WeightedSignalRow(trade_date=date(2026, 1, 2), symbol="PETR4", signal="long"),
            WeightedSignalRow(trade_date=date(2026, 1, 2), symbol="PETR4", signal="short"),
        ]
    )

    with pytest.raises(
        ValueError,
        match="WeightedSignals validation failed: found 2 duplicate keys",
    ):
        validate_weighted_signals(weighted_signals)
```

For dataframe validation, assert useful context such as rule name, row count,
columns, or a sample of offending rows when those fields are part of the
diagnostic contract.

## Rollback And Cleanup

If a function mutates state and then fails, assert the post-failure state
directly. Drive the failing branch, then inspect the object, filesystem,
database, registry, cache, or emitted events that the function owns.

```python
def test_upload_removes_temporary_file_when_publish_fails(tmp_path: Path) -> None:
    staging_file = tmp_path / "statement.csv"
    publisher = FailingPublisher()

    with pytest.raises(PublishError):
        upload_statement(staging_file, publisher=publisher)

    assert not staging_file.exists()
```

For transactions, check the state at the durable boundary:

```python
def test_rejected_order_does_not_insert_execution(order_repository: OrderRepository) -> None:
    order = make_order_request(quantity=0)

    with pytest.raises(ValueError):
        submit_order(order, repository=order_repository)

    assert order_repository.executions_for(order.account_id) == []
```

Do not infer rollback from a later successful operation. The failed operation
has its own cleanup contract.

## Boundary Translation

Boundary tests verify how domain failures become the boundary shape the caller
sees.

```python
def test_validation_error_becomes_bad_request(api_client: TestClient) -> None:
    response = api_client.post("/orders", json={"symbol": "PETR4", "quantity": 0})

    assert response.status_code == 400
    assert response.json()["error_code"] == "invalid_order"
```

For CLI commands, assert exit code and stderr. For queues, assert retry,
dead-letter, or acknowledgement behavior. For jobs, assert the recorded job
state and diagnostic payload.

## Warnings

Use `pytest.warns` when a recoverable condition is the contract.

```python
def test_warns_for_deprecated_strategy_name() -> None:
    with pytest.warns(DeprecationWarning, match="mean_reversion_v1"):
        load_strategy("mean_reversion_v1")
```

CI can turn project warnings into errors for broad feedback. Individual tests
still use `pytest.warns` when the warning itself is the behavior.

## Optional And Result-Shaped APIs

Some Python APIs express absence with `None` or a private sentinel because
absence is a normal lookup result.

```python
def test_returns_none_for_missing_security() -> None:
    repository = SecurityRepository([Security(symbol="PETR4")])

    assert repository.find("VALE3") is None
```

When a project uses an explicit result object at a boundary, test both
branches and inspect the structured failure.

```python
def test_import_reports_rejected_rows() -> None:
    result = import_statement(make_invalid_statement())

    assert not result.accepted
    assert result.rejected_rows == 3
    assert result.error_code == "invalid_statement"
```

Keep result-shaped helpers local to APIs that use that contract. Exceptions
remain the normal Python shape for failures inside a domain.
