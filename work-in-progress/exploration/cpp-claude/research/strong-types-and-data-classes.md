# Strong types and data classes in Python (2026)

## Mapping table

| C++ idiom                                      | Python equivalent                                                                                                                                      |
|------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| `lib::strong_type<T, Tag>` for a domain id     | `NewType("OrderId", str)` for the simplest case; `Annotated[str, Brand("OrderId")]` when validation must travel with the type.                          |
| `lib::strong_type<T, Tag>` with validation     | A small frozen pydantic `BaseModel` or `dataclass(frozen=True)` wrapping the value, with a parse-don't-validate factory returning `Result` / raising.   |
| `lib::fixed_string<N>` (bounded text)          | No memory-layout analogue; closest semantic match is `Annotated[str, StringConstraints(max_length=N)]` (pydantic) or a frozen wrapper that validates.   |
| Plain aggregate `struct`                       | `@dataclass(frozen=True, slots=True)` for internal domain models; `@attrs.frozen(slots=True)` if attrs is already in the codebase.                      |
| Designated initialisers                        | dataclass constructors are keyword-based by default; enforce with `@dataclass(kw_only=True)`. attrs and pydantic do the same with `kw_only=True`.        |
| `static constexpr` discriminator on a variant  | A `Literal[...]` class attribute on each dataclass; the union is a `type Event = Placed \| Canceled` discriminated by that attribute.                   |
| `std::variant` exhaustive `lib::match`         | `match`/`case` over the union with a `case _ as never: assert_never(never)` final arm; pyright/pyrefly verify exhaustiveness.                            |

## When to choose which

- **NewType** -- when the type is purely an identity wrapper, the underlying
  primitive is fine, and the value enters the domain already validated
  (parsed at the boundary, then carried as a `NewType` for the rest of the
  call chain). Cheap, zero runtime cost.
- **frozen dataclass / attrs.frozen** -- internal domain value objects,
  trusted inputs, zero-cost equality/hashing, slots reduce memory ~40%.
  This is the default for plain "carry data" structs from C++.
- **pydantic BaseModel** -- the value crosses a trust boundary (HTTP body,
  config file, CLI flags, JSON from disk, DB row coming back through a
  driver that loses typing). pydantic v2 is the parse-don't-validate
  primitive: a factory that returns `Model | ValidationError`. Pay the 2-3x
  overhead at the boundary, not in the hot loop. Internal flow uses the
  frozen dataclass form.
- **attrs.frozen** -- when finer per-field validators are needed and
  pydantic is overkill; or when slots + converters compose more cleanly.

## "Strong type" recipe

For the equivalent of `lib::strong_type<uint64_t, OrderIdTag>`:

```python
from typing import NewType

OrderId = NewType("OrderId", str)
UserOrderId = NewType("UserOrderId", str)

def place_order(order_id: OrderId, user_order_id: UserOrderId) -> None: ...
```

Pyright/pyrefly will reject `place_order(user_order_id, order_id)`. To
construct: `OrderId("O-123")`. The constructor is identity at runtime; the
distinction lives in the type system. This is the closest Python gets to
the "compile-time check, zero runtime cost" property of `lib::strong_type`.

For values that must be parsed (a symbol that can be empty, a quantity that
can be negative):

```python
@dataclass(frozen=True, slots=True)
class Symbol:
    value: str

    @classmethod
    def parse(cls, raw: str) -> Symbol:
        if not raw or not raw.isalnum():
            raise ValueError(f"invalid symbol {raw!r}")
        return cls(raw)
```

Or pydantic with `Annotated`:

```python
from typing import Annotated
from pydantic import AfterValidator

def _validate_symbol(v: str) -> str:
    if not v or not v.isalnum():
        raise ValueError("invalid symbol")
    return v

Symbol = Annotated[str, AfterValidator(_validate_symbol)]
```

Either way, the symbol type carries its proof; the rest of the program never
re-checks.

## Sources

- https://docs.pydantic.dev/latest/concepts/types/
- https://docs.pydantic.dev/latest/concepts/validation_decorator/
- https://tildalice.io/python-dataclasses-pydantic-attrs/
- https://dataclass-settings.readthedocs.io/en/stable/class_compatibility.html
