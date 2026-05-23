# Data pipeline style

factors2 mixes four compute layers - **dbt**, **ibis**, **duckdb SQL**,
and **pandas** - and three orchestration layers - **dlt** for source
ingest, **Dagster** for jobs/schedules, in-process Python for the
backtest engine. The lesson the new guideline can lift: pick the right
layer per stage, keep each layer's boundary clean.

## Layer responsibilities, as factors2 uses them

| Stage                                 | Tool      | Why                                          |
|---------------------------------------|-----------|----------------------------------------------|
| HTTP / file ingest                    | dlt       | incremental state, file metadata, dispositions |
| CSV reshape at the ingest boundary    | ibis      | declarative, can run on duckdb               |
| Warehouse transforms                  | dbt       | SQL-first, tested, schema-managed            |
| In-memory query against warehouse     | duckdb SQL via the python connector | one-shot reads |
| In-memory rolling math per symbol     | pandas    | groupby + transform is the actual hot loop   |
| Orchestration                         | Dagster   | schedules, asset selection, partitions       |
| Strategy parameters / variants        | Pydantic  | typed config + (de)serialisation             |

This is a sensible spread. The new guideline can encode it almost
verbatim.

## What factors2 does well

### dlt source as a function that builds resources from a config table

```python
# etl_factors/sources/economatica/__init__.py:209-406
@dlt.source(max_table_nesting=0, parallelized=True)
def economatica(bucket_url: str = dlt.config.value) -> list[DltResource]:
    def make_resource(name, columns, *, glob=None, write_disposition, ...): ...

    resources = []
    resources.append(make_resource("security_list", SECURITY_LIST, write_disposition="replace", ...))
    resources.append(make_resource("stock_borrow", STOCK_BORROW, write_disposition="merge", ...))
    ...
    return resources
```

The `make_resource` factory is the imperative shell; the per-table
config (snapshot vs incremental, write disposition, primary key,
transforms) is data. New tables = new `make_resource(...)` call. The
column definitions live in their own file (`column_definitions.py`).

**PROMOTE** the "config-as-data + tiny factory" shape.

### ibis as the *only* place column-name normalisation happens

```python
# etl_factors/sources/economatica/__init__.py:43-158
def process_csv(file, columns, *, primary_key=None, filter_nulls=None, ...):
    ...
    # derive columns rename map from columns
    column_names = {v: k for k, (v, _) in columns.items()}
    column_types = {k: v for k, (_, v) in columns.items()}

    con = ibis.duckdb.connect()
    table = con.read_csv(file, encoding=..., sep=..., decimal_separator=..., ...)
    table = table.rename(column_names)
    if "trade_date" in table.columns:
        table = table.filter(table.trade_date >= "2000-01-01")
    if "asset" in table.columns:
        table = table.mutate(symbol=table.asset.re_extract(...))
        table = table.mutate(market=table.asset.re_extract(...))
        if drop_otc:
            table = table.filter(~table.symbol.endswith("B"))
    for col in quarter_columns:
        ...
    table = table.drop_null(primary_key, how="any")
    ...
    arrow_table = table.to_pyarrow()
```

This is *exactly* the layered-data-pipeline pattern: the messy vendor
CSV is parsed at exactly one boundary, into a normalised pyarrow
table with a documented schema. Everything downstream gets a clean
table.

**PROMOTE.** The guideline phrasing: "parse at the boundary, then
work on normalised types".

### Dagster: declarative selections, one schedule, narrow scope

```python
# dagster_factors/defs/scheduling/scheduling.py:20-58
economatica_downloads_selection = dg.AssetSelection.key_prefixes(["economatica", "downloads"])
economatica_raw_selection = dg.AssetSelection.key_prefixes(["economatica", "raw"])
bcb_selection = dg.AssetSelection.groups("bcb")
ingest_selection = economatica_raw_selection | bcb_selection
models_selection = dg.AssetSelection.all() - economatica_downloads_selection - ingest_selection

economatica_downloads_job = dg.define_asset_job(name="...", selection=economatica_downloads_selection)
factors_ingest_job = dg.define_asset_job(name="...", selection=ingest_selection)
factors_models_job = dg.define_asset_job(name="...", selection=models_selection)

@dg.schedule(job=economatica_downloads_job, cron_schedule="0 7 * * 1-5", execution_timezone="America/Sao_Paulo")
def economatica_downloads_schedule(context): ...
```

