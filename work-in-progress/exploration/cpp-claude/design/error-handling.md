# Analysis: cpp-design-principles/error-handling.md

Disposition: **PYTHON_SPECIFIC_VARIANT** -- the high-level principles
("errors live in the domain that produces them", "errors do not cross
unrelated boundaries", "top-level functions catch everything", "specific
handlers before catch-alls") all survive, but the machinery is wholly
different: Python uses **exceptions**, not result types, as the in-domain
failure channel, and the LEAF / `std::expected` distinction collapses.

## Section-by-section

### Two result types (`lib::result<T>` vs `std::expected<T, E>`)

- **Classification:** DROP the two-result-types frame; replace with the
  Python error-channel discussion.
- **C++ principle:** A LEAF-style result inside a domain, an
  `expected<T, E>` at the boundary, never let LEAF cross the seam.
- **Python rendering:** Exceptions inside a domain (subclass a domain
  base exception per domain); convert at the boundary if the caller
  needs a different shape. Three concrete patterns:

  1. **Pure exceptions, single domain hierarchy.** The default. Each
     domain owns a base exception (`class RoutingError(Exception):`)
     and concrete subclasses (`class UnknownOrder(RoutingError):`).
  2. **`Result` library** (e.g. `result` package, or a small in-house
     `Result = Ok[T] | Err[E]`). Only useful when the consuming domain
     wants type-checker-visible failure branches. Recommend as the
     **boundary** form for code that needs to be exhaustive on errors,
     not as the in-domain form.
  3. **`typing.Never` / `assert_never`** for fallthrough on exhaustive
     enum/union dispatch.

### Domain error types (`error_code.hpp` + `errors.hpp`)

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Each domain ships `error_code.hpp` (enum +
  category) and `errors.hpp` (structured payloads).
- **Python rendering:** Each domain ships an `errors.py` (or `exceptions.py`)
  with:

  - A domain base exception: `class RoutingError(Exception):`
  - Concrete structured exceptions: `@dataclass(frozen=True, slots=True)
    class UnknownOrder(RoutingError): order_id: OrderId` (frozen
    dataclass methods compose with `Exception.__init__` via a small
    `__post_init__`).
  - An `ErrorCode` `IntEnum` registered with a per-domain prefix when
    interop with non-Python observers (Prometheus codes, gRPC status,
    JSON serialization) matters. Otherwise just the exception class.

  Example:

  ```python
  from dataclasses import dataclass
  from enum import IntEnum

  class ErrorCode(IntEnum):
      INVALID_FIELD_VALUE = ROUTING_PREFIX + 1
      MISSING_REQUIRED_FIELD = ROUTING_PREFIX + 2
      UNKNOWN_ORDER = ROUTING_PREFIX + 3

  class RoutingError(Exception):
      """Base exception for the order_routing domain."""

  @dataclass(frozen=True, slots=True)
  class UnknownOrder(RoutingError):
      order_id: OrderId

      def __post_init__(self) -> None:
          super().__init__(f"unknown order {self.order_id}")
  ```

  The `__post_init__` keeps `str(exc)` meaningful for logs while
  the structured fields stay available to typed handlers.

### Handlers match on the structured type

- **Classification:** PORTS_AS_IS (the principle), PORTS_WITH_ADAPTATION
  (the spelling).
- **C++ principle:** Ordered handlers, specific before catch-all.
- **Python rendering:** Same. Use `try` / `except` with the most-specific
  exception type first, the domain base second, and a final `except
  Exception` only when the top-level function genuinely catches
  everything. `match` over a caught exception is the closest Python
  comes to LEAF's tuple-of-handlers:

  ```python
  try:
      route(request)
  except DuplicateOrder | DuplicateRequest as e:
      log.warning("duplicate %s", e)
  except InvalidFieldValue as e:
      reject(request, e.field, e.value)
  except RoutingError:
      log.exception("unhandled routing error")
  ```

### Exceptions do not cross component boundaries

- **Classification:** PORTS_AS_IS (the principle inverts: Python
  defaults to exceptions traveling, so the rule is "stop them at the
  module boundary unless the caller speaks the same exception
  vocabulary").
- **Python rendering:** Same rule. Document it as: a public function
  raises exceptions from its own domain hierarchy plus whitelisted
  stdlib types (`ValueError`, `KeyError`). It does not propagate
  exceptions from sibling domains; it catches and translates them at
  the seam.

### Top-level functions catch everything

- **Classification:** PORTS_AS_IS.
- **C++ principle:** A top-level function is responsible for catching
  everything it produces.
- **Python rendering:** Same rule. The Python "top-level" surfaces are:

  - FastAPI / Starlette route handlers (use `exception_handler`).
  - asyncio / anyio task functions launched via `task_group.start_soon`.
  - Dagster `@asset` / `@op` functions.
  - Click / Typer command handlers.
  - Test entry points (let exceptions propagate; pytest handles them).

  Document a `@catch_all_to_log` decorator helper as the recommended
  spelling for the cases where a runtime is one frame above the
  top-level function.

### Diagnostic context

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Capture `source_location` and stack traces on
  error construction.
- **Python rendering:** Python tracebacks are free; the equivalent of
  "attach a stacktrace" is "let the exception propagate or capture
  `traceback.format_exception(exc)`". For structured logging, attach
  the exception object to a `logger.exception(...)` call; loguru does
  this automatically with `@logger.catch`.

### `BOOST_LEAF_ASSIGN` / `BOOST_LEAF_CHECK` / `result.value()` traps

- **Classification:** DROP. These are C++-specific.
- **Replace with:** A short section on **two Python traps** in the
  same shape:

  - **`raise` inside a `finally` masks the original exception.** Use
    `raise X from None` only when you genuinely want to suppress the
    cause; otherwise let Python chain via `__cause__`.
  - **Bare `except:` and `except Exception:` are not the same.** The
    bare form swallows `KeyboardInterrupt` and `SystemExit`; the
    rule is `except Exception` at the top-level, never `except:`.

## Suggested Python file: `python-design-principles/error-handling.md`

A wholly rewritten chapter. The themes carry over (domain ownership,
boundaries, top-level catch-all, structured context, ordering of
handlers) but the C++ macro vocabulary is replaced with exception
class design, `except` clause ordering, and a short discussion of
when a `Result[T, E]` library is worth its weight.
