# Comments And Docstrings

Comments and docstrings are part of the design surface. Names, types, and
helper functions should carry most of the model, but some intent is cheaper to
read in prose. Use comments and docstrings to explain present behavior,
non-obvious reasoning, protocol order, domain rules, and public contracts.

Write in present tense. Keep the comment close to the code it explains.

## Guiding Comments

A guiding comment names the next block before the reader parses it. It is most
useful in functions that proceed through phases:

```python
def validate_weighted_signals(
    weighted_signals: DataFrame[WeightedSignals],
) -> DataFrame[WeightedSignals]:
    # Check facts that require relationships across rows.
    _validate_weighted_signals_non_empty(weighted_signals)
    _validate_weighted_signals_unique_keys(weighted_signals)
    _validate_weighted_signals_side_weight_sums(weighted_signals)

    # Apply the column-level schema after domain invariants have useful samples.
    return WeightedSignals.validate(weighted_signals)
```

The comments name why the groups are separate. They do not restate each call.

Use guiding comments for:

- Protocol ordering that affects callers or receivers.
- Transaction, rollback, or idempotency reasoning.
- Concurrency preconditions and ownership.
- Performance-sensitive ordering.
- Domain rules that a line of code relies on.
- Data cleaning choices that would look arbitrary otherwise.

## Present-Tense Contracts

Phrase preconditions and invariants as facts the code relies on:

```python
# Precondition: the session lock is held while next_sequence is read and incremented.
next_sequence = self._next_sequence
self._next_sequence += 1
```

This is a present contract. It tells a reviewer what must be true for the code
to be correct.

Avoid comments that describe old designs, moved behavior, commented-out
alternatives, or non-responsibilities. Code shows current behavior, and git
history shows previous behavior. Durable design decisions belong in ADRs.

## Negative Documentation

Negative documentation describes absent behavior instead of current behavior.
It often appears during refactors:

```python
# Bad: deduplication moved upstream.
orders = normalize_orders(raw_orders)

# Good: accept the upstream batch as the unit of normalization.
orders = normalize_orders(raw_orders)
```

The good comment names the present rule. If the line needs no present rule,
delete the comment.

Non-responsibility comments have the same problem:

```python
# Bad: this converter does not route, persist, or resolve status.
def convert_vendor_trade(payload: object) -> Trade: ...

# Good: convert the vendor trade payload into the market-data trade model.
def convert_vendor_trade(payload: object) -> Trade: ...
```

Describe what the code does. Put architectural ownership rules in the owning
guide or module.

## Docstrings

Use docstrings for public modules, public classes, public functions, public
methods, and non-obvious private validators. A docstring should tell the
reader what the object is for, what contract it exposes, and what domain rule
it implements.

Do not repeat type hints:

```python
def split_date_range(
    start_date: date,
    end_date: date,
    chunk_size: timedelta,
) -> list[DateRange]:
    """Split a closed date range into chunks no larger than chunk_size."""
```

Document a parameter only when the type and name are not enough:

```python
def annualized_volatility(
    returns: Series,
    *,
    annualization_factor: int,
) -> Series:
    """Return volatility scaled to the period represented by annualization_factor."""
```

The docstring explains the semantic role of the factor, not that it is an
integer.

## Algorithms And Domain Rationale

Algorithmic docstrings should separate intent from implementation choices:

```python
class LinearRegressionSlope(BaseModel):
    """Estimate trend strength from the slope of log prices.

    The indicator regresses natural-log prices on time so the slope is
    scale-independent across symbols. The result is reported as a decimal
    change per period.
    """
```

When an algorithm depends on a paper, regulation, protocol, or source-specific
data decision, summarize the relevant choice in the docstring and cite the
durable source. Put long derivations, ADRs, and source-specific cleaning rules
in durable documentation, then keep the docstring short.

## Validation Comments

Validators often benefit from names more than comments. Prefer one named
predicate or helper per rule:

```python
def _validate_unique_symbol_keys(weighted_signals: DataFrame[WeightedSignals]) -> None:
    duplicated = weighted_signals.duplicated(
        subset=["trade_date", "market", "symbol"],
        keep=False,
    )
    _raise_validation_error(
        weighted_signals,
        duplicated,
        context="WeightedSignals",
        columns=["trade_date", "market", "symbol"],
        message="duplicate symbol keys",
    )
```

The function name and structured error carry the rule. Add a docstring only
when the invariant needs domain context that the name cannot carry.

## Inline Comments

Use inline comments sparingly. They are useful when a line contains a dense
expression or a domain-specific constant:

```python
annualized = daily_volatility * sqrt(252)  # Brazilian equity trading days.
```

For repeated constants, prefer a named constant:

```python
BRAZIL_EQUITY_TRADING_DAYS = 252

annualized = daily_volatility * sqrt(BRAZIL_EQUITY_TRADING_DAYS)
```

## Commented-Out Code

Delete commented-out code. If a branch represents the current domain rule,
write the rule as code, a test, or a present-tense comment.

```python
def mark_to_market(close_price: float | None, prior_price: float) -> float:
    if close_price is None:
        # Missing closes carry the prior mark until the next priced session.
        return prior_price
    return close_price
```

The comment states the active rule. The inactive alternative belongs in
history, not beside current code.

## Review Checklist

Review comments and docstrings with the same care as code:

- The comment describes current behavior.
- The comment explains intent, ordering, or domain reasoning.
- The code and comment make the same claim.
- The docstring adds information beyond the signature.
- The prose uses domain vocabulary and present tense.
- Long rationale lives in durable documentation or an ADR.
