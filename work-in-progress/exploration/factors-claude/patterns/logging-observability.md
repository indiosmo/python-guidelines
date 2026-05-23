# Logging and observability

factors2 has **loguru** as a dependency, has **one** module using it
correctly, has **35+** `print()` calls in `src/pyfactors/`, and has
no consistent log-level discipline. The guideline has the easiest
job here: pick loguru, use it consistently, ban `print`.

## What factors2 does well

### Structured logging with key=value kwargs in `proxy_downloader.py`

```python
# etl_factors/utils/proxy_downloader.py:49-57
logger.info(
    "ProxyDownloader initialized",
    base_url=self.base_url,
    proxy_count=len(self.proxies),
    max_concurrent=max_concurrent,
    max_requests_per_second=max_requests_per_second,
    max_retries=max_retries,
    timeout=timeout,
)

# lines 73, 92
logger.info("Starting batch download", query_count=len(queries))
logger.info("Batch download completed", total=len(queries), successful=success_count, failed=len(exceptions))
```

Static message (so a log aggregator can group by it) + kwargs for the
variable parts (so a log aggregator can filter by them). This is the
shape the new guideline should standardise on.

**PROMOTE.** Guideline phrasing: *"the log message is static text;
the variable parts go in kwargs. Static = groupable, kwargs =
filterable."*

### Dagster's `context.log` for asset-level events

```python
# dagster_factors/components/economatica_downloads.py:62
context.log.info(f"downloaded {screen} to {path}")
```

Inside Dagster ops/assets, use `context.log` (which routes to the
Dagster UI and the unified log stream) instead of a top-level
`logger`. This is the right boundary - Dagster owns the run context,
the asset reports back through it.

**PROMOTE.** *"In Dagster ops/assets, use `context.log`. Library
code under those ops uses its own loguru logger; the Dagster
integration captures both."*

## What factors2 does poorly

### 35+ `print()` calls in production code

```python
# pipeline/stages.py:307-308
print("universe time: ", timeit.default_timer() - t1)
print("universe size: ", universe.memory_usage(deep=True).sum() / 1024 / 1024, "MB")

# pipeline/stages.py:329
print("indicators time: ", timeit.default_timer() - t1)

# backtest/simulation.py:328-332
print(
    f"[{self.params.name}] Auto-liquidating {order.market}:{order.symbol} "
    f"on {next_trade_date} "
    f"with order {order.side.value} {order.size}"
)

# backtest/server.py:44, 102, 103, 118, 123, 129, 131, 132
print(f"Worker initialized in {timeit.default_timer() - t0}")
print("scores time: ", timeit.default_timer() - t0)
print("scores size: ", scores.memory_usage(deep=True).sum() / 1024 / 1024, "MB")
print("waiting results")
print("simulation time: ", timeit.default_timer() - t2)
...
```

All of these should be structured `logger.info(...)` calls. Concrete
problems with `print`:

- Goes to stdout, not stderr; mingles with normal output.
- No level - the auto-liquidation message is operationally important
  ("we forced a position closed") but indistinguishable in tone from
  the timing prints.
- No timestamp, no module path, no correlation IDs.
- Cannot be silenced or rerouted per environment.
- Cannot be parsed: `print("universe time: ", x)` produces
  `universe time:  3.4` with the timing baked into the message.

**AVOID.** Replace with `logger.info(...)` calls; treat each as a
documented log line.

### `logging.error(...)` (stdlib) in one module, `loguru.logger` in another

```python
# pyfactors/database/dataset.py:1, 178-180
import logging
...
except Exception as e:
    logging.error(f"Error loading dataset from database: {e}")
    raise
```

The codebase has loguru in its deps; this module uses stdlib
`logging`. The result is that this one log line will not be configured
the same way as the loguru calls in `proxy_downloader.py`, and `str(e)`
will not include the traceback.

**AVOID** mixing stdlib `logging` with loguru. Pick one.

### `logger.error(f"... {str(e)}")` instead of `logger.exception(...)`

```python
# etl_factors/utils/proxy_downloader.py:118-123
logger.error(
    f"Request failed after all retries: {url_with_params} | {str(e)}",
    query_params=query_params,
    error=str(e),
    proxy=self._pick_proxy(idx),
)
```

The log line says "request failed" but does not include the traceback.
loguru's `logger.exception(...)` (inside an `except` block) includes
the full traceback by default. The current call also passes `error=str(e)`
as a kwarg *and* in the message - the kwarg is the structured place,
keep one.

**ADAPT.** *"Inside an `except` block, use `logger.exception(...)`,
not `logger.error(f"... {str(e)}")`. The exception class, message,
and traceback go in the structured log automatically."*

### `f"{x}"` interpolated into a static-looking log message

```python
# proxy_downloader.py:111
logger.info(f"Downloading: {url_with_params}", proxy=self._pick_proxy(idx))
```

`url_with_params` is variable, so the message is no longer groupable
across calls. The kwargs already carry the variable data; the message
should be static:

```python
logger.info("Downloading URL", url=url_with_params, proxy=self._pick_proxy(idx))
```

**ADAPT.** *"If the log message contains an f-string interpolation
of variable data, you've collapsed the structure. Move the variable
to kwargs."*

### `try: ... except: ...; st.toast(str(e))` as the only diagnostics

The Streamlit handlers in `portfolio_management/app.py` are the only
record of what went wrong - no log line, no error tracking, just a
transient UI toast that disappears in a few seconds.

**AVOID.** At trust boundaries, log the failure *and* surface it to
the user. Two different audiences, two different channels.

## Log-level discipline (currently absent)

There is no consistent rule for which level to use. `proxy_downloader.py`
uses `info` for normal events, `warning` for HTTP error responses,
`error` for "after all retries". Dagster uses `info` for "downloaded
file". Everything else uses `print`. The guideline can lift a clear
levelling rule (loguru defaults to: trace, debug, info, success,
warning, error, critical):

| Level     | Use for                                                                 |
|-----------|-------------------------------------------------------------------------|
| trace     | per-row diagnostics that should never run in production                 |
| debug     | per-call diagnostics (request/response, intermediate values) - off by default in prod |
| info      | one line per significant business event (job started, asset materialised) |
| success   | optional - operational milestones                                        |
| warning   | recoverable degraded behaviour (retry succeeded after a transient failure) |
| error     | operation failed; caller will see it                                     |
| critical  | process cannot continue                                                  |

## Recommendations for the new guideline

1. **One logger, loguru.** Stdlib `logging` is allowed only when a
   third-party library writes to it (and you redirect those to loguru).
2. **Static message, variable kwargs.** `logger.info("downloaded
   file", screen=..., path=..., elapsed_s=...)` not
   `logger.info(f"downloaded {screen} to {path}")`.
3. **Inside `except` blocks, use `logger.exception(...)`.** Includes
   the traceback automatically, attaches the exception object to the
   structured log.
4. **No `print` in `src/`.** CI rule (`ruff` `T201`).
5. **No timing via `print(...timeit.default_timer()...)`.** Use a
   context manager / decorator that emits one structured log line on
   exit (`elapsed_s=...`), or use pytest-benchmark in `benchmarks/`.
6. **In Dagster ops/assets, use `context.log`.** Library code uses
   its own loguru; Dagster captures both.
7. **At the trust boundary, log *and* notify the user.** The toast/HTTP
   response is for the user; the log is for the operator.
8. **Adopt a log-level table** (above) and stick to it. The level is
   the API: "if you see ERROR in prod, page on-call".
