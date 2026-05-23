# Analysis: cpp-debugging-principles/logging.md

Disposition: **PORTS_WITH_ADAPTATION**. The principles (severity family,
zero-cost disabled levels, throttled hot-path variants, domain-type
formatters, logger as variant-backed cross-cutting service) port. The
mechanics are different because Python's `logging` is already a
mature stdlib facility and `loguru` is a strong opinionated alternative.

## Section-by-section

### Severity family

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ severities:** `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL`.
- **Python equivalent:** stdlib `logging` has `DEBUG (10)`, `INFO (20)`,
  `WARNING (30)`, `ERROR (40)`, `CRITICAL (50)`. No `TRACE` by default.
  Add it as a custom level:
  ```python
  logging.TRACE = 5
  logging.addLevelName(logging.TRACE, "TRACE")
  ```
  loguru ships `trace`/`debug`/`info`/`success`/`warning`/`error`/`critical`
  out of the box.

  Recommend documenting the severity family with the same `TRACE` ->
  hot-path-detail-compiled-out-in-release rule, implemented in Python as:
  `if logger.isEnabledFor(TRACE): logger.log(TRACE, ...)`. The
  `isEnabledFor` check is the Python equivalent of the C++ compile-time
  preprocessor cutoff.

### Compile-time level cutoff

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** Disabled levels expand to `(void)0`; no runtime cost.
- **Python rendering:** Python has no preprocessor, so disabled levels
  always have **some** runtime cost. The closest approximation:

  - **Use `logger.debug("fmt %s", arg)`** instead of
    `logger.debug(f"fmt {arg}")`. The first form skips arg formatting
    when DEBUG is disabled; the f-string formats unconditionally.
  - **Guard expensive formats with `isEnabledFor`:**
    ```python
    if logger.isEnabledFor(logging.DEBUG):
        logger.debug("payload: %s", expensive_dump(payload))
    ```
  - **For trace-level hot paths**, write a tiny helper that returns
    early on `isEnabledFor`; or, if the rate is truly extreme, gate
    the call site on a module-level constant: `if _TRACE_ENABLED:
    logger.trace(...)`. The constant resolves once at import time;
    Python's peephole optimizer turns the gated branch into a no-op
    when False.

  Document that **true zero-cost** is not available; the goal is "as
  cheap as Python lets us".

### Throttled variants for hot call sites

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same idea. Implement a tiny throttle wrapper:

  ```python
  import time
  from collections import defaultdict

  _last_logged: dict[tuple[str, int], float] = defaultdict(float)

  def log_warn_throttled(interval: float, msg: str, *args) -> None:
      key = (sys._getframe(1).f_code.co_filename, sys._getframe(1).f_lineno)
      now = time.monotonic()
      if now - _last_logged[key] >= interval:
          _last_logged[key] = now
          logger.warning(msg, *args)
  ```

  The `(filename, lineno)` key is the Python equivalent of "call-site
  location". For production hot paths, prefer a more robust solution
  (`pythonjsonlogger`, structlog) and put rate-limiting on the
  collector side.

### Formatter support for domain types

- **Classification:** PORTS_AS_IS.
- **C++ principle:** A project formatter covers strong types,
  optionals, error codes; the call site never has to `.get()` or
  `to_string()`.
- **Python rendering:** Two options:

  - **Implement `__str__` / `__repr__`** on every domain dataclass.
    The format string `"%s"` formats them naturally; the call site
    is `logger.info("order %s settled", order_id)` -- no unwrap.
  - **Use structlog with key/value logging**: the renderer handles
    serialization centrally:
    ```python
    log.info("order_settled", order_id=order_id, reason=settle_reason)
    ```
    The structured form composes better with log aggregators (Loki,
    Datadog) and removes the formatter question entirely.

  Recommend `structlog` for new projects, loguru for "rich
  human-readable logs", stdlib `logging` for libraries that should not
  impose a logger on consumers.

### Logger as a variant-backed cross-cutting service

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** A `std::variant<console_logger, file_logger,
  null_logger>` configured once at startup.
- **Python rendering:** The Python equivalent is the
  `contextvars`-backed module singleton from
  `design/cross-cutting.md`, or simply "configure stdlib `logging` once
  at startup". Either works; the second is simpler and lines up with
  the way Python libraries already expect to be used:

  ```python
  # config/logging_setup.py
  import logging
  import logging.config

  def configure(settings: Settings) -> None:
      logging.config.dictConfig({
          "version": 1,
          "handlers": {
              "console": {"class": "logging.StreamHandler", "level": "INFO"},
              "file": {"class": "logging.FileHandler", "filename": settings.log_path},
          },
          "root": {"level": "INFO", "handlers": ["console", "file"]},
      })
  ```

  Tests install a no-op handler via the `caplog` fixture or via
  `logging.getLogger().handlers = [logging.NullHandler()]`.

## Suggested Python file: `python-debugging-principles/logging.md`

A rewritten chapter that recommends one of `stdlib logging` /
`loguru` / `structlog` per project type, documents the
no-zero-cost-disabled-levels caveat with mitigations, and ports the
throttling and domain-formatter rules. The cross-cutting service shape
links back to `design/cross-cutting.md`.
