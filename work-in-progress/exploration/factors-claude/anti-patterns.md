# Anti-patterns observed in factors2

One consolidated list. Each entry: name, citation, why it's a
problem, and what the new guideline should require instead.

## A. f-string SQL composition

**Where**: `pyfactors/database/database_queries.py:17-238` (the
non-psycopg.sql functions).

```python
# database_queries.py:28
where_conditions.append(f"trade_date >= '{start_date}'::DATE")
```

**Why**: this is the canonical SQL-injection shape. The fact that
`start_date: date` happens to be safe today is irrelevant to the rule
- the next caller will pass a string from a UI form. The same file
also contains the correct pattern (`psycopg.sql.SQL(...).format(sql.Literal(...))`
at lines 285-294), so the team knows what to do.

**Guideline**: never f-string SQL. Use the connector's SQL
composition API (`psycopg.sql`, ibis, `sqlalchemy.text` with
bindparams). Enforce via ruff `S608`.

## B. `except Exception` swallowing

**Where**:
- `pyfactors/portfolio_management/app.py:57, 90, 136, 149, 161, 187, 202, 232`
- `etl_factors/sources/brasilapi/fetch_cnpj.py:38`
- `dashboards/app.py:263` (already `# noqa: BLE001 - surface in UI`)
- `pyfactors/database/dataset.py:178` (log-and-re-raise)

```python
# portfolio_management/app.py:50-58
def update_position_book():
    try:
        ...
    except Exception as e:
        st.toast(str(e))
```

**Why**: loses exception class, loses traceback, makes
"validation error from user input" indistinguishable from "DB is
down" indistinguishable from "programmer typo on `st.session_state`".

**Guideline**: `except Exception` is allowed at exactly one place per
call stack - the boundary that maps internal failures to the next
protocol (UI/HTTP/Dagster). Catch by class everywhere else. At the
boundary, log via `logger.exception(...)`, not `str(e)`.

## C. `print(...)` for production logging

**Where**: 35+ occurrences across `pyfactors/`, including
- `pipeline/stages.py:307, 308, 329`
- `backtest/server.py:44, 102, 103, 118, 123, 129, 131, 132`
- `backtest/simulation.py:328-332` (auto-liquidation - an
  operationally important event)

**Why**: no level, no timestamp, no structure, can't be filtered or
rerouted, mingles with stdout.

**Guideline**: no `print` in `src/`. Enforce via ruff `T201`. Use
`loguru.logger.info(...)` with static message + structured kwargs.

## D. `timeit.default_timer()` + `print` for timing

**Where**: `pipeline/stages.py:298, 320, 332`,
`backtest/server.py:38, 84, 105, 112, 127`.

```python
t1 = timeit.default_timer()
...
print("universe time: ", timeit.default_timer() - t1)
```

**Why**: timing data trapped in stdout; can't aggregate, can't
threshold, can't alert. The codebase already has `pyinstrument` and
`pytest-benchmark` in deps for the development case.

**Guideline**: timing goes through a `with timed_block("..."): ...`
context manager that emits one structured `logger.info(...)` on exit.
Profiling goes through `pyinstrument`. Microbenchmarks go in
`benchmarks/`.

## E. Trailing `raise ValueError("Unknown ...")` on a closed enum

**Where**: `pyfactors/core/__init__.py:11-18`.

```python
class SeriesPeriod(str, Enum):
    DAILY = "daily"; WEEKLY = "weekly"; MONTHLY = "monthly"

    @property
    def annualization_factor(self) -> int:
        if self == SeriesPeriod.DAILY: return 252
        elif self == SeriesPeriod.WEEKLY: return 52
        elif self == SeriesPeriod.MONTHLY: return 12
        else: raise ValueError(f"Unknown period: {self}")
```

**Why**: the enum is closed; the trailing `raise` will never fire,
deprives the type checker of an exhaustiveness check, and the
codebase's own `doc/guidelines.md:76-90` argues for the right pattern.

**Guideline**: closed enums + `Literal` types are dispatched with
`match` + `assert_never`, or with a lookup dict. Never with chained
`if/elif/else: raise`.

## F. `hasattr` + `inspect.signature` to recover structure from a
typed union

**Where**: `pyfactors/indicators/computed_indicator.py:28-43`.

