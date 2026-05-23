# Documentation

factors2's `doc/` folder is small (5 files + 1 ADR) but the writing
quality is high enough that the new guideline can lift several
practices directly. The shape:

```
doc/
├── adr/
│   └── 0001-schedule-only-economatica-downloads.md
├── economatica_decisions.md       # source-specific data decisions
├── guidelines.md                  # local coding guidelines (this is the seed of what we're porting)
├── linear_regression_ln.md        # mathematical justification for an indicator
└── setup.md                       # local development setup
```

`one-pagers/` exists at the repo root but was empty when I read it.

## What factors2 does well

### ADR template

`doc/adr/0001-schedule-only-economatica-downloads.md` is a clean ADR:

- Title is the decision, not a topic.
- `Status:` + `Date:`
- `## Context` describes the current state and the constraint, in
  this case the four-tier recoverability classification of every
  asset family.
- `## Decision` is one paragraph; the rest is justification.
- `## Consequences` enumerates the second-order effects (operator
  workflow, daemon dependency, what future similar decisions should
  look like).
- The "alternatives considered and why rejected" paragraph is one
  paragraph, not a separate section - and *that's* what makes the
  ADR readable.

**PROMOTE.** Copy this as the ADR template for the new repo.

### Per-indicator mathematical justification next to the indicator

`doc/linear_regression_ln.md` is a 5-page derivation explaining *why*
linear regression on `ln(P(t))` (vs raw prices, vs log-returns) gives
a scale-invariant slope. It is referenced from the indicator's docstring:

```python
# indicators/technical/linear_regression_slope.py:8
- Log prices: Uses natural logarithm of prices (ln P(t)) rather than raw prices or log-returns.
  As documented in the project's doc/linear_regression_ln.md, this approach ensures...
```

The pairing is the right shape: math derivation in `doc/`, summary +
reference in the docstring. The reader who wants the why has it; the
reader who just wants the code doesn't have to scroll.

**PROMOTE.** *"For any non-obvious algorithmic choice, write the math
in `doc/`, summarise it in 2-3 lines in the docstring, and link from
the docstring to the doc."*

### Source-specific "data decisions" file

`doc/economatica_decisions.md` documents non-obvious data-cleaning
decisions: "ignore symbols with missing CNPJ", "always use last
symbol in chain for the same CNPJ", "do not join consolidated and
nonconsolidated data". Each is one paragraph; each cites concrete
examples (e.g. "ALLL3 -> RUMO3 -> RAIL3"). The file is short and the
decisions are findable.

**PROMOTE.** *"For each upstream data source, keep a
`doc/<source>_decisions.md` that documents the cleaning rules and
the *reason* for each. Find-and-replace later when the rule
changes."*

### Indicator docstrings cite the paper they implement

```python
# indicators/technical/sharpe_ratio.py:1-28
"""
SharpeRatio Indicator

In portfolio analysis and quantitative finance, ... volatility, rewarding strategies ...

Our implementation follows these principles:
- **Returns calculation**: simple percentage changes (Pₜ / Pₜ₋₁ – 1), ...
- **Excess returns**: subtracts either a scalar or time-series risk-free rate ...
- **Unbiased volatility estimate**: uses the sample standard deviation (ddof=1) ...
- **Annualization**: multiplies the raw ratio by √(annualization_factor) ...

References:
- Title: Portfolio Risk in Multiple Frequencies
  Authors: Mustafa U. Torun; Ali N. Akansu; Marco Avellaneda
  Year: 2011
...

Links:
https://ieeexplore.ieee.org/document/5999595
"""
```

Three sections: what it computes, the implementation principles
(each one a footnote on the math choice), and the citations. The
citations include both bibliographic info and durable URLs.

**PROMOTE.** *"Indicator/algorithm docstrings have three sections:
intent, implementation principles (one bullet per non-obvious
choice), references. Citations include title, authors, year, and
URL."*

### Local guideline doc that the team already follows (mostly)

`doc/guidelines.md` is the seed document for this whole port. It
already articulates:

- decimals, not percentages, for stored indicator values
- descriptive names ("`annualized_volatility()`, not `volatility()`")
- type-safe args (closed enums over open ints) with `match`-based
  dispatch
- type-safe returns (`ScalarValue`/`PercentValue` instead of bare
  `float`)
