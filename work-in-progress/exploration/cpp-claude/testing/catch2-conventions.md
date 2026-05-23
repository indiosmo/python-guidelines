# Analysis: cpp-testing-principles/catch2-conventions.md

Disposition: **PYTHON_SPECIFIC_VARIANT**. The C++ chapter is about
Catch2 registration, CTest discovery, tag spelling, and assertion choice.
The Python equivalent is pytest: collection, marks, fixtures,
parametrize, and `assert` vs `pytest.raises`. Same purpose, different
machinery.

## Section-by-section

### Test discovery and selection (`catch_discover_tests`, `TEST_PREFIX`)

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** Each `TEST_CASE` becomes its own CTest entry; a
  prefix scopes the module's tests.
- **Python rendering:** pytest's collection follows file/module/class
  structure automatically. The equivalent of `TEST_PREFIX` is the
  pytest path / nodeid: `tests/order_routing/test_router.py::test_route_new_order`.
  Selection by node id:
  - `pytest tests/order_routing/` (by directory)
  - `pytest -k "router and not slow"` (by keyword)
  - `pytest -m "smoke"` (by mark)
  - `pytest tests/order_routing/test_router.py::test_route_new_order` (specific).

### Test serialization (`RUN_SERIAL`)

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** Tests that touch shared state must not run in
  parallel.
- **Python rendering:** pytest-xdist is the parallel runner; the
  serialization escape hatch is `@pytest.mark.serial` plus
  `--dist=loadgroup` configuration:

  ```python
  @pytest.mark.serial  # custom mark, registered in pytest.ini
  def test_touches_global_singleton(): ...
  ```

  Plus the conftest setup:
  ```python
  def pytest_collection_modifyitems(config, items):
      for item in items:
          if "serial" in item.keywords:
              item.add_marker(pytest.mark.xdist_group("serial"))
  ```

### Tags

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ tags:** `[module][component][feature]` bracketed tokens.
- **Python equivalent:** `@pytest.mark.module_name`, with all marks
  registered in `pyproject.toml` so pytest does not warn:

  ```toml
  [tool.pytest.ini_options]
  markers = [
      "slow: tests that take more than a second",
      "serial: tests that touch process-wide state",
      "integration: requires external services",
      "smoke: minimal happy-path coverage",
  ]
  ```

  Filter at the command line: `pytest -m "not slow"`.

### Assertions

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ `REQUIRE` / `CHECK`:** abort vs continue.
- **Python equivalent:** pytest only has `assert`. The equivalent of
  "abort vs continue" is:
  - **Use a single `assert` per logical fact**; pytest reports the
    first failing assertion in a test. To run multiple checks against
    a single setup, write multiple short tests or use `pytest.subtests`
    (third-party) for "continue after failure" semantics.
  - **`pytest.raises(SomeError)` context manager** for expected
    exceptions; this is the equivalent of `REQUIRE_THROWS_AS`. With
    `match="regex"` for the message check.
  - **`pytest.approx(0.1)` for floating-point comparisons** (the
    equivalent of Catch2's `Approx`).

### `CAPTURE` for table-driven tests

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ `CAPTURE`:** prints the variable when an assertion fails.
- **Python equivalent:** pytest already prints the failed assertion's
  expression and intermediate values via its assertion rewriter. For
  table-driven tests, `@pytest.mark.parametrize` with `ids=` gives each
  row a name:

  ```python
  @pytest.mark.parametrize(
      "raw,expected",
      [("0", 0), ("42", 42), ("-17", -17)],
      ids=["zero", "positive", "negative"],
  )
  def test_parse_int_decimal(raw, expected):
      assert parse_int(raw) == expected
  ```

## Suggested Python file: `python-testing-principles/pytest-conventions.md`

Rename from "catch2-conventions" to "pytest-conventions". Covers:
collection rules, nodeids, marks (with registration), parametrize ids,
fixtures and scopes, pytest-xdist and serial-test policy, plugin
inventory (pytest-benchmark, pytest-approvaltests, pyfakefs, pytest-cov,
pytest-asyncio / pytest-anyio).
