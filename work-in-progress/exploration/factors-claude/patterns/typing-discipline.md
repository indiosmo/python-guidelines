# Typing discipline

## What factors2 does well

### Pydantic-discriminated-union as the primary sum type

The codebase consistently models "this thing has N implementations" as
a tagged union of Pydantic models, with a single Pydantic field acting
as the discriminator. The recipe:

```python
# pipeline/scoring_strategies/binary.py:11-14
class Binary(BaseModel):
    strategy: Literal["binary"] = Field(default="binary", frozen=True)
    nulls: Literal["zero", "negative"]
    percentile: float = Field(default=1 / 3, gt=0, le=0.5)

# pipeline/scoring_strategies/__init__.py:9
Strategy = SumRanks | Binary | GaussianRankScore | LinearRankScore | ZScore

# pipeline/scoring_strategy.py:14-16
class ScoringStrategy(BaseModel):
    strategy: Annotated[Strategy, Field(discriminator="strategy")]
    untie: UntieStrategy
```

This is the Python analogue of the C++ `lib::match` + sum-type pattern
and it lets Pydantic both serialise and validate the choice from a
config dict. Same shape repeats in:

- indicators: `pyfactors/indicators/__init__.py:43-85`, `indicators/computed_indicator.py:12-14`
- scoring: `pyfactors/pipeline/scoring_strategies/__init__.py:9`
- signal: `pyfactors/pipeline/signal_strategies/__init__.py` (use `Strategy` in `signal_strategy.py:25`)
- weighting, portfolio, untie, restriction, liquidation - same shape.

**PROMOTE.** This is the right Python idiom for "open at config time,
closed at runtime" subsystems. Guideline should call it out by name:
*"Annotated Pydantic discriminated union"*.

### `match` + `assert_never` over Literal fields

```python
# indicators/operations/interval_indicator.py:30-36
def _compute_column(self, grouped: SeriesGroupBy) -> pd.Series:
    match self.interval_method:
        case "change":
            return grouped.diff(self.interval)
        case "growth":
            return grouped.pct_change(self.interval, fill_method=None) * 100
        case _:
            assert_never(self.interval_method)
```

`self.interval_method: Literal["change", "growth"]` plus `assert_never`
gives a type-checker error the moment a new variant is added without a
matching arm. Used in `binary.py:29-35`, `z_score.py:28-35`,
`interval_indicator.py:30-36`.

**PROMOTE.** This is the canonical exhaustive-match pattern Python 3.13
gives us, and the guideline should require it whenever a `Literal` is
matched on.

### Pydantic Field constraints as the primary precondition language

```python
# indicators/technical/linear_regression_slope.py:38-40
window: int = Field(ge=2)
offset: int = Field(ge=0, default=0)
short_term_reversal_period: int = Field(ge=0, default=0)
```

Constraints that live in the type are checked once at construction;
the function body never has to repeat them. `model_validator(mode="after")`
covers the cross-field cases (`linear_regression_slope.py:42-52` rejects
`short_term_reversal_period >= window`).

**PROMOTE.** "Parse don't validate" in Pydantic-flavoured Python.

### pandera DataFrameModel for dataframe-shaped boundaries

`pyfactors/database/table_models.py` and `pipeline/models.py:28-91` lift
pandas DataFrames into named, validated types (`UniverseTable`,
`Scores`, `Signals`, `WeightedSignals`). Downstream functions declare
`pa.typing.DataFrame[Scores]` in their signatures. The shape lets the
type checker carry intent across function boundaries, even if pandera
itself can only enforce it at the `.validate()` call.

**PROMOTE** for any DataFrame that crosses a function boundary.

## What factors2 does poorly

### A 40-arm union without a discriminator

```python
# indicators/functions.py:43-85
Function = (
    AbnormalVolume
    | AmihudIlliquidity
    | AnnualizedDownsideVolatility
    | ... 40 more ...
)
```

Each variant has its own `function_name: Literal["..."]`, but the
union is used directly (not as `Annotated[..., Field(discriminator=...)]`).
The wrapper then probes structure at runtime:

```python
# indicators/computed_indicator.py:28-43
def deps(self) -> list["ComputedIndicator"]:
    if hasattr(self.function, "deps"):
        deps = self.function.deps()  # type: ignore
        ...

def compute_rolling(self, df: pd.DataFrame, reference_data: ReferenceData) -> pd.DataFrame:
    if len(inspect.signature(self.function.compute_rolling).parameters) > 1:
        return self.function.compute_rolling(df, reference_data)  # type: ignore
    else:
        return self.function.compute_rolling(df)  # type: ignore
```

This is the failure mode of an unstructured union: the wrapper has to
use `hasattr` + `inspect` + `# type: ignore` to recover the structure
the type system already had. Three smells in one function:

- `hasattr(self.function, "deps")` - asking "are you a member of the
  subset that has dependencies?" at runtime, when this could be a
  protocol.
- `inspect.signature(...).parameters` - asking "did your subclass
  override the unary or the binary form?" at runtime, when *one*
  signature for the whole family would erase the question.