- named over positional args (`*, period`)
- test every expected property + return type
- ISO `yyyy-mm-dd` for dates
- preconditions raise; no defensive fixup
- shim/fork/vendor third-party libraries

That document, plus this exploration, **is** the seed of the new
Python guidelines repo. The bullets above are all promotable.

**PROMOTE everything in that doc.** Some bullets need expansion (the
"shim/fork/vendor" advice could use a decision matrix), some need
softening (vendoring small libraries is fine; vendoring large ones
is a maintenance debt). But the document's altitude is right.

## What factors2 does poorly

### `setup.md` is a checklist of `cp ... example` commands with no rationale

```
## env variables
Copy and edit the sample env:
`cp docker/dagster/.env.example .env`

## dlt setup
Clear dlt state:
`rm -rf ~/.dlt`
...
```

The doc tells you *what* to run but not *why* you'd need to clear
dlt state, or *when* (only on first setup, or after every checkout?).
A first-time reader can copy-paste, but the second-time reader can't
debug.

**ADAPT.** Each command gets a one-line "why" comment. Same shape
as the indicator docstring's "implementation principles" section.

### Code map is missing

There is no `doc/architecture.md` or `doc/codemap.md` showing how
`pyfactors`, `etl_factors`, `dagster_factors`, `dbt_factors`,
`dashboards` connect. A new contributor has to read every
`__init__.py` to figure out which package imports which. The
`pyproject.toml` already lists `pydeps` as a dev dep; running it
once and committing the SVG would close this gap.

**ADAPT.** A two-paragraph `doc/architecture.md` plus a `pydeps`
SVG; refresh on every CI run.

### `one-pagers/` directory exists but is empty

Nothing in it. Either pruned dirs or future intent - the new repo
should not commit empty placeholders.

**ADAPT.** Either populate (a one-pager on each indicator, similar
to `linear_regression_ln.md`) or remove.

### Docstring style is inconsistent

Some indicators have a multi-paragraph docstring with citations
(`sharpe_ratio.py`, `linear_regression_slope.py`); others are
docstring-free or single-line (`annualized_volatility.py:11-15`).
Some validation functions have docstrings explaining the invariant
(`stages.py:62-77`); others are bare (`stages.py:186-189`).

**ADAPT.** Pick a per-file-type rule:
- public Pydantic model / strategy: at least 1-paragraph docstring
  with intent + reference if applicable.
- private validator: one-line docstring naming the invariant.
- internal helper: optional, only if the name doesn't explain itself.

### Old-style numpy "Args:" docstrings mixed with prose docstrings

```python
# core/datetime.py:13-23
"""
Split a date range into chunks where each chunk's range is limited to `chunk_size`.

Args:
    start_date: Start date in YYYY-MM-DD format
    end_date: End date in YYYY-MM-DD format
    chunk_size: timedelta object defining chunk size
    align_bounds_to_year: If True, ensure chunks don't span across year boundaries
                         based on the chunk_size years (e.g., if chunk_size is 3 years,
                         align to 3-year boundaries)
"""
```

The `Args:` section duplicates the type hints (`start_date: str | date`
in the signature already says the same thing). The third bullet
(`align_bounds_to_year`) is the only one carrying information beyond
the type. The other three are noise.

**ADAPT.** *"Docstrings describe intent and non-obvious behaviour.
Don't repeat the type. Document parameters individually only when
the type doesn't explain them."*

## Recommendations for the new guideline

1. **ADR per non-trivial decision.** Use the 0001 template
   (Context / Decision / Consequences); keep alternatives-considered
   inside one paragraph of the Decision section.
2. **For each upstream data source, a `doc/<source>_decisions.md`.**
   One paragraph per cleaning rule with the *why* and a concrete
   example.
3. **For non-obvious algorithmic choices, a `doc/<algorithm>.md` with
   the math derivation; link from the docstring; summarise in
   2-3 lines in the docstring.**
4. **Citations in algorithm docstrings include title, authors, year,
   URL.** All four.
5. **Architecture overview + `pydeps` SVG in `doc/architecture.md`,
   refreshed in CI.**
6. **Docstrings describe intent, not types.** No `Args:` blocks that
   merely re-list the signature; only the parameters whose meaning
   isn't covered by the type hint get a line.
7. **No empty placeholder directories.** `one-pagers/` either has
   content or doesn't exist.
8. **The local `guidelines.md` already articulates the right
   altitude.** The new repo's guidelines are an *expansion* of that
   doc, not a replacement.
