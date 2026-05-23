# Pydantic vs Pandera vs dataclasses - which at which boundary

factors2 uses three modelling tools, and (mostly) uses them at
distinct boundaries. Where it's clean, the guideline can copy the
mapping. Where it's mixed, the guideline should rule.

## The mapping factors2 actually uses

| Layer                          | Tool                | Example                                                    |
|--------------------------------|---------------------|------------------------------------------------------------|
| Config / strategy variants     | Pydantic v2         | `Binary`, `ZScore`, `AnnualizedVolatility`, `Order`        |
| Cross-process domain records   | Pydantic v2         | `Position`, `PositionBook`, `Trade`, `SimulationParams`    |
| Plain "passive" results        | `@dataclass`        | `StepResult`, `SimulationResult`, `PipelineConfig`         |
| DataFrame contracts            | pandera             | `UniverseTable`, `Scores`, `Signals`, `WeightedSignals`    |
| HTTP/API serialisation         | Pydantic v2         | `Rate`, `FetchSeriesParams` (with aliases + serialisers)   |
| Cached reference data wrappers | hand-rolled classes | `InterestRates`, `Benchmarks`, `MarketPrices`              |

The last row is the one weak spot - more on that below.

## Pydantic (PROMOTE the patterns observed)

### Strategy classes carry their own discriminator and constraints

```python
# indicators/technical/annualized_volatility.py:11-15
class AnnualizedVolatility(BaseModel):
    function_name: Literal["annualized_volatility"] = Field(default="annualized_volatility", frozen=True)
    window: int = Field(gt=0)
    series_period: SeriesPeriod
    column: Literal["returns"] = "returns"
```

The class doubles as runtime config, type-checked argument, and JSON
serialisation contract. `Field(gt=0)` means the precondition is checked
once at construction, never again inside `compute_rolling`.

### Cross-field invariants via `model_validator(mode="after")`

```python
# indicators/technical/linear_regression_slope.py:42-52
@model_validator(mode="after")
def validate_short_term_reversal_period(self) -> Self:
    if self.short_term_reversal_period >= self.window:
        raise ValueError(...)
    if self.short_term_reversal_period == 1:
        raise ValueError(...)
    return self
```

Same trick used in `PortfolioDefinition` (`pipeline/portfolio_definition.py:28-33`)
to delegate to a free function, which is then individually testable.
Useful pattern: the validator is a thin wrapper around a free function.

### HTTP request/response models with aliases + serialisers

```python
# etl_factors/sources/bcb/api.py:38-65
class Rate(BaseModel):
    trade_date: date = Field(alias="data")
    rate_pct: float = Field(alias="valor")

    @field_validator("trade_date", mode="before")
    @classmethod
    def parse_date(cls, v) -> date | Any:
        if isinstance(v, str):
            return datetime.strptime(v, DATE_FMT).date()
        return v


class FetchSeriesParams(BaseModel):
    series_code: str = Field(description="Series code like '12' for CDI")
    start_date: date = Field(serialization_alias="dataInicial")
    end_date: date | None = Field(default=None, serialization_alias="dataFinal")

    @field_serializer("start_date", "end_date", when_used="always")
    def _serialize_date(self, v: date | None) -> str | None:
        return v.strftime(DATE_FMT) if v else None
```

This is the textbook way to keep upstream wire formats (the BCB API's
`data` / `valor` / `dd/MM/yyyy`) out of the domain model. **PROMOTE.**

## Pandera (PROMOTE the layered pattern)

`pandera` covers what `pandera` can cover (columns, dtypes,
nullability, simple per-row constraints) and the codebase wraps each
schema check with a hand-written cross-row validator. The pattern is
visible in `pipeline/stages.py:62-183`:

```python
def validate_weighted_signals(weighted_signals):
    _validate_weighted_signals_non_empty(weighted_signals)
    _validate_weighted_signals_unique_keys(weighted_signals)
    _validate_weighted_signals_scores(weighted_signals)
    _validate_weighted_signals_close_prices(weighted_signals)
    _validate_weighted_signals_active_weights(weighted_signals)
    _validate_weighted_signals_side_weight_sums(weighted_signals)
    return WeightedSignals.validate(weighted_signals)
```

Each helper is one paragraph, named for the invariant it enforces,
shows a sample of bad rows in the message, and exits early when it
has nothing to do. Tests cover each helper individually
(`tests/pyfactors/pipeline/test_validate_weighted_signals.py`).

**PROMOTE.** The guideline should say explicitly: pandera enforces
the schema, you write the cross-row invariants, and you write them as
named single-purpose predicates that compose into one public validator.

### Use pandera schemas in signatures

```python
# pipeline/stages.py:167-183
def validate_weighted_signals(
    weighted_signals: pa.typing.DataFrame[WeightedSignals],
) -> pa.typing.DataFrame[WeightedSignals]: ...
```

The signature carries intent across the call site even when the
validator itself does the runtime check. **PROMOTE.**

