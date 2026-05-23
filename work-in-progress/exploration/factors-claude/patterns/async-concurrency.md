# Async and concurrency

factors2 has three distinct concurrency stories: **asyncio + httpx** in
the dlt sources, **asyncio + aiohttp + tenacity + aiolimiter** in the
proxy downloader, and **Ray actors** for parallel backtest simulations.
The async usage is small (one good module, one mediocre module) and
the Ray usage is small but correct.

## What factors2 does well

### `httpx.AsyncClient` as a context manager, scoped to one resource

```python
# etl_factors/sources/bcb/__init__.py:17-79
def create_client() -> httpx.AsyncClient:
    return httpx.AsyncClient(timeout=httpx.Timeout(30.0))

@dlt.resource(name="cdi_daily_rates", primary_key=("trade_date",), ...)
async def cdi_daily_rates(updated_at=dlt.sources.incremental("trade_date", ...)) -> Any:
    start_date = updated_at.start_value
    end_date = date.today()
    async with create_client() as client:
        date_ranges = split_date_range(start_date=start_date, end_date=end_date, chunk_size=relativedelta(years=MAX_CHUNK_YEARS))
        for chunk_start, chunk_end in date_ranges:
            params = api.FetchSeriesParams(series_code=CDI_SERIES, start_date=chunk_start, end_date=chunk_end)
            data = await api.fetch_series(request, params, client)
            if not data:
                continue
            df = api.RateDF.validate(pd.DataFrame(data))
            arrow_table = pa.Table.from_pandas(df)
            arrow_table = fix_primary_key_schema(arrow_table, ["trade_date"])
            yield arrow_table
```

Single `async with`, one client per resource, an explicit chunking
function (`split_date_range`) keeps the loop bounded.
**PROMOTE** as the canonical "talk to a paginated HTTP API" shape.

### Pydantic at the wire boundary, ibis/pandas after

The same module uses Pydantic for serialisation
(`FetchSeriesParams.model_dump(mode="json", by_alias=True, exclude_none=True)`,
`api.py:73-78`) and pandera for the result frame
(`RateDF.validate(pd.DataFrame(data))`). Wire formatting + domain
validation live in named types, not in the loop. **PROMOTE.**

### `httpx` for sync HTTP with explicit timeout + raise

```python
# etl_factors/sources/economatica/downloads.py:65-67
with httpx.Client(timeout=httpx.Timeout(DOWNLOAD_TIMEOUT_SECONDS), follow_redirects=True) as client:
    response = client.get(url)
    response.raise_for_status()
    body = response.content
```

Timeout is named and explicit (600 seconds, with a comment justifying
it on line 25-26), `raise_for_status()` is called, the failure mode
("downstream loader does not run") is documented on lines 54-56. This
is a small block but it's the *right* shape. **PROMOTE.**

### Ray actor pool with a one-time deserialisation cost

```python
# backtest/server.py:35-54
@ray.remote
class SimulationWorker:
    def __init__(self, refdata: ReferenceData) -> None:
        t0 = timeit.default_timer()
        self.refdata = refdata
        self.refdata.market_prices.build_dict()  # cache dict for O(1) lookup
        ...

    def run_simulation(self, v):
        params, scores = v[0], v[1]
        return _run(params, scores, self.refdata)
```

The reference data is deserialised once per worker, then `build_dict()`
builds the lookup cache once per worker - the rest of the workload is
pure simulation. The driver code in `run_simulation` uses
`workers.map_unordered(...)` and pulls results as they arrive
(`server.py:115-120`).

This is the right shape: stateful actors for heavyweight initialisation,
stateless tasks for the per-job work. **PROMOTE.**

## What factors2 does poorly

### `try: ... except Exception as e: ... await error_callback(e, ...)`

```python
# etl_factors/utils/proxy_downloader.py:113-124
async with self.semaphore:
    url_with_params = f"{self.base_url}?{urlencode(query_params)}"
    logger.info(f"Downloading: {url_with_params}", proxy=self._pick_proxy(idx))
    try:
        data = await self._fetch_one(session, query_params, idx)
        await success_callback(data, query_params)
    except Exception as e:
        url_with_params = f"{self.base_url}?{urlencode(query_params)}"
        logger.error(f"Request failed after all retries: {url_with_params} | {str(e)}", ...)
        await error_callback(e, query_params)
```

