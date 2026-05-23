# Templates: concepts and dispatch

Overall disposition: PORTS_WITH_ADAPTATION.

Rewrite as `generics-and-protocols.md`. Python generics are for static
analysis and editor feedback, not code generation. Use them when they clarify
the contract; avoid clever type plumbing that makes Python harder to read.

## Concept constraints

Disposition: PORTS_WITH_ADAPTATION.

C++ concepts map to `Protocol`, bounded `TypeVar`, constrained `TypeVar`, and
overloads.

```python
class Serializable(Protocol):
    def write_json(self) -> JsonValue: ...


def publish(message: Serializable) -> None:
    sink.write(message.write_json())
```

With PEP 695 syntax:

```python
def first[T](items: Sequence[T]) -> T:
    return items[0]
```

## Constrain templates intended for a specific shape

Disposition: PORTS_WITH_ADAPTATION.

Keep the rule. If a function assumes an object has methods or attributes, say
so in a `Protocol` instead of accepting `Any`.

```python
class Outcome(Protocol[T]):
    result_type: type[T]

    @classmethod
    def error(cls, error: Exception) -> Self: ...


def make_error_handlers[T](outcome: type[Outcome[T]], request: Request) -> Handlers:
    ...
```

Use `Any` at genuine escape hatches: untyped libraries, JSON-like values,
dynamic plugin dispatch, or exploratory code.

## Compile-time dispatch with inline `requires`

Disposition: PYTHON_SPECIFIC_VARIANT.

Python should not use ad hoc `hasattr` as a static-dispatch substitute unless
the dynamic behavior is truly intended. Prefer protocols or pattern matching.

```python
class HasRequestId(Protocol):
    request_id: RequestId


def request_id_of(event: object) -> RequestId | None:
    if isinstance(event, OrderPlaced | OrderCanceled):
        return event.request_id
    return None
```

If many alternatives share a field, a shared base dataclass or protocol may be
the clearer design.

## Dependent false for template `static_assert`s

Disposition: DROP.

No Python analogue is worth a chapter section. The closest idiom is
`assert_never` for unreachable union alternatives.

```python
def handle(event: Event) -> None:
    match event:
        case OrderPlaced():
            ...
        case OrderCanceled():
            ...
    assert_never(event)
```

## Additional Python topics to add

Disposition: PYTHON_SPECIFIC_VARIANT.

The Python guide should include sections absent from the C++ chapter:

- PEP 695 `class Box[T]` and `def f[T](...)` syntax.
- `type X = ...` aliases.
- `ParamSpec` for decorators.
- `Self` for fluent APIs and alternate constructors.
- Variance by default inference and when explicit variance still appears.
- `TypedDict` for dictionary-shaped boundaries.
- `Protocol` versus ABC versus duck typing.
