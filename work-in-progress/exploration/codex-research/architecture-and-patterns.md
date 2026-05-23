# factors2 Architecture and Code Pattern Notes

Research slice: factors2 architecture and Python guideline patterns.

Source code read from `/home/msi/python_workspace/factors2`. These are research notes for a later writing phase, not a guide draft.

## Representative Files Reviewed

- `pyproject.toml`
- `doc/guidelines.md`
- `src/pyfactors/core/__init__.py`
- `src/pyfactors/core/datetime.py`
- `src/pyfactors/database/data_source.py`
- `src/pyfactors/database/dataset_queries.py`
- `src/pyfactors/database/table_models.py`
- `src/pyfactors/database/universe_query.py`
- `src/pyfactors/indicators/computed_indicator.py`
- `src/pyfactors/indicators/functions.py`
- `src/pyfactors/indicators/technical/rolling_capm_beta.py`
- `src/pyfactors/pipeline/models.py`
- `src/pyfactors/pipeline/portfolio_definition.py`
- `src/pyfactors/pipeline/rebalance_frequency.py`
- `src/pyfactors/pipeline/scoring_strategy.py`
- `src/pyfactors/pipeline/scoring_strategies/__init__.py`
- `src/pyfactors/pipeline/scoring_strategies/linear.py`
- `src/pyfactors/pipeline/signal_strategy.py`
- `src/pyfactors/pipeline/signal_strategies/__init__.py`
- `src/pyfactors/pipeline/signal_strategies/keep_n.py`
- `src/pyfactors/pipeline/stages.py`
- `src/pyfactors/pipeline/portfolio_strategies/static_exposure.py`
- `src/pyfactors/backtest/models.py`
- `src/pyfactors/backtest/simulation.py`
- `src/pyfactors/backtest/slippage_model.py`
- `src/pyfactors/backtest/stats.py`
- `src/dagster_factors/definitions.py`
- `src/dagster_factors/components/economatica_downloads.py`
- `src/dagster_factors/components/partitioned_dlt_load_collection.py`
- `src/dagster_factors/defs/economatica/defs.yaml`
- `src/dagster_factors/defs/economatica/loads.py`
- `src/dagster_factors/defs/dbt/defs.yaml`
- `src/dbt_factors/models/market_data/daily_rollup.sql`
- `src/dbt_factors/models/market_data/schema.yml`
- `src/dbt_factors/README.md`
- `src/etl_factors/sources/economatica/downloads.py`
- `tests/pyfactors/backtest/test_simulation_params_validation.py`
- `tests/pyfactors/database/test_universe_query.py`
- `tests/pyfactors/pipeline/test_portfolio_definition_validation.py`
- `tests/pyfactors/pipeline/test_validate_weighted_signals.py`

## Architecture Observations

The codebase has three major layers:

1. A Python domain and analytics library under `pyfactors`.
2. Data ingestion and orchestration under `etl_factors`, `dagster_factors`, and `dbt_factors`.
3. User-facing or operational entry points under dashboards, backtest examples, and portfolio management.

The Python library is centered on a pipeline that transforms a market universe into scores, signals, weights, orders, trades, and simulation results. The pipeline is not represented by a generic graph engine. It is mostly explicit function and method composition:

```text
fetch_universe
compute_indicators
rank_universe
scoring_strategy.compute_scores
signal_strategy.compute_signals
weighting_strategy.compute_weights
portfolio_strategy.rebalance
simulation.step
```

That explicit sequence is a useful Python style point. It keeps domain order visible and makes it easy to put validation at each boundary. The DAG-like behavior appears where it pays for itself: computed indicators expose dependencies through `deps()`, Dagster assets declare ingestion dependencies, and dbt models use `ref()`.

`pyproject.toml` shows a modern, heavily typed stack: Python 3.13, Pydantic 2, Pandera, pandas, numpy, DuckDB, Dagster, dbt, Ruff, mypy, pyright, pytype, and approval tests. The lint configuration deliberately relaxes some complexity rules (`PLR0911`, `PLR0912`, `PLR0913`, `PLR0915`) and test magic-number rules. That is a pragmatic signal: this codebase values type and schema checks, but does not force every trading or data function into tiny fragments when domain logic is naturally branchy.

## Dependency and Import Patterns

Imports mostly flow inward:

- Strategy leaf modules import `pyfactors.pipeline.models` and implement one method.
- Wrapper modules such as `scoring_strategy.py`, `signal_strategy.py`, `portfolio_strategy.py`, `slippage_model.py`, and `trading_cost_model.py` import strategy unions and delegate to the concrete object.
- Backtest imports pipeline models and strategy wrappers, then composes them into a stateful simulation.
- Dagster imports ETL source definitions and component classes. The core `pyfactors` library does not appear to depend on Dagster.
- dbt models are wired to Dagster through dbt metadata and a Dagster dbt component, rather than Python code manually importing SQL models.

The import pattern is worth preserving in guidelines: keep orchestration dependencies at the edges. Domain code can expose typed configuration objects and pure-ish operations. Dagster, dbt, HTTP clients, and filesystem concerns belong in the ingestion and orchestration layer.

