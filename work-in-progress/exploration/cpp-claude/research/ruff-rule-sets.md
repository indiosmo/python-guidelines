# Recommended ruff rule sets for these guidelines

## Recommended `[tool.ruff.lint]` for the Python guidelines

```toml
[tool.ruff.lint]
select = [
    "E", "W",         # pycodestyle
    "F",              # pyflakes
    "I",              # isort
    "N",              # pep8-naming
    "UP",             # pyupgrade -- forces modern syntax (PEP 604, 695, etc.)
    "B",              # flake8-bugbear -- mutable default args, broad except, etc.
    "C4",             # comprehensions
    "SIM",            # simplifications
    "RET",            # return statements
    "RSE",            # raise statements (no parens)
    "PIE",            # misc. correctness
    "PT",             # pytest
    "FURB",           # refurb -- modern Python idioms
    "PERF",           # performance hints
    "RUF",            # ruff-specific (e.g. mutable class default detection)
    "ANN",            # require type annotations on public surface
    "ASYNC",          # asyncio antipatterns (blocking calls in coroutines)
    "TID",            # tidy imports
    "TCH",            # type-checking blocks
    "FBT",            # boolean trap (positional bool args)
    "D",              # pydocstyle (optional; selective per-file ignore)
    "S",              # bandit (security)
    "PL",             # pylint subset
    "TRY",            # tryceratops -- exception antipatterns
    "EM",             # exception messages (no inline literals)
    "PTH",            # use pathlib over os.path
    "T20",            # forbid print() in production code
    "ARG",            # unused arguments
    "DTZ",            # require timezone-aware datetimes
]

ignore = [
    "ANN401",         # allow `Any` selectively (override per-file when justified)
    "D203", "D213",   # pydocstyle conflicts
    "PLR0913",        # too many arguments -- factory funcs trip this legitimately
    "S101",           # asserts allowed in tests
    "TRY003",         # long messages in raise -- some are unavoidable
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "ANN", "D", "PLR2004", "ARG"]
"**/conftest.py" = ["ANN", "D"]
```

## Rule sets that map directly to C++ guideline themes

| C++ guideline                                    | Ruff coverage                                                                            |
|--------------------------------------------------|------------------------------------------------------------------------------------------|
| "Parse, don't validate" / strong typing surface  | `ANN` (require annotations), `UP` (modern syntax for `X \| Y`).                          |
| "Functional core, imperative shell"              | `ASYNC` (no `time.sleep` inside `async def`), `T20` (no `print` deep in libs).            |
| "Push effects to the edge"                        | `S110`/`S112` (silent except), `TRY` (exception antipatterns).                            |
| "Errors do not cross unrelated boundaries"        | `TRY002` (no `raise Exception(...)`), `TRY003`.                                           |
| Designated initializers                           | `B006` (mutable default), `FBT` (no positional bool).                                    |
| Const placement / readability                     | `SIM`, `RET`, `RUF`.                                                                      |
| Iterator invalidation / `for` clarity             | `PERF`, `C4`.                                                                             |
| Comments style                                    | `D` (pydocstyle) at the strictness level the team can tolerate; otherwise just `D401`.   |

## Notes

- `ASYNC` catches the common "I blocked the event loop with `time.sleep`" or
  "I used `requests.get` instead of `httpx`" mistakes -- they are the Python
  equivalent of dropping a `std::mutex` into the inner functional layer.
- `FURB` is the closest ruff has to "compile-time correctness" linting --
  it flags use of legacy APIs that have a better modern form.
- `T20` (no `print`) lines up directly with the C++ "logging is a
  cross-cutting service, not `std::cout`" rule.

## Sources

- https://docs.astral.sh/ruff/rules/
- https://blog.marcosalonso.dev/the-complete-python-code-quality-stack-in-2026-ruff-mypy
