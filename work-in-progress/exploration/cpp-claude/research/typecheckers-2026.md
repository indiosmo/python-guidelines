# Python type checkers in 2026: pyright, pyrefly, ty, mypy

## TL;DR

Recommend **pyright** as the default for these guidelines, with **pyrefly** called
out as a faster alternative once it stabilises typing-spec conformance. Mention
mypy only as the legacy fallback. Skip `ty` (Astral's checker) for now -- it has
the worst typing-spec conformance of the four.

## Comparison

| Tool     | Owner        | Speed (relative)    | Typing-spec conformance | PEP 695 | Notes                                                                                |
|----------|--------------|---------------------|-------------------------|---------|--------------------------------------------------------------------------------------|
| pyright  | Microsoft    | baseline (slow)     | leader (early PEP supp.) | yes    | Strongest IDE story (Pylance). Strict mode catches the most.                          |
| pyrefly  | Meta         | 10-50x pyright      | 87.8% of conformance suite | yes  | Rust-implemented. Default settings already aggressive. Good for large monorepos.      |
| ty       | Astral       | 10-50x pyright      | 53.2% (worst)            | yes    | Newest, conformance still catching up; bundle ergonomics with ruff/uv attractive.     |
| mypy     | Python core  | 1.5-3x pyright now  | 57%                      | partial | Safe default if already in CI; behind on PEP 695, conformance, speed.                 |

## Recommendation for the Python guidelines

- **Default:** `pyright --strict` (or equivalent `[tool.pyright]` config). Enables
  `reportMissingTypeStubs`, `reportUnknownVariableType`, `reportUnknownArgumentType`,
  `reportImplicitOverride`, etc. The strict mode is what catches the kinds of bugs
  the C++ "compile-time correctness" rules catch in C++.
- **Mention pyrefly** as the swap-in when the codebase grows past comfortable
  pyright cold-start times (numpy-scale: 70s pyright vs 4.8s pyrefly).
- **Configure ruff to be strict** about implicit `Any` (via `ANN` rules), and rely
  on the type checker for the rest.

## Sources

- https://pydevtools.com/handbook/explanation/how-do-mypy-pyright-and-ty-compare/
- https://pyrefly.org/blog/speed-and-memory-comparison/
- https://pyrefly.org/blog/typing-conformance-comparison/
- https://www.pkgpulse.com/blog/pyrefly-vs-ty-python-type-checkers-2026
- https://dasroot.net/posts/2026/05/pyrefly-v1-0-fast-python-type-checker-challenging-mypy/
