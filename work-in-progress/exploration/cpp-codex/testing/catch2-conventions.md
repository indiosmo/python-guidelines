# Catch2 and CTest conventions

Overall disposition: PYTHON_SPECIFIC_VARIANT.

Rewrite as `pytest-conventions.md`. The intent ports, but every mechanic is
pytest-specific.

## Snippet conventions

Disposition: PYTHON_SPECIFIC_VARIANT.

Use pytest examples, plain `assert`, explicit type annotations for helpers,
and `snake_case` test names.

```python
def test_parser_accepts_empty_input() -> None:
    assert parse("") == []
```

## Test discovery and selection

Disposition: PYTHON_SPECIFIC_VARIANT.

pytest discovers `test_*.py` files and `test_*` functions. Use node ids and
markers instead of CTest prefixes.

```bash
uv run pytest tests/parser/test_empty.py::test_parser_accepts_empty_input
uv run pytest -m "not slow"
```

Register custom markers in `pyproject.toml` to catch spelling mistakes.

## Test serialization

Disposition: PYTHON_SPECIFIC_VARIANT.

Use `pytest-xdist` grouping or avoid parallelism for tests that touch shared
process state. Markers such as `serial` can drive CI selection.

```python
@pytest.mark.serial
def test_global_provider_restore() -> None:
    ...
```

## Tags

Disposition: PYTHON_SPECIFIC_VARIANT.

Use pytest markers. Keep the C++ rule's spirit: one stable spelling per axis.
Prefer a small controlled marker set: `slow`, `serial`, `integration`,
`network`, `gui`.

## Assertions

Disposition: PORTS_WITH_ADAPTATION.

Map `REQUIRE` to assertions that guard setup. In pytest, a failed `assert`
stops the test, so use helper functions to keep failures close to the cause.
Use `pytest.approx` for approximate numeric checks and `pytest.raises` for
exceptions.

```python
with pytest.raises(InvalidSymbol, match="empty"):
    parse_symbol("")
```

Use assertion messages sparingly; pytest's introspection is usually clearer
than a restated condition.
