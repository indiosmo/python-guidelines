# Analysis: cpp-design-principles/preprocessor-macros.md

Disposition: **PYTHON_SPECIFIC_VARIANT**. Python has no preprocessor.
The chapter's purpose was "when and how to reach for a code-generation
mechanism the type system cannot replace"; the Python equivalent is
**decorators**, with **metaclasses** and **`__init_subclass__`** as the
heavier escape hatches.

## Section-by-section

### Macros are a last resort

- **Classification:** PORTS_AS_IS (the principle).
- **C++ principle:** Reach for the preprocessor only when the alternative
  is repeated boilerplate the type system cannot remove.
- **Python rendering:** Same rule, restated for Python's equivalents:

  - **Decorators** are the default. A decorator that adds logging,
    caching, validation, or retry logic to a function is the right
    tool. Examples: `@functools.cache`, `@functools.cached_property`,
    `@contextlib.contextmanager`, `@dataclass`, `@pydantic.validate_call`.
  - **`__init_subclass__`** is the right tool for "every subclass of
    `Base` must do X". It runs once per subclass declaration; it does
    not require a metaclass.
  - **Metaclasses** are the last resort. They compose poorly and
    confuse readers; only reach for one when neither a decorator nor
    `__init_subclass__` can do the job (typically: when you need to
    intercept attribute access at class-definition time).

### Narrow sub-headers vs umbrellas

- **Classification:** DROP.
- **Reason:** No C preprocessor; there is no umbrella-include cost in
  Python. Python imports are already module-granular.

### Indirect three-token paste for argument expansion

- **Classification:** DROP.
- **Reason:** No equivalent. Closest concern is name collisions in
  `globals()` for code-gen tools (Jinja-templated Python files), which
  is its own topic.

### Variadic dispatch via `__VA_OPT__`

- **Classification:** DROP.
- **Replace with:** Python's variadic dispatch primitive is
  `functools.singledispatch` / `functools.singledispatchmethod`:

  ```python
  @functools.singledispatch
  def check(value, /):
      raise TypeError(f"no check for {type(value).__name__}")

  @check.register
  def _(value: int) -> None: ...
  @check.register
  def _(value: str) -> None: ...
  ```

  And `*args` / `**kwargs` for parameter-pack-flavoured arguments.

### Domain macros are not preprocessor utilities

- **Classification:** PORTS_AS_IS (the principle).
- **C++ principle:** A macro that abstracts a project concept is a
  domain abstraction, not a preprocessor utility; it lives with the
  domain type it shapes.
- **Python rendering:** Same. A decorator that registers an error
  category or wires a dependency lives in the domain's module, not in
  a project-wide `utilities.py`. The shape rule is
  "decorators-near-the-domain"; the structural rule is the same.

## New section to add: decorator design principles

The Python guide should add a short companion section on **when to write
a decorator vs a plain function vs a class**:

- Prefer a plain function when the caller's site is fine with calling
  it directly.
- Reach for a decorator when the same wrapping logic applies at many
  call sites and the wrapping logic is structural (logging, caching,
  validation, retry, transaction).
- Use `functools.wraps` on every custom decorator so the wrapped
  function keeps its `__name__`, `__doc__`, and `__wrapped__`.
- Type a decorator with `ParamSpec` (PEP 612) / PEP 695 generics so the
  wrapped function's signature survives to callers.
- Class decorators (the equivalent of the C++ `LIB_DEFINE_ERROR_CATEGORY`
  macro) are valid; they sit alongside `@dataclass` and `@attrs.define`.

## Suggested Python file: `python-design-principles/decorators-and-metaprogramming.md`

A wholly rewritten chapter. The C++ "preprocessor as last resort"
discipline ports to "metaclasses as last resort"; the
"decorator-design rules" subsection is new and is what most Python
codebases actually need.
