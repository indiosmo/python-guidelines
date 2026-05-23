# Testing philosophy

Overall disposition: PORTS_AS_IS.

The testing philosophy ports directly. Python tests need the same independent
oracle discipline because dynamic code makes tautological tests especially easy
to write.

## Test behavior, not implementation

Disposition: PORTS_AS_IS.

Keep the section. Render examples in pytest:

```python
def test_rectangle_area() -> None:
    rectangle = Rectangle(width=6, height=4)
    assert rectangle.area() == 24
```

Bad:

```python
def test_rectangle_area() -> None:
    rectangle = Rectangle(width=6, height=4)
    assert rectangle.area() == rectangle.width * rectangle.height
```

Expected values should come from a specification, domain rule, fixture, bug
report, or public contract.

## Understanding intent before writing tests

Disposition: PORTS_AS_IS.

Keep the reading order: domain first, interface second, surrounding context
third, implementation last. Python-specific sources include API docs,
OpenAPI schemas, Pydantic/msgspec models, CLI help, database constraints, and
public fixtures.

Keep the bug concentration list: boundaries, state transitions, error paths,
implicit assumptions, combinatorial interactions, and semantic correctness.

## Component vs integration tests

Disposition: PORTS_AS_IS.

Keep it. Component tests call a function or class directly. Integration tests
prove several components cooperate through a realistic boundary.

In Python, integration tests often involve fixture-managed resources:
temporary files, HTTP test clients, database transactions, subprocesses,
message queues, or event loops. They should assert coordination that one
component test cannot.

## What an integration test should assert

Disposition: PORTS_AS_IS.

Keep the list. Add Python examples:

- FastAPI/ASGI handler translates a domain exception into the documented HTTP
  response.
- A queue consumer commits an offset only after the side effect succeeds.
- A repository and service agree on transaction rollback.
- A background task emits the expected callback without corrupting state.

Avoid rechecking pure calculations already covered in component tests.
