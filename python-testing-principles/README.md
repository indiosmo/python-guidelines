# Python Testing Principles

Python tests should verify behavior, contracts, and invariants through
independent evidence. Pytest is the default runner. Test helpers should make
domain setup concise while keeping expected outcomes separate from the
implementation under test.

## Navigation

- [philosophy.md](philosophy.md) covers behavior-first testing, independent
  oracles, component scope, integration scope, determinism, public contracts,
  and focused feedback.
- [pytest-conventions.md](pytest-conventions.md) covers discovery, layout,
  names, assertions, parametrization, fixtures, marks, plugin policy, parallel
  runs, and command surfaces.
- [test-patterns.md](test-patterns.md) covers separate tests, parametrized
  cases, shared setup, dataframe assertions, validation tests,
  type-checker tests, and benchmarks.
- [test-helpers.md](test-helpers.md) covers domain factories, presets, fixture
  factories, dataframe builders, fakes, context-local providers, integration
  probes, and helper placement.
- [error-path-testing.md](error-path-testing.md) covers `pytest.raises`, stable
  message and attribute assertions, predicate tests, rollback, cleanup,
  boundary translation, warnings, and optional APIs.
- [condition-based-waiting.md](condition-based-waiting.md) covers predicate
  waits, monotonic clocks, events, queues, async timeouts, subprocess
  readiness, external effects, timing tests, and failure messages.
- [approval-tests.md](approval-tests.md) covers approval fit, pytest shape,
  deterministic rendering, scrubbers, scenario aggregation, review discipline,
  file placement, and direct assertions.

Start with `philosophy.md` for intent, then use the remaining files as
reference while writing or reviewing tests.