```python
def deps(self) -> list["ComputedIndicator"]:
    if hasattr(self.function, "deps"):
        deps = self.function.deps()  # type: ignore
        ...

def compute_rolling(self, df, reference_data):
    if len(inspect.signature(self.function.compute_rolling).parameters) > 1:
        return self.function.compute_rolling(df, reference_data)  # type: ignore
    else:
        return self.function.compute_rolling(df)  # type: ignore
```

**Why**: the 40-arm `Function = A | B | C | ...` union
(`indicators/functions.py:43-85`) has no discriminator, and the
variants don't agree on a method shape. Three `# type: ignore` and
two runtime introspection calls to work around it.

**Guideline**: if you need `hasattr` or `inspect` on a member of a
typed union, either (a) give all variants the same method signature
(a Protocol), or (b) give the union an explicit
`Annotated[..., Field(discriminator=...)]` so a `match` does the
right thing statically.

## G. `cast(...)` in tests to fake a typed empty DataFrame

**Where**: `tests/pyfactors/backtest/util.py:78-81`.

```python
signals = cast(
    DataFrame[WeightedSignals],
    pd.DataFrame(columns=[...]),
)
```

**Why**: the cast lies to the type checker. If `WeightedSignals` ever
adds a column, no one notices.

**Guideline**: no `cast(...)` in tests. Write `WeightedSignals.empty() -> DataFrame[WeightedSignals]`
on the schema instead.

## H. `.copy()` inside the indicator orchestrator loop

**Where**: `pipeline/stages.py:243, 251`.

```python
inputs = universe.copy()
for indicator in indicators:
    if indicator.indicator_column_name not in universe:
        columns.append(indicator.indicator_column_name)
        indicator_df = indicator.compute_rolling(inputs.copy(), reference_data)
        inputs = inputs.merge(indicator_df, ...)
```

**Why**: workaround for indicators that mutate their input
(`linear_regression_slope.py:102-136` writes scratch `_log_close`,
`_pos`, `_y`, `_pos_y`, `_slope_long`, `_slope_short` columns to
`df`). For 1M rows x 40 indicators, the cost is real.

**Guideline**: indicators are pure - read input columns, return only
the new column plus join keys, never mutate the input. The
orchestrator joins. Remove the defensive copy.

## I. Stateful caches that can drift from their backing DataFrame

**Where**: `pyfactors/pipeline/models.py:139-225`
(`InterestRates`, `Benchmarks`, `MarketPrices`).

```python
class InterestRates:
    def __init__(self, df, *, build_dict=True):
        self.df = df.copy()
        self.dict = {}
        if build_dict: self.build_dict()
```

**Why**: both `self.df` and `self.dict` are public; the dict is
derived from the df; nothing prevents a caller from mutating `self.df`
without calling `build_dict()` again. The fact that `ReferenceData.build_dicts()`
exists to rebuild all three caches is the giveaway that this *does*
happen.

**Guideline**: caches are immutable (`@dataclass(frozen=True)` + build
in `__post_init__`) or hidden (`functools.cached_property` so they
rebuild on demand). Never expose both the source and the derived
cache as public attributes.

## J. Hand-rolled `argparse + getenv` settings class

**Where**: `pyfactors/database/postgres_settings.py:1-43`.

```python
@dataclass
class PostgresSettings:
    host: str = ""
    port: int = 5432
    ...
    def __init__(self, *, env: bool = False, args: argparse.Namespace | None = None):
        if env: self.from_env()
        if args: self.from_args(args)
```

**Why**: re-implements what `pydantic-settings` already does (typed,
env-var-aware, CLI-aware settings). The custom `__init__` defeats
the dataclass-generated init.

**Guideline**: use `pydantic-settings.BaseSettings` for config from
env/CLI/file. The combination of typed fields, `env_prefix`, and
nested model loading covers this case in 10 lines.

## K. Commented-out alternative branches

**Where**: `pyfactors/backtest/simulation.py:31`.

```python
if close_price is not None:
    position.mtm_price = close_price
    # raise RuntimeError(f"missing close price for symbol {position.symbol} trade date {trade_date}")
```

