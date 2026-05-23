# State and invariants

The C++ guidelines call out *exception safety, transactions, rollback
with scope guards, idempotency for retried operations*. The Python
analogues factors2 uses are: idempotent loaders via dlt write
dispositions, `with conn.transaction()` for SQL writes, replayability
expressed as a dbt vs Dagster split, and per-step `validate_step_result`
to enforce invariants on a mutable simulation state.

## What factors2 does well

### Replayable vs irrecoverable, made explicit at the orchestration layer

`doc/adr/0001-schedule-only-economatica-downloads.md` is the cleanest
statement of the idea anywhere in the repo. Quoting:

> Economatica downloads pull daily snapshot files from the vendor's
> bucket. The vendor overwrites those files every day, so a snapshot
> that is not fetched on its own date is gone.
> Economatica raw / dlt loads ingest the downloaded files into the
> warehouse. They can re-run at any time against whatever files are
> present locally.
> BCB ingestion pulls public series from the central bank. The full
> history is always available upstream.
> dbt models transform data already in the warehouse. They are
> replayable from their inputs at any point.

The whole pipeline is classified into four tiers of recoverability,
the *only* scheduled job is the one that loses data on a miss, and
everything else is "rerun on demand". This is the right shape for
idempotency: not "every step is idempotent" (impossible: vendor
rotation), but "every step that *can* be idempotent is, and the rest
is wrapped in a separate schedule with its own alerting".

**PROMOTE.** This goes verbatim into the orchestration chapter:
*"classify each step by recoverability, schedule only the irrecoverable
ones, document the rest as 'rerun on demand' so missed runs are not
silent failures."*

### dlt write dispositions encode idempotency intent

```python
# etl_factors/sources/economatica/__init__.py:226-261
if snapshot:
    files_resource = latest_file(bucket_url=bucket_url, file_glob=glob)
else:
    files_resource = filesystem(bucket_url=bucket_url, file_glob=glob)
    files_resource.apply_hints(incremental=dlt.sources.incremental("modification_date"))

resource = files_resource | read_csv(..., tag_source_mtime=not snapshot)

hints: dict = {
    "file_format": "parquet",
    "write_disposition": write_disposition,  # "merge" or "replace"
    "table_name": table_name,
    "primary_key": primary_key,
    "merge_key": merge_key,
}
if not snapshot:
    hints["columns"] = {"_source_mtime": {"dedup_sort": "desc"}}
```

Each table picks its write disposition (`replace` for snapshots like
`security_list`, `merge` for incremental tables with a primary key),
and the snapshot vs incremental choice flows through `latest_file()`
vs `filesystem() + apply_hints(incremental=...)`. The `_source_mtime`
tagging on incrementals plus `dedup_sort: "desc"` is the explicit
"same row twice -> keep the newer" rule.

**PROMOTE.** Guideline phrasing: "loaders declare their write
disposition; snapshot tables use `replace`, incrementals use `merge`
on a primary key plus a deduplication sort. The retry semantics are
in the disposition, not in the caller."

### `with conn.transaction()` for multi-statement writes

```python
# database/database_queries.py:241-271
def add_blacklist_entry(conn: psycopg.Connection, market: str, symbol: str, start_date: date) -> None:
    with conn.transaction():
        with conn.cursor() as cur:
            cur.execute("INSERT INTO ranking.blacklist (...) VALUES (...)", (...))


def remove_blacklist_entry(conn: psycopg.Connection, market: str, symbol: str, end_date: date) -> None:
    with conn.transaction():
        with conn.cursor() as cur:
            cur.execute("UPDATE ranking.blacklist b SET ... WHERE ...", (...))
```

And the COPY loader:

```python
# etl_factors/sources/directory/csv_to_postgres.py:30-35
with conn.transaction():
    cur.execute(sql.SQL("TRUNCATE TABLE {} CASCADE").format(sql.Identifier(SCHEMA, table_name)))
    with open(csv_path, encoding="utf-8") as fh:
        with cur.copy(stmt) as copy:
            copy.write(fh.read())
```

The `TRUNCATE + COPY` is atomic from the readers' point of view -
either the table has the old data or the new, never a mixture. This
is the "transactional boundary" the C++ guidelines describe.

**PROMOTE.** Multi-statement writes are wrapped in
`with conn.transaction()`; in dlt, the equivalent is the `merge`
write disposition with a primary key.

### `validate_step_result` re-checks invariants on the *mutable* `PositionBook`

```python
# backtest/simulation.py:246-262
self.result.steps.append(
    validate_step_result(
        StepResult(
            trade_date=trade_date,
            trades=trades,
            book=self._book.model_copy(deep=True),
            mark_to_market_nav=self._book.nav(),
            ...
        )
    )
)
```

