# Analysis: cpp-debugging-principles/root-cause-tracing.md

Disposition: **PORTS_AS_IS**. The principle (trace the bad value
backward through the call chain; fix at the root; reinforce intermediate
layers) is language-independent. The tooling notes change.

## Section-by-section

### When to trace backward

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Identical guidance.

### Tracing procedure (observe -> immediate cause -> walk up -> root)

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Identical. The "empty routing key" example
  ports verbatim with `dataclass` instead of `struct`. Python's
  default-initialized empty string is the same trap (default
  `field(default="")`).

### Instrumenting when manual tracing runs out

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ snippet:** `cpptrace::generate_trace().to_string()` printed to
  log.
- **Python equivalents:**

  ```python
  import traceback
  import logging

  def lookup(self, key: str) -> Destination:
      if not key:
          logging.debug(
              "router.lookup: empty key\n%s",
              "".join(traceback.format_stack()),
          )
          raise InvalidArgument("empty key in routing table lookup")
  ```

  For richer traces:
  - `loguru` automatically captures backtraces when an exception
    propagates and `logger.opt(exception=True).debug(...)` is used.
  - `rich.traceback.install()` makes uncaught exception traces much
    more readable -- recommend for development.
  - `pdb` / `ipdb` for interactive step-debugging at the failure
    site; `breakpoint()` (PEP 553) is the stdlib entry point.

### Log before the failure, not after

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. Python's `logging.debug` is the
  primitive; in pytest tests, use the `caplog` fixture to capture log
  output (or print to `sys.stderr` via `pytest -s`).

### Finding the test that pollutes state

- **Classification:** PORTS_AS_IS.
- **Python rendering:** pytest's `--collect-only` lists tests; bisect
  by selecting halves with `pytest tests/...[half_of_files]`. For
  order-dependent flakes, `pytest-randomly` is the canonical "shuffle
  with a recorded seed, replay to bisect" plugin -- the Python
  equivalent of Catch2's `--shuffle --rng-seed`.

### Reusing the mock clock in a debug harness

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same idea. The `contextvars`-backed clock
  from `design/cross-cutting.md` is the seam; install a mock clock
  driven from a debug script and replay the timing-dependent bug
  deterministically against the real binary.

### Low-perturbation observability

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. Python primitives:
  - `py-spy` for sampling-profile inspection of a live process
    (zero-instrumentation).
  - Sampled counters via `prometheus_client.Counter.inc()` -- cheap
    and out-of-band.
  - For event-stream debugging without log spam, an `asyncio.Queue`
    drained by a debug task.

## Suggested Python file: `python-debugging-principles/root-cause-tracing.md`

Verbatim port with the C++-specific tracing library calls swapped for
`traceback.format_stack` / `loguru` / `rich.traceback`, and a note on
`pytest-randomly` for shuffle-based bisection.
