# Debugging guides: detailed outlines

For each proposed file: scope, h2 outline, cpp source mapping, factors2
patterns it cites, and the research notes that back it.

---

## `python-debugging-principles/root-cause-tracing.md`

Scope: trace the bad value backward through the call chain; fix at the root;
reinforce intermediate layers. Tools: traceback, breakpoint, pdb, loguru
backtraces, rich.traceback, pytest-randomly for order-dependent flakes.

Disposition: PORT_AS_IS.

cpp source: `cpp-debugging-principles/root-cause-tracing.md`. Claude
analysis: `cpp-claude/debugging/root-cause-tracing.md`. Codex analysis:
`cpp-codex/debugging/root-cause-tracing.md`.

### Section outline

1. **When to trace backward.** Same guidance as cpp: a bad value showed
   up somewhere, and the cheapest fix is the one nearest the source.
2. **Tracing procedure.** Observe -> immediate cause -> walk up the call
   chain -> root. Render the cpp "empty routing key" example with a
   `@dataclass` instead of a `struct`. Default-initialised empty string
   is the same trap.
3. **Instrumenting when manual tracing runs out.**
   - `traceback.format_stack()` for a one-shot dump.
   - `loguru.logger.opt(exception=True).debug(...)` or
     `@logger.catch` for automatic backtrace capture.
   - `rich.traceback.install()` for development-mode tracebacks.
   - `breakpoint()` (PEP 553) for interactive step-debugging.
4. **Log before the failure, not after.** Cite the cpp rule. In pytest,
   `caplog` captures emitted logs; `pytest -s` exposes stdout.
5. **Read chained exceptions.** `__cause__`, `__context__`,
   `__suppress_context__`. The chain is often the root-cause trail. Cite
   `cpp-codex/debugging/root-cause-tracing.md` and
   `factors-claude/patterns/error-handling.md` for the
   "log-and-re-raise loses the chain" anti-pattern.
6. **Finding the test that pollutes state.** `pytest --collect-only` +
   bisect. `pytest-randomly` shuffles with a recorded seed; replay with
   the seed to bisect order-dependent flakes. The Python equivalent of
   Catch2's `--shuffle --rng-seed`.
7. **`faulthandler` for stuck or crashing processes.** Enable in
   `__main__`; install on `SIGUSR1` so an operator can dump from outside.
8. **Reusing the mock clock in a debug harness.** The
   contextvars-backed clock from `cross-cutting-services.md` is the
   seam; replay a timing-dependent bug deterministically against the
   real binary.
9. **Low-perturbation observability.** `py-spy` for live-process sampling
   (zero instrumentation); `prometheus_client.Counter.inc()` for sampled
   counters; an `asyncio.Queue` drained by a debug task for event-stream
   debugging without log spam.
10. **Async-specific tracing.** Name async tasks (`asyncio.create_task(...,
    name="...")`); log request ids; treat cancellation as a control path,
    not a generic failure. Cite
    `cpp-codex/agent-context/cpp-agent-context.md` debugging additions.

Backing research: `cpp-claude/debugging/root-cause-tracing.md`,
`cpp-claude/research/observability-and-debugging.md`.

---

## `python-debugging-principles/defense-in-depth.md`

Scope: layered validation. Parse at the boundary; business-logic
validation; environment guards; debug instrumentation.

Disposition: PORT_AS_IS.

cpp source: `cpp-debugging-principles/defense-in-depth.md`. Claude analysis:
`cpp-claude/debugging/defense-in-depth.md`. Codex analysis:
`cpp-codex/debugging/defense-in-depth.md`.

### Section outline

1. **Core principle.** Make the bug class structurally impossible. The
   Python type-system half is weaker than cpp's at runtime, but the rule
   of thumb ports.
2. **Why one validation is not enough.** Layers defend against
   monkey-patching during tests, against direct attribute mutation, against
   stale caches.
3. **Types first: parse, don't validate.** Cross-link to
   `types-and-correctness.md`. The two-line summary: a
   `Symbol.parse(raw)` factory carries the check once; downstream code
   that takes `Symbol` as a parameter no longer re-validates.
4. **Layer 1: parse at the boundary.** Cite the
   `create_order(raw: RawOrderParams) -> Order` example. Pydantic
   alternative: define `RawOrderParams` as a `BaseModel` and let
   pydantic do the parsing on construction.
5. **Layer 2: business-logic validation.** Domain invariants the type
   system cannot express (balance check, universe check) live as plain
   function calls that raise on failure. Cite
   `factors-claude/patterns/state-invariants.md`'s
   `validate_step_result` for the named-predicate, one-rule-one-raise
   pattern.
6. **Layer 3: environment guards.** Gate on a settings flag or env var
   (`settings.TEST_MODE`, `settings.DRY_RUN`). Settings come from one
   well-known place; never consulted at module load.
7. **Layer 4: debug instrumentation.** `log.debug(...)` for entry state;
   `log.exception(...)` for the failure path. loguru's `@logger.catch`
   wraps the imperative-shell entry point.
