# Analysis: cpp-testing-principles/philosophy.md

Disposition: **PORTS_AS_IS**. The entire chapter is language-agnostic.

## Section-by-section

### Test behavior, not implementation

- **Classification:** PORTS_AS_IS.
- **C++ principle:** A test verifies the intended semantics of the
  code; never re-derive the expected value from the implementation.
- **Python rendering:** Identical. The rectangle example translates
  verbatim:

  ```python
  def test_rectangle_area():
      r = Rectangle(width=6, height=4)
      assert r.area() == 24  # 6 * 4, from the definition of area
  ```

  The URL percent-encode example translates directly.

### Understanding intent before writing tests

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same procedure. The Python "reading order" is
  almost the same: read public APIs and docstrings first, callers
  second, implementation last. Stub files and PEP 484 type hints are
  the Python equivalent of "headers" in the reading order.

### Component vs integration tests

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same distinction. pytest's parametrize / fixtures
  scoping (`function` / `class` / `module` / `session`) supports the
  shape; recommend the default `function` scope for component tests and
  `module`/`session` for genuinely shared integration setup.

## Suggested Python file: `python-testing-principles/philosophy.md`

Verbatim port with the rectangle and URL examples in Python.
