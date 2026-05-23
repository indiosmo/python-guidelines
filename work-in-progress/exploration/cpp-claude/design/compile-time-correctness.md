# Analysis: cpp-design-principles/compile-time-correctness.md

Disposition: **PYTHON_SPECIFIC_VARIANT**. The principle (push checks into
something earlier than runtime) ports; the mechanism changes wholesale:
Python has no compile step, so "compile-time" becomes "type-checker time"
(pyright/pyrefly --strict) plus "parse-once boundary validation"
(pydantic / dataclass factories).

## Section-by-section

### Strong typing

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Wrap primitives in strong types so the compiler
  catches argument swaps.
- **Python rendering:** Two tiers (see `research/strong-types-and-data-classes.md`):

  - `NewType` for the lightweight "distinct identity over the same
    primitive" case:

    ```python
    AccountId = NewType("AccountId", str)
    Symbol = NewType("Symbol", str)
    OrderQty = NewType("OrderQty", int)
    Price = NewType("Price", float)

    def place_order(account: AccountId, symbol: Symbol,
                    qty: OrderQty, px: Price) -> None: ...
    ```

    `pyright --strict` rejects `place_order(symbol, account, qty, px)`
    at type-check time. There is no runtime cost; `NewType` is identity.

  - Frozen dataclass / `Annotated` + pydantic when the type needs to
    carry validation past the boundary (see "Parse, don't validate"
    below).

### Per-domain `types` namespace

- **Classification:** PORTS_AS_IS (with a Python-specific simplification).
- **C++ principle:** Every domain ships a `types.hpp` inside a nested
  `types` namespace.
- **Python rendering:** Every domain package ships a `types.py`. Python
  already has the namespacing for free: `from order_routing import types
  as routing_types`, then `routing_types.OrderId`. Drop the C++ caveat
  about field-vs-type name shadowing (Python module attributes do not
  shadow dataclass field names).

### Parse, don't validate

- **Classification:** PORTS_AS_IS.
- **C++ principle:** A `types::symbol` whose factory takes
  `std::string_view` and returns `expected<...>` moves the check to a
  single site; downstream code trusts the refined type.
- **Python rendering:** Identical. The Python factory raises a domain
  exception or returns a `Result`:

  ```python
  @dataclass(frozen=True, slots=True)
  class Symbol:
      value: str

      @classmethod
      def parse(cls, raw: str) -> Symbol:
          if not raw or not raw.isalnum():
              raise InvalidSymbol(value=raw)
          return cls(raw)

  def place_order(symbol: Symbol, ...) -> None:
      # `symbol` is well-formed by construction. No re-validation.
      ...
  ```

  Or with pydantic v2:

  ```python
  from typing import Annotated
  from pydantic import AfterValidator

  def _check_symbol(v: str) -> str:
      if not v or not v.isalnum():
          raise ValueError("invalid symbol")
      return v

  Symbol = Annotated[str, AfterValidator(_check_symbol)]
  ```

  Recommend the pydantic spelling at the boundary (HTTP input, config
  file, JSON), the frozen dataclass spelling internally.

### Data structs as plain aggregates

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Structs that carry data should be aggregates;
  `const` non-static members break reflection, move semantics,
  triviality.
- **Python rendering:** Internal data classes are `@dataclass(frozen=True,
  slots=True)`. Frozen makes them immutable (the Python equivalent of
  "every field is `const`"), and unlike C++, frozen does not break
  copy/replace -- `dataclasses.replace(obj, field=new)` returns a new
  instance.

  For the "static constexpr discriminator" pattern, use `Literal[...]`
  class attributes:

  ```python
  @dataclass(frozen=True, slots=True)
  class OrderPlaced:
      kind: Literal["placed"] = "placed"
      order_id: OrderId
      request_id: RequestId

  @dataclass(frozen=True, slots=True)
  class OrderCanceled:
      kind: Literal["canceled"] = "canceled"
      order_id: OrderId
      request_id: RequestId

  type OrderEvent = OrderPlaced | OrderCanceled
  ```

  Pydantic v2's discriminated unions key on the same `Literal` field;
  this pattern travels both ways.

### Exhaustiveness checking

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Prefer `enum class` + `switch` without `default`;
  fallthrough returns a structured error.
- **Python rendering:** `enum.Enum` (or `StrEnum`/`IntEnum`) plus
  `match` / `case` with a final `case _ as never: assert_never(never)`
  arm. See `research/exhaustive-matching.md`.

  Boundary parsers need an explicit `case _:` that raises a structured
  exception, same as the C++ "default arm required" rule.

### When `std::unreachable()` is appropriate

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Reserve for `constexpr` helpers and logging
  fallbacks; otherwise return a structured error.
- **Python rendering:** Python's equivalent of `std::unreachable` is
  `typing.assert_never(value)`. It raises at runtime if reached --
  there is no UB equivalent. Prefer raising a domain exception for
  caller-visible failures; use `assert_never` for "the type checker
  proves this branch is unreachable, but I want a runtime trip if I'm
  wrong".

### `if/else` chains need an explicit `else`

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same. `match` over an enum has the same
  exhaustiveness story; an `if/elif` chain does not, so end with
  `else: assert_never(...)` or `else: raise ...`.

### Designated initializers

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Prefer designated initializers; trailing commas;
  omit defaulted fields; brace elision for class types.
- **Python rendering:** Identical. Dataclass / pydantic constructors
  are already keyword-based; recommend `@dataclass(kw_only=True)` to
  ban positional construction outright. Trailing-comma rule comes from
  ruff (`COM812`). Omit defaulted fields rule is automatic (Python
  defaults apply when arguments are not passed). Brace-elision has no
  Python analogue; the equivalent ergonomic win is the `NewType` /
  type-alias spelling that lets `Symbol("X")` read as cleanly as
  `types::symbol{"X"}`.

## Suggested Python file: `python-design-principles/types-and-correctness.md`

Title shift from "compile-time" to "types-and-correctness" because the
mechanism shifted. The chapter covers NewType, strong-type dataclasses,
parse-don't-validate, discriminated unions with `Literal`, `enum.Enum`
+ exhaustive `match`, and pyright/pyrefly --strict configuration.
