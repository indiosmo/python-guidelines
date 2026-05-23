# Analysis: cpp-design-principles/templates.md

Disposition: **PYTHON_SPECIFIC_VARIANT**. Templates and concepts are
C++-specific; Python's generics and `Protocol`-based structural typing
solve overlapping problems in a different shape.

## Section-by-section

### Concept constraints

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** A concept names the operations a template parameter
  must support; misuse fails at the call site with a readable diagnostic.
- **Python rendering:** A `Protocol` plays the same role. The "fails at
  the call site with a readable diagnostic" property requires
  `pyright --strict` or `pyrefly`; at runtime Python is happy to call
  methods that do not exist until they fail. Example:

  ```python
  from typing import Protocol

  class Serializable(Protocol):
      def write_json(self, target: JsonValue) -> None: ...

  def publish[T: Serializable](message: T) -> None: ...

  publish(42)  # pyright: "int" is incompatible with "Serializable"
  ```

  Concept composition (`A && B`) becomes Protocol inheritance:

  ```python
  class TimestampedEvent(Serializable, Protocol):
      timestamp: datetime
  ```

### Constrain templates intended for a specific shape

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** A bare `typename T` hides the contract in the body;
  a concept names it in the signature.
- **Python rendering:** A bare `T` in `def foo[T](x: T) -> T: ...` is
  fine for genuinely-generic helpers. As soon as the body assumes a
  shape (`x.error_code`, `x.fault()`), constrain with a Protocol bound
  (`def foo[T: SomeProtocol](x: T) -> T:`). The narrower the bound, the
  earlier the type checker catches misuse.

  The C++ "concept paired with `is_X_v` trait" pattern (which checks
  "is this a specialisation of `outcome<R, E>`") has no clean Python
  analogue. Python generics do not distinguish "any object with the
  right shape" from "an instance of a specific generic class"; the
  type checker treats them as equivalent. If the distinction must hold
  at runtime, use `isinstance(x, Outcome)` (assuming `Outcome` is an
  actual class, not a Protocol).

### Compile-time dispatch with inline `requires`

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** `if constexpr (requires { alt.request_id; })`
  picks a branch based on whether an expression is well-formed for
  the current template parameter.
- **Python rendering:** Two shapes:

  - **Runtime capability check** with `hasattr`:

    ```python
    def request_id_of(event: OrderEvent) -> str | None:
        if hasattr(event, "request_id"):
            return event.request_id
        return None
    ```

    The type checker does not narrow `event` to "things with
    `request_id`" inside the branch. To help it:

  - **Protocol-based narrowing** with `isinstance`:

    ```python
    @runtime_checkable
    class HasRequestId(Protocol):
        request_id: str

    def request_id_of(event: OrderEvent) -> str | None:
        if isinstance(event, HasRequestId):
            return event.request_id
        return None
    ```

  - **`TypeIs` user-defined narrowing** (PEP 742, available in
    `typing_extensions` and typing in 3.13+):

    ```python
    from typing import TypeIs

    def has_request_id(event: OrderEvent) -> TypeIs[OrderPlaced | OrderCanceled]:
        return hasattr(event, "request_id")
    ```

    This is the cleanest spelling when the union is small and stable.

### Dependent false for template `static_assert`s

- **Classification:** DROP.
- **Reason:** Python's `assert_never(value)` already lives at runtime
  and is type-checker-aware; there is no instantiation lifecycle to
  defer the assertion through.

## Suggested Python file: `python-design-principles/generics-and-protocols.md`

A rewritten chapter covering: PEP 695 generic syntax, `Protocol` and
when to reach for it vs ABC, `TypeIs` / `TypeGuard` for runtime
narrowing, `assert_never` for exhaustiveness, and pyright/pyrefly
strict-mode settings that surface generic misuse early.
