# Module layout and DAG shape

The C++ guidelines call for *domain core vs imperative shell*,
*forward-only dependencies*, and *per-domain `types` namespace*. The
Python equivalents factors2 lives by are: a five-subpackage split
under `src/`, `__init__.py` exports as the public API, discriminated
unions assembled in `__init__.py` files, and a clear "library
package vs Dagster package vs ETL package" separation.

## Five sub-packages, named for the layer they own

```
src/
├── pyfactors/           # the library: indicators, backtest, portfolio_management, ...
├── etl_factors/         # dlt sources, file loaders
├── dagster_factors/     # Dagster components, definitions, schedules
├── dbt_factors/         # dbt project: models, seeds, snapshots
└── dashboards/          # Streamlit app
```

Each name is the layer it owns. `pyfactors` has no Dagster import.
`dagster_factors` imports `etl_factors` and `pyfactors` (in the
asset bodies) but not the other way around. `dbt_factors` is pure
SQL/yaml. **PROMOTE.**

## What factors2 does well

### `__init__.py` assembles the discriminated union

```python
# pipeline/scoring_strategies/__init__.py:1-11
from typing import get_args

from .binary import Binary
from .gaussian import GaussianRankScore
from .linear import LinearRankScore
from .sum_ranks import SumRanks
from .z_score import ZScore

Strategy = SumRanks | Binary | GaussianRankScore | LinearRankScore | ZScore

__all__ = [cls.__name__ for cls in get_args(Strategy)]  # pyright: ignore[reportUnsupportedDunderAll]
```

The `__init__.py` is the *single point* where the variant list is
assembled. Adding a new variant means adding a file in the subpackage
plus one line each in `__init__.py` (import + `|`). Same shape in
`indicators/__init__.py`, `signal_strategies/__init__.py`, etc.

**PROMOTE.** Guideline phrasing: *"discriminated-union types are
assembled in the subpackage's `__init__.py`; the `Strategy` alias is
the public API."*

### Subpackage cohesion: one concept per subpackage

```
src/pyfactors/
├── core/                       # cross-cutting constants and small helpers
├── indicators/
│   ├── fundamental/            # one file per fundamental indicator
│   ├── technical/              # one file per technical indicator
│   └── operations/             # composite indicators (IntervalIndicator)
├── pipeline/
│   ├── scoring_strategies/     # variants of one strategy concept
│   ├── signal_strategies/
│   ├── weighting_strategies/
│   ├── portfolio_strategies/
│   ├── untie_strategies/
│   ├── restrictions/
│   └── liquidation_rules/
├── ranking/
├── screening/
├── backtest/
│   ├── slippage_models/
│   └── trading_cost_models/
├── database/
└── portfolio_management/
```

Every leaf directory is *one concept*. There are no `utils/` or
`helpers/` folders inside `pyfactors`. The one `util.py` at the
`etl_factors` root is one tiny pyarrow helper. **PROMOTE.**

### Tests mirror the source tree

```
tests/pyfactors/indicators/technical/test_annualized_volatility.py
matches
src/pyfactors/indicators/technical/annualized_volatility.py
```

You can always find the test for any source file by replacing
`src/` with `tests/` and prefixing the filename with `test_`. No
guesswork. **PROMOTE.**

## What factors2 does poorly

### `defs/` vs `components/` inside `dagster_factors` is unclear

```
src/dagster_factors/
├── __init__.py
├── definitions.py             # entry point
├── components/                # custom Dagster Components
│   ├── dlt_load_collection_with_pool.py
│   ├── economatica_downloads.py
│   └── partitioned_dlt_load_collection.py
└── defs/                      # asset/schedule definitions
    ├── bcb/loads.py
    ├── economatica/loads.py
    └── scheduling/scheduling.py
```

`defs/` is Dagster's auto-loaded definitions folder (`definitions.py`
calls `dg.load_from_defs_folder(...)`). `components/` is custom
Component classes registered via the entry point in `pyproject.toml`
(`"dagster_dg_cli.registry_modules" = {dagster_factors = "dagster_factors.components"}`).
The distinction is real but undocumented in `doc/`.

