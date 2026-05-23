# Analysis: cpp-design-principles/cross-cutting.md

Disposition: **PYTHON_SPECIFIC_VARIANT**. The principle (cross-cutting
services configured once at startup; downstream code reaches them through
free functions; tests substitute by swapping the implementation, no
mocking framework) ports directly. The mechanism is wholly different:
Python has no `std::variant` performance argument, no ODR hazards under
dynamic linking, and module-level globals are already a single address.
The Python guide should rewrite the chapter around `contextvars`,
module-level singletons, and `pytest` fixtures.

## Section-by-section

### The problem

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Threading clocks/loggers through every constructor
  distorts every signature; polymorphic singletons trade that for vtable
  cost and inheritance coupling.
- **Python rendering:** Same problem statement. Python's traps are
  slightly different:

  - Threading `clock`/`logger` through every constructor: same
    annoyance, same readability loss.
  - Mocking via `unittest.mock.patch` at the module level: cheap to
    write, but the patched name is sometimes the wrong import (the
    consumer imported the symbol into its own module's namespace and
    the test patched the source module). The cross-cutting pattern
    sidesteps this.

### The pattern: variant + free functions

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** A `std::variant<Impl1, Impl2, ...>` global plus
  free functions that dispatch via `lib::match`.
- **Python rendering:** Two cleaner Python shapes, recommended in order:

  - **Module-level singleton with `contextvars` override:**

    ```python
    # logging_service.py
    from contextvars import ContextVar
    from typing import Protocol

    class Logger(Protocol):
        def log(self, level: Level, message: str) -> None: ...

    class _NullLogger:
        def log(self, level: Level, message: str) -> None: ...

    _logger: ContextVar[Logger] = ContextVar("logger", default=_NullLogger())

    def configure(impl: Logger) -> None:
        _logger.set(impl)

    def log(level: Level, message: str) -> None:
        _logger.get().log(level, message)
    ```

    `contextvars` is the equivalent of a per-task / per-thread override
    -- it composes with asyncio (each task sees its own value) and with
    threads (each thread sees its own value). A test fixture sets a
    fresh value in setup and the `ContextVar` machinery restores the
    previous value automatically when the test's context exits.

  - **`functools.singledispatch` for dispatch shape**: same as the C++
    `lib::match` over the variant; useful when the implementation is
    chosen by **type** of the request rather than by the active
    service. Less common for clock/logger.

  Drop the `std::variant` argument entirely; Python's module-level
  singleton is already a single address (one module object) and there
  is no dynamic-linking ODR concern. The closed-set-vs-open-set
  trade-off survives: a Protocol-typed slot accepts any conforming
  implementation, so the set is genuinely open in Python.

### Test substitution

- **Classification:** PORTS_AS_IS.
- **Python rendering:** A pytest fixture is the natural shape:

  ```python
  @pytest.fixture
  def mock_clock():
      token = clock._clock.set(MockClock(now_value=datetime(2026, 1, 1)))
      yield clock._clock.get()
      clock._clock.reset(token)
  ```

  The `set` / `reset` token pair is the equivalent of the C++
  destructor restoring the previous value.

### Catalog: logger, timer, clock

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same three services, same shape. The Python
  recommended implementations:
  - **logger:** loguru (default) or stdlib `logging` with a single
    `getLogger(__name__)` per module; configured once at startup.
  - **clock:** A `Protocol` with `now() -> datetime`; production
    impl wraps `datetime.now(tz=UTC)`, test impl is mutable.
  - **timer:** asyncio's `loop.call_later` for the production case;
    a manual / fake-clock-driven timer for tests.

### The match helper

- **Classification:** DROP.
- **Reason:** Python's `match` / `case` is built in; `functools.singledispatch`
  covers the type-keyed dispatch case. No project-specific helper needed.

## Suggested Python file: `python-design-principles/cross-cutting.md`

A rewritten chapter built around `contextvars`-backed module singletons
and pytest fixtures. The C++ "why a variant, not an inline template
variable" argument is dropped; the testability story stays.
