# Sanitizer presets

Overall disposition: PYTHON_SPECIFIC_VARIANT.

Rewrite as `tooling.md`. C++ sanitizers do not have a direct Python analogue.
The Python chapter should cover static analysis, linting, runtime diagnostics,
profilers, memory tools, async debugging, and fuzz/property testing.

## Ship a preset per sanitizer

Disposition: PYTHON_SPECIFIC_VARIANT.

Replace with stable tool commands, usually through uv:

```bash
uv run ruff check .
uv run ruff format --check .
uv run pyright
uv run pytest
```

If using pyrefly:

```bash
uv run pyrefly check
```

Keep separate CI jobs for lint, type checking, tests, coverage, and slow
diagnostic suites when runtime cost differs.

## Suppression files live next to the build, version-controlled

Disposition: PORTS_WITH_ADAPTATION.

Keep the policy. Suppressions and ignores belong in version-controlled config:
`pyproject.toml`, `pyrightconfig.json`, Ruff ignores, coverage config,
pytest markers, and tool-specific allowlists. Inline ignores need a reason.

```python
value = dynamic_library_call()  # type: ignore[unknown]  # third-party API is untyped
```

## Triage before suppressing

Disposition: PORTS_AS_IS.

Keep it. A type error, Ruff warning, flaky test marker, or coverage exclusion
is evidence first. Suppress only after understanding ownership and documenting
why the tool cannot be satisfied cleanly.

## ThreadSanitizer is special-cased

Disposition: PYTHON_SPECIFIC_VARIANT.

Replace with Python concurrency diagnostics:

- `PYTHONASYNCIODEBUG=1` and event-loop debug mode.
- `faulthandler` for stuck or crashing processes.
- pytest timeouts for hangs.
- py-spy for low-perturbation sampling.
- memray or tracemalloc for memory growth.
- thread dumps for deadlocks.

## Python-specific tooling catalog

Disposition: PYTHON_SPECIFIC_VARIANT.

Recommended sections:

- Static types: pyright or pyrefly.
- Lint and format: Ruff.
- Tests and coverage: pytest and coverage.py.
- Async diagnostics: asyncio debug mode, task names, cancellation logging.
- Runtime crash and hang evidence: faulthandler and signal-triggered dumps.
- Profiling: py-spy, pyinstrument, cProfile, pyperf.
- Memory: memray, tracemalloc, objgraph when appropriate.
- Native boundaries: ASan-enabled wheels or extension builds when the project
  owns C/C++/Rust extensions.
