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

This guidance scales. A single tool or library uses one package and one
`pyproject.toml`. A platform with several deployable units, shared libraries,
orchestration, and infrastructure uses a workspace monorepo. Start at the size
the work needs and grow into the larger shape when a second deployable unit or a
shared library appears.

## Single-Package Projects

A one-off tool or a standalone library does not need a workspace, a `docker/`
tree, or an orchestration layer. It needs one package, one set of dependencies,
and tests that import it the way callers will.

```text
annualized_volatility/
  pyproject.toml
  src/annualized_volatility/
    __init__.py
  tests/
    test_annualized_volatility.py
```

Keep the `src/` layout even here. It costs almost nothing and it buys the same
import honesty the larger shape relies on: pytest exercises the installed
package name rather than whatever happens to sit in the current directory. The
single root `pyproject.toml` owns runtime dependencies, entry points, and tool
configuration directly, with no workspace indirection.

Reach for the monorepo shape below when the project grows a second deployable
unit, a library that more than one unit imports, or infrastructure and
orchestration that need their own homes. Until then, the extra directories are
scaffolding without tenants.

## Repository Shape

A Python monorepo benefits from a small number of top-level areas with clear
ownership:

- The repository root owns workspace metadata, lockfiles, global tool
  configuration, CI configuration, and developer command entry points.
- `src/` owns Python packages, applications, orchestration code, and data
  pipeline code.
- `docker/` owns image build definitions and the compose stacks that wire
  services together.
- `db/` owns shared database schemas and the migrations that maintain them.
- `tests/` mirrors the package paths it exercises.
- `benchmarks/` owns performance tests that run outside the normal correctness
  loop.
- `etc/` or `scripts/` owns maintenance tools that are not imported by product
  code.

The philosophy that holds this together is strict isolation of concerns: a hard
boundary between infrastructure, source code, and the individual deployable
units inside it. Each area can be read, owned, and changed without dragging the
others along.

Put source packages under `src/`, and name each top-level package for the layer
or domain it owns. Names such as `common`, `shared`, `helpers`, and `utils`
usually mean the boundary has not been named yet. Prefer names such as
`market_data`, `risk_model`, `etl_sources`, `orchestration`, or `dashboards`
when those are the concepts the package owns.

```text
repo/
  pyproject.toml
  uv.lock
  compose.yaml
  .github/
  docker/
    images/
    stacks/
  db/
    main_postgres/
    analytics_mongo/
  src/
    libs/
      market_data/
        pyproject.toml
        src/market_data/
    apps/
      portfolio_api/
        pyproject.toml
        src/portfolio_api/
        portfolio_api/migrations/
    orchestration/
      daily_jobs/
        workspace.yaml
        pyproject.toml
        src/daily_jobs/
    dbt/
      finance/
      marketing/
```

## Workspace Members

Use a root `pyproject.toml` to define the workspace and shared tool policy.
Each deployable package or reusable library gets its own child `pyproject.toml`
with the runtime dependencies it owns.

The root `pyproject.toml` does not carry application dependencies such as
`pandas` or `django`. It declares the workspace members and houses global
configuration for tooling such as Ruff, Mypy, and pytest. Each child
`pyproject.toml` under `libs/`, `apps/`, and `orchestration/` declares only the
dependencies that package needs.

This keeps dependency pressure local. A dashboard can depend on Streamlit, a
Dagster code location can depend on Dagster, and a domain library can stay
small enough to test and reuse without installing the whole platform.

When an application needs a shared library, it references the local path rather
than a published version:

```toml
[dependencies]
django = "^4.2"
market_data = { path = "../../libs/market_data", develop = true }
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

## Infrastructure and Builds

By keeping all application logic, libraries, and pipelines inside `src/`, you
make tooling configuration trivial: point Ruff, Mypy, and pytest at `src/` and
they naturally ignore Docker definitions, CI configuration, and database
schemas.

Image builds live in `docker/images/` rather than beside the code they package.
A deployable unit often needs files from several places at once: its own code, a
shared library, and a dbt project. A Dockerfile pinned inside one `src/` folder
cannot reach the others, so the build definitions live in one central place and
copy from across the tree.

Run Docker builds and `docker compose` commands from the repository root. The
build context is the whole monorepo, which lets a Dockerfile copy files from
anywhere in it. This is the rule that makes the centralized build directory
work.

Use compose profiles to start only the services a task needs. A service that
declares a profile starts only when that profile is requested, so a developer
working on the API does not have to run the data stack.

```yaml
# docker/stacks/data-stack.yml
services:
  dagster_de:
    profiles: ["data", "all"]

# docker/stacks/app-stack.yml
services:
  django_api:
    profiles: ["app", "all"]
```

```text
docker compose --profile data up
```

## Orchestration: Dagster and dbt

Keep orchestration configuration ignorant of the applications and libraries
around it. A Dagster `workspace.yaml` points the daemon at individual code
locations and knows nothing about your APIs or shared libraries. Each code
location is its own workspace member with its own `pyproject.toml`.

Keep dbt projects separate from the Python that orchestrates them. Analytics
engineers then develop, test, and run dbt natively through the dbt CLI without
needing to know Python or Dagster. Dagster wraps those project directories at
runtime through `dagster-dbt`.

## Databases

Distinguish shared database schemas from migrations an application owns.

Shared databases live under `db/`, one directory per database, each maintained
by a migration tool such as Flyway or Atlas. These are operational inputs owned
at the repository level because more than one unit depends on their shape.

Migrations that belong to a single application live inside that application's
package, next to the code that defines the models. They are owned by the
application and version with it.

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
