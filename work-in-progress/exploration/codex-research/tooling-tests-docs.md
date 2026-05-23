# factors2 tooling, tests, docs, and operations research notes

## Scope

Read-only exploration of `/home/msi/python_workspace/factors2` as an example modern Python codebase. Focus areas were project metadata, uv usage, pytest conventions, static analysis configuration, documentation, benchmarks, and operational scripts.

These are research notes for a later writing phase, not a guide draft.

## Representative files reviewed

- `pyproject.toml`
- `uv.lock`
- `.python-version`
- `.pre-commit-config.yaml`
- `.github/workflows/python.yml`
- `.github/workflows/web.yml`
- `Makefile`
- `tests/conftest.py`
- `tests/pyfactors/indicators/technical/test_annualized_volatility.py`
- `tests/pyfactors/indicators/util.py`
- `tests/pyfactors/pipeline/test_portfolio_definition_validation.py`
- `tests/pyfactors/pipeline/util.py`
- `tests/pyfactors/database/test_universe_query.py`
- `tests/pyfactors/database/test_universe_query.test_query_builder.approved.txt`
- `benchmarks/__init__.py`
- `benchmarks/test_benchmark_annualized_volatility.py`
- `doc/setup.md`
- `doc/guidelines.md`
- `doc/adr/0001-schedule-only-economatica-downloads.md`
- `src/pyfactors/indicators/README.md`
- `database/postgres/factors/README.md`
- `src/dbt_factors/README.md`
- `etc/update_dataset.py`
- `etc/indicator_portfolios_sql_from_md.py`
- `etc/dependency_graph.sh`
- `etc/dagster_sample.yaml`
- `.sqlfluff`

## Project layout and dependency management observations

- The project uses a `src/` layout with multiple import roots: `pyfactors`, `etl_factors`, `dagster_factors`, `dashboards`, and `dbt_factors`.
- Runtime dependencies and development dependencies are declared in `pyproject.toml`, with dev tooling grouped under `[dependency-groups].dev`.
- The repository pins the interpreter in `.python-version` as `3.13` and declares `requires-python = ">=3.13"`.
- `uv.lock` is committed. CI installs with `uv sync --all-extras --dev`; Makefile commands consistently use `uv run`.
- `pyproject.toml` declares `readme = "README.md"`, but the root listing did not include a root `README.md`. That is a concrete docs/metadata drift signal.
- `Makefile` provides a thin task facade for setup, sync, lock, test, benchmark, coverage, lint, format, mypy, and pyright.
- `pytest` is configured with `testpaths = ["tests"]` and `pythonpath = ["src"]`. Benchmarks live outside normal test discovery and are run explicitly with `uv run pytest benchmarks/`.

## Existing good practices to port into Python guidelines

- Prefer `src/` layout and configure tests to import from `src` deliberately. This avoids accidental imports from the repository root and makes installed-package behavior easier to reason about.
- Commit the lockfile when applications, pipelines, or operational tools need reproducible environments.
- Provide one obvious command surface for routine development: `uv sync`, `uv run pytest`, `uv run ruff check`, `uv run ruff format`, `uv run mypy`, and `uv run pyright`.
- Keep benchmark tests separate from correctness tests. factors2 uses `benchmarks/` plus pytest-benchmark markers and a Makefile target, so normal CI/test loops do not accidentally run million-row benchmarks.
- Exercise code through domain-shaped data rather than implementation plumbing. Indicator tests construct realistic multi-symbol, multi-date DataFrames and compare complete outputs with `pd.testing.assert_frame_equal`.
- Use focused test helper modules close to test domains. Examples: `tests/pyfactors/indicators/util.py`, `tests/pyfactors/pipeline/util.py`, and `tests/pyfactors/backtest/util.py`.
- Test validation boundaries with explicit exception messages. Portfolio and pipeline tests use `pytest.raises(..., match=...)` for domain contract failures.
- Use approval tests for large generated text artifacts such as SQL. `test_universe_query.py` verifies SQL output while separately asserting query parameters.
- Put project-specific semantic conventions near the code they govern. `src/pyfactors/indicators/README.md` documents DataFrame ordering, fill conventions, expected non-null columns, and copy assumptions for indicator functions.
- Use ADRs for operational decisions that have real tradeoffs. The Economatica scheduling ADR records context, decision, rejected alternative, and consequences around irrecoverable vendor snapshots.
- Keep operational commands executable through `uv run` when possible. CI and Makefile mostly avoid relying on an activated shell.

## Friction points or legacy practices to avoid recommending

