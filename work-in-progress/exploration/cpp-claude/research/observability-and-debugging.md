# Observability and debugging tools (Python 2026)

## Sanitizers -> Python analogues

The C++ `sanitizers.md` ships per-tool build presets for asan, tsan, msan,
ubsan, filc. Python has no direct analogue (no UB, GC manages memory, no
data races at the C level under the GIL). The Python guide should replace
the chapter with the equivalent **runtime-checker stack** plus
**profilers**:

| C++ sanitizer | Python equivalent                                                                                                  |
|---------------|--------------------------------------------------------------------------------------------------------------------|
| asan          | Not applicable.                                                                                                    |
| ubsan         | Not applicable.                                                                                                    |
| msan          | Not applicable.                                                                                                    |
| tsan          | `PYTHONASYNCIODEBUG=1` for asyncio; `python -X dev` for general dev-mode warnings; `faulthandler` for native crashes. |
| filc / lsan   | `tracemalloc`, `memray` for memory leaks; `objgraph` for reference-cycle hunting.                                 |
| Static checks | pyright/pyrefly strict, ruff with the rule set in `ruff-rule-sets.md`.                                              |

## Profilers

| Tool         | When to reach for it                                                                                     |
|--------------|----------------------------------------------------------------------------------------------------------|
| `pyinstrument` | Sampling profiler with low overhead; default for everyday "where is the time going" inspection. Has an HTML report that reads well.    |
| `py-spy`     | Production-safe sampling profiler; attaches to a running PID without restart. Use for live diagnosis. |
| `memray`     | Memory profiler from Bloomberg; tracks every allocation including C extensions; can attach to a live process. The endgame for "what is holding this memory?". |
| `scalene`    | CPU + memory + GPU sampling profiler with line-level attribution. Heavier than pyinstrument but more granular.                            |
| `cProfile`   | Stdlib deterministic profiler. Use for unit-level micro-comparisons.                                                                       |

## Debug-mode toggles to recommend

- `python -X dev` -- enables `ResourceWarning`, `DeprecationWarning` by
  default, runs malloc in debug mode where supported.
- `PYTHONASYNCIODEBUG=1` -- slow callback warnings, never-awaited coroutine
  warnings, sync calls detection.
- `PYTHONTRACEMALLOC=N` -- track top N allocation frames.
- `faulthandler.enable()` -- prints a Python traceback on segfault.

## Logging

`loguru` (used in factors2) is a reasonable default for new projects.
`structlog` is the better choice when log lines need to be parsed by a
collector (Loki, Datadog, CloudWatch). The C++ "logger as cross-cutting
variant-backed global" pattern maps to: pick one logger module, configure
it once near `main`, expose narrow functions that downstream code calls.
Avoid threading a `Logger` instance through every constructor; the
stdlib `logging.getLogger(__name__)` pattern is the default escape hatch.

## Sources

- https://github.com/bloomberg/memray
- https://bloomberg.github.io/memray/
- https://docs.python.org/3/library/asyncio-dev.html
- https://docs.python.org/3/library/tracemalloc.html
