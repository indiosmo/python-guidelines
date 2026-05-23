# Analysis: cpp-testing-principles/test-patterns.md

Disposition: **PORTS_WITH_ADAPTATION**. Catch2's `TEST_CASE` / `SECTION`
/ `GENERATE` / `TEMPLATE_TEST_CASE` family becomes pytest's `def test_`
+ `pytest.mark.parametrize` + `subtests` patterns. Same purpose,
different vocabulary.

## Section-by-section

### Choosing a test pattern

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ vocabulary -> Python vocabulary:**

  | Catch2 construct                     | pytest equivalent                                                                     |
  |--------------------------------------|----------------------------------------------------------------------------------------|
  | Separate `TEST_CASE`                 | Separate `def test_`. Default. Each function is one scenario.                          |
  | `SECTION` (shared setup, branching)  | A fixture for the shared setup + separate `def test_` per branch; or use `pytest-subtests` when the branches must share intermediate locals. |
  | `GENERATE(values...)`                | `@pytest.mark.parametrize("name", [values])`                                            |
  | `TEMPLATE_TEST_CASE(types...)`       | `@pytest.mark.parametrize("cls", [TypeA, TypeB])` -- Python has no static template machinery; the dynamic form is fine.                  |
  | `GENERATE(table<...>({rows}))`       | `@pytest.mark.parametrize("input,expected", [(...)], ids=[...])`                       |

  Examples:

  ```python
  # separate function per scenario
  def test_parse_int_success(): ...
  def test_parse_int_invalid_input(): ...

  # shared setup via fixture
  @pytest.fixture
  def policy():
      return PasswordPolicy(min_length=8, max_length=64)

  def test_validate_password_within_bounds(policy):
      assert validate_password("correct_horse", policy)

  def test_validate_password_too_short(policy):
      assert not validate_password("short", policy)

  # generated scalar inputs
  @pytest.mark.parametrize("c", [" ", "\t", "\n", "\r"])
  def test_is_whitespace(c):
      assert is_whitespace(c)

  # same logic across types
  @pytest.mark.parametrize("container_cls", [list, collections.deque])
  def test_push_back(container_cls):
      c = container_cls()
      c.append(42)
      assert len(c) == 1
  ```

### `TEMPLATE_TEST_CASE` -> pytest parametrize on class

- **Classification:** PORTS_AS_IS (the principle).
- **C++ principle:** Pays off only when type substitution alone is
  enough; otherwise the per-type dispatch costs more than two
  `SECTION`s.
- **Python rendering:** Same advice. Python's dynamic typing makes
  the cross-type test even cheaper -- a parametrize over classes
  reads naturally, no `if constexpr` needed. The smell to watch for
  is the same: if the test body has `if cls is X: ... else: ...`
  branches, write two tests instead.

### Composed patterns

- **Classification:** PORTS_AS_IS.
- **Python rendering:** pytest's parametrize stacks:

  ```python
  @pytest.mark.parametrize("role", [Peer.CLIENT, Peer.SERVER])
  @pytest.mark.parametrize("scenario", ["partial_send", "peer_closes"])
  def test_tcp_connection_data_transfer_transitions(role, scenario):
      conn = make_established_connection(role)
      ...
  ```

  Runs four sub-cases just like the Catch2 composition.

### Compile-time tests

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ `static_assert`:** type-level contracts checked at compile
  time.
- **Python equivalent:** **type-checker-only tests**. Use
  `assert_type` (PEP 488 / `typing_extensions`) for "the inferred
  type at this expression is exactly X":

  ```python
  from typing import assert_type

  def test_parse_returns_symbol_type():
      result = parse_symbol("AAPL")
      assert_type(result, Symbol)  # type-checker error if wrong
  ```

  Or write a `tests/typing/test_*.py` that pyright/pyrefly checks
  separately. `static_assert(serializable<T>)` becomes "a function
  parameter typed as `T: SerializableProto` -- if T doesn't satisfy
  the protocol, the type checker complains at the call site".

  The dependent-false / `dependent_false_v` trick has no analogue;
  Python's `assert_never` covers the runtime exhaustiveness case.

### Table-driven testing

- **Classification:** PORTS_AS_IS.
- **Python rendering:** `pytest.mark.parametrize` is the table; the
  `ids=` argument provides the label column. For the "many columns"
  case the C++ guide recommends "separate vectors", the Python
  equivalent is a list of typed dataclasses:

  ```python
  @dataclass(frozen=True, slots=True)
  class TestCase:
      label: str
      outcome: str
      year: int
      month: int
      day: int

  @dataclass(frozen=True, slots=True)
  class Expected:
      year: int; month: int; day: int
      day_of_week: int; day_of_year: int; iso_week: int

  CASES: list[tuple[TestCase, Expected]] = [
      (TestCase("in range", "no rollover", 2024, 3, 1),
       Expected(2024, 3, 1, 5, 61, 9)),
      ...
  ]

  @pytest.mark.parametrize("case,expected", CASES,
                           ids=lambda x: x.label if hasattr(x, "label") else None)
  def test_normalize_date(case, expected):
      result = normalize_date(case.year, case.month, case.day)
      assert result.year == expected.year
      ...
  ```

## Suggested Python file: `python-testing-principles/test-patterns.md`

A near-verbatim port with the Catch2 -> pytest vocabulary swap, a
short subsection on the `assert_type` runtime check, and a callout for
`pytest-subtests` when the test really needs the "branch after shared
setup" shape.