## Dataclasses (PROMOTE - but understand the boundary)

Plain `@dataclass` is used for "I'm a container, I don't validate":

```python
# backtest/models.py:146-165
@dataclass
class StepResult:
    trade_date: date
    book: PositionBook
    trades: list[Trade]
    mark_to_market_nav: float
    ...

@dataclass
class SimulationResult:
    simulation_name: str
    params: SimulationParams
    steps: list[StepResult] = field(default_factory=list)
```

The fields inside are already validated (`book: PositionBook` is a
Pydantic model, `trades: list[Trade]` is a list of Pydantic models),
so the `StepResult` wrapper does not need another validation pass.
A separate free function `validate_step_result(...)` checks the
cross-field invariants.

**PROMOTE.** Use `@dataclass` for aggregates whose components already
validate themselves. Use Pydantic when the aggregate is itself a
trust boundary.

### `@dataclass(frozen=True)` for value-semantics records

```python
# pipeline/models.py:99-122
@dataclass(frozen=True)
class TradeDates:
    dates: list[date]
    prev_date: dict[date, date | None]
    next_date: dict[date, date | None]

    @classmethod
    def from_dataframe(cls, dates_df: pa.typing.DataFrame[DatesTable]) -> Self: ...
```

A frozen dataclass with a `from_dataframe` classmethod is a nice
factory pattern: the dataframe lives at the boundary, the in-memory
object is the typed denormalised form.

## Where the boundary slips (AVOID)

### Hand-rolled "wrap a DataFrame and build a dict" classes

```python
# pipeline/models.py:139-188
class InterestRates:
    def __init__(self, df: pd.DataFrame, *, build_dict=True) -> None:
        self.df = df.copy()
        self.dict: dict[tuple[date, str], float] = {}
        if build_dict:
            self.build_dict()

    def get_rate(self, trade_date: date, symbol: str) -> float | None:
        data = self.dict.get((trade_date, symbol))
        return data if data else None

    def build_dict(self) -> None:
        self.dict = {...}


class Benchmarks:                # near-identical
class MarketPrices:              # near-identical with a 3-tuple value
class QuarterlyIndicators:       # just `df.copy()`
class SecurityList:              # `df.copy()` + json normalisation
class Blacklist:                 # just `df.copy()`
```

Six classes, three of them with the same `__init__(df, *, build_dict)`
+ `build_dict()` shape, all reinventing the same lookup-cache idea.
The `get_rate` / `get_close` methods do `dict.get(...)` and `return
data if data else None`, which collapses `None` and `0.0` to `None`
(real bug, though probably not hit because rates and prices are
positive).

**AVOID.** Either:
- collapse to one generic `KeyedFrame[K, V]` class, or
- give each a Pydantic / dataclass facade and put the dict logic in
  a free function, or
- use pandera + a small free `lookup(df, key) -> V | None`.

### Streamlit handlers as the validation boundary

```python
# portfolio_management/app.py:50-58
def update_position_book():
    try:
        df = st.session_state.position_df.rename(columns={"price": "mtm_price"})
        df["avg_price"] = df["mtm_price"]
        st.session_state.position_book = PositionBook.from_dataframe(df, nav=st.session_state.nav)
    except Exception as e:
        st.toast(str(e))
```

The right thing is for `PositionBook.from_dataframe` to consume a
validated `PositionsTable` (pandera) and for the handler to map
`pa.errors.SchemaError` (and `ValidationError` from Pydantic) into
`st.toast(...)` explicitly, instead of catching `Exception`.

**AVOID** `except Exception` at trust boundaries; **PROMOTE**
catching `pydantic.ValidationError` and `pandera.errors.SchemaError`
by name at exactly the layer where you can translate them into UI
state or HTTP responses.

## Recommendations for the new guideline

1. Pydantic v2 is the default for: strategy/config variants,
   domain records that cross processes (orders, positions, trades),
   HTTP request/response shapes. Constraints live in `Field(...)`;
   cross-field rules live in `model_validator(mode="after")` and
   delegate to a free function so the rule is testable in isolation.
2. Pandera `DataFrameModel` is the default for any DataFrame that
   crosses a function boundary. Column / dtype / nullability live in
   the schema; cross-row invariants live in single-purpose free
   functions, composed into one public `validate_*` entry point.
3. `@dataclass` (and `@dataclass(frozen=True)`) is the default for
   "aggregate of already-validated parts" - results, configs,
   in-memory denormalised forms.
4. One record style per layer. Picking Pydantic for the in-memory
   form *and* then writing a parallel hand-rolled class around the
   same data (the `InterestRates` / `Benchmarks` / `MarketPrices`
   trio) is the failure mode to call out.
5. At the trust boundary (HTTP edge, UI handler, Dagster op edge),
   catch the *named* exception class (`pydantic.ValidationError`,
   `pandera.errors.SchemaError`, `httpx.HTTPStatusError`), translate
   it, and let everything else bubble.