**ADAPT.** Either rename or add a one-paragraph README at the
`dagster_factors/` root explaining the split. This is the only
package where the layout requires reading external Dagster docs to
follow.

### `pyfactors/pipeline/` is the load-bearing junk drawer

`pipeline/` contains:

- pure config/data types (`models.py`, `portfolio_definition.py`)
- variant subpackages (`scoring_strategies/`, etc.)
- top-level wrappers (`scoring_strategy.py`, `signal_strategy.py`, ...)
- orchestration (`stages.py`)
- domain rules (`liquidation_rule.py`, `restriction.py`, ...)
- pure functions (`lookback_window.py`)

Four kinds of code in one folder. The wrapper-vs-variant split is
clean (`scoring_strategy.py` wraps `scoring_strategies/`), but the
orchestration code (`stages.py`) and the cross-cutting types
(`models.py`) are jumbled into the same directory.

**ADAPT.** Two refinements:
1. Move `stages.py` (the orchestration) into its own `pipeline/orchestration.py`
   or out of `pipeline/` entirely.
2. The "wrapper module + variants subpackage" pair is a real
   pattern: `scoring_strategy.py` + `scoring_strategies/`. Document
   it. Possibly rename to `scoring/strategy.py` + `scoring/variants/`
   to put the relationship in the folder structure.

### Circular-import risk in `models.py`

`pipeline/models.py:99` declares `TradeDates.from_dataframe(dates_df: pa.typing.DataFrame[DatesTable])`,
where `DatesTable` is imported from `pyfactors/database/table_models.py`.
`pipeline/` should be a higher-level concept than `database/`, so
this is a downward import - fine in this direction. But several
strategy files in `pipeline/scoring_strategies/` import from
`pipeline/models.py` (`Rankings`, `ReferenceData`), and `models.py`
imports from `database/`. The chain is `database/ -> pipeline/models.py
-> pipeline/scoring_strategies/* -> pipeline/scoring_strategy.py`.
No cycle today, but the chain is fragile.

**ADAPT.** *"Models that are shared between layers live in a `domain/`
package (or a single `domain.py` module). Pipeline imports from
`domain`; database imports from `domain`; no cross-imports between
`pipeline` and `database`."*

### No `pydeps` output committed despite the dep being listed

`pydeps` is in `[dependency-groups].dev` (`pyproject.toml:71`) but
there is no `make` target, no committed graph image, no README link.
The information is available but not visible.

**ADAPT.** Add `make deps-graph` that runs `pydeps src/pyfactors -o
doc/pyfactors-deps.svg --max-bacon=2 --cluster` and commit the SVG;
or document the dep removal.

### No `pyfactors_web/` in `src/` but referenced everywhere

The pre-commit config and the CI workflow both reference `src/pyfactors_web/`
(pnpm formatter, separate web CI), but the directory does not exist
in the repo I read. Either it lives elsewhere or it was removed
without cleanup.

**ADAPT.** Audit and remove dangling references; the test of a clean
module layout is that every path in the config is real.

## Recommendations for the new guideline

1. **Top-level packages name the layer they own.** Library, ETL,
   orchestration, dashboards, SQL transforms. No `common/`, `shared/`,
   `utils/` at the top level.
2. **Subpackages are one concept each.** No `helpers/` or `utils/`
   folders inside a subpackage. If you have one, the right move is
   either (a) inline the helper into its only caller, or (b) lift it
   to a sibling subpackage that names what it does.
3. **`__init__.py` is the public API.** Re-export the variants;
   assemble the discriminated-union type alias; set `__all__`.
4. **Tests mirror `src/`.** Every test file is at the same relative
   path as the source file it covers.
5. **Wrapper module + variants subpackage** is the canonical shape
   for "interface + N implementations": `signal_strategy.py` (wrapper)
   + `signal_strategies/` (variants).
6. **Cross-layer data types live in their own module/package.** No
   `pipeline` importing from `database` (or vice versa) for shared
   types - both import from `domain`.
7. **Run `pydeps` in CI** to fail on new cycles; commit the rendered
   SVG so the structure is visible without running tools.