8. **A worked example.** Config loader returning empty names -> pydantic
   factory with `EntryName.parse`; registry runs the runtime layers on
   top.
9. **Where validation belongs.**
   - `raise DomainError(...)` for caller-visible failures (the cpp
     `error_code::invalid_input` equivalent).
   - `assert condition, "explanation"` for programmer-error invariants.
     Note: `assert` is removed under `python -O`; do not rely on it for
     production behaviour.
   - `raise AssertionError(...)` or a custom `InternalError` exception
     for invariants that must hold even under `-O`.
10. **Where validation does not belong.** Not on the hot path; not after
    the type system already proves the property; not as a substitute for
    a proper boundary parse.

Backing research: `cpp-claude/debugging/defense-in-depth.md`,
`codex-research/generalizable-cpp-testing-debugging.md`.

---

## `python-debugging-principles/logging.md`

Scope: one logger per project (loguru recommended), severity discipline,
static message + structured kwargs, `logger.exception` in `except`,
throttling on hot paths.

Disposition: PORT_WITH_ADAPTATION.

cpp source: `cpp-debugging-principles/logging.md`. Claude analysis:
`cpp-claude/debugging/logging.md`. Codex analysis:
`cpp-codex/debugging/logging.md`.

### Section outline

1. **One logger per project.** loguru is the recommended default for new
   projects; stdlib `logging` with `logging.config.dictConfig` is the
   fallback for libraries that should not impose a logger on consumers;
   structlog is the alternative when key/value rendering is the primary
   need. Cite `factors-claude/anti-patterns.md` entry L (mixing stdlib
   `logging` and loguru is the failure mode).
2. **Severity discipline.** Stdlib has `DEBUG (10)`, `INFO (20)`,
   `WARNING (30)`, `ERROR (40)`, `CRITICAL (50)`. loguru adds `trace`
   and `success`. Document the level table from
   `factors-claude/patterns/logging-observability.md`:
   - `trace`: per-row diagnostics, never in production.
   - `debug`: per-call diagnostics, off by default in prod.
   - `info`: one line per significant business event.
   - `success`: optional operational milestones.
   - `warning`: recoverable degraded behaviour.
   - `error`: operation failed; caller sees it.
   - `critical`: process cannot continue.
3. **No true zero-cost disabled levels.** Python cannot compile out
   disabled calls. Mitigations:
   - `logger.debug("fmt %s", arg)` over f-strings (skips formatting
     when disabled).
   - `if logger.isEnabledFor(logging.DEBUG): logger.debug("...",
     expensive_dump(payload))` for expensive formats.
   - Module-level `_TRACE_ENABLED` constant for the truly hot case.
4. **Static message + structured kwargs.** `logger.info("downloaded
   file", screen=..., path=...)` over `logger.info(f"downloaded {screen}
   to {path}")`. Cite
   `factors-claude/patterns/logging-observability.md`: static =
   groupable, kwargs = filterable. Cite
   `factors-claude/anti-patterns.md` entry O (f-string-interpolated URL
   in the message kills grouping).
5. **`logger.exception(...)` inside `except`.** Includes the traceback
   automatically; attaches the exception object to the structured log.
   Cite `factors-claude/anti-patterns.md` entry M (`logger.error(f"...
   {str(e)}")` loses the traceback).
6. **Never log-and-re-raise.** Cross-link to `error-handling.md`. Pick
   one: propagate or `logger.exception(...)` and stop. To add context,
   `raise NewError(...) from e`.
7. **Throttled variants for hot call sites.** A throttle wrapper keyed
   on `(filename, lineno)` from `sys._getframe`. For high-rate
   production, push throttling to the collector side.
8. **Formatter support for domain types.** Implement `__str__` /
   `__repr__` on every domain dataclass; structured kwargs handle
   serialisation centrally.
9. **Configure logging once at startup.** Cite
   `cpp-claude/design/cross-cutting.md`. Tests install a no-op handler
   via the `caplog` fixture; in Dagster ops/assets, use `context.log`
   for asset-level events. Cite
   `factors-claude/patterns/logging-observability.md`.
10. **No `print` in `src/`.** ruff `T201` enforces. Cite
    `factors-claude/anti-patterns.md` entry C (35+ print calls).
11. **No `timeit.default_timer() + print` for timing.** Use a
    `with timed_block("..."): ...` context manager that emits one
    structured log line on exit; use `pyinstrument` / `pytest-benchmark`
    for profiling and microbenchmarks. Cite
    `factors-claude/anti-patterns.md` entry D.
12. **At trust boundaries, log AND notify the user.** Two different
    audiences, two different channels.

Backing research: `cpp-claude/debugging/logging.md`,
`factors-claude/patterns/logging-observability.md`,
`factors-claude/anti-patterns.md`.

---

## `python-debugging-principles/static-analysis-and-runtime-checks.md`

Scope: the Python equivalent of cpp sanitizers. pyright/pyrefly + ruff CI
gate; `python -X dev` + `-W error` for tests; tracemalloc / memray / py-spy
cookbook; suppression policy; free-threaded caveat.

