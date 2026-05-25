# Logging

Logging is the debugging surface that remains after the failing request, job,
or test is gone. A useful log line has a stable event name, structured context,
an intentional severity, and one owning logging stack.

Choose one logging stack per project surface. Libraries should emit through
stdlib `logging` unless the repository has a stronger policy. Applications can
standardize on stdlib logging, loguru, or structlog, then bridge third-party
library logs at startup.

## Choose The Stack

Use one stack for an application or deployable unit. This matrix scores common
choices from 1 to 5, where 5 is strongest for the criterion.

| Stack | Library Fit | App Ergonomics | Fields | Cost | Bridging | Total |
|---|---:|---:|---:|---:|---:|---:|
| stdlib `logging` | 5 | 3 | 3 | 5 | 5 | 21 |
| loguru | 2 | 5 | 4 | 3 | 3 | 17 |
| structlog | 3 | 4 | 5 | 3 | 4 | 19 |

Default to stdlib logging for reusable libraries because callers already know
how to configure it. Use loguru when an application values concise setup,
rich tracebacks, and ergonomic local logging. Use structlog when logs are a
strict key/value product consumed by collectors, alerting, or audit workflows.

## Shape Log Lines As Events

Make the message a stable event name. Put request IDs, domain identifiers,
counts, durations, file paths, URLs, and exception details in structured
fields.

```python
logger.info(
    "batch download completed",
    extra={
        "query_count": len(queries),
        "successful_count": successful_count,
        "failed_count": failed_count,
        "elapsed_seconds": elapsed_seconds,
    },
)
```

For loguru or structlog, bind fields through the stack's native API:

```python
logger.bind(
    query_count=len(queries),
    successful_count=successful_count,
    failed_count=failed_count,
    elapsed_seconds=elapsed_seconds,
).info("batch download completed")
```

Stable messages group cleanly in search and metrics. Structured fields let
operators filter and aggregate without parsing prose.

## Use Severity Consistently

Severity is part of the contract with operators and tests:

- `debug`: investigation details that are useful during development or a
  targeted capture session.
- `info`: significant business or operational events, such as a job start,
  asset materialization, accepted command, or completed batch.
- `warning`: recoverable degraded behavior, retries, fallbacks, or accepted
  inputs that deserve attention.
- `error`: an operation failed and the boundary reports or translates the
  failure.
- `critical`: the process, worker, or service cannot continue.

If the chosen stack has `trace` or `success`, define their meaning in project
policy before using them. Keep per-row or per-message diagnostics off by
default in production paths.

## Keep Disabled Levels Cheap

Python does not remove disabled log calls at compile time. Prefer lazy
formatting and guard expensive argument construction.

```python
import logging


logger.debug("computed portfolio weights for %s accounts", account_count)

if logger.isEnabledFor(logging.DEBUG):
    logger.debug(
        "portfolio weight sample",
        extra={"sample": weights.head(10).to_dict(orient="records")},
    )
```

Build large dataframe samples, serialized payloads, stack traces, and
expensive diagnostic strings only when the level is enabled or the diagnostic
is part of an explicit capture mode.

## Log Exceptions At Stopping Boundaries

Log a traceback where the current layer stops, translates, retries for the
last time, or returns a boundary response. In an exception handler, use the
stack's exception-aware call so the traceback and chained causes are present.

```python
from collections.abc import Sequence


def cli_main(argv: Sequence[str]) -> int:
    try:
        run_command(argv)
    except ConfigurationError:
        logger.exception("command failed during configuration")
        return 2
    except Exception:
        logger.exception("command failed unexpectedly")
        return 1
    return 0
```

When a layer adds context and propagates the failure, raise a new exception
with chaining and let the boundary log it.

```python
try:
    response = client.get(url)
    response.raise_for_status()
except httpx.HTTPError as exc:
    raise VendorDownloadError(vendor="market-data", url=url) from exc
```

This keeps one traceback log per failure path and preserves the causal chain.

## Control Volume

High-volume diagnostics need a volume policy. Use sampling, throttling,
counters, histograms, or bounded debug capture instead of emitting one log
line per item on a hot path.

```python
if rejected_count:
    logger.warning(
        "orders rejected by risk checks",
        extra={
            "rejected_count": rejected_count,
            "sample_order_ids": rejected_order_ids[:5],
        },
    )
```

For timing, prefer a common timer context manager or profiler command over
ad hoc status output. Timed logs should use a stable event name and a numeric
duration field.

## Test Logging Behavior

Use pytest's `caplog` for stdlib logging. For loguru or structlog, provide a
project fixture that captures through the configured sink. Tests that change
handlers, levels, filters, processors, or warning policies should restore them
with fixtures or context managers.

Assert logs when the log line is part of the contract: boundary translation,
audit events, warning-level degradation, retry exhaustion, or operational
milestones. Use direct assertions for behavior that logs merely describe.