One tradeoff is that union registries in package `__init__.py` files create central import points:

- `pipeline/scoring_strategies/__init__.py`
- `pipeline/signal_strategies/__init__.py`
- `pipeline/weighting_strategies/__init__.py`
- `pipeline/portfolio_strategies/__init__.py`
- `indicators/functions.py`

This works well with Pydantic discriminated unions, but the registry must be edited when a new strategy is added. That is acceptable when the registry is intentionally the public configuration surface. It should be called out as a feature, not hidden behind dynamic import magic.

## Declarative Configuration and Strategy Objects

The strongest recurring pattern is "declarative configuration object plus imperative implementation method."

Examples:

- `ScoringStrategy` wraps a discriminated union of scoring strategies and an `UntieStrategy`.
- `SignalStrategy` wraps a discriminated union of signal strategies plus restrictions.
- `RebalanceFrequency` wraps `Daily`, `BiWeekly`, `Monthly`, or `DayOfTheWeek`.
- `SlippageModel` and `TradingCostModel` wrap model unions.
- `ComputedIndicator` wraps a discriminated union of indicator functions.

Concrete strategies are small Pydantic models with literal discriminator fields:

```python
class KeepN(BaseModel):
    strategy: Literal["keep_n"] = Field(default="keep_n", frozen=True)
    keep: Literal["top", "bottom", "both"]
    n: int | float
    by: Literal["symbols", "percent"]
```

This is a good guideline pattern for Python systems with user-configurable behavior:

- Use Pydantic models for configuration that crosses API, CLI, YAML, or storage boundaries.
- Use discriminated unions when a field selects one of several concrete strategies.
- Put the domain verb on the strategy (`compute_scores`, `compute_signals`, `rebalance`, `compute`) instead of forcing a generic interface with vague method names.
- Keep the union registry explicit when the allowed variants are part of the supported contract.

## DataFrame Boundaries and Validation

The codebase uses Pandera `DataFrameModel` classes for table and pipeline-stage shapes:

- database tables: `CalendarTable`, `PricesTable`, `UniverseTable`, `QuarterlyIndicatorsTable`
- pipeline stages: `Scores`, `RestrictedSymbols`, `RestrictedScores`, `Signals`, `WeightedSignals`

The good pattern is boundary validation. Query functions return `TableModel.validate(df)`. Pipeline functions return `Scores.validate`, `Rankings.validate`, or `WeightedSignals.validate`. This gives loose pandas code a typed shell without pretending pandas can be statically typed at every expression.

The code also distinguishes schema validation from domain invariants. `validate_weighted_signals` checks facts that Pandera cannot express cleanly:

- the final weighted dataframe is non-empty
- each `(trade_date, market, symbol)` appears once
- active rows have finite scores
- scored rows have finite, positive close prices
- active weights are finite and positive
- active weights sum to 1.0 per date and side

That pattern is worth preserving:

- Schema validators check shape and dtypes.
- Domain validators check cross-row and cross-field invariants.
- Error messages include counts and samples of offending rows.

The tests for `validate_weighted_signals` are concrete and property-oriented. They assert the domain error, not internal call order.

## Invariants and Configuration Validation

The codebase validates invariants close to the configuration object that owns them:

- `PortfolioDefinition` rejects `RiskParity` paired with one-sided signal strategies.
- `StaticExposure` rejects `abs(net_exposure) > gross_exposure`.
- `KeepN` checks that `n` is an integer for symbol counts and between 0 and 100 for percentages.
- `SimulationParams` validates paired `HardToBorrow` restriction and liquidation rules.
- `RebalanceFrequency` variants encode allowed rebalance modes as typed objects rather than free-form strings.

This supports a guideline topic: "Make invalid configurations unrepresentable where reasonable, and validate cross-object invariants at the aggregate boundary." The aggregate boundary matters. `RiskParity` cannot validate itself against a signal strategy it does not own, so `PortfolioDefinition` owns that check.

One pattern to be careful with: `SimulationParams.model_post_init` mutates `self.signal_strategy.restrictions` by appending a `DelistedSymbols` restriction. Hidden mutation during model initialization can surprise callers who reuse a `SignalStrategy` instance. A future guideline should prefer deriving an effective configuration or copying before mutation when a validated model receives nested mutable objects.

## DAG-like Structures

There are three DAG-like systems:

1. Indicator dependencies in `ComputedIndicator.deps()` and `extract_computed_indicators`.
2. Dagster assets and components under `dagster_factors`.
3. dbt models using `ref()` and dbt metadata for Dagster asset keys.

`extract_computed_indicators` performs a depth-first dependency expansion and returns dependencies before dependents. It is simple and readable, but it does not appear to deduplicate repeated dependencies or detect cycles. For a small indicator graph, this is acceptable. For a larger graph, guidelines should recommend making graph invariants explicit:

- identify nodes by stable keys
- deduplicate repeated dependencies
- reject cycles with a clear error
- test ordering behavior with shared dependencies

