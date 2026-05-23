# Tooling baseline

Extracted from `pyproject.toml`, `Makefile`, `.pre-commit-config.yaml`,
`.sqlfluff`, `.lintr`, and `.github/workflows/`. Most of this is
*already* what a modern Python project should look like; the new
guideline can lift it almost verbatim.

## Python version, dependency manager

- `requires-python = ">=3.13"` (`pyproject.toml:6`)
- `uv` is the dependency manager; `Makefile:22-25` runs `uv sync`
  + `uv pip install -e .`
- `dependency-groups.dev` keeps dev-only deps out of the runtime
  install (`pyproject.toml:58-85`)
- Pre-commit hook `uv-lock` + `uv-sync --locked --all-packages` runs
  on every commit (`.pre-commit-config.yaml:8-13`). This is the
  enforcement: `uv.lock` is real, drift is rejected at commit time.
- No `.python-version` file is committed in the repo I read, even
  though CI references one (`setup-python@v5` with
  `python-version-file: ".python-version"`). Worth fixing.

**Lift verbatim**: `requires-python = ">=3.13"`, uv as the manager,
`uv-lock` + `uv-sync --locked` in pre-commit, `dev` group via
`dependency-groups`.

## Ruff

```toml
# pyproject.toml:98-116
[tool.ruff]
line-length = 120
include = ["src/**/*.py", "tests/**/*.py", "examples/**/*.py"]
respect-gitignore = true

[tool.ruff.lint]
# E501 ignore line length, formatter will fix it.
# PLR0911 too many return statements (style)
# PLR0912 too many branches (style)
# PLR0913 too many arguments (style)
# PLR0915 too many statements (style)
# PLC0414 import alias does not rename (conflicts with unused import)
ignore = ["E501", "PLR0911", "PLR0912", "PLR0913", "PLR0915", "PLC0414"]
select = ["E", "F", "B", "I", "N", "Q", "UP", "C4", "ARG", "NPY", "PD", "PERF", "PL"]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["ARG001", "ARG002", "ARG005", "PLR2004"]
```

Notes:
- **Broad rule selection** (`E`, `F`, `B`, `I`, `N`, `Q`, `UP`, `C4`,
  `ARG`, `NPY`, `PD`, `PERF`, `PL`) - more than the ruff default.
- **Style-of-coding `PL` rules ignored** (too-many-returns/branches/args/statements).
  This is a deliberate softening; the new guideline can either keep
  or relax further depending on review preference.
- **Tests get `ARG*` and `PLR2004` magic-number relaxed** - fixtures
  and magic-number assertions are normal in tests.
- `respect-gitignore = true` keeps generated files out of the lint
  surface.
- `[tool.black] line-length = 120` (`pyproject.toml:118-119`) - vestigial.
  ruff-format is used in pre-commit; the black config is unused. The
  new repo should drop it.

**Add** (suggested for the new guideline):
- `T201` (no print) - the codebase has 35+ `print()` calls that
  would catch.
- `S` (security) family for new code, especially `S608` (SQL
  injection via f-string).
- `RUF` family - newer, mostly Python-3.12+ idioms.
- `SIM` (simplify) - catches `if x: return True else: return False`
  shapes.

## Type checkers

- mypy **and** pyright are both enforced in CI
  (`.github/workflows/*.yml:48-52`).
