# PEP 695 `type X = ...` vs legacy `X: TypeAlias = ...`

(No external research run; this is a local note for the
reconciliation pass. Mark as TODO if the parent wants me to dispatch
a research subagent.)

## What factors2 uses

Zero PEP 695 type-alias statements (`grep -rn "^type " src/` returns
nothing). Every alias in the codebase is the legacy form:

```python
# pyfactors/indicators/__init__.py:4
Indicator = CachedIndicator | ComputedIndicator

# pyfactors/database/data_source.py:8
DataSource = str | Dataset

# pyfactors/pipeline/scoring_strategies/__init__.py:9
Strategy = SumRanks | Binary | GaussianRankScore | LinearRankScore | ZScore
```

## What the new guideline should say

For Python 3.13+ (which factors2 already requires), `type X = ...` is
the recommended form. It:
- creates an alias whose binding is *lazy* (no forward-reference
  trickery needed for self-referential or mutually-recursive aliases)
- carries `TypeAliasType` semantics that mypy/pyright treat as a
  first-class alias rather than "X is just bound to the union"
- shortens `from typing import TypeAlias; X: TypeAlias = Foo | Bar`
  to `type X = Foo | Bar`

For sum-type discriminated unions:

```python
# Recommended
type Strategy = SumRanks | Binary | GaussianRankScore | LinearRankScore | ZScore

class ScoringStrategy(BaseModel):
    strategy: Annotated[Strategy, Field(discriminator="strategy")]
    untie: UntieStrategy
```

`Annotated[Strategy, Field(...)]` resolves through the alias the same
way as before; pyright/mypy both understand the discriminator.

The remaining question: Pydantic discriminated unions specifically
need the alias to be a `Union[...]`-compatible thing at the point of
field resolution. As of pydantic 2.7+, `type X = A | B | C` works in
that slot. Worth pinning if the new guideline standardises on PEP 695.

## Caveat

`type X = ...` is a *statement*; legacy aliases were *assignments*.
If runtime code does anything like `for cls in get_args(Strategy):`
(which factors2 does in `pipeline/scoring_strategies/__init__.py:11`),
you need `typing.get_args(Strategy.__value__)` for the PEP 695 form.
This is a real behaviour difference that's easy to miss.

Recommendation: prefer PEP 695 in new code; if you find yourself
doing reflection on the alias (`get_args`, `get_origin`), the legacy
form is still acceptable, but consider switching to a static
`__all__` literal instead.
