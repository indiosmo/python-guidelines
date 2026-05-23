# Antipatterns Surfaced from factors2

Research notes for the python-guidelines project. Antipatterns identified by reading the factors2 codebase at `/home/msi/python_workspace/factors2/`, organized by category. Each entry includes the pattern, why it matters, and the idiomatic alternative. Entries marked [CODEBASE] were found in factors2; entries marked [RESEARCH] come from broader modern-Python literature.

Source files reviewed span `src/pyfactors/`, `src/etl_factors/`, `src/dashboards/`, `tests/`, and configuration files.

---

## Architectural

### God module with mixed domain models

`src/pyfactors/pipeline/models.py` (~360 lines) holds TradeDates, InterestRates, Benchmarks, MarketPrices, QuarterlyIndicators, SecurityList, Blacklist, Order, Position, PositionBook, and ReferenceData in one file. These serve different pipeline stages and have no shared base.

**Why it matters:** Import overhead grows, single-file diffs are noisy, testing one model forces loading all of them. When two models evolve independently (e.g. Position vs InterestRates), merge conflicts appear for unrelated changes.

**Alternative:** One module per concept or per pipeline stage: `interest_rates.py`, `position.py`, `reference_data.py`. Keep a `models/__init__.py` that re-exports for backward compatibility during migration.

### God function mixing orchestration and computation

`src/pyfactors/backtest/server.py:57-134` -- `run_simulation` handles lookback window computation, portfolio assembly, scoring, Ray remote dispatch, and result collection in ~80 lines.

**Why it matters:** Untestable without standing up Ray. Each concern has different failure modes, making diagnosis slow.

**Alternative:** Separate pure functions (`compute_lookback`, `build_portfolio_config`) from the orchestration shell that wires them and submits to Ray.

### Massive union type as registry

`src/pyfactors/indicators/functions.py:43-85` defines `Function` as a union of 40+ indicator classes:

```python
Function = (
    AbnormalVolume
    | AmihudIlliquidity
    | AnnualizedDownsideVolatility
    | ...  # 40 more
)
```

**Why it matters:** Adding an indicator means editing this file. Forgetting to add it is a silent bug -- no indicator, no error. The union also slows type-checker passes.

**Alternative:** A registry pattern with auto-discovery:

```python
INDICATORS: dict[str, type[IndicatorFunction]] = {
    cls.__name__: cls for cls in IndicatorFunction.__subclasses__()
}
```

Or explicit registration via a decorator, which keeps control while avoiding the central file.

---

## Data Modeling

### Stringly-typed signal values

`src/pyfactors/pipeline/models.py:78` uses string literals for signal types validated only at the Pandera schema level:

```python
signal: pa.typing.Series[pa.Category] = pa.Field(
    isin=["long", "neutral", "short"], coerce=True
)
```

**Why it matters:** Typos in business logic ("Long" vs "long") pass type-checking and only fail at runtime. Refactoring signal names requires grep instead of IDE rename.

**Alternative:** A `SignalType(str, Enum)` propagated through the pipeline. Pandera can validate against enum members.

### Overly broad dict types for structured config

`src/pyfactors/backtest/reports/config.py:11-13`:

```python
class ChartConfig(BaseModel):
    trace: dict[str, Any] = Field(default_factory=dict)
    layout: dict[str, Any] = Field(default_factory=dict)
```

**Why it matters:** No IDE autocompletion, no validation of Plotly trace/layout keys, runtime KeyError instead of static error.

**Alternative:** TypedDict or Pydantic model mirroring the subset of Plotly schema the project actually uses.

### Frozen dataclass that mutates its fields

`src/pyfactors/pipeline/models.py:267-270`:

```python
@dataclass(frozen=True)
class ReferenceData:
    ...
    def build_dicts(self) -> None:
        self.interest_rates.build_dict()  # type: ignore
```

**Why it matters:** `frozen=True` signals immutability. Circumventing it with `# type: ignore` breaks the contract that callers rely on, and silences mypy for the exact line where a bug would surface.

**Alternative:** Build the dicts in `__post_init__`, or use a classmethod factory that returns a fully initialized instance.

### Redundant None guard on dict.get

`src/pyfactors/pipeline/models.py:150-151, 176-177, 201-203`:

```python
def get_rate(self, trade_date: date, symbol: str) -> float | None:
    data = self.dict.get((trade_date, symbol))
    return data if data else None
```