Three jobs, exactly one schedule, written as set algebra over asset
selections. The corresponding ADR
(`doc/adr/0001-schedule-only-economatica-downloads.md`) says exactly
why this is the shape: only the vendor-rotated downloads are
irrecoverable, so only they are scheduled.

**PROMOTE.** *"Schedule durability concerns, not pipelines. One
durability concern, one schedule."* This is gold and goes straight
into the Python guidelines.

### Custom Dagster components factor out repeated wiring

```python
# dagster_factors/components/economatica_downloads.py:35-67
@dataclass
class EconomaticaDownloads(dg.Component, dg.Resolvable):
    pool: str | None = None
    bucket_url_env: str = "SOURCES__ECONOMATICA__BUCKET_URL"
    partitions_def: ResolvedPartitionsDef = None

    def build_defs(self, context: dg.ComponentLoadContext) -> dg.Definitions:
        def make(screen: str) -> dg.AssetsDefinition:
            @dg.asset(name=screen, key_prefix=["economatica", "downloads"], ...)
            def _download(context: dg.AssetExecutionContext) -> dg.MaterializeResult:
                bucket_url = os.environ[bucket_url_env]
                file_date = date.fromisoformat(context.partition_key)
                path = download_screen(screen, bucket_url, file_date)
                context.log.info(f"downloaded {screen} to {path}")
                return dg.MaterializeResult(metadata={"path": dg.MetadataValue.path(path)})
            return _download

        return dg.Definitions(assets=[make(screen) for screen in sorted(SCREEN_URLS)])
```

The component owns the "loop over `SCREEN_URLS`" wiring; the per-asset
body delegates to the pure `download_screen(screen, bucket_url, file_date)`
function in `etl_factors`. The orchestration is at the edge, the
business logic is in a library function. This is the functional core /
imperative shell pattern, applied to Dagster.

**PROMOTE.** *"Dagster assets are a one-line delegation to a function
in a library package."*

### dbt for warehouse transforms; Python for everything else

The `dbt_factors/` project handles the warehouse transforms - models,
seeds, snapshots, tests, dbt_packages. The Python pipeline does not
re-implement what dbt does. The split is consistent: SQL stays in
dbt, Python stays in dbt orchestration via `dagster-dbt`.

**PROMOTE.** Don't recompute in pandas what dbt has already
materialised; use the warehouse as the source of truth.

## What factors2 does poorly

### f-string SQL composition

`pyfactors/database/database_queries.py` builds queries with
`f"trade_date >= '{start_date}'::DATE"` and similar:

```python
# database_queries.py:17-35
def fetch_stocks(start_date: date | None, end_date: date | None):
    where_conditions = []
    if start_date is not None:
        where_conditions.append(f"trade_date >= '{start_date}'::DATE")
    if end_date is not None:
        where_conditions.append(f"trade_date <= '{end_date}'::DATE")
    where_clause = "AND " + " AND ".join(where_conditions) if where_conditions else ""
    query = f"""
        SELECT * EXCLUDE (instrument_id, issuer_id) FROM market_data.daily_rollup
        WHERE quarter_date is not null
        {where_clause}
        ORDER BY trade_date, market, symbol
    """
    df = duckdb.sql(query).df()
```

Same shape repeats in `fetch_interest_rates`, `fetch_benchmarks`,
`fetch_quarterly_indicators`, `fetch_calendar`, `fetch_blacklist`,
`get_blacklisted_symbols`. Two of them embed a Postgres query string
inside a `duckdb.sql(f"select * from postgres_query('pg', '{query}')")`
- a double interpolation.

The same file *also* contains the correct pattern alongside, using
`psycopg.sql.SQL(...).format(sql.Literal(...))` (`database_queries.py:285-294`).
The new guideline should pick one and ban the other.

**AVOID.** Guideline: "use the connector's SQL composition API
(`psycopg.sql`, ibis, sqlalchemy.text + bindparams). Never f-string
SQL." Even though `start_date: date` is "safe", the consistency rule
matters: a future date string from a UI form will inherit the
f-string and be unsafe.