Dagster and dbt are used where the graph has operational consequences. `definitions.py` delegates loading to Dagster's defs folder. `defs.yaml` declares downloads, partitioning, resource dependencies, and environment requirements. The dbt component points at the dbt project, and dbt model metadata maps SQL models to Dagster asset groups.

This is a good architectural boundary: in-process Python pipeline composition stays explicit, while durable orchestration uses tools that already know how to model assets, partitions, dependencies, and materializations.

## Dynamic and Pragmatic Python Examples

Several examples show where Python should stay dynamic or pragmatic rather than over-typed.

Pandas internals:

- Most indicator implementations operate on `pd.DataFrame` and `pd.Series` directly.
- `RollingCAPMBeta.compute_rolling` creates temporary columns, uses groupby transforms, and returns a small dataframe with join keys plus the computed indicator.
- Over-typing every intermediate column would add noise and would not make pandas expressions substantially safer.

DataFrame wrappers:

- `MarketPrices`, `InterestRates`, and `Benchmarks` keep the original dataframe and build dictionaries for O(1) lookup.
- The dict keys and return types are annotated, but the source dataframe remains a dataframe.
- This is a practical compromise between pandas flexibility and simulation performance.

Dynamic indicator dispatch:

- `ComputedIndicator.compute_rolling` uses `inspect.signature` to support functions that may or may not need `ReferenceData`.
- This is less type-safe than a uniform protocol, but it keeps many indicator implementations simple.
- A future guideline should not ban this pattern. It should require a reason, a small adapter boundary, and tests that exercise both call shapes.

Query construction:

- `make_universe_query` builds SQL text dynamically while keeping values in a separate params list for filters and benchmark symbols.
- The generated SQL is tested with approval tests.
- This is a pragmatic pattern for SQL that must vary structurally, but the code also shows a risk: some dates and offsets are interpolated into SQL strings. A guideline should distinguish safe structural SQL generation from value interpolation and recommend parameters wherever the database API supports them.

Pydantic plus pandas:

- Pydantic models own configuration and small domain objects.
- Pandera owns dataframe shape.
- pandas owns vectorized transformation.
- Trying to force one typing mechanism across all three would make the code worse.

## Patterns Worth Preserving in Python Guidelines

- Use typed configuration objects for behavior that can be selected declaratively.
- Use discriminated unions when configuration selects a concrete strategy.
- Keep strategy registries explicit when they define the supported public surface.
- Put invariants at the object that has enough context to validate them.
- Validate dataframes at system boundaries, not after every pandas expression.
- Pair schema validation with separate domain-invariant validation.
- Include offending row samples in dataframe validation errors.
- Use domain vocabulary in names: `trade_date`, `benchmark_rate`, `risk_free_rate`, `ranking_nodes`, `restricted_buys`, `weighted_signals`, `rebalance_frequency`, `liquidation_rules`.
- Prefer explicit pipeline stages over generic graph abstractions until graph behavior is a real domain concept.
- Let orchestration tools own durable dependency graphs, partitions, and asset materialization.
- Use approval tests for generated SQL or other large structured strings.
- Use enums or literal unions for small closed domains.
- Keep pandas code readable and vectorized instead of wrapping every operation in small abstractions.

## Patterns to Improve or Avoid in Future Guidance

- Avoid hidden mutation of nested configuration objects during model initialization. Prefer immutable models, copied nested objects, or explicit effective-configuration builders.
- Avoid broad `type: ignore` on dynamic dispatch unless the dynamic boundary is deliberate and tested.
- Avoid comments that narrate obvious pandas operations. Comments should explain domain assumptions, numerical choices, or non-obvious invariants.
- Avoid placeholder or starter documentation in project-owned docs. `src/dbt_factors/README.md` still contains stock dbt starter text, which does not explain this project's dbt role.
- Avoid large domain model files that mix stage schemas, lookup caches, orders, positions, and books indefinitely. `pipeline/models.py` is convenient now, but it may become hard to navigate as the domain grows.
- Avoid SQL value interpolation where parameters can be used. Dates, offsets, and identifiers need different handling, but the distinction should be explicit.
- Avoid unbounded dependency expansion in DAG-like helpers if the graph can grow. Add deduplication and cycle detection when dependencies become shared or user-extensible.
- Avoid log and print calls in reusable library functions for timing and operational messages. Use logging or instrumentation hooks at orchestration boundaries.

## Candidate Guide Topics Grounded in factors2

- Strategy objects and discriminated unions for declarative behavior.
- DataFrame validation boundaries with Pandera and Pydantic.
- Where to put invariants in Python domain models.
- Building explicit pipelines before reaching for graph frameworks.
- When to use a real orchestration DAG instead of in-process composition.
- Testing generated SQL with approval tests.
- Keeping pandas dynamic without giving up contracts.
- Designing extension registries that are explicit and maintainable.
- Separating configuration models, runtime state, and vectorized data transformations.
- Writing useful validation errors for batch data.
- Handling domain-specific closed sets with `Enum`, `Literal`, and Pydantic validators.
- Treating dynamic dispatch as an adapter boundary.
