# Analysis: cpp-debugging-principles/sanitizers.md

Disposition: **PYTHON_SPECIFIC_VARIANT**. Python has no equivalent of
ASan/MSan/UBSan/TSan/Fil-C; the runtime classes of failures these
sanitizers catch (use-after-free, undefined behaviour, data races) do
not exist at the Python level. The C++ chapter's purpose -- "ship build
presets that catch a class of defect deterministically, with version-
controlled suppressions" -- maps to **Python's static-checker stack +
dev-mode toggles + profilers**.

## Section-by-section

### Per-tool presets and suppression files

- **Classification:** DROP the per-sanitizer mechanics; **REPLACE** with
  the Python equivalent stack.

- **Python equivalent:**

  | Failure class C++ sanitizer catches  | Python equivalent                                                                                  |
  |--------------------------------------|----------------------------------------------------------------------------------------------------|
  | use-after-free / dangling reference  | Not applicable. GC manages object lifetime.                                                        |
  | uninitialised reads                  | Not applicable.                                                                                    |
  | undefined integer overflow           | Not applicable (Python ints are arbitrary precision).                                              |
  | data races                           | Free-threaded 3.13t/3.14t: no analogue tool yet; under-GIL: not possible at Python level (but a C extension can race; address by writing the extension carefully). |
  | memory leaks                          | `tracemalloc`, `memray`, `objgraph` -- run in dev/CI as a debug step.                              |
  | misuse of typing system              | pyright/pyrefly --strict; ruff `ANN`/`PLE`/`PYI` rules.                                              |

### Recommended Python "sanitizer-equivalent" stack

A single chapter that recommends the following presets / CI jobs:

1. **`pyright --strict` (or `pyrefly --strict`)** as a CI gate. This is
   the largest single source of "bug class made impossible" leverage
   in Python.
2. **`ruff check` with the rule set from
   `research/ruff-rule-sets.md`** as a CI gate.
3. **`python -X dev pytest`** for the test run. `dev` mode upgrades
   warnings (ResourceWarning, DeprecationWarning) to actionable
   failures.
4. **`pytest -W error`** to fail tests on any warning.
5. **`memray run` on the test suite** in a nightly CI job, with a
   regression baseline. Catches new allocations / leaks.
6. **`pytest --co` with `pytest-randomly`** in a nightly CI job to
   detect order-dependent flakes.
7. **`py-spy dump` ready to run on production** as an operational
   primitive, not a CI gate.

### Suppression policy

- **C++:** per-sanitizer suppression file, comment-documented,
  triaged before silencing.
- **Python equivalent:** type-checker pragmas (`# pyright: ignore[...]`),
  ruff per-line ignores (`# noqa: RULE_CODE`), and warning filters
  (`warnings.filterwarnings("ignore", category=..., module="...")`).

  Same discipline: never add a silencer without a comment naming
  what was triaged and why. ruff and pyright support per-file ignores
  in `pyproject.toml`; suppressed code lines need an explanation in
  the source as a trailing comment.

### "ThreadSanitizer is special-cased"

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **Python equivalent:** Under the GIL, Python-level data races are
  not possible. Under free-threaded Python (3.13t/3.14t),
  application-level races can occur and there is **no equivalent of
  TSan** today. The mitigations are:
  - Default to the "owning thread" discipline from `design/runtime.md`.
  - Use `threading.Lock` explicitly around any state genuinely shared
    across threads.
  - Run the test suite under `python3.13t -X gil=0` periodically to
    surface "I assumed GIL" mistakes.
  - C-extension authors writing for free-threaded Python should use
    the upstream ASan/TSan for the extension build, not for Python.

## Suggested Python file: `python-debugging-principles/static-analysis-and-runtime-checks.md`

Rename the chapter; the Python equivalent is "static analysis +
runtime checks + profilers", not "sanitizers". Cover:

- The pyright/pyrefly + ruff CI stack.
- `python -X dev` / `-W error` for test runs.
- `tracemalloc` / `memray` / `py-spy` cookbook.
- The free-threaded Python caveat (no TSan analogue; rely on the
  ownership discipline).
- The suppression policy (comment-documented, triaged).
