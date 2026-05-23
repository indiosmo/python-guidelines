# Error-path testing

Overall disposition: PORTS_WITH_ADAPTATION.

The success/failure/rollback obligations port. The mechanics change because
Python code should usually raise domain exceptions internally and translate at
boundaries.

## Testing `expected`-style return types

Disposition: PYTHON_SPECIFIC_VARIANT.

This is not the default Python shape. Keep a smaller section for explicit
boundary result objects only when the project uses them.

```python
result = parse_int_result("abc")
assert isinstance(result, Err)
assert result.error is ParseError.INVALID_INPUT
```

Do not invent result objects for ordinary internal Python code unless the
domain benefits from value-level composition.

## Testing exception-throwing code

Disposition: PORTS_WITH_ADAPTATION.

This becomes the main section. Use `pytest.raises` and inspect structured
exception fields.

```python
def test_parse_symbol_rejects_empty() -> None:
    with pytest.raises(InvalidSymbol) as raised:
        parse_symbol("")

    assert raised.value.raw == ""
```

Use `match` only for user-visible message text. Prefer attributes for domain
facts.

## Exception safety and scope-guard rollback

Disposition: PORTS_AS_IS.

Keep the obligation. Trigger the failing branch and inspect post-call state
directly.

```python
def test_registry_invalid_entry_rolls_back_insert() -> None:
    registry = Registry()

    with pytest.raises(InvalidEntry):
        registry.register_entry(bad_entry())

    assert bad_entry_id() not in registry.entries
```

For async functions:

```python
async def test_consumer_rolls_back_on_publish_failure() -> None:
    with pytest.raises(PublishFailed):
        await consumer.consume(message)

    assert repository.find(message.id) is None
```

## Boundary translation tests

Disposition: PYTHON_SPECIFIC_VARIANT.

Add a Python-specific section. Public boundaries should catch domain
exceptions and translate them into the caller's vocabulary.

```python
def test_http_handler_translates_invalid_symbol(client: TestClient) -> None:
    response = client.post("/orders", json={"symbol": ""})

    assert response.status_code == 422
    assert response.json()["code"] == "routing.invalid_symbol"
```

Task and CLI top levels need the same coverage: log, swallow, retry, return an
exit code, or publish an error event according to the contract.