**Why it matters:** `dict.get` already returns `None` when the key is missing. The conditional also incorrectly converts a legitimate `0.0` value to `None` because `0.0` is falsy.

**Alternative:** `return self.dict.get((trade_date, symbol))`

### Implicit coercion hiding data quality issues

Multiple Pandera models use `coerce=True` without documenting what coercions are expected:

```python
score: float = pa.Field(nullable=True, coerce=True)
```

**Why it matters:** Silent coercion masks upstream data problems. A string `"NaN"` or `"inf"` coerced to float can propagate through the pipeline without any validation error.

**Alternative:** Validate data types at ingestion boundaries; use `coerce=False` in domain models and fix upstream producers.

---

## Error Handling

### Bare `except Exception` with toast-and-forget

`src/pyfactors/portfolio_management/app.py:51-58, 86-92`:

```python
try:
    df = pd.read_excel(st.session_state.uploaded_file)
except Exception as e:
    st.toast(str(e))
    return
```

**Why it matters:** Catches `MemoryError`, `KeyboardInterrupt` (in some contexts), and everything else. No logging, so production failures are invisible. The toast shows the raw exception message, which may be unhelpful or leak internals.

**Alternative:** Catch the specific failures (`ValueError`, `FileNotFoundError`), log with `exc_info=True`, and show a user-appropriate message.

### Over-broad try blocks

`src/pyfactors/database/dataset.py:178-180` wraps ~7 distinct operations (proxy setup, HTTP fetch, schema validation, DuckDB load) in a single try/except. A DNS failure and a schema mismatch produce the same generic error message.

**Alternative:** Wrap each logical step individually, or use a pipeline pattern where each step can report its own failure context.

---

## Security

### SQL construction via f-string interpolation

`src/pyfactors/database/database_queries.py:21-26, 84-90, 116, 142-148`:

```python
where_conditions.append(f"trade_date >= '{start_date}'::DATE")
query = f"""SELECT * FROM market_data.daily_rollup
    WHERE quarter_date is not null {where_clause}"""
df = duckdb.sql(query).df()
```

**Why it matters:** Classic SQL injection surface. Even if inputs are trusted today, nothing prevents a future caller from passing unsanitized strings. DuckDB supports parameterized queries.

**Alternative:**

```python
df = duckdb.sql(
    "SELECT * FROM market_data.daily_rollup WHERE trade_date >= ?",
    [start_date]
).df()
```

### Credentials in repository-tracked .env file

`/home/msi/python_workspace/factors2/.env` contains `PGPASSWORD=factorsro`. The `.gitignore` may exclude it, but the file exists in the working tree alongside `.env.example`.

**Alternative:** Use a secrets manager or OS-level environment variables. Never store passwords in files that might be committed.

---

## Performance

### iterrows() for object construction

`src/pyfactors/pipeline/models.py:304-319`:

```python
for _, row in positions_df.iterrows():
    Position(market=row["market"], symbol=row["symbol"], ...)
```

**Why it matters:** `iterrows()` is the slowest pandas iteration method. Each call boxes every cell value into a Python object and constructs a Series per row.

**Alternative:** `positions_df.to_dict(orient='records')` produces plain dicts, which can be unpacked directly into Position constructors.

### Deep copy in hot loop

`src/pyfactors/backtest/simulation.py:251`:

```python
book = self._book.model_copy(deep=True)
```

Called on every simulation step (~252 times per year of daily data). Deep copy traverses every nested object.

**Alternative:** If Position objects are treated as immutable after creation, a shallow copy is sufficient. Otherwise, snapshot only the fields that change.

### Dict rebuild without caching

`src/pyfactors/pipeline/models.py` -- `InterestRates.build_dict()`, `Benchmarks.build_dict()`, `MarketPrices.build_dict()` each rebuild a lookup dict from a DataFrame. Nothing prevents calling `build_dict()` multiple times.

**Alternative:** Build once in `__post_init__` or `__init__`, or guard with an `_initialized` flag.

---

## API Design

### Boolean trap parameter

`src/pyfactors/backtest/simulation.py:119`:

```python
def execute_order(self, ..., liquidation: bool = False) -> Trade:
```

At the call site, `execute_order(..., True)` is opaque.

**Alternative:** An enum (`ExecutionContext.LIQUIDATION`) or a separate `liquidate_position` method.

