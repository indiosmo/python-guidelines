# Tooling Baseline

A Python project should have one command surface that developers, CI, and
automation all use. Tooling works best when the lockfile, formatter, linter,
type checker, test runner, and pre-commit hooks describe the same project
scope.

Use `pyproject.toml` as the default metadata and tool configuration surface.
Use `uv` for environments, dependency resolution, lockfile updates, and
command execution. Add a thin task facade only when it shortens common commands
without becoming a second source of truth.

## Dependency Workflow

Use uv workspaces for monorepos and uv projects for single-package
repositories. Commit `uv.lock` for applications, data pipelines, operational
tools, and other repositories where reproducible environments matter.

Keep runtime dependencies in package metadata and development-only tools in
dependency groups such as `test`, `lint`, `typing`, `docs`, `benchmark`, and
`security`. Development dependencies should earn their place through a named
command, CI job, pre-commit hook, or documented workflow.

Common commands should use the same shape everywhere:

```sh
uv sync
uv lock
uv run pytest
uv run ruff check .
uv run ruff format .
uv run pyright
```

A task facade can make the project easier to use when each target is a thin
wrapper:

```makefile
sync:
	uv sync

test:
	uv run pytest

lint:
	uv run ruff check .

format:
	uv run ruff format .

typecheck:
	uv run pyright

ci: lint typecheck test
```

## Ruff

Use Ruff for linting, import sorting, formatting, modernization, and fast
bug-pattern checks. Choose one formatter per repository. When Ruff formats the
project, run Ruff format in CI and pre-commit, and remove parallel Black policy
from the active command surface.

Treat Ruff as lint and format, not as the semantic type checker. Ruff can catch
annotation hygiene, import discipline, async mistakes, pytest style, security
patterns, and pandas issues, but the type checker owns type relationships.

A useful first-pass rule set includes:

- `E`, `W`, and `F` for pycodestyle and pyflakes.
- `I` for import sorting.
- `UP` for modern Python syntax.
- `B`, `C4`, `SIM`, `RET`, `PIE`, `RUF`, and `PERF` for common defect and
  maintainability checks.
- `PT` for pytest conventions.
- `ASYNC` for event-loop hazards.
- `TID` and `TCH` for import and type-checking hygiene.
- `S` for security checks, tuned to project context.
- `T20` to keep `print` out of production code.
- `ARG` for unused arguments, with test-specific relaxations.
- `NPY` and `PD` for NumPy and pandas projects.

Use per-file ignores sparingly and keep them tied to the file shape. Tests
often need different rules for fixtures, magic numbers, and invalid-input
cases.

## Type Checking

Use Pyright as the default type checker. Start with a configuration that gives
useful feedback without forcing the whole repository into strict mode at once.
Move stable packages, public APIs, domain models, adapters, and reusable
helpers toward stricter checking by package or directory.

Run mypy alongside Pyright only when it earns the cost: existing CI depends on
it, a library plugin is valuable, or the project has already standardized on
mypy diagnostics. Treat ty and pyrefly as fast alternatives after evaluating
their diagnostics, typing-spec behavior, editor support, and suppression model
against the project.

Keep type checker configuration portable. Developer-specific virtual
environment paths belong in local settings or command-line options.

## Pytest

Use pytest as the correctness test runner. Configure test discovery explicitly:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
```

Use marks, node IDs, file paths, and `-k` expressions for focused loops. Keep
benchmarks outside normal test discovery and run them with an explicit command
such as `uv run pytest benchmarks/`.

When approval tests are part of the project, configure a deterministic reporter
and pair broad approval checks with direct assertions for narrow facts such as
query parameters, record counts, and boundary conditions.

## Pre-Commit

Use pre-commit for quick checks that protect the working tree before CI. Good
hooks include uv lockfile checks, Ruff lint, Ruff format, merge-conflict
checks, private-key detection, SQL lint for dbt projects, and project-specific
formatters for non-Python subtrees.

Install hooks for branch changes when the team wants the environment to track
the current checkout:

```yaml
default_install_hook_types:
  - pre-commit
  - post-checkout
  - post-merge
  - post-rewrite
```

Keep expensive type checks and broad test suites in CI or an explicit local
target unless the project has measured that pre-commit remains fast enough.

## CI

CI should run the same command families developers run locally:

```sh
uv sync --locked --all-extras --dev
uv run ruff format --check .
uv run ruff check .
uv run pyright
uv run pytest
```

Add project-specific jobs where they carry real feedback: SQL lint for dbt,
security scanning for dependency and secret exposure, coverage gates for
mature packages, package metadata checks, import-boundary checks, and scheduled
benchmarks.

Cache dependency downloads by the lockfile. Pin tool versions where the tool
is installed outside the lockfile, and keep pre-commit, local commands, and CI
on the same versions.

## Review Scaffolding

Use CODEOWNERS, pull request templates, and issue templates when they improve
review routing and capture recurring context. Keep templates short. They
should ask for behavior, risk, tests, rollout notes, and screenshots or
artifacts only when those apply to the project.