Disposition: PYTHON_SPECIFIC.

cpp source: `cpp-debugging-principles/sanitizers.md`. Claude analysis:
`cpp-claude/debugging/sanitizers.md`. Codex analysis:
`cpp-codex/debugging/sanitizers.md`.

### Section outline

1. **What the cpp chapter was actually about.** Ship build presets that
   catch a class of defect deterministically, with version-controlled
   suppressions, triaged before silencing. That model ports; the tools
   change.
2. **Failure-class mapping.** Table:
   - use-after-free / dangling reference -> GC handles; not applicable.
   - uninitialised reads -> not applicable.
   - integer overflow -> arbitrary precision; not applicable.
   - data races -> under-GIL, Python-level races are impossible; under
     free-threaded 3.13t/3.14t, no analogue tool exists yet; rely on
     the ownership discipline.
   - memory leaks -> `tracemalloc`, `memray`, `objgraph`.
   - misuse of the typing system -> pyright/pyrefly strict; ruff `ANN` /
     `PLE` / `PYI` rules.
3. **The recommended Python "sanitizer-equivalent" stack.** Seven CI
   jobs:
   1. `pyright --strict` (or `pyrefly --strict`).
   2. `ruff check` with the rule set from
      `cpp-claude/research/ruff-rule-sets.md`.
   3. `ruff format --check --diff`.
   4. `python -X dev` + `pytest -W error` for the test run.
   5. `memray run` on the test suite in a nightly job, with a baseline.
   6. `pytest --collect-only` + `pytest-randomly` in a nightly job to
      detect order-dependent flakes.
   7. `py-spy dump` ready to run on production as an operational
      primitive (not a CI gate).
4. **Async diagnostics.**
   - `PYTHONASYNCIODEBUG=1` for slow callbacks, unawaited coroutines,
     accidental cross-thread access.
   - `asyncio.run(..., debug=True)` for the same in-process.
   - `anyio.run(main, backend_options={"debug": True})` for anyio.
   - `pytest-timeout` for hangs.
5. **Memory tools cookbook.**
   - `tracemalloc.start(); snapshot1 = tracemalloc.take_snapshot()` ...
     `snapshot2 = tracemalloc.take_snapshot(); top = snapshot2.compare_to(snapshot1, "lineno")`.
   - `memray run` produces a flamegraph; recommend the
     `--native --follow-fork` flags for the common case.
   - `objgraph.show_growth()` for the "what's leaking?" investigation.
6. **CPU profilers cookbook.**
   - `pyinstrument` for everyday CPU profiling; produces a readable
     tree.
   - `py-spy record -o out.svg -- python script.py` for sampling without
     instrumentation; `py-spy dump --pid` on a live process.
   - `cProfile` for stdlib-only environments.
   - `scalene` for line-level CPU+memory combined.
7. **Suppression policy.** Same discipline as cpp:
   - `# pyright: ignore[<rule>]` with a trailing comment naming what was
     triaged.
   - `# noqa: <rule>` for ruff suppressions, same comment rule.
   - `warnings.filterwarnings(...)` in a project-wide
     `warning_filters.py`, version-controlled.
   - Per-file ignores in `pyproject.toml`'s
     `[tool.ruff.lint.per-file-ignores]` and `[tool.pyright]` overrides
     for tests, scripts, generated code.
8. **Free-threaded caveat.** No TSan analogue. Mitigations: default to
   the owning-thread discipline; use `threading.Lock` explicitly around
   state shared across threads; run the test suite under `python3.13t
   -X gil=0` periodically to surface "I assumed GIL" mistakes; C
   extensions writing for free-threaded Python should use the upstream
   ASan/TSan for the extension build, not for Python.
9. **`faulthandler` for production-side crashes and hangs.** Enable in
   `__main__`; install on a signal so an operator can dump.
10. **Coverage discipline.** `pytest-cov` produces a report; CI gate on a
    documented threshold; reduce only via PR. Cite
    `factors-claude/tooling-baseline.md` for the gap (factors2 has
    pytest-cov but no gate).

Backing research: `cpp-claude/debugging/sanitizers.md`,
`cpp-claude/research/observability-and-debugging.md`,
`cpp-claude/research/ruff-rule-sets.md`,
`cpp-claude/research/typecheckers-2026.md`.

---

## Cross-cutting debugging rules

Apply across all debugging files; repeat in `root-cause-tracing.md` opener
and in the agent-context.

1. **Fix at the root, reinforce at the layers.** Patching the symptom
   without finding the source recurs.
2. **Reproduce before hypothesising.** A reproducer is the contract.
3. **One hypothesis at a time.** Multiple simultaneous fixes hide which
   one worked.
4. **Read chained exceptions.** `__cause__`, `__context__`.
5. **Treat cancellation as a control path.** Not a generic failure.
6. **No `print` in `src/`; no `time.sleep` in coroutines; no
   `logging.error(f"...{e}")` in `except`.** These are the recurring
   anti-patterns in the factors2 codebase and they all reduce diagnostic
   yield.