### Two-phase initialization with order dependency

`src/pyfactors/database/postgres_settings.py:23-35`:

```python
def __init__(self, *, env: bool = False, args: Namespace | None = None) -> None:
    if env:
        self.from_env()
    if args:
        self.from_args(args)
```

Callers must know that args overrides env (because it runs second). If neither flag is set, the object is partially initialized.

**Alternative:** Classmethods (`PostgresSettings.from_env()`, `PostgresSettings.from_args(args)`) that return fully constructed instances.

---

## Logging and Instrumentation

### print() in production code

`src/pyfactors/backtest/server.py:44, 102-132` and `src/pyfactors/backtest/simulation.py:328` use `print()` for timing and status output.

**Why it matters:** No log level control, no timestamps, no structured fields, clutters test output, mixes with stdout data.

**Alternative:** `logging.getLogger(__name__).info(...)` with structured context.

### Ad-hoc timing with timeit scattered through code

`src/pyfactors/backtest/server.py:37-45`:

```python
t0 = timeit.default_timer()
...
print("scores time: ", timeit.default_timer() - t0)
```

**Alternative:** A `@timed` decorator or context manager that logs elapsed time, and can be disabled in production.

---

## Type Safety

### Excessive type: ignore comments

`src/pyfactors/indicators/computed_indicator.py:30,41,43` and `src/pyfactors/pipeline/models.py:48` suppress type errors rather than fixing the types. Each `# type: ignore` is a hole in the type safety net.

**Alternative:** Fix the underlying type signatures. If the indicator function hierarchy needs polymorphism, use a Protocol or TypeVar to express it.

### Pydantic v1 Config class in v2 codebase

`src/pyfactors/pipeline/models.py:48`:

```python
class RestrictedSymbols(pa.DataFrameModel):
    class Config:  # type: ignore
        strict = True
```

**Why it matters:** Pydantic v2 uses `model_config = ConfigDict(...)`. The v1 style is a migration artifact that may behave differently or be deprecated. (Note: this is Pandera's DataFrameModel, which uses its own `Config` class -- verify whether Pandera mirrors Pydantic's migration path.)

---

## Testing

### Session-scoped autouse fixture with global side effects

`tests/conftest.py:6-10`:

```python
@pytest.fixture(scope="session", autouse=True)
def enable_copy_on_write():
    pd.options.mode.copy_on_write = True
    yield
```

**Why it matters:** Every test in the suite runs under this setting. A test that fails only without copy-on-write will pass in CI but break when copy-on-write becomes default (pandas 3.0). The fixture doesn't restore the original value.

**Alternative:** Set copy-on-write via `pyproject.toml` pandas config if available, or use a fixture that saves and restores the original setting.

---

## Constants and Magic Numbers

### Hardcoded trading-day count

`src/pyfactors/backtest/simulation.py:59`:

```python
daily_rate = (1 + rate_pct / 100.0) ** (1 / 252.0) - 1
```

252 appears in multiple files without a named constant. The codebase has `SeriesPeriod.DAILY.annualization_factor` returning 252, but it is not used here.

**Alternative:** Use the existing `SeriesPeriod.DAILY.annualization_factor` or define `TRADING_DAYS_PER_YEAR = 252` in a shared constants module and reference it everywhere.

---

## Modern Python Missed Opportunities

### Enum exhaustiveness via else/raise instead of match

`src/pyfactors/core/__init__.py:10-18`:

```python
@property
def annualization_factor(self) -> int:
    if self == SeriesPeriod.DAILY:
        return 252
    elif self == SeriesPeriod.WEEKLY:
        return 52
    elif self == SeriesPeriod.MONTHLY:
        return 12
    else:
        raise ValueError(f"Unknown period: {self}")
```

The else branch is dead code if the enum is complete, but it signals that the author worried about future variants. Python 3.10+ `match` with type narrowing makes exhaustiveness visible:

```python
match self:
    case SeriesPeriod.DAILY:
        return 252
    case SeriesPeriod.WEEKLY:
        return 52
    case SeriesPeriod.MONTHLY:
        return 12
```

Or use a dict mapping, which is naturally exhaustive (KeyError on miss) and avoids branching entirely.

---

*Section below to be populated with findings from online research on modern Python antipatterns not covered by ruff/linters.*

## Research-Based Antipatterns (Beyond Linter Coverage)

(pending)
