# Logging

Overall disposition: PORTS_WITH_ADAPTATION.

The logging goals port, but C++ macro and compile-time cutoff mechanics do not.
Python should use the stdlib logging stack by default unless a project has
chosen structlog, loguru, or another structured logger.

## Severity family

Disposition: PORTS_AS_IS.

Keep the severity family, mapped to Python names: `DEBUG`, `INFO`, `WARNING`,
`ERROR`, `CRITICAL`. Use `TRACE` only if the project configures a custom level.

## Compile-time level cutoff

Disposition: PYTHON_SPECIFIC_VARIANT.

Replace with lazy logging and configuration policy. Python cannot compile out
disabled logging calls in the same way. Use parameterized logging so disabled
levels avoid formatting.

```python
logger.debug("order %s settled with reason %s", order_id, settle_reason)
```

Guard expensive argument construction:

```python
if logger.isEnabledFor(logging.DEBUG):
    logger.debug("book snapshot %s", render_snapshot(book))
```

## Throttled variants for hot call sites

Disposition: PORTS_WITH_ADAPTATION.

Keep it. Implement as a logging filter, helper, or metrics-driven sampler.

```python
throttled_warning("queue_depth", interval=1.0, message="queue depth %s", args=(depth,))
```

Throttle before expensive formatting.

## Formatter support for domain types

Disposition: PORTS_WITH_ADAPTATION.

Keep the principle. Domain values should format naturally or expose structured
fields. Prefer structured `extra` fields over preformatted strings when the
logging backend supports them.

```python
logger.info(
    "order settled",
    extra={"order_id": str(order_id), "settle_reason": settle_reason.value},
)
```

## Logger as a variant-backed cross-cutting service

Disposition: PYTHON_SPECIFIC_VARIANT.

Replace with logging configuration at the application edge and test capture
through pytest `caplog`, monkeypatching, or provider fixtures.

```python
def test_logs_duplicate_order(caplog: pytest.LogCaptureFixture) -> None:
    with caplog.at_level(logging.WARNING):
        handle_duplicate(order)

    assert "duplicate order" in caplog.text
```

Warn against adding logging handlers at import time in widely imported modules.
