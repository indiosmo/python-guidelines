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

## Research-Based Antipatterns (Beyond Linter Coverage)

Sourced from Python community blogs, PyCon/EuroPython talks, and practitioner articles (2024-2026). Each entry explains why linters cannot catch the problem.

---

### Dataclass `__post_init__` abuse

Using `__post_init__` for complex initialization with partially-initialized fields (`field = None` computed later) instead of explicit factory methods.

```python
# antipattern
@dataclass
class MyTimeKeeper:
    tz: datetime.timezone
    ts: datetime.datetime = None  # partially initialized

    def __post_init__(self):
        self.ts = datetime.datetime.now(self.tz)

# fix: classmethod factory
@dataclass
class MyTimeKeeper:
    ts: datetime.datetime

    @classmethod
    def now(cls, tz: datetime.timezone):
        return cls(datetime.datetime.now(tz=tz))
```

**Why linters miss it:** Whether a field should be a constructor parameter or derived is a design decision.

Source: [Patterns and Antipatterns for Dataclasses - Devin J. Cornell](https://devinjcornell.com/post/dsp0_patterns_for_dataclasses.html)

### Mutable fields in frozen dataclasses (false immutability)

`frozen=True` prevents attribute reassignment but does not prevent mutating container contents.

```python
# antipattern
@dataclass(frozen=True)
class Config:
    allowed_hosts: list[str]

config = Config(allowed_hosts=["a.com"])
config.allowed_hosts.append("evil.com")  # succeeds

# fix: use immutable containers
@dataclass(frozen=True)
class Config:
    allowed_hosts: tuple[str, ...]
```

**Why linters miss it:** mypy enforces attribute assignment but not `.append()` on a mutable container.

Source: [Statically enforcing frozen dataclasses - Redowan](https://rednafi.com/python/statically_enforcing_frozen_dataclasses/)

### Missing `slots=True` on dataclasses

Without `__slots__`, typos in attribute names silently create new dynamic attributes instead of raising `AttributeError`.

```python
# antipattern
@dataclass
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
p.z = 3.0  # silent new attribute, likely a typo

# fix
@dataclass(slots=True)
class Point:
    x: float
    y: float
```

Also saves ~50% memory per instance.

**Why linters miss it:** In dynamic or partially typed code, attribute access on unknown types slips through type checkers.

Source: [Patterns and Antipatterns for Dataclasses - Devin J. Cornell](https://devinjcornell.com/post/dsp0_patterns_for_dataclasses.html)

### Over-engineering with ABCs when Protocol or plain functions suffice

ABC hierarchies force nominal inheritance. When there are few implementations and the interface is small, structural typing (Protocol) decouples modules better.

```python
# antipattern: forced inheritance
class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount: float) -> bool: ...

class StripeGateway(PaymentGateway):  # must inherit
    def charge(self, amount: float) -> bool: ...

# fix: structural typing
class PaymentGateway(Protocol):
    def charge(self, amount: float) -> bool: ...

class StripeGateway:  # no inheritance needed
    def charge(self, amount: float) -> bool: ...
```

**Why linters miss it:** Both approaches are valid Python. The choice is a design judgment about coupling.

Source: [Interfaces: ABC vs. Protocols - Oleg Sinavski](https://sinavski.com/post/1_abc_vs_protocols/)

### Import-time side effects

Executing significant work (database connections, HTTP clients, registry population) at module import time causes circular import failures, makes modules untestable in isolation, and fails silently when instrumentation is not yet configured.

```python
# antipattern
# db.py
connection = psycopg2.connect("dbname=prod")  # runs on import

# fix: lazy initialization
_connection = None

def get_connection():
    global _connection
    if _connection is None:
        _connection = psycopg2.connect("dbname=prod")
    return _connection
```

**Why linters miss it:** Any expression at module scope is legal Python. Whether it constitutes a side effect depends on what it does at runtime.

Source: [Don't run code at import time - Ben Kuhn](https://www.benkuhn.net/importtime/), [Python at Scale: Strict Modules - Instagram Engineering](https://instagram-engineering.com/python-at-scale-strict-modules-c0bb9245c834)

---

### Over-mocking: testing implementation details

Mocking internal methods and asserting they were called, rather than testing the output. Tests break when code is refactored even though behavior is unchanged.

```python
# antipattern
def test_get_full_name():
    user = User()
    user.get_first_name = Mock(return_value="Jane")
    user.get_last_name = Mock(return_value="Smith")
    result = user.get_full_name()
    user.get_first_name.assert_called_once()  # testing internals

# fix: test the contract
def test_get_full_name():
    user = User(first_name="Jane", last_name="Smith")
    assert user.get_full_name() == "Jane Smith"
```

**Why linters miss it:** The mock code is valid. Whether a test asserts behavior vs. implementation is a design judgment.

Source: [Common Mocking Problems - Pytest with Eric](https://pytest-with-eric.com/mocking/pytest-common-mocking-problems/)

### Deep mock chains (mocks returning mocks)

Chaining `.return_value` across multiple layers creates tests that mirror the internal call graph rather than testing behavior.

```python
# antipattern
mock_client = Mock()
mock_data = Mock()
mock_data.get_temperature.return_value = 72
mock_client.fetch_weather.return_value = mock_data

# fix: use lightweight fakes
class FakeWeatherClient:
    def fetch_weather(self, location):
        return WeatherData(temperature=72, humidity=55)
```

**Why linters miss it:** Mock chains are valid Python. The decision between mocks and fakes is architectural.

Source: [Common Mocking Problems - Pytest with Eric](https://pytest-with-eric.com/mocking/pytest-common-mocking-problems/)

### Mock signature violations (not using autospec)

Regular `Mock()` objects accept any arguments regardless of the real object's signature. Tests pass with invalid call signatures.

```python
# antipattern
mock_processor = Mock()
mock_processor.process_payment(100, "USD", "EXTRA_ARG")  # real method takes 2 args
# passes

# fix
mock_processor = create_autospec(PaymentProcessor)
mock_processor.process_payment(100, "USD", "EXTRA_ARG")  # TypeError at test time
```

**Why linters miss it:** Mock is a dynamic proxy. Type checkers see `Mock` type, not the original interface.

Source: [Common Mocking Problems - Pytest with Eric](https://pytest-with-eric.com/mocking/pytest-common-mocking-problems/)

### Snapshot/approval testing without review discipline

Snapshot tests capture output on first run. When behavior changes, developers run `--update-snapshot` without reviewing what changed, turning tests into rubber stamps.

**Why linters miss it:** The test code is valid. Whether the developer reviewed the snapshot diff is a process problem.

Source: [Snapshot Testing: A New Era of Reliability - EuroPython 2025](https://ep2025.europython.eu/session/snapshot-testing-a-new-era-of-reliability/)

---

### Blocking the event loop with sync calls in async functions

Calling synchronous I/O (`requests.get`, `time.sleep`, `open().read()`) inside `async def`. The `async` keyword does not make the body non-blocking.

```python
# antipattern
async def fetch_data():
    response = requests.get("https://api.example.com/data")  # blocks event loop

# fix
async def fetch_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
```

**Why linters miss it:** `requests.get()` in an async function is syntactically valid. Whether a call blocks the event loop is a runtime property. The tool [BlockBuster](https://dev.to/cbornet/introducing-blockbuster-is-my-asyncio-event-loop-blocked-3487) was created specifically to detect this at runtime.

Source: [BlockBuster - DEV Community](https://dev.to/cbornet/introducing-blockbuster-is-my-asyncio-event-loop-blocked-3487)

### `asyncio.gather` with `return_exceptions=True` swallowing errors

Exceptions are returned as values in the results list. If the caller doesn't inspect for exception instances, errors are silently lost.

```python
# antipattern
results = await asyncio.gather(fetch_users(), fetch_orders(), return_exceptions=True)
users, orders = results  # orders might be an Exception object

# fix
results = await asyncio.gather(fetch_users(), fetch_orders(), return_exceptions=True)
errors = [r for r in results if isinstance(r, BaseException)]
if errors:
    raise ExceptionGroup("gather failures", errors)
```

**Why linters miss it:** The code is type-valid. The return type is `list[T | BaseException]`, but unpacking assumes all elements are `T`.

Source: [Asyncio gather() Handle Exceptions - SuperFastPython](https://superfastpython.com/asyncio-gather-exception/)

### `asyncio.to_thread` for CPU-bound work

`asyncio.to_thread` runs code in a thread pool, which does not bypass the GIL. CPU-bound Python code in a thread pool still contends with the event loop thread.

```python
# antipattern
result = await asyncio.to_thread(heavy_computation, data)  # GIL contention

# fix: use ProcessPoolExecutor
loop = asyncio.get_running_loop()
with concurrent.futures.ProcessPoolExecutor() as pool:
    result = await loop.run_in_executor(pool, heavy_computation, data)
```

**Why linters miss it:** Whether a function is CPU-bound or I/O-bound is a runtime property.

Source: [BBC Cloudfit: Mixing Synchronous and Asynchronous Code](https://bbc.github.io/cloudfit-public-docs/asyncio/asyncio-part-5.html)

---

### `Any` as a permanent escape hatch

Using `Any` to silence type errors during gradual typing adoption, then never replacing it. `Any` is compatible with every type, so it propagates through the codebase and hides real bugs.

```python
# antipattern
def process_response(data: Any) -> Any:
    return data["result"]["items"]

# fix
class ApiResponse(TypedDict):
    result: ResultPayload

def process_response(data: ApiResponse) -> list[Item]:
    return data["result"]["items"]
```

**Why linters miss it:** `Any` is intentionally designed to be compatible with everything. Mypy's `--strict` flags untyped definitions but not explicit `Any` annotations.

Source: [Resisting the Any type - Jared Khan](https://jaredkhan.com/blog/resist-the-any-type)

### TypeGuard vs TypeIs confusion

`TypeGuard` (Python 3.10) only narrows the `if` branch, leaving the `else` branch with the full union type. `TypeIs` (Python 3.13) narrows both branches correctly.

```python
# antipattern: TypeGuard leaves else branch unnarrowed
def is_number(val: str | int | float) -> TypeGuard[int | float]:
    return isinstance(val, (int, float))

def process(val: str | int | float):
    if is_number(val):
        print(val + 1)      # val is int | float
    else:
        print(val.upper())  # val is STILL str | int | float -- wrong

# fix: TypeIs narrows both branches
def is_number(val: str | int | float) -> TypeIs[int | float]:
    return isinstance(val, (int, float))
```

**Why linters miss it:** TypeGuard's behavior is by-design per PEP 647. The narrowing semantics are documented but subtle.

Source: [TypeIs does what I thought TypeGuard would do - Redowan](https://rednafi.com/python/typeguard-vs-typeis/)

---

### Upper-bound version constraints on library dependencies

Capping dependencies with `<N` or Poetry's `^` operator creates unsolvable dependency conflicts in Python's flat dependency model.

```toml
# antipattern
dependencies = ["requests>=2.28,<3"]  # blocks users from future versions

# fix: floor only
dependencies = ["requests>=2.28"]
```

**Why linters miss it:** Version specifiers are metadata strings in `pyproject.toml`, not Python code.

Source: [Should You Use Upper Bound Version Constraints?](https://iscinumpy.dev/post/bound-version-constraints/)

### Flat layout masking import errors

In flat layout (`mypackage/` at project root), running tests from the project directory finds the local source before the installed package. Tests pass locally but imports fail in production.

```
# antipattern: flat layout
myproject/
    mypackage/
        __init__.py
    tests/
        test_core.py  # imports work because CWD is on sys.path

# fix: src layout
myproject/
    src/
        mypackage/
            __init__.py
    tests/
        test_core.py  # must install package; tests the installed version
```

**Why linters miss it:** The imports are valid Python. Whether they resolve from the local directory or installed package depends on `sys.path` at runtime.

Source: [src layout vs flat layout - Python Packaging User Guide](https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/)

---

### N+1 queries in SQLAlchemy (lazy loading default)

SQLAlchemy defaults to lazy loading for relationships. Iterating over parent objects and accessing a relationship fires one query per parent.

```python
# antipattern: N+1 queries
users = session.query(User).all()
for user in users:
    print(user.orders)  # each access hits the database

# fix: eager load
users = session.query(User).options(selectinload(User.orders)).all()
```

**Why linters miss it:** The lazy-loaded attribute access is valid Python. Whether it triggers a query depends on the ORM's runtime descriptor protocol.

Source: [How to Defeat the N+1 Problem](https://hevalhazalkurt.com/blog/how-to-defeat-the-n1-problem-with-joinedload-selectinload-and-subqueryload/)

### Pandas `.apply(axis=1)` instead of vectorized operations

`.apply()` with `axis=1` runs a Python function per row, bypassing NumPy's vectorized C operations. Orders of magnitude slower.

```python
# antipattern
df["category"] = df.apply(
    lambda row: "top3" if row["country"] in top_3 else "other", axis=1
)

# fix
df["category"] = np.where(df["country"].isin(top_3), "top3", "other")
```

**Why linters miss it:** `.apply()` is a legitimate pandas API. Whether a vectorized alternative exists is a domain-knowledge question.

Source: [4 Pandas Anti-Patterns and How to Fix Them](https://www.aidancooper.co.uk/pandas-anti-patterns/)

### Not using `category` dtype for low-cardinality columns

Keeping string columns as `object` dtype when they have few distinct values wastes memory and slows group-by operations.

```python
# fix
df["age_certification"] = df["age_certification"].astype("category")
```

**Why linters miss it:** Column dtypes are runtime properties. No static tool can analyze data cardinality.

Source: [4 Pandas Anti-Patterns and How to Fix Them](https://www.aidancooper.co.uk/pandas-anti-patterns/)

### Materializing when a generator would suffice

Creating a full list in memory when the data is only iterated once.

```python
# antipattern
has_error = any([validate(item) for item in million_items])

# fix: generator stops at first True
has_error = any(validate(item) for item in million_items)
```

**Why linters miss it:** Some linters flag this inside `any()`/`all()` but not the general case.

Source: [Generator Expressions vs List Comprehensions - Performance Comparison](https://www.usefulfunctions.co.uk/2025/11/07/generator-expressions-vs-list-comprehensions-performance/)

### ORM overhead on read-heavy paths

Using full ORM objects for read-only dashboards where you only need data, not persistence behavior.

```python
# antipattern: full ORM hydration for a read-only query
orders = session.query(Order).filter(Order.status == "shipped").all()

# fix: raw query + dataclass
@dataclass(slots=True)
class OrderSummary:
    id: int
    total: float

rows = session.execute(text("SELECT id, total FROM orders WHERE status = :s"), {"s": "shipped"})
return [OrderSummary(**row._mapping) for row in rows]
```

**Why linters miss it:** ORM usage is valid code. The performance difference is only visible in profiling.

Source: [Raw+DC: The ORM pattern of 2026? - Michael Kennedy](https://mkennedy.codes/posts/raw-dc-the-orm-pattern-of-2026/)

---

### Not using `match/case` for structural decomposition

Chains of `if/elif` with `isinstance` or dictionary key checks where `match/case` (Python 3.10+) would express intent more clearly and extract values in one step.

```python
# antipattern
if event["type"] == "click" and "x" in event and "y" in event:
    handle_click(event["x"], event["y"])

# fix
match event:
    case {"type": "click", "x": x, "y": y}:
        handle_click(x, y)
```

**Why linters miss it:** Both approaches produce identical behavior.

Source: [How to use match-case in Python - DEV Community](https://dev.to/guzmanojero/how-to-use-match-case-in-python-hint-not-for-if-statements-491e)

### Not using `ExceptionGroup` / `except*` for concurrent error handling

When multiple operations can fail independently, raising only the first exception loses the rest.

```python
# antipattern
errors = []
for task in tasks:
    try:
        task.run()
    except Exception as e:
        errors.append(e)
if errors:
    raise errors[0]  # other errors lost

# fix (Python 3.11+)
if errors:
    raise ExceptionGroup("batch processing failed", errors)
```

Callers can handle specific types with `except*`.

**Why linters miss it:** Raising a single exception from a list is valid code. Whether the caller needs all errors is a design requirement.

Source: [PEP 654 - Exception Groups and except*](https://peps.python.org/pep-0654/)

### Still using third-party TOML parsers when `tomllib` is in stdlib

Adding `toml` or `tomli` as dependencies when `tomllib` has been in the standard library since Python 3.11.

```python
# antipattern
import toml
config = toml.load("config.toml")

# fix
import tomllib
with open("config.toml", "rb") as f:
    config = tomllib.load(f)
```

**Why linters miss it:** The import is valid. Whether a stdlib replacement exists is a knowledge question.

Source: [Python 3.11 Preview: TOML and tomllib - Real Python](https://realpython.com/python311-tomllib/)

### Not using Python 3.12+ `type` statement and generic syntax

Still using verbose `TypeVar` + `Generic[T]` when Python 3.12+ provides native syntax.

```python
# antipattern
T = TypeVar("T")
class Stack(Generic[T]):
    def push(self, item: T) -> None: ...

# fix (Python 3.12+)
class Stack[T]:
    def push(self, item: T) -> None: ...
```

**Why linters miss it:** Both forms are valid. Ruff has upgrade rules but they require opt-in and `target-version = "py312"`.

Source: [Modernizing Superseded Typing Features - typing docs](https://typing.python.org/en/latest/guides/modernizing.html)

---

### Raw `os.environ` sprawl

Accessing environment variables with `os.environ`/`os.getenv` scattered across the codebase. Each call site has its own default, type coercion, and failure mode. Missing variables fail late.

```python
# antipattern: scattered across 15 files
db_port = int(os.environ.get("DB_PORT", "5432"))
api_key = os.environ["API_KEY"]  # KeyError at runtime if missing

# fix: centralized, validated at startup
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    db_port: int = 5432
    api_key: str  # required: fails fast at startup
```

**Why linters miss it:** `os.environ` is a valid dict. Whether variables exist or have correct types is a runtime concern.

Source: [Pydantic Settings: A Safer Config Option](https://stuart.mchattie.net/posts/2026/03/07/pydantic-settings-safer-config/)

### `.env` files as a security boundary

Treating `.env` files as secure storage for production secrets. They are plaintext on disk, often accidentally committed.

**Fix:** Use a secrets manager in production. `.env` only for local development, always in `.gitignore`. Use `SecretStr` in Pydantic settings to prevent accidental logging.

**Why linters miss it:** `.env` files are not Python code.

Source: [Stop Putting Secrets in .env Files](https://jonmagic.com/posts/stop-putting-secrets-in-dotenv-files/), [Python Secrets Management - GitGuardian](https://blog.gitguardian.com/how-to-handle-secrets-in-python/)

### Hardcoded defaults that should be explicit failures

Providing default values for configuration that should never have a default in production. The default lets the application start silently misconfigured.

```python
# antipattern
REDIS_URL = os.environ.get("REDIS_URL", "redis://localhost:6379")
# in production, silently connects to nonexistent localhost

# fix: no default for required config
class Settings(BaseSettings):
    redis_url: str  # fails fast at startup if missing
```

**Why linters miss it:** Whether a default is appropriate depends on the deployment context, which is invisible to static analysis.

Source: [Pydantic Settings documentation](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)

---

## Sources

Codebase antipatterns: direct analysis of `/home/msi/python_workspace/factors2/`.

Research antipatterns:
- [Patterns and Antipatterns for Dataclasses - Devin J. Cornell](https://devinjcornell.com/post/dsp0_patterns_for_dataclasses.html)
- [Statically enforcing frozen dataclasses - Redowan](https://rednafi.com/python/statically_enforcing_frozen_dataclasses/)
- [Interfaces: ABC vs. Protocols - Oleg Sinavski](https://sinavski.com/post/1_abc_vs_protocols/)
- [Don't run code at import time - Ben Kuhn](https://www.benkuhn.net/importtime/)
- [Python at Scale: Strict Modules - Instagram Engineering](https://instagram-engineering.com/python-at-scale-strict-modules-c0bb9245c834)
- [Common Mocking Problems - Pytest with Eric](https://pytest-with-eric.com/mocking/pytest-common-mocking-problems/)
- [Snapshot Testing: A New Era of Reliability - EuroPython 2025](https://ep2025.europython.eu/session/snapshot-testing-a-new-era-of-reliability/)
- [BlockBuster - DEV Community](https://dev.to/cbornet/introducing-blockbuster-is-my-asyncio-event-loop-blocked-3487)
- [Asyncio gather() Handle Exceptions - SuperFastPython](https://superfastpython.com/asyncio-gather-exception/)
- [BBC Cloudfit: Mixing Sync and Async Code](https://bbc.github.io/cloudfit-public-docs/asyncio/asyncio-part-5.html)
- [Resisting the Any type - Jared Khan](https://jaredkhan.com/blog/resist-the-any-type)
- [TypeIs does what I thought TypeGuard would do - Redowan](https://rednafi.com/python/typeguard-vs-typeis/)
- [Should You Use Upper Bound Version Constraints?](https://iscinumpy.dev/post/bound-version-constraints/)
- [src layout vs flat layout - Python Packaging User Guide](https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/)
- [How to Defeat the N+1 Problem](https://hevalhazalkurt.com/blog/how-to-defeat-the-n1-problem-with-joinedload-selectinload-and-subqueryload/)
- [4 Pandas Anti-Patterns and How to Fix Them](https://www.aidancooper.co.uk/pandas-anti-patterns/)
- [Raw+DC: The ORM pattern of 2026? - Michael Kennedy](https://mkennedy.codes/posts/raw-dc-the-orm-pattern-of-2026/)
- [Generator Expressions vs List Comprehensions](https://www.usefulfunctions.co.uk/2025/11/07/generator-expressions-vs-list-comprehensions-performance/)
- [PEP 654 - Exception Groups and except*](https://peps.python.org/pep-0654/)
- [Python 3.11 Preview: TOML and tomllib - Real Python](https://realpython.com/python311-tomllib/)
- [Modernizing Superseded Typing Features - typing docs](https://typing.python.org/en/latest/guides/modernizing.html)
- [Stop Putting Secrets in .env Files](https://jonmagic.com/posts/stop-putting-secrets-in-dotenv-files/)
- [Pydantic Settings: A Safer Config Option](https://stuart.mchattie.net/posts/2026/03/07/pydantic-settings-safer-config/)
- [Python Secrets Management - GitGuardian](https://blog.gitguardian.com/how-to-handle-secrets-in-python/)