### `print()` for timing inside the production hot path

```python
# pipeline/stages.py:307-308
print("universe time: ", timeit.default_timer() - t1)
print("universe size: ", universe.memory_usage(deep=True).sum() / 1024 / 1024, "MB")
```

Repeats in `stages.py:329`, `backtest/server.py:44,102,103,118,123,129,131,132`.
The codebase has loguru in its dependencies and uses it in
`etl_factors/utils/proxy_downloader.py`. A consistent
`logger.info("universe ready", elapsed_s=..., size_mb=...)` is the
shape the rest of the codebase already supports.

**AVOID.** See `logging-observability.md`.

### `.copy()` inside the hot loop

```python
# pipeline/stages.py:243-260
def compute_indicators(universe, indicators, reference_data):
    inputs = universe.copy()
    ...
    for indicator in indicators:
        if indicator.indicator_column_name not in universe:
            columns.append(indicator.indicator_column_name)
            indicator_df = indicator.compute_rolling(inputs.copy(), reference_data)
            inputs = inputs.merge(indicator_df, on=[...], how="inner")
```

Every indicator gets a fresh `inputs.copy()` because each
`compute_rolling` mutates its argument (it writes `_log_close`, `_pos`
etc. columns in `LinearRegressionSlope`). The mutation is the actual
bug; the defensive copy is the workaround. Compare to
`AnnualizedVolatility` (`indicators/technical/annualized_volatility.py:27-35`)
which uses `groupby(...).transform(...)` and returns only the new
column, no mutation.

**AVOID.** The guideline phrasing: "indicators are pure. They read
the input columns they need, return the new column. The orchestrator
joins. The copy is unnecessary because nobody mutates."

### `MarketPrices`/`InterestRates`/`Benchmarks` reinventing `merge_asof`

```python
# pipeline/models.py:191-225
class MarketPrices:
    def __init__(self, df, *, build_dict=True):
        self.prices_df = df.copy()
        self._prices_dict: dict[tuple[date, str, str], tuple[float, float, float]] = {}
        if build_dict:
            self.build_dict()

    def get_close(self, trade_date, market, symbol):
        price_data = self._prices_dict.get((trade_date, market, symbol))
        return None if not price_data or pd.isna(price_data[0]) else price_data[0]
    ...
```

The dict is rebuilt every time the data changes; lookup is O(1) but
the cost of rebuilding the dict from a 1M-row DataFrame is borne every
time. Either:
- keep the dict and treat it as an index (build once, mutate alongside
  the DataFrame), or
- drop the dict and use `df.set_index([...]).loc[...]`, or
- if the access is "give me the value as of this date for this
  symbol", `merge_asof` is exactly the join.

The current shape is the worst of both: a defensive `df.copy()`, a
hand-rolled dict, and a `get_close`/`get_vwap`/`get_borrow_rate`
trio that all look up the same key three times when the simulation
loop wants all three.

**ADAPT.** The pattern (typed lookup class wrapping a DataFrame) is
fine; the implementation is repetitive. Generalise to one
`KeyedFrame[K, V]` class, or replace with `set_index`.

## Recommendations for the new guideline

1. *Choose the layer per stage.* Ingest in dlt; reshape in ibis at
   the boundary; transform in dbt; query the warehouse with parametrised
   SQL or ibis; do rolling math per symbol in pandas; orchestrate
   in Dagster.
2. *Never f-string SQL.* Use the connector's SQL composition API
   (psycopg.sql, ibis, sqlalchemy bindparams). Same rule even when
   the value "happens to be safe today".
3. *Dagster components / assets are thin.* The asset body is a one-line
   delegation to a function in a library package. The library function
   has no Dagster imports.
4. *One schedule, one durability concern.* Schedule the things that
   lose data on a miss; everything else is "materialize on demand".
5. *Indicators are pure: input columns -> output column.* No mutation
   of the input DataFrame; no defensive `.copy()` in the orchestrator.
6. *Wrap DataFrames into typed lookup classes if you must, but generalise
   the wrapper.* Six near-identical "wrap a frame, build a dict, expose
   `get_X` methods" classes are five too many.