- Avoid declaring metadata that points at missing files. `pyproject.toml` references `README.md`, but no root README was present in the repository listing.
- Avoid duplicate top-level YAML keys. `.pre-commit-config.yaml` has two `repos:` keys, which likely causes one block to override the other under normal YAML parsing.
- Avoid comments that say "latest version" beside a pinned pre-commit revision. The revision is static; the comment drifts immediately.
- Avoid leaving generated or starter docs in durable project docs. `src/dbt_factors/README.md` still reads like the dbt starter template rather than project-specific documentation.
- Avoid docs that are only a manual transcript of operational steps when those steps should become scripts. `database/postgres/factors/README.md` explicitly says the database initialization steps should move into an `initialize_db.sh`-style workflow.
- Avoid dangerous or hidden behavior in operational scripts. `etc/indicator_portfolios_sql_from_md.py` parses Markdown with regex and evaluates extracted Python expressions with `eval`; that may be acceptable for trusted local maintenance, but guidelines should not present it as a default pattern.
- Avoid scripts that silently return success on invalid user input. `etc/update_dataset.py` prints date parse errors and returns from `main()` without a nonzero exit code.
- Avoid hard-coded absolute local paths in sample operational config. `etc/dagster_sample.yaml` includes `/home/msi/.dagster/...`, which is useful locally but should be templated or documented as an example-only value.
- Avoid making CI, Makefile, pre-commit, and tool config disagree. factors2 has `ruff include = ["src/**/*.py", "tests/**/*.py", "examples/**/*.py"]`; benchmarks are run separately but not included in Ruff's configured include list, while `make lint` invokes `ruff check .`.
- Avoid excessive test comments that restate the line of code. Several helpers include comments like "Add signal column" immediately before assigning the signal column. The code is clearer without the comment.

## Candidate guide topics and content hooks

### Project layout

- Recommend `src/` layout for projects intended to run in CI or ship as packages.
- Explain when multiple top-level packages under `src/` are reasonable: separable domains such as core library, ETL, orchestration, and dashboards.
- Require root metadata to be checked against the files it names, especially `readme`, license, package names, and Python version files.

### uv and command surfaces

- Standardize on `uv sync` for environment creation and `uv run` for repeatable commands.
- Use a Makefile or equivalent task runner as a thin facade, not a second source of dependency or environment truth.
- Include explicit targets for test, benchmark, coverage, lint, format, mypy, and pyright when those tools are part of the project contract.
- In CI, install from the lockfile and use the same commands developers run locally.

### Pytest conventions

- Keep correctness tests under `tests/` and long-running benchmarks under `benchmarks/`.
- Use domain-specific factory helpers for complex tabular test data.
- Prefer full-output assertions for DataFrame transformations when the output is tractable.
- Use approval tests for generated SQL or other large structured text, and pair them with direct assertions for important parameters.
- Make test names describe behavior and contract boundaries, especially validation failures.
- Keep autouse fixtures rare and session-scoped only when they establish a global runtime mode. The pandas copy-on-write fixture is a useful example.

### Static analysis and formatting

- Put Ruff, pytest, mypy, pytype, and project-specific tool configuration in `pyproject.toml` unless a tool requires its own file.
- Treat Ruff as both linter and formatter when possible; avoid carrying Black config unless Black is still run.
- If using both mypy and pyright, state why both are used and define which one is authoritative for which classes of findings.
- Make per-file ignores narrow and explain them in present-tense terms. Test fixture unused-argument ignores are reasonable; broad ignores should be periodically reviewed.
- Keep tool scopes aligned across local commands and CI. If benchmarks should be linted, include them explicitly; if not, document the boundary.

### Documentation

- Use ADRs for decisions with meaningful operational consequences.
- Put local setup docs in one durable place, but keep them runnable and current by preferring commands that call scripts over prose-only sequences.
- Put domain conventions near the code that depends on them, especially DataFrame schemas, ordering assumptions, time/date conventions, and unit conventions.
- Avoid starter-template READMEs in durable docs.
- Avoid root metadata drift by having CI or a lightweight check assert that declared files exist.

### Operational scripts

- Put maintenance scripts in a clear location such as `etc/` only when they are not product/runtime modules.
- Give scripts `argparse`, date parsing, `Path.expanduser()`, and explicit inputs.
- Return nonzero exit codes on user input errors.
- Prefer structured parsers over regex plus `eval` for generated SQL or config migration scripts.
- Keep local-only sample config marked as sample data and avoid hard-coded personal paths in reusable templates.

## Notes on avoiding documentation drift

- Documentation should describe durable contracts rather than repeat obvious implementation details. For example, document DataFrame ordering, fill guarantees, units, and operational deadlines; do not narrate every assignment in helper code.
- Docs that name commands should call stable project entry points: `make test`, `uv run pytest`, `uv run mypy src`, or a real script. This makes drift visible when commands break.
- CI should run the same commands that docs recommend. If docs say `uv run pytest`, CI should not use a materially different invocation without explaining why.
- Keep generated examples and starter docs out of durable docs unless they are edited into project-specific runbooks.
- Prefer ADRs when documenting "why this workflow exists." The Economatica ADR is useful because it captures the non-obvious business constraint: missed vendor snapshots cannot be recovered.
- Periodically check packaging metadata and docs together: root README existence, Python version consistency between `.python-version` and `pyproject.toml`, lockfile freshness, and tool version consistency between pre-commit, CI, and dependency groups.
