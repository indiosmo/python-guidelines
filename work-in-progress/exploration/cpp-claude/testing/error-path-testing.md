# Analysis: cpp-testing-principles/error-path-testing.md

Disposition: **PORTS_WITH_ADAPTATION**. The principle (test the success
path, the failure path, and the rollback after a failed mutation) ports
directly. The mechanics change: Python's primary error channel is
exceptions, so the `expected`-vs-`throws` split collapses into a single
`pytest.raises` shape (plus an optional `Result` library for the codebases
that use one).

## Section-by-section

### Testing `expected`-style return types

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ snippet:**
  ```cpp
  auto result = parse_int("42", 10);
  REQUIRE(result);
  CHECK(*result == 42);
  ```
- **Python equivalent (the default: exceptions):**
  ```python
  def test_parse_int_decimal_positive():
      assert parse_int("125255") == 125255
  ```
  And for the failure path:
  ```python
  def test_parse_int_decimal_invalid():
      with pytest.raises(ParseError, match="invalid_input"):
          parse_int("abc")
  ```
- **Python equivalent (codebases using a Result library):**
  ```python
  from result import Ok, Err

  def test_parse_int_ok():
      result = parse_int("42")
      assert result == Ok(42)

  def test_parse_int_err():
      result = parse_int("abc")
      assert result == Err(ParseError.INVALID_INPUT)
  ```

### Testing exception-throwing code

- **Classification:** PORTS_AS_IS (the principle); PORTS_WITH_ADAPTATION
  (the spelling).
- **Python equivalent:**

  ```python
  def test_checked_multiply_overflow():
      with pytest.raises(OverflowError, match="overflow"):
          checked_multiply(sys.maxsize, 2)

  def test_checked_multiply_ok():
      assert checked_multiply(100, 100) == 10_000
  ```

  `pytest.raises(SomeException, match="regex")` is the equivalent of
  `REQUIRE_THROWS_MATCHES`. For "exception with structured payload"
  (the Python equivalent of a typed LEAF handler), bind the caught
  exception:

  ```python
  with pytest.raises(InvalidFieldValue) as exc_info:
      to_internal(WireSide.UNKNOWN)
  assert exc_info.value.field == "side"
  ```

### Exception safety and scope-guard rollback

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Trigger the failing branch, inspect post-call
  state to confirm rollback fired.
- **Python rendering:** Same shape. Use the dataclass `__eq__` to assert
  on the post-call state. The bad anti-example (asserting cleanup via
  a follow-up "good" call) translates verbatim.

  ```python
  def test_registry_invalid_entry_rolls_back():
      r = Registry()
      with pytest.raises(InvalidEntry):
          r.register_entry(bad_entry)
      # Rollback fired; no trace of bad_entry.
      assert bad_entry.id not in r.entries
  ```

### Result-library testing (when one is used)

- **Classification:** PORTS_WITH_ADAPTATION (new content for the
  Python guide).
- **Recommended approach:** Provide a small fixture or helper that
  unwraps an `Ok` value while asserting on the success path:

  ```python
  def unwrap_ok(result: Result[T, E]) -> T:
      match result:
          case Ok(value): return value
          case Err(error): pytest.fail(f"expected Ok, got Err({error!r})")
  ```

  The test reads as:
  ```python
  def test_parse_int_ok():
      value = unwrap_ok(parse_int("42"))
      assert value == 42
  ```

  This is the equivalent of the C++ `LIB_REQUIRE_RESULT` macro the
  C++ guide recommends.

## Suggested Python file: `python-testing-principles/error-path-testing.md`

A rewritten chapter that defaults to the `pytest.raises` shape (the
common case in Python) and devotes a short subsection to the `Result`
library style for codebases that use one. The rollback / state-after-
failure rule ports verbatim.
