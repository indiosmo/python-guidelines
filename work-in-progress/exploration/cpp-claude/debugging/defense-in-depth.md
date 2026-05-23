# Analysis: cpp-debugging-principles/defense-in-depth.md

Disposition: **PORTS_AS_IS** for the principle and the four-layer
structure. The "types first: parse, don't validate" subsection ports
with the Python adaptations from `design/compile-time-correctness.md`.

## Section-by-section

### Core principle: make the bug class structurally impossible

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Identical. The Python language gives the
  type-system half of the property weaker enforcement than C++
  (everything is checked at type-check time, not runtime), but the
  rule of thumb -- "a value that cannot exist also cannot be wrong"
  -- ports verbatim.

### Why one validation is not enough

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same argument. Add a Python-specific
  consideration: monkey-patching during tests sometimes bypasses
  validation layers; the rule of "validate at every layer the data
  passes through" defends against that, too.

### Types first: parse, don't validate

- **Classification:** PORTS_AS_IS.
- **Python rendering:** See `design/compile-time-correctness.md` for
  the full treatment. Two-line summary: a `Symbol.parse(raw)` factory
  that raises a domain exception or returns `Result[Symbol, ParseError]`
  carries the check once; downstream code that takes `Symbol` as a
  parameter no longer re-validates.

### Layer 1: parse at the boundary

- **Classification:** PORTS_AS_IS.
- **C++ snippet:** `parse_symbol`, `parse_order_qty`, `parse_side`
  composed via `BOOST_LEAF_ASSIGN`.
- **Python rendering:**

  ```python
  def create_order(raw: RawOrderParams) -> Order:
      symbol = Symbol.parse(raw.symbol)        # may raise InvalidSymbol
      quantity = OrderQty.parse(raw.quantity)  # may raise InvalidQty
      side = Side.parse(raw.side)              # may raise InvalidSide

      return Order(symbol=symbol, quantity=quantity, side=side)
  ```

  Pydantic alternative: define `RawOrderParams` as a pydantic
  `BaseModel` and let pydantic do the parsing on construction. The
  rest of the program operates on the typed `Order`.

### Layer 2: business-logic validation

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Identical. Domain-level invariants that the
  type system cannot express (balance check, universe check) live
  here, as plain function calls that raise on failure.

### Layer 3: environment guards

- **Classification:** PORTS_AS_IS.
- **C++ snippet:** `#ifdef BUILD_TEST_MODE` plus a runtime
  `config_.dry_run` check.
- **Python equivalent:** No preprocessor; gate on an environment
  variable, a settings flag, or a build-time module that gets
  swapped at install time. The recommended shape:

  ```python
  def send_to_exchange(order: Order) -> None:
      if settings.TEST_MODE and order.route == Route.PRODUCTION:
          raise RefusedInTestMode(order.id)

      if settings.DRY_RUN:
          log.info("dry run: would send order %s", order.id)
          return

      self._transport.send(order)
  ```

  Settings come from one well-known place (a `settings.py` reading
  env vars and a config file once at startup); they are not consulted
  module-load-time.

### Layer 4: debug instrumentation

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Identical. `log.debug(...)` for the entry
  state, `log.exception(...)` for the failure path. loguru's
  `@logger.catch` decorator wraps both in one line for an
  imperative-shell entry point.

### A worked example

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Identical. The "config loader returning empty
  names" example becomes a pydantic / dataclass with an
  `EntryName.parse` factory; the registry runs the runtime layers on
  top.

### Applying the pattern

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same steps.

### Where validation belongs, and where it does not

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rules. The Python-specific "assertions for
  programmer errors, returns for caller errors" rule becomes:
  - `raise SomeDomainError(...)` for caller-visible failures (the
    Python equivalent of `error_code::invalid_input`).
  - `assert condition, "explanation"` for programmer-error invariants
    -- bearing in mind that `assert` is removed under `python -O`, so
    do not rely on it for production behaviour.
  - `raise AssertionError(...)` or a custom `InternalError` exception
    for invariants that must hold even under `-O`.

## Suggested Python file: `python-debugging-principles/defense-in-depth.md`

Verbatim port with the Python idiom substitutions; the principle
travels unchanged.
