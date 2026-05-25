# Python Debugging Principles

Debugging should find the origin of a bad value, state transition, missing
validation, or unexpected side effect. A change that hides the symptom is not
the same as a fix. Reproduce the failure, read the full traceback or diagnostic
output, state one hypothesis, test it with a focused experiment, then fix the
root cause.

## Navigation

- [root-cause-tracing.md](root-cause-tracing.md) covers exact symptoms,
  traceback reading, chained and grouped failures, backward value tracing,
  focused probes, flaky-test isolation, low-perturbation observability, and
  fixing the origin.
- [defense-in-depth.md](defense-in-depth.md) covers boundary parsing, domain
  validation, environment guards, assertions for internal impossibility,
  evidence preservation, and layer tests.
- [logging.md](logging.md) covers logging stack choice, event-shaped messages,
  severity, disabled-level cost, exception logging, volume control, and log
  assertions.
- [static-analysis-and-runtime-checks.md](static-analysis-and-runtime-checks.md)
  covers Pyright, optional mypy, Ruff, warnings, Python development mode,
  asyncio debug mode, fault diagnostics, memory diagnostics, suppression
  policy, and scheduled diagnostic runs.

Start with `root-cause-tracing.md` during an active failure. Use
`defense-in-depth.md` after the cause is known to close the bug class.
