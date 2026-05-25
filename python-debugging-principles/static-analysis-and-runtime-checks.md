# Static Analysis And Runtime Checks

Static analysis and runtime diagnostics turn broad defect classes into fast,
repeatable feedback. They complement root-cause tracing: checkers catch many
mistakes before execution, and diagnostic runs add evidence when behavior
depends on runtime state, async scheduling, native extensions, memory, or
warnings.

Expose these checks through the same command surface developers and CI use.
For Python projects in this guide set, that usually means `uv run` commands
and tool configuration in `pyproject.toml`.

## Static Analysis Baseline

Use Pyright as the default type checker. It gives strong feedback for public
APIs, domain values, adapters, protocols, discriminated unions, and reusable
helpers.

```sh
uv run pyright
```

Run mypy alongside Pyright when an existing project depends on it, when a
plugin covers important framework behavior, or when CI already treats mypy as
part of the contract.

```sh
uv run mypy src
```

Evaluate ty or pyrefly when fast whole-repository feedback matters. Treat a
checker change as a project decision: diagnostics, suppression comments,
editor support, standards conformance, and CI runtime all affect the team.

## Ruff As The Fast Defect Pass

Use Ruff for linting, import sorting, formatting, modernization, and common
bug-pattern checks.

```sh
uv run ruff check .
uv run ruff format --check .
```

The debugging-oriented rule families are the ones that catch misleading or
fragile code before it reaches runtime:

- `F`, `B`, `RUF`, `SIM`, `RET`, and `PERF` for common defects and confusing
  control flow.
- `ASYNC` for event-loop mistakes.
- `T20` for production `print` calls.
- `PT` for pytest patterns that affect test diagnosis.
- `TID` and `TCH` for import and type-checking hygiene.
- `S` for security-sensitive patterns, tuned to the project.
- `NPY` and `PD` for NumPy and pandas projects.

Per-file ignores should name a narrow file shape, such as tests that need
magic numbers or fixture arguments. Broad repository-wide ignores deserve the
same review as any other tool policy.

## Warnings As Diagnostics

Warnings are runtime evidence. Run tests in a mode that keeps project warnings
visible and escalates the categories the project treats as defects.

```sh
uv run pytest -W error::DeprecationWarning -W error::ResourceWarning
```

Use filters deliberately. A warning filter should name the category, module,
and reason whenever possible. Treat an unscoped ignore as temporary triage
work, not as a stable policy.

## Python Development Mode

`python -X dev` enables additional runtime checks and warning behavior. Use it
for diagnostic CI jobs, focused reproductions, and local investigation loops.

```sh
uv run python -X dev -m pytest tests
```

Development mode is useful for resource leaks, deprecations, encoding issues,
and other runtime signals that normal execution may keep quiet.

## Asyncio Debug Mode

Async failures often depend on scheduling. Enable asyncio debug mode when
diagnosing slow callbacks, never-awaited coroutines, leaked tasks, or blocking
work inside async paths.

```sh
PYTHONASYNCIODEBUG=1 uv run python -X dev -m pytest tests/async
```

Keep timeouts explicit in async tests. A timed-out test should name the
condition it waited for, not merely report that the event loop stalled.

## Fault And Memory Diagnostics

Use `faulthandler` for crashes, hangs, and native-extension failures. It can
dump Python stack traces when a process receives a fatal signal or after a
timeout.

```python
import faulthandler

faulthandler.enable()
faulthandler.dump_traceback_later(60, repeat=True)
```

Use `tracemalloc` for allocation origins inside Python code:

```sh
PYTHONTRACEMALLOC=25 uv run pytest tests/memory
```

Use memray when memory pressure involves native extensions, large dataframes,
or allocator-level questions:

```sh
uv run memray run -o profile.bin -m pytest tests/memory
uv run memray flamegraph profile.bin
```

Use py-spy when a process is already running or the profiler needs low
overhead attachment:

```sh
uv run py-spy top --pid 12345
uv run py-spy record --pid 12345 --output profile.svg
```

Use pyinstrument, scalene, or cProfile when the question is local CPU time and
the project exposes those tools through its command surface.

## Suppression Policy

A suppression is a triaged finding with a narrow scope. It should explain why
the checker, linter, warning, or diagnostic report is acceptable at that site.

```python
from typing import cast

payload = cast(OrderPayload, raw_payload)  # trusted test fixture builds this shape
```

Prefer local suppressions near the checked fact. Repeated suppressions around
the same dependency usually point to a typed adapter, local stub, parser, or
test helper. A suppression with no nearby reason sends the next debugging
session back to the beginning.

## Put Diagnostic Runs On A Schedule

Not every runtime diagnostic belongs in the normal fast loop. Keep the command
surface explicit:

```makefile
lint:
	uv run ruff check .

typecheck:
	uv run pyright

test:
	uv run pytest

test-dev:
	uv run python -X dev -m pytest

test-warnings:
	uv run pytest -W error::DeprecationWarning -W error::ResourceWarning

profile-memory:
	uv run memray run -o profile.bin -m pytest tests/memory
```

Run the fast checks on every change. Run heavier diagnostics before risky
changes land, on a schedule, or when the symptom points at the defect class
they expose.