- mypy uses the `pydantic.mypy` plugin (`pyproject.toml:128-133`).
- mypy ignores missing imports for `ibis.*` and `plotly.*` (those
  libraries' stubs are weak; the project does not vendor stubs).
- pyright runs with default config (no `pyrightconfig.json`); it
  reads `pyproject.toml`'s `[tool.pyright]` if present (none here).

**Lift**: both checkers in CI. Run mypy first (better error messages
on Pydantic), pyright second (catches things mypy misses around
discriminated unions). The new guideline can specify a `[tool.pyright]`
block with `reportMissingTypeStubs = false` for the ibis/plotly
class of weak-stubs issue.

## pytest

```toml
# pyproject.toml:87-92
[tool.pytest.ini_options]
addopts = "--approvaltests-use-reporter=PythonNative"
testpaths = ["tests"]
pythonpath = ["src"]
```

- approvaltests is the default reporter (no GUI diff tool prompts)
- `pythonpath = ["src"]` means tests import as `from pyfactors.x import y`
  rather than `from src.pyfactors.x import y`
- Tests live in `tests/`, benchmarks in `benchmarks/`, both
  pytest-discoverable

**Lift verbatim**, including the approvaltests reporter setting.

## pre-commit

```yaml
# .pre-commit-config.yaml
default_install_hook_types:
  - pre-commit
  - post-checkout
  - post-merge
  - post-rewrite

repos:
  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.6.14
    hooks:
      - id: uv-lock
      - id: uv-sync
        args: ["--locked", "--all-packages"]

repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.6
    hooks:
      - id: ruff
      - id: ruff-format

  - repo: local
    hooks:
      - id: pnpm-format
        name: Format frontend code with prettier
        entry: bash -c 'cd src/pyfactors_web && pnpm run format'
        language: system
        files: ^src/pyfactors_web/
        pass_filenames: false
```

Notes:
- **`default_install_hook_types` includes `post-checkout`,
  `post-merge`, `post-rewrite`.** This means `uv sync` runs not just
  on commit but every time you switch branches, merge, or rewrite
  history. Result: your venv never drifts from your branch. **Lift
  verbatim - this is the cleanest pre-commit setup I've seen in a
  Python repo.**
- **Two top-level `repos:` keys** - YAML allows it but most parsers
  merge them. Cosmetic; can collapse.
- **The `pnpm-format` hook references `src/pyfactors_web/`, which
  does not exist in the repo I read.** Either restore or remove.
- **No mypy/pyright hook in pre-commit.** Type checks are CI-only.
  Defensible (they're slow); worth a guideline note.

**Add** (suggested):
- `check-merge-conflict` from `pre-commit-hooks` to catch unresolved
  conflicts.
- `detect-private-key` from `pre-commit-hooks` for the secrets
  case.
- An sqlfluff hook for the dbt SQL.

## sqlfluff

```ini
# .sqlfluff
[sqlfluff]
large_file_skip_byte_limit = 0
max_line_length = 0
dialect = postgres
templater = dbt

[sqlfluff:templater:dbt]
project_dir = src/dbt_factors
```

- `dialect = postgres` - matches the warehouse
- `templater = dbt` - resolves Jinja before linting
- `max_line_length = 0` - disabled, presumably because dbt's
  generated SQL can be wide
- No rule selection - uses sqlfluff defaults

**Lift** the dialect/templater shape. The line-length disable is a
defensible default; the alternative is to set it generously
(`max_line_length = 160`).

## Makefile

```makefile
# 14 targets, all thin wrappers over uv run
setup:        uv sync + uv pip install -e .
sync:         uv sync
lock:         uv lock
test:         uv run pytest
benchmark:    uv run pytest benchmarks/
test-coverage: uv run pytest --cov=src --cov-report=term --cov-report=html tests/
lint:         uv run ruff check .
lint-fix:     uv run ruff check . --fix
format:       uv run ruff format .
check-fix:    lint-fix + format
mypy:         uv run mypy src
pyright:      uv run pyright src
```

Notes:
- All targets are one-liners. No Bash logic that could rot. **PROMOTE.**
- `make help` is the entry point. **PROMOTE.**
- No `make ci` target that runs everything. The CI workflow runs each
  command separately. Worth adding: `ci: lint format mypy pyright test`
  so contributors can run the same set locally.
- No `make profile` target despite `pyinstrument` being a dep.
- No `make deps-graph` target despite `pydeps` being a dep.

**Lift verbatim** with the additions above.

## CI (`.github/workflows/python-ci.yml`)

```yaml
on:
  pull_request:
    branches: [main]
    paths-ignore: ['src/pyfactors_web/**']

steps:
  - uses: actions/checkout@v4
  - uses: astral-sh/setup-uv@v5
    with:
      enable-cache: true
      cache-dependency-glob: "uv.lock"
  - uses: actions/setup-python@v5
    with:
      python-version-file: ".python-version"
  - run: uv sync --all-extras --dev
  - run: uv run pytest tests
  - uses: astral-sh/ruff-action@v3
    with:
      version: "0.14.6"
      args: format --check --diff
  - uses: astral-sh/ruff-action@v3
    with:
      version: "0.14.6"
      args: check
  - run: uv run mypy src
  - run: uv run pyright src
```

Notes:
- **`paths-ignore: ['src/pyfactors_web/**']`** so frontend-only PRs
  skip the Python CI. Matched by a separate `Web CI` workflow that
  runs only on `src/pyfactors_web/**`. Sensible split. **PROMOTE.**
- **`enable-cache: true`** + `cache-dependency-glob: "uv.lock"` - uv
  caches keyed on the lockfile. Free speedup. **PROMOTE.**
- **Pinned ruff version** (`0.14.6`) matches the pre-commit pin.
  Same exact version locally and in CI; no "works on my machine".
  **PROMOTE.**
- **`uv sync --all-extras --dev`** in CI installs everything; locally
  the Makefile only does `uv sync`. Mild divergence, but `--all-extras`
  is the safer CI default.
- **Tests run before lint.** Mild smell - lint failures are usually
  quicker to surface. Easy to flip.

## What's missing that the new guideline should add

1. **No CODEOWNERS, no PR template, no issue templates.** Lightweight
   touches that scale code review.
2. **No `make ci` aggregator target.**
3. **No `pydeps` invocation; no `pyinstrument` invocation.** Two dev
   deps without entry points.
4. **No security scan in CI** (`pip-audit`, `safety`, `gitleaks`).
5. **No coverage gate** despite `pytest-cov` being installed - the
   `make test-coverage` target produces a report but nothing fails on
   regression.
6. **No SQL lint in pre-commit/CI** despite sqlfluff being installed.
7. **No nightly job** that runs `make benchmark` and posts to a tracker.

## Recommendations for the new guideline

1. **`requires-python = ">=3.13"`; uv as the manager; lockfile
   enforced via pre-commit (`uv-lock` + `uv-sync --locked`).**
2. **Pre-commit hooks installed for `pre-commit`, `post-checkout`,
   `post-merge`, `post-rewrite`** so the venv tracks the branch.
3. **Ruff with broad rule selection** (`E F B I N Q UP C4 ARG NPY PD
   PERF PL`) plus `T201`, `S`, `RUF`, `SIM`. Style-of-coding `PL`
   rules ignored.
4. **mypy + pyright both enforced in CI**, mypy with the
   `pydantic.mypy` plugin.
5. **pytest + pythonpath=src + approvaltests-PythonNative reporter.**
6. **sqlfluff for dbt SQL** with `dialect = postgres, templater = dbt`,
   wired into pre-commit and CI.
7. **Thin Makefile** wrapping uv: `setup sync lock test benchmark
   test-coverage lint lint-fix format check-fix mypy pyright ci`.
8. **Pinned tool versions** across pre-commit, Makefile, and CI:
   the version is the contract.
9. **Coverage gate in CI** with a documented threshold; reduce only
   via PR.
10. **Add what's missing**: CODEOWNERS, PR template, security scan,
    SQL lint, nightly benchmark.
