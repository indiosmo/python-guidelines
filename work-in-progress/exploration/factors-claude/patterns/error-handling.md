# Error handling

The C++ guidelines distinguish *contract violations* (raise / refuse to
run) from *expected failures* (typed `expected<T, E>` result). Python
doesn't have `expected<>`, so the equivalent in this codebase is
"raise the right exception class at the right boundary, and translate
it explicitly where the next layer needs a different shape".

factors2 does this well *inside* the pure-Python library and poorly
*at* the boundaries.

## What factors2 does well

### Named, single-purpose preconditions; one `raise` per rule

```python
# pipeline/stages.py:62-93
def _validate_weighted_signals_non_empty(weighted_signals):
    if weighted_signals.empty:
        raise ValueError("WeightedSignals validation failed: dataframe is empty")


def _validate_weighted_signals_unique_keys(weighted_signals):
    duplicated = weighted_signals.duplicated(subset=SYMBOL_JOIN_KEY, keep=False)
    _raise_validation_error(weighted_signals, duplicated, context="WeightedSignals", ...)


def _validate_weighted_signals_scores(weighted_signals):
    active = weighted_signals.loc[weighted_signals["signal"].isin(["long", "short"])]
    invalid = ~np.isfinite(active["score"].to_numpy(dtype=float))
    _raise_validation_error(active, invalid, context="WeightedSignals", ...)
```

Each predicate is one named function; each function raises exactly
once; the public `validate_*` is a flat list of those calls. Result:
when a rule fires in production, the message includes both the rule
name (in the function name and the message prefix) and a sample of
offending rows. The matching tests cover exactly one rule each, and
assert the exact message text via `re.escape(...)`.

**PROMOTE.** Guideline phrasing:
- one rule -> one named predicate -> one `raise`
- compose predicates into one public validator
- message includes context, rule, count, sample
- tests pair one assertion to one rule

### Helper that uniformly reports rule violations

```python
# pipeline/stages.py:40-59
def _sample_rows(df, columns, n=5):
    return df.loc[:, columns].head(n).to_dict(orient="records")

def _raise_validation_error(df, invalid, *, context, columns, message):
    if not bool(np.any(invalid)):
        return
    bad_rows = df.loc[invalid]
    sample = _sample_rows(bad_rows, columns)
    count = int(np.sum(invalid))
    raise ValueError(f"{context} validation failed: found {count} {message}. Sample: {sample}")
```

The helper enforces the same shape across all rules. **PROMOTE.**

### `_validate_step_result` - one rule per `if`, prefixed identically

`backtest/models.py:1022-1179` is 150 lines of "if X then raise
ValueError('StepResult validation failed: ...')", each block 2-4
lines, each calling out exactly one invariant. It reads as a checklist.

```python
# backtest/models.py:1106-1145
def _validate_position(key, position):
    key_market, key_symbol = key

    if (position.market, position.symbol) != (key_market, key_symbol):
        raise ValueError(
            "StepResult validation failed: "
            f"book position key {key!r} does not match contained position ..."
        )
    if not _is_non_empty_string(position.market):
        raise ValueError(f"StepResult validation failed: position {key!r} market must be a non-empty string")
    ...
    if position.position >= 0 and position.accrued_borrow_fee != 0:
        raise ValueError(f"StepResult validation failed: non-short position {key!r} cannot carry accrued_borrow_fee")
```

