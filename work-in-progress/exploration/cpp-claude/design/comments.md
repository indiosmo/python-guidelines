# Analysis: cpp-design-principles/comments.md

Disposition: **PORTS_AS_IS**. The entire chapter is language-agnostic.
Two small adaptations: Python uses `"""docstring"""` for class/function
documentation (where C++ uses `/* ... */`) and `#` for guiding comments
inside function bodies (where C++ uses `//`).

## Section-by-section

### Guiding comments

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. Use `# ` (with the trailing space)
  for line and block comments, `"""..."""` for module / class /
  function docstrings.

### Non-obvious lines

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. The examples translate directly --
  a dict pop with a tuple key, a chained method call, a generator
  expression that materialises lazily, a hidden `iter(...)` semantics
  difference, a known PEP-695 type-alias gotcha.

### Blocks and flow

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. Consecutive block comments read as
  the function's table of contents.

### What not to comment

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule, with one Python-specific addition:
  do not comment on the **GIL** or **free-threaded behaviour** at
  every call site -- document the threading contract once on the
  module or class, as `runtime.md` covers.

  The "negative documentation" anti-pattern (from `~/.claude/CLAUDE.md`)
  ports directly. The "past-tense framing", "delta framing", and
  "non-responsibility framing" tells are language-agnostic; the
  examples just need `//` -> `#` substitutions.

### Precondition phrasing

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same. The Python idiom for a load-bearing
  precondition is a one-line `# precondition: ...` comment, OR a
  defensive `assert` (when the precondition is also a useful runtime
  check, not just documentation). Recommend the comment form for
  conventions a reader needs to understand, and the assert form when
  the program should crash loudly if the precondition fails.

### Shape

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Convention:
  - Module docstring at the top of every file (PEP 257).
  - Class docstring on every public class.
  - Function/method docstring for any public function (private helpers
    can rely on naming and types; ruff's `D` rules let the team pick
    the strictness).
  - `# ` for guiding comments inside function bodies.
  - Avoid C++-style block comments with `#` lines that span many rows;
    if a comment is more than two lines and explains structural intent,
    promote it to a docstring on the enclosing function.

## Suggested Python file: `python-design-principles/comments.md`

Verbatim port with the `//` -> `#` and `/* */` -> `"""..."""` substitutions,
and a short paragraph on docstring style (Google / NumPy / Sphinx --
recommend Google for new code, NumPy for codebases already using
sphinx-style autodoc; configure ruff's `D` rules to match the choice).
