# Test patterns

Overall disposition: PORTS_WITH_ADAPTATION.

The selection logic ports, but Catch2 constructs become pytest functions,
fixtures, parametrization, and markers.

## Choosing a test pattern

Disposition: PORTS_WITH_ADAPTATION.

Separate `TEST_CASE` maps to separate test functions:

```python
def test_parse_int_success() -> None: ...
def test_parse_int_invalid_input() -> None: ...
```

`SECTION` maps to either separate tests sharing a fixture, or a parametrized
test when the setup and assertion shape are the same.

```python
@pytest.fixture
def password_policy() -> PasswordPolicy:
    return PasswordPolicy(min_length=8, max_length=64)


def test_password_within_bounds(password_policy: PasswordPolicy) -> None:
    assert validate_password("correct_horse", password_policy)
```

`GENERATE` maps to `pytest.mark.parametrize`.

```python
@pytest.mark.parametrize("character", [" ", "\t", "\n", "\r"])
def test_is_whitespace(character: str) -> None:
    assert is_whitespace(character)
```

Typed tests map to parametrized fixtures or protocols. Use them only when the
same assertion body naturally covers all implementations.

## Compile-time tests

Disposition: PYTHON_SPECIFIC_VARIANT.

Python type-level tests are type-checker tests. Options:

- Keep representative snippets under `tests/typing/` and run pyright/pyrefly.
- Use `typing.assert_type` in checked code when useful.
- Use `assert_never` to force union exhaustiveness.

```python
def describe(event: Event) -> str:
    match event:
        case OrderPlaced():
            return "placed"
        case OrderCanceled():
            return "canceled"
    assert_never(event)
```

Runtime pytest cannot prove a static type contract by itself.

## Table-driven testing

Disposition: PORTS_WITH_ADAPTATION.

Use `pytest.mark.parametrize` with row ids. Prefer explicit dataclass rows for
wide cases.

```python
@dataclass(frozen=True)
class DateCase:
    label: str
    year: int
    month: int
    day: int
    expected: NormalizedDate


@pytest.mark.parametrize("case", CASES, ids=lambda case: case.label)
def test_normalize_date(case: DateCase) -> None:
    assert normalize_date(case.year, case.month, case.day) == case.expected
```

Use `pytest.param(..., id="...")` when inline rows are shorter.

## Property tests

Disposition: PYTHON_SPECIFIC_VARIANT.

Add a Python-specific section for Hypothesis. It should complement, not
replace, example-based tests. Use it when invariants are easier to state than
examples are to enumerate.

```python
@given(st.lists(st.integers()))
def test_sort_preserves_items(values: list[int]) -> None:
    assert sorted(sorted(values)) == sorted(values)
```
