# factors2 - executive summary

A read of `/home/msi/python_workspace/factors2/` for inputs into the
Python-guidelines port of the C++ guidelines. The C++ guidelines are
architectural ("functional core / imperative shell", "parse don't validate",
"contracts at the edge"), so this read focuses on the same altitude: where
does factors2 succeed at applying that style in Python, and where does it slip.

## Codebase shape

- Monorepo with five top-level sub-packages under `src/`:
  - `pyfactors`     - pure-Python indicator/backtest/portfolio engine.
                       This is the "library" half.
  - `etl_factors`   - dlt-based sources and CSV/HTTP loaders.
  - `dagster_factors` - Dagster project: components, definitions, schedules.
  - `dbt_factors`   - dbt project (postgres + duckdb).
  - `dashboards`    - Streamlit UI on top of `pyfactors`.
- Stack: Python 3.13, uv, ruff, mypy + pyright (both, in CI), pre-commit
  with `uv-lock`/`uv-sync` + ruff + a pnpm bridge for the FE.
- Data: pydantic v2 for in-memory domain models, pandera for DataFrame
  schemas, dataclasses for plain records, ibis/duckdb/pandas for compute,
  dlt for ingestion, dbt for warehouse transforms.
- Testing: pytest + approvaltests + pytest-benchmark + pyfakefs (declared,
  largely unused). Tests are colocated by package shape under `tests/`,
  benchmarks live in `benchmarks/`.
- The `doc/` folder is small but the content is unusually high-signal:
  `guidelines.md` already states many of the principles the new repo will
  formalise (parse-don't-validate, prefer-named-args, type-safe args /
  returns, contract-style preconditions), and `adr/0001-*.md` is a clean
  ADR template the new repo can copy.

## Strongest patterns to PROMOTE

1. **Discriminated unions of Pydantic models** for plugin-shaped
   subsystems (indicators, scoring/signal/weighting strategies, untie,
   restrictions, liquidation rules). The pattern: each variant pins
   `function_name: Literal["x"] = Field(default="x", frozen=True)`, the
   sibling `__init__.py` exposes `Strategy = A | B | C | ...`, and the
   wrapper holds `Annotated[Strategy, Field(discriminator="function_name")]`.
   See `pyfactors/indicators/__init__.py:43`, `indicators/computed_indicator.py:14`,
   `pipeline/scoring_strategies/__init__.py:9`. This is the Python
   equivalent of the C++ sum-type-with-`lib::match` pattern.
2. **Layered validation: pandera at the dataframe boundary, hand-written
   cross-row invariants behind it**. `pipeline/stages.py:62-183` and
   `backtest/models.py:1022-1179` are textbook examples - one small,
   named predicate per invariant; one composing validator that calls them
   in order; the schema stays for column/dtype, the imperative code stays
   for cross-row rules. The matching tests in
   `tests/pyfactors/backtest/test_validate_step_result.py` are one
   assertion per test, error message asserted by `re.escape(...)`.
3. **`match` + `assert_never` on `Literal` types** for exhaustive
   dispatch (`indicators/operations/interval_indicator.py:30-36`,
   `scoring_strategies/binary.py:29-35`, `scoring_strategies/z_score.py:28-35`).
   This is the Python translation of C++ sum-type matching and is the
   single most important typing pattern in the codebase.
4. **Functional core / imperative shell at the boundaries.**
   `etl_factors/sources/economatica/__init__.py` is declarative
   (`make_resource(...)` for each table), `dagster_factors/components/*`
   keeps the Dagster glue at the boundary, and `pyfactors/backtest/simulation.py`
   keeps the trading rules as free functions (`mark_to_market`,
   `accrue_daily_borrow_fees`, `execute_order`, `process_trade`) with
   the `Simulation` class purely a driver.
5. **ADR practice.** `doc/adr/0001-schedule-only-economatica-downloads.md`
   is short, names the decision, the rejected alternative, and the
   consequences. Use it as the template.
