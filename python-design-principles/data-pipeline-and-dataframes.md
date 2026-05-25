# Data Pipelines And Dataframes

Data pipelines should make boundaries explicit. Ingest messy data once, parse
it into named contracts, validate dataframe shapes at module and system
boundaries, and let each compute layer do the work it is good at. Keep pandas,
Polars, SQL, ibis, dbt, dlt, and Dagster code native to those tools instead of
wrapping every expression in a generic abstraction.

Use dataframe code where tabular operations are the domain. Use domain values
around the dataframe so boundaries, configuration, errors, and tests stay
readable.

## Choose The Layer Per Stage

Pick the tool that owns the stage's real contract:

- dlt for source ingestion, incremental state, file metadata, dispositions,
  and source resources.
- ibis, DuckDB, or another query engine for declarative reshape at messy ingest
  boundaries.
- dbt for warehouse transforms, SQL-first models, schema tests, snapshots, and
  lineage.
- SQL composition APIs or query builders for application-owned database reads.
- pandas or Polars for in-memory tabular calculations.
- Pandera for dataframe schemas at Python boundaries.
- Dagster for durable orchestration, partitions, asset selection, resources,
  and schedules.

In-process Python pipelines should stay explicit until the graph itself has
operational meaning. Use Dagster, dbt, dlt, or another orchestration tool when
the graph needs materialization, lineage, partitions, retries, or schedules.

## Parse At The Boundary

Foreign data should become project data at one boundary. HTTP payloads,
vendor CSV files, object-store paths, queue messages, and UI uploads should be
parsed by adapters that name the source format and emit normalized values or
frames.

Pydantic fits scalar and record-shaped boundaries. Pandera fits dataframe
boundaries. A query engine fits messy tabular reshape when column normalization
and filtering are easier to express relationally.

```python
from datetime import date

import pandera.pandas as pa
from pandera.typing import DataFrame, Series
from pydantic import BaseModel, Field, field_serializer


class RateRequest(BaseModel):
    series_code: str
    start_date: date = Field(serialization_alias="dataInicial")
    end_date: date | None = Field(default=None, serialization_alias="dataFinal")

    @field_serializer("start_date", "end_date", when_used="always")
    def serialize_date(self, value: date | None) -> str | None:
        return value.strftime("%d/%m/%Y") if value is not None else None


class Rates(pa.DataFrameModel):
    trade_date: Series[pa.DateTime]
    rate: Series[float]
```

After parsing, downstream code should use domain names such as `trade_date`,
`symbol`, `market`, `score`, `weight`, and `benchmark_rate`, rather than vendor
column names.

## Dataframe Schemas

Use Pandera schemas for dataframe values that cross module, package, pipeline,
job, or test-helper boundaries. A schema should carry column names, dtypes,
nullability, simple value constraints, index expectations, and coercion policy.

```python
import pandera.pandas as pa
from pandera.typing import DataFrame, Series


class WeightedSignals(pa.DataFrameModel):
    trade_date: Series[pa.DateTime]
    symbol: Series[str]
    side: Series[str]
    score: Series[float]
    weight: Series[float] = pa.Field(ge=-1.0, le=1.0)
```

Annotate signatures with schema types when they clarify the contract:

```python
def validate_weighted_signals(
    weighted_signals: DataFrame[WeightedSignals],
) -> DataFrame[WeightedSignals]:
    _validate_non_empty(weighted_signals)
    _validate_unique_keys(weighted_signals)
    _validate_side_weight_sums(weighted_signals)
    return WeightedSignals.validate(weighted_signals)
```

Pandera owns table shape. Named validators own cross-row and cross-column
invariants that need domain context, such as uniqueness, sortedness, matching
side totals, complete date coverage, or absence of stale reference data. Keep
each invariant small and test it directly.

Use coercion deliberately. Ingestion schemas may normalize source types.
Domain-stage schemas should usually reject unexpected types so data quality
failures stay visible.

## Dataframe Mutation Ownership

Dataframe transforms need a clear mutation contract. Pure transforms read
input columns and return a new frame with join keys plus output columns. The
orchestrator joins the outputs. In-place transforms should own the frame they
mutate.

