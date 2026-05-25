# Project Layout

Python project layout should make the runtime boundary visible. A reviewer
should be able to tell which code is a reusable library, which code is an
application, which code is orchestration, and which files are generated or
operational inputs. The layout should also make imports honest: tests and tools
exercise the package through the same import names production code uses.

Use `src/` layout for packages that run in CI, ship as installable artifacts,
or need confidence that tests import the installed package path. Keep
infrastructure, databases, generated outputs, and local maintenance files
outside the source package tree.

## Repository Shape

A Python monorepo benefits from a small number of top-level areas with clear
ownership:

- The repository root owns workspace metadata, lockfiles, global tool
  configuration, CI configuration, and developer command entry points.
- `src/` owns Python packages, applications, orchestration code, and data
  pipeline code.
- `docker/`, `db/`, and similar infrastructure directories own process
  wiring, image builds, migrations, and service-level configuration.
- `tests/` mirrors the package paths it exercises.
- `benchmarks/` owns performance tests that run outside the normal correctness
  loop.
- `etc/` or `scripts/` owns maintenance tools that are not imported by product
  code.

Put source packages under `src/`, and name each top-level package for the layer
or domain it owns. Names such as `common`, `shared`, `helpers`, and `utils`
usually mean the boundary has not been named yet. Prefer names such as
`market_data`, `risk_model`, `etl_sources`, `orchestration`, or `dashboards`
when those are the concepts the package owns.

## Workspace Members

Use a root `pyproject.toml` to define the workspace and shared tool policy.
Each deployable package or reusable library gets its own child `pyproject.toml`
with the runtime dependencies it owns.

This keeps dependency pressure local. A dashboard can depend on Streamlit, a
Dagster code location can depend on Dagster, and a domain library can stay
small enough to test and reuse without installing the whole platform.

```text
repo/
  pyproject.toml
  uv.lock
  src/
    libs/
      market_data/
        pyproject.toml
        src/market_data/
    apps/
      portfolio_api/
        pyproject.toml
        src/portfolio_api/
    orchestration/
      daily_jobs/
        pyproject.toml
        src/daily_jobs/
    dbt/
      finance/
```

The root owns workspace discovery and shared tool configuration. Child package
metadata owns runtime dependencies, entry points, optional extras, and package
data for that package.

## Package Layers

Keep package dependency arrows forward and acyclic. Domain packages own
vocabulary, types, invariants, and errors. Adapter packages translate foreign
data into domain values. Application and orchestration packages own runtime
wiring such as event loops, process pools, resources, settings, and logging.

A practical dependency shape is:

```text
application or orchestration
  imports adapters and domain packages

adapters
  import domain packages and vendor libraries

domain packages
  import standard library, small local domain modules, and narrow support
  libraries
```

Shared cross-layer domain types belong in a domain package. Database code and
pipeline code should import those types from the domain package instead of
importing each other to reach a model class.

Use `__init__.py` as the package's public API when a package exposes a small,
stable surface. Re-export supported variants there, assemble public union type
aliases there, and keep private module structure free to change.

## Tests

Mirror the source tree in `tests/` so readers can find coverage by path:

```text
src/market_data/indicators/annualized_volatility.py
tests/market_data/indicators/test_annualized_volatility.py
```

Configure pytest to import from `src/` deliberately. This makes tests exercise
the package import name instead of accidental current-directory imports.

Long-running benchmarks belong under `benchmarks/` and run through an explicit
command. Keep them separate from correctness tests so normal test loops stay
fast and deterministic.

## Generated Files

Generated artifacts belong in ignored output directories unless they are
intentional, reviewed inputs. Python caches, package build outputs, dbt target
directories, logs, coverage reports, rendered dependency graphs, and local
notebook outputs should not live beside source modules as if they were code.

When a generated artifact is useful in review, commit the stable output and
wire the command that refreshes it into the project command surface. A
dependency graph is useful only when contributors can regenerate it and the
review can explain what changed.

## Operational Files

Maintenance scripts should have explicit inputs, normal CLI parsing, and
nonzero exit codes for invalid user input. Use `pathlib.Path`, parse dates
through standard libraries or domain parsers, and prefer structured parsers
over regular expressions plus dynamic evaluation for code or SQL generation.

Sample configuration should be portable. Personal absolute paths, local
credentials, and machine-specific defaults belong in local files or examples
that clearly mark placeholder values.