Note that the *last* rule is a domain rule ("long positions cannot
accrue borrow fees"), sitting in the same checklist as the basic
finite-number rules. The flatness is the point.

**PROMOTE.** This is the imperative-shell version of
"contract-style preconditions" from the C++ guidelines.

## What factors2 does poorly

### `except Exception` at the trust boundary, swallowing into `st.toast`

```python
# portfolio_management/app.py:50-58
def update_position_book():
    try:
        ...
    except Exception as e:
        st.toast(str(e))

# same pattern at lines 86-91, 126-137, 140-150, 153-165, 168-188, 191-203, 206-235
```

Eight handlers in one file. None of them distinguishes
`pydantic.ValidationError` (bad user input, retryable in the UI) from
`psycopg.OperationalError` (DB down, not retryable in the UI) from
`KeyError` on `st.session_state` (programmer bug, should crash).
The user sees the same toast in every case.

In `fetch_cnpj.py:38` the same shape eats every exception in a tight
loop, including `KeyboardInterrupt` (it's a subclass of
`BaseException`, not `Exception` - so this one is safe, but only by
luck). In `proxy_downloader.py:116` the catch is right after a
`@retry` decorator: by the time control reaches the `except`, tenacity
has already retried 5 times, so the catch is genuinely the last-resort
boundary - but logging `str(e)` loses the chain.

**AVOID.** Guideline rules to lift:
- `except Exception` is allowed at exactly one place per call stack:
  the boundary that maps internal failures to the next protocol (UI,
  HTTP response, Dagster op result). Everywhere else, name the class.
- At that boundary, log `traceback.format_exc()` or use
  `logger.exception(...)`, not `str(e)`.
- Map known exception classes to user-visible outcomes; let unknown
  exception classes crash.

### `logging.error(f"...{e}"); raise` - the worst of both worlds

```python
# database/dataset.py:178-180
except Exception as e:
    logging.error(f"Error loading dataset from database: {e}")
    raise
```

Logging the exception then re-raising it doubles the noise: the log
has a one-line message with no traceback; the propagated exception
will be logged again upstream, this time with the traceback. Either
let it propagate, or log `logger.exception(...)` (which prints the
traceback) and *don't* re-raise.

**AVOID.** Pick one: re-raise OR log-with-traceback; not both. If you
catch to add diagnostic context, raise a new exception `from e`.

### Translating HTTP errors implicitly via `response.raise_for_status()`

```python
# etl_factors/sources/economatica/downloads.py:65-67
with httpx.Client(...) as client:
    response = client.get(url)
    response.raise_for_status()
    body = response.content
```

This is actually fine - the docstring at line 56 says exactly what it
intends: *"Raises `httpx.HTTPStatusError` on non-2xx responses so the
calling Dagster asset fails and the downstream loader does not run."*
The contract is documented and the exception class is explicit.
**PROMOTE** the documented intent alongside the raise.

### Two competing styles in `simulation.py:202-203`, `226-227`, `377-380`

```python
if risk_free_rate is None:
    raise ValueError(f"missing rate for trade_date {trade_date}")

if benchmark_close is None:
    raise RuntimeError(f"Simulation no benchmark index close for trade date {trade_date}")
```

`ValueError` is the right class when the *input is wrong*; `RuntimeError`
is the catch-all for "I cannot proceed". These both mean the same thing
("upstream data is missing") and should be the same class. The
guideline can lift: *"`ValueError` for caller-supplied bad input,
`RuntimeError` only for genuine programmer-error states; for missing
upstream data, define a domain exception (`MissingMarketData`,
`MissingBenchmarkData`) and let the caller catch by class."*

**ADAPT.** Same name for the same kind of failure; define domain
exception classes for the systemic cases.

### `mark_to_market` swallowing missing prices

```python
# backtest/simulation.py:23-31
def mark_to_market(trade_date, book, reference_data):
    for position in book.positions.values():
        close_price = reference_data.market_prices.get_close(trade_date, position.market, position.symbol)
        if close_price is not None:
            position.mtm_price = close_price
            # raise RuntimeError(f"missing close price for symbol {position.symbol} trade date {trade_date}")
```

The commented-out `raise` is the giveaway: someone decided to silently
keep the previous `mtm_price` rather than raise. There's no comment
explaining why, no test pinning the behaviour, and a downstream
`assert_step_result` will accept the stale NAV. Either the behaviour
is wrong (raise), or it is right (write a one-line comment explaining
"on a missing close, carry yesterday's price; subsequent NAV reflects
yesterday's mark") and delete the commented line.

**AVOID** commented-out alternative branches. The guideline can lift:
*"commented-out code is a refactoring leftover; delete it or write
the rule down".*

## Recommendations for the new guideline

1. *Preconditions raise; raise immediately.* One predicate, one rule,
   one named `raise ValueError(...)`. Compose into one public validator
   per type/boundary.
2. *Validator messages carry context, rule, count, sample.* The
   `_raise_validation_error` helper in `pipeline/stages.py` is the
   template.
3. *Catch by class.* `except Exception` is only legal at one place per
   call stack: the boundary that maps to the next protocol. Define
   domain exceptions for systemic failure modes (missing data, vendor
   timeout, unfilled contract).
4. *Never log-and-re-raise.* Either propagate, or
   `logger.exception(...)` and stop. To add context, `raise NewError(...) from e`.
5. *Document the boundary contract.* `response.raise_for_status()` is
   fine when the docstring says "raises HTTPStatusError so the upstream
   asset fails". Implicit raises without that note are landmines.
6. *No commented-out alternative branches.* Code shows what it does;
   git history shows what it used to do.