By the time we reach the except, tenacity has already retried 5 times
(line 126-129). Catching `Exception` here is the boundary - but the
error callback receives the bare exception with no chain, the log
shows `str(e)` (no traceback), and the success/error split is bifurcated
inside the function.

`asyncio.gather(*tasks, return_exceptions=True)` on line 86 also
collapses exceptions to "got one in the list, look later". With Python
3.11+, `asyncio.TaskGroup` + `*ExceptionGroup` would let the caller
handle the partial-failure case explicitly.

**ADAPT.** Three things to lift into the guideline:
- `asyncio.gather(return_exceptions=True)` is fine for "fire-and-forget"
  batch downloads, but for anything where you want structured error
  handling, prefer `asyncio.TaskGroup` + `except*`.
- Use `logger.exception(...)` inside async exception handlers, not
  `logger.error(f"... {str(e)}")`. The latter loses the traceback.
- Callbacks (`success_callback`, `error_callback`) couple this class
  to its consumer's error model; consider an async iterator that
  yields a `Result` sum type instead.

### Blocking calls inside an async function

```python
# etl_factors/sources/bcb/__init__.py - cdi_daily_rates
df = api.RateDF.validate(pd.DataFrame(data))
arrow_table = pa.Table.from_pandas(df)
```

`pd.DataFrame(...)`, pandera validation, and `pa.Table.from_pandas`
are all CPU/blocking work; they run on the event loop thread. For
chunk sizes of 10 years of daily rates this is small (a few MB), so
it doesn't matter. But the pattern is a smell: an `async def` that
mixes `await` with blocking I/O makes it ambiguous what the loop is
doing.

**ADAPT.** Guideline: "if the function does CPU-bound work that takes
longer than a single network round-trip, move it out of `async def`
with `asyncio.to_thread(...)` or compute it after the loop completes."

### `time.sleep` and `requests` inside a loop in fetch_cnpj.py

```python
# etl_factors/sources/brasilapi/fetch_cnpj.py:1-46
MAX_PER_SECOND = 5
SLEEP_TIME = 1 / MAX_PER_SECOND

def main(input_file, output_file):
    ...
    for cnpj in cnpjs:
        try:
            response = requests.get(API_URL.format(cnpj))
            if response.status_code == requests.codes.ok:
                ...
            else:
                ...
        except Exception as e:
            ...
        time.sleep(SLEEP_TIME)
```

This file is presumably "throwaway script" - but it's in `src/`, not
in `scripts/`, and the team already has `proxy_downloader.py` doing
the right thing with `aiolimiter` and `httpx.AsyncClient`. The
guideline should call out: "the right tool for rate-limited HTTP
fetching exists in this repo; don't write a new sync `time.sleep`
loop next door."

**AVOID** new `time.sleep` rate limiting; reuse the existing
`aiolimiter` shape.

## Recommendations for the new guideline

1. **HTTP**: prefer `httpx` over `requests`; prefer `httpx.AsyncClient`
   over `aiohttp.ClientSession` for new code unless you need
   `aiohttp_socks`-style transports. One client per scope, explicit
   `httpx.Timeout`, `response.raise_for_status()` documented in the
   docstring.
2. **Rate limiting**: `aiolimiter.AsyncLimiter` for request budgets;
   `asyncio.Semaphore` for in-flight concurrency. Both used together
   is fine; don't write `time.sleep` loops in `src/`.
3. **Retries**: `tenacity` with `stop_after_attempt` + `wait_exponential_jitter`
   wrapped around the *single-attempt* function; the wrapper's
   `try/except` is the *post-retry* boundary, and should
   `logger.exception(...)` and either re-raise or return a typed
   failure.
4. **Concurrency primitives**: prefer `asyncio.TaskGroup` (Python
   3.11+) over `asyncio.gather(return_exceptions=True)` when callers
   want structured error handling.
5. **Blocking in async**: short pandas/pyarrow work is fine inline;
   anything longer than a few ms goes through `asyncio.to_thread(...)`
   or runs after the gather/await.
6. **Heavy parallelism**: Ray actors for stateful initialisation +
   stateless per-job work. The actor's `__init__` carries the
   one-time deserialisation cost; `map_unordered` over the work; pull
   results as they complete.