**Why**: the comment is the previous behaviour; the new behaviour
(silently carry yesterday's price) is undocumented and untested. A
reader has to choose between "the comment is wrong" and "the code
is wrong".

**Guideline**: commented-out code is a refactoring leftover; delete
it. If the alternative is worth keeping, write down the rule that
made you pick this branch. (Same rule as `~/.claude/CLAUDE.md`'s
"negative documentation" anti-pattern.)

## L. Mixing stdlib `logging` with `loguru`

**Where**: `pyfactors/database/dataset.py:1, 178-180` (stdlib
`logging.error`) alongside `etl_factors/utils/proxy_downloader.py:13`
(loguru `logger.info/error/exception`).

**Why**: two log streams; one is not configured the same way as the
other.

**Guideline**: one logger across the project. loguru is already a
dependency; everything else should redirect to it (`loguru` has a
helper for stdlib `logging` interception).

## M. `logger.error(f"... {str(e)}")` inside `except`

**Where**: `etl_factors/utils/proxy_downloader.py:118-123`,
`pyfactors/database/dataset.py:179`.

**Why**: drops the traceback. `loguru.logger.exception(...)` (inside
`except`) includes it automatically.

**Guideline**: inside an `except` block, `logger.exception(...)`. To
log without re-raising, that's all you need. To re-raise with added
context, `raise NewError(...) from e`. Never log-and-re-raise.

## N. Setting a global pandas mode in `conftest.py`

**Where**: `tests/conftest.py:6-10` (CoW on for tests),
`pyfactors/portfolio_management/app.py:239` (CoW on for the app).

**Why**: behaviour diverges between the test session and any other
importer of `pyfactors`. The library secretly depends on CoW being
on; an importer who doesn't enable it gets different behaviour.

**Guideline**: global library modes go in `pyfactors/__init__.py`
with a comment documenting the contract, or the library is rewritten
to not depend on the mode. Never set it only in `conftest.py`.

## O. `f"{base_url}?{urlencode(query_params)}"` in `logger.info`

**Where**: `etl_factors/utils/proxy_downloader.py:111, 117`.

```python
logger.info(f"Downloading: {url_with_params}", proxy=self._pick_proxy(idx))
```

**Why**: the variable URL is baked into the message, so a log
aggregator can no longer group by message. Kwargs are the structured
slot; the message should be static.

**Guideline**: static message, variable kwargs. Always.

## P. Two competing exception classes for the same kind of failure

**Where**: `backtest/simulation.py:203` raises `ValueError("missing rate for trade_date ...")`;
`backtest/simulation.py:226, 363-365, 377` raise `RuntimeError("...missing benchmark...", "no price available")`.

**Why**: same systemic cause ("upstream data is missing"), two
exception classes. Callers cannot catch one to handle both.

**Guideline**: define domain exception classes for systemic failure
modes (`MissingMarketData`, `MissingBenchmarkData`, `MissingRate`).
`ValueError` is for caller-supplied bad input; `RuntimeError` is for
genuine programmer-error states; everything else is a named domain
exception.

## Q. `pyfakefs`, `pyinstrument`, `pydeps` listed but unused

**Where**: `pyproject.toml:71, 72, 75` declares them; no use in
`src/` or `tests/`; no `make` target uses them.

**Why**: dev deps that nobody invokes erode the signal of the
dependency list. Worse, they're declared at the version pinning level,
which means the dep resolver still has to satisfy them.

**Guideline**: every dev dep has a documented invocation (a `make`
target, a `doc/` file, a test that uses it). If it doesn't, remove it.

## R. Empty `__init__.py` exports / `__all__` via `get_args`

**Where**: `pyfactors/pipeline/scoring_strategies/__init__.py:11`.

```python
__all__ = [cls.__name__ for cls in get_args(Strategy)]  # pyright: ignore[reportUnsupportedDunderAll]
```

**Why**: technically clever but `__all__` should be a static literal
so static analysers can use it. The pyright ignore is the giveaway.

**Guideline**: `__all__` is a static literal list of strings. The
discriminated union variant list is what generates it conceptually,
but the file writes it out.

## S. `time.sleep` rate limiting next to a proper aiolimiter implementation

**Where**: `etl_factors/sources/brasilapi/fetch_cnpj.py:11-42` does
`time.sleep(1/MAX_PER_SECOND)`; `etl_factors/utils/proxy_downloader.py`
does it correctly with `aiolimiter`.

**Why**: two implementations of the same need, the worse one written
later (or earlier - either way, the team has the right shape and
didn't reuse it).

**Guideline**: if the right shape exists in the repo, use it.
Otherwise, lift it into a shared module.

## T. Empty placeholder directories

**Where**: `one-pagers/` at the repo root is empty.

**Why**: looks like intent but documents nothing.

**Guideline**: no empty directories committed.