6. **Tooling baseline that already matches modern Python.** uv,
   ruff (E/F/B/I/N/Q/UP/C4/ARG/NPY/PD/PERF/PL), mypy + pyright in CI,
   pre-commit with `uv-lock` + `uv-sync --locked` keeping `uv.lock`
   honest, separate jobs for the FE.

## Strongest anti-patterns to AVOID

1. **f-string interpolation of dates and filter args into raw SQL**:
   `pyfactors/database/database_queries.py:28,92,118,144,188,216,237`.
   The same module also contains the *correct* pattern alongside, using
   `psycopg.sql` (lines 285-294). New guideline: parameterise or
   use the SQL composition API; never f-string into a query.
2. **`except Exception as e: ... toast(e)`** swallowing in
   `pyfactors/portfolio_management/app.py:57,90,136,149,161,187,202,232`
   and `etl_factors/sources/brasilapi/fetch_cnpj.py:38`. The Streamlit
   case has a partial excuse (UI must not crash), but the result is the
   same: error class is lost, stack trace is gone, retry is silent.
3. **`print(...)` for production logging** in
   `pyfactors/pipeline/stages.py:307,308,329` (universe time/size),
   `pyfactors/backtest/server.py:44,102-103,118,123,129,131,132`
   (worker startup, simulation timing). The codebase has loguru in its
   deps and uses it in one place (`etl_factors/utils/proxy_downloader.py`);
   the rest is `print` + `timeit.default_timer()`. Standardise on
   structured logging.
4. **Trailing `raise ValueError(f"Unknown ...: {self}")` after an
   exhaustive `if/elif`** in `pyfactors/core/__init__.py:11-18`. The
   enum is closed, the right tool is `match` + `assert_never`, and
   the guideline doc literally argues for it on lines 76-90.
5. **Reaching into a Pydantic model and `function.deps()  # type: ignore`**:
   `indicators/computed_indicator.py:30,41,43` and the `Function` type
   in `indicators/functions.py:43-85` is a 40-member `A | B | C | ...`
   without a discriminator. The wrapper then uses `hasattr` + `inspect.signature`
   to decide which signature to call. New guideline: if you reach for
   `hasattr` on a union member, your union is the wrong shape -- give
   the variants a common protocol or make them all accept the same
   arguments.
6. **Per-call `.copy()` + ad-hoc `timeit.default_timer()` benchmarking
   inside production code** (`pipeline/stages.py:243,251,298,307,320,332`).
   Move timing to decorators/context managers or to actual benchmarks;
   move copies to the boundary where mutation is meaningful.
7. **`InterestRates`, `Benchmarks`, `MarketPrices` as hand-rolled
   plain classes when the rest of the codebase is Pydantic/dataclass**
   (`pipeline/models.py:139-225`). They each build a dict in `__init__`,
   re-implement getters with `None`-or-value semantics, and copy a
   `pd.DataFrame` defensively. The new guideline should pick one
   record style per layer and stick with it.

## Tooling baseline (one-line recap)

uv-managed `pyproject.toml`, ruff (line 120, broad rule selection,
test-folder ignores), mypy (+ pydantic plugin) **and** pyright both
enforced in CI, pre-commit pulling `uv-lock`/`uv-sync --locked` so the
lockfile is real, no `.python-version` pinned in the repo despite the
CI step that reads it, sqlfluff configured for dbt+postgres,
Makefile thin wrapper over uv. Full extract: `tooling-baseline.md`.

## What this read does NOT cover

- I read `dbt_factors` only as filenames; I did not read the SQL models.
- `dashboards/` was sampled (`portfolio_management/app.py`); the
  Streamlit UI is broadly the source of the `except Exception` smell.
- The proxy downloader is the only place async + retries are used; one
  data point is not a pattern.
- I did not run pydeps; the per-file imports look acyclic but a real
  cycle check is left to the reconciliation pass.