The simulation mutates `_book` in place (positions get modified by
`process_trade`, cash accrues), but every step ends with a deep copy
that is then handed to `validate_step_result` (`models.py:1171-1179`).
The validator checks 30+ invariants over the snapshot
(`test_validate_step_result.py` covers each one). If any invariant
fails, the simulation raises and stops, before the next step compounds
the damage.

**PROMOTE.** *"After mutating shared state, validate the snapshot
before persisting it."* This is the in-process analogue of the
"transaction commits only if invariants hold" pattern.

## What factors2 does poorly

### Stateful caches that can drift from their backing DataFrame

```python
# pipeline/models.py:139-188
class InterestRates:
    def __init__(self, df: pd.DataFrame, *, build_dict=True):
        self.df = df.copy()
        self.dict: dict[tuple[date, str], float] = {}
        if build_dict:
            self.build_dict()

    def build_dict(self) -> None:
        self.dict = {...}
```

`self.df` is public, `self.dict` is derived from it, and there is no
contract preventing a caller from mutating `self.df` and forgetting
to call `build_dict()` again. The `ReferenceData.build_dicts()` method
exists (`models.py:267-270`) to rebuild all three caches at once,
which is the giveaway that this happens often enough to matter.

**AVOID.** Either:
- make the wrapper frozen (`@dataclass(frozen=True)` + the dict built
  in `__post_init__`), or
- expose only the dict and hide the DataFrame, or
- use `functools.cached_property` so the dict is rebuilt lazily on
  first access after invalidation.

The current shape is a sharp edge - looks fine in tests, can drift
in production when the data is updated incrementally.

### `mark_to_market` silently leaves stale `mtm_price` on missing data

```python
# backtest/simulation.py:23-31
def mark_to_market(trade_date, book, reference_data):
    for position in book.positions.values():
        close_price = reference_data.market_prices.get_close(trade_date, position.market, position.symbol)
        if close_price is not None:
            position.mtm_price = close_price
            # raise RuntimeError(f"missing close price for symbol {position.symbol} trade date {trade_date}")
```

The commented-out raise is the previous behaviour. Now the position
keeps yesterday's mark, the book's `nav()` reflects a stale price,
and `validate_step_result` will pass because there's no invariant
that says "the mark must be from today". A position with a stale
mark for 5 trading days is indistinguishable from one that was just
marked.

**AVOID.** Either raise (the commented-out behaviour) or track
`mtm_trade_date` on the `Position` so the staleness is visible in
the snapshot. Don't carry yesterday's mark silently.

### Idempotency-by-trusting-`_dlt_id` instead of by primary key

dbt models drop `_dlt_id` and `_dlt_load_id` columns
(`database_queries.py:145`: `SELECT * EXCLUDE (_dlt_id, _dlt_load_id)`).
That's fine. But several Pydantic loader configs declare primary keys
that span 4 columns (`["symbol", "market", "balance_date", "disclosure_date"]`
in `economatica/__init__.py:389`) and the corresponding pandera
schema does not assert uniqueness over them
(`table_models.py:56-65`). If a vendor file ever contains duplicate
rows on the same primary key, dlt will keep the newer (good), pandera
will not flag it (silent risk).

**ADAPT.** Lift the primary key declaration into the pandera schema:
`class Config: ... unique = [...]` or a `@pa.dataframe_check` that
asserts uniqueness on the same columns.

## Recommendations for the new guideline

1. **Classify steps by recoverability.** *Vendor-rotated -> schedule
   and alert; replayable from upstream -> "rerun on demand";
   transforms over data already in the warehouse -> dbt.* Document
   the classification in an ADR; never schedule a replayable step
   "just in case".
2. **Loaders declare write disposition.** Snapshot tables use
   `replace`; incremental tables use `merge` on the primary key plus
   a tiebreak sort. Idempotency is in the disposition, not in the
   caller.
3. **Wrap multi-statement writes in a transaction.** `with conn.transaction():`
   for psycopg; `merge` disposition for dlt; one ADR-level decision
   per data store.
4. **Validate the snapshot before persisting it.** After mutating
   shared state, copy + validate; if the validator raises, the
   mutation is rolled back (or the process stops, depending on the
   layer).
5. **Caches must be immutable or self-rebuilding.** A class that
   exposes both `df` and `dict[...]` where one is derived from the
   other is a footgun; freeze or hide one.
6. **No silent staleness.** Carrying yesterday's value when today's
   is missing is fine *if and only if* the staleness is part of the
   recorded state. Otherwise raise.