- `# type: ignore` three times in a row - the canonical sign that the
  declared type and the actual contract no longer agree.

**AVOID.** Guideline: if you need `hasattr` on a member of a typed
union, either (a) give all members the same method shape (a Protocol),
or (b) give the union a real discriminator so the wrapper can `match`
on it.

### Stringly-typed enum values that don't use `match` for dispatch

```python
# core/__init__.py:1-18
class SeriesPeriod(str, Enum):
    DAILY = "daily"
    WEEKLY = "weekly"
    MONTHLY = "monthly"

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

Two problems. First, the chained `if/elif/else: raise` is exactly
the pattern `match` + `assert_never` exists to remove; the trailing
`raise` will never fire on a correct call site and just defers the
type-checker's job to runtime. Second, the project's own
`guidelines.md:60-90` argues for the right pattern (carry data on
the enum, dispatch via `match`) and then doesn't follow its own
advice here.

Cleaner:
```python
class SeriesPeriod(StrEnum):
    DAILY = "daily"
    WEEKLY = "weekly"
    MONTHLY = "monthly"

_ANNUALIZATION: dict[SeriesPeriod, int] = {
    SeriesPeriod.DAILY: 252,
    SeriesPeriod.WEEKLY: 52,
    SeriesPeriod.MONTHLY: 12,
}

@property
def annualization_factor(self) -> int:
    return _ANNUALIZATION[self]
```

or use `match` + `assert_never` to keep the same exhaustiveness check
the rest of the codebase uses.

**AVOID** the trailing `raise` on a closed enum.

### `cast(...)` in tests for "construct an empty DataFrame of this schema"

```python
# tests/pyfactors/backtest/util.py:78-81
signals = cast(
    DataFrame[WeightedSignals],
    pd.DataFrame(columns=[...]),
)
```

The `cast` lies to the type checker; if the columns ever drift from
the `WeightedSignals` schema, no one notices. The cleaner move is to
write a `WeightedSignals.empty()` helper that returns an empty,
validated frame. (Same flavour of issue: `dataset.py:126` `return cls(**validated_data)  # type: ignore[arg-type]`.)

**ADAPT** - cast is occasionally fine at SDK boundaries, but never to
paper over a missing constructor.

### typing escape valves audit

Across `src/`:
- `# type: ignore` - 26 occurrences. Mostly clustered around
  ibis fluent calls (`# type: ignore` on `.re_extract`, `.notnull`,
  `.endswith`, `.mutate(...)`), where the ibis stubs are weak; a
  handful are genuine type laundering (the `function.deps()` case
  above, the `Dataset.load` `**validated_data` case).
- `from typing import Any` - 9 occurrences. Most are SDK boundaries
  (httpx response JSON, ibis tables, Pydantic before-validators).
- `cast(...)` - 3 occurrences, mostly to massage empty DataFrames or
  inputs to dataclass `replace`.
- `TYPE_CHECKING` - **0 occurrences.** Notable - the codebase has no
  type-only imports. Either runtime imports are cheap enough (likely)
  or the team has not had a good reason to reach for it yet. Worth
  calling out: the guideline can say "we have not needed
  `TYPE_CHECKING` so far".
- `NewType` / `ParamSpec` / `Protocol(` - **0 occurrences.** No phantom
  types, no callable-shape generics, no structural protocols. Domain
  modelling is done with Pydantic + Literal + Enum instead.
- PEP 695 `type X = ...` aliases - **0 occurrences.** All aliases are
  the legacy `Foo = A | B | C` form. The new guideline can prefer PEP
  695 form (`type Foo = A | B | C`) for new code.

**Heuristic to lift into the guideline:**
"`# type: ignore` is acceptable when the upstream library's stubs are
the bug, but never when *your own* model needs it. If your own code
needs `# type: ignore`, the type is wrong, not the usage."

## Recommendations for the new guideline

1. Discriminated Pydantic unions are the default for plugin-shaped
   subsystems. Each variant carries a `Literal["x"]` tag in a frozen
   default field; the parent uses `Annotated[Union, Field(discriminator="tag")]`.
2. `match` + `assert_never` is the only correct way to dispatch on a
   closed `Literal` or `Enum`. Trailing `raise ValueError("Unknown ...")`
   is an anti-pattern that bypasses static exhaustiveness.
3. Pandera `DataFrameModel` types are required at any function boundary
   that takes or returns a DataFrame. Inside the function, regular
   `pd.DataFrame` is fine.
4. `# type: ignore` is allowed only over upstream library calls (ibis,
   pandera, dagster Resolver). Over your own model -> rewrite the model.
5. `cast(...)` is allowed at SDK boundaries to recover dynamism we
   know is correct. Banned in tests; write a constructor instead.
6. No code change needed for `TYPE_CHECKING`, `NewType`, `Protocol`,
   `ParamSpec` - reach for them only when a concrete problem demands
   them. Python is gradually typed; over-typing is its own anti-pattern.