```python
def compute_score(
    observations: DataFrame[Observations],
) -> DataFrame[Scores]:
    score = (
        observations.groupby("trade_date", sort=False, observed=True)["value"]
        .rank(method="first", ascending=False)
        .astype(float)
    )
    scores = observations[["trade_date", "symbol"]].assign(score=score)
    return Scores.validate(scores)
```

Keep scratch values local. Scratch columns prefixed with `_` still mutate the
caller's frame when they are assigned into the input. Local series, temporary
frames, and `assign` keep ownership visible.

Avoid defensive copies in orchestration loops. They hide the mutation bug and
scale with the full frame size. Fix the transform contract instead.

## SQL And Generated Queries

Values enter SQL through bind parameters or typed SQL composition objects.
String formatting may choose static clauses, table names from an allowlist, or
predefined orderings, but boundary values such as dates, symbols, identifiers,
and user inputs should use the database API.

```python
from datetime import date

import duckdb
import pandas as pd


def fetch_prices(
    connection: duckdb.DuckDBPyConnection,
    start_date: date,
    end_date: date,
) -> pd.DataFrame:
    return connection.execute(
        """
        select trade_date, symbol, close
        from market_data.daily_prices
        where trade_date between ? and ?
        order by trade_date, symbol
        """,
        [start_date, end_date],
    ).df()
```

For generated SQL, use a structured input language and approval tests. Keep
the accepted input smaller than arbitrary Python unless the input file is
intentionally trusted code.

## Dagster, dbt, And Schedules

Dagster assets should be thin orchestration wrappers. The asset body reads
resources and context, delegates to a library function, and returns metadata.
The library function owns the domain work and remains importable without
Dagster.

```python
import dagster as dg


@dg.asset
def daily_prices(context: dg.AssetExecutionContext) -> dg.MaterializeResult:
    row_count = load_daily_prices(partition_key=context.partition_key)
    return dg.MaterializeResult(metadata={"row_count": row_count})
```

Use dbt for warehouse transforms that are naturally SQL models. Let dbt tests,
models, snapshots, and lineage own warehouse behavior. Python should call or
orchestrate dbt rather than recomputing the same warehouse transform in pandas.

Schedule durability concerns. Vendor-rotated downloads, expiring files, and
missed external windows need schedules and alerts. Replayable transforms can
often be materialized on demand from persisted inputs.

## pandas, Polars, ibis, And SQL

Choose dataframe and query tools by workload:

- pandas fits mature ecosystem integrations, moderate in-memory datasets,
  rolling per-group calculations, and codebases already standardized on it.
- Polars fits larger in-memory columnar work, lazy query plans, and fast
  expression pipelines when the dependency is accepted.
- ibis fits backend-portable relational expressions and ingest normalization
  that can run on DuckDB or a warehouse.
- SQL fits persisted relational data, joins close to storage, and transforms
  that operations teams inspect through warehouse tools.

Keep native expressions in the tool's own vocabulary unless the project truly
needs backend independence. Readable tool-native code is usually easier to
review and profile.

## Dataframe Tests

Test dataframe behavior with the dataframe library's assertion helpers,
Pandera validation, and domain invariants. Sort rows and columns when order is
not part of the contract. Use explicit tolerances for floating-point results.

```python
import pandas as pd
from pandas.testing import assert_frame_equal


def test_compute_score_ranks_by_trade_date(observations):
    actual = compute_score(observations)

    expected = pd.DataFrame(
        {
            "trade_date": pd.to_datetime(["2026-01-02", "2026-01-02"]),
            "symbol": ["AAA", "BBB"],
            "score": [1.0, 2.0],
        }
    )
    assert_frame_equal(actual.reset_index(drop=True), expected)
```

Use approval tests for large generated SQL, reports, JSON, or configuration
where the full artifact matters. Pair broad approval with direct assertions for
security-sensitive values, bind parameters, row counts, and boundary cases.

## Review Checklist

- The stage uses the tool that owns the contract: dlt, ibis, dbt, SQL, pandas,
  Polars, Pandera, or Dagster.
- Foreign data is parsed once at the adapter boundary.
- Dataframes that cross boundaries have named schemas.
- Cross-row invariants live in named validators with focused tests.
- Transforms state whether they are pure or own the frame they mutate.
- SQL values use parameters or typed composition APIs.
- Dagster assets delegate domain work to library functions.
- Dataframe tests assert schema, full output, invariants, or tolerances as
  appropriate.
