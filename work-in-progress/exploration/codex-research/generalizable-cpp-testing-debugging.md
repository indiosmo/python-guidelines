# Generalizable C++ Testing and Debugging Guidance for Python

Research notes for porting general testing and debugging principles from the C++ guidelines into modern Python guidance. Target Python stack: pytest, uv, ruff, and static type checking.

This is not a full guide draft.

## Source files reviewed

Local Python documentation:

- `/home/msi/repos/python-guidelines/README.md`
  - Empty at time of review.
- `/home/msi/repos/python-guidelines/MONOREPO.md`
  - Establishes root-level tooling conventions and strict separation between infrastructure, source code, and deployable units.
  - Assumes root workspace configuration for uv, ruff, pytest, and mypy or equivalent static type checking.

C++ testing principles:

- `/home/msi/repos/cpp-guidelines/cpp-testing-principles/README.md`
- `/home/msi/repos/cpp-guidelines/cpp-testing-principles/philosophy.md`
- `/home/msi/repos/cpp-guidelines/cpp-testing-principles/catch2-conventions.md`
- `/home/msi/repos/cpp-guidelines/cpp-testing-principles/test-patterns.md`
- `/home/msi/repos/cpp-guidelines/cpp-testing-principles/test-helpers.md`
- `/home/msi/repos/cpp-guidelines/cpp-testing-principles/error-path-testing.md`
- `/home/msi/repos/cpp-guidelines/cpp-testing-principles/condition-based-waiting.md`
- `/home/msi/repos/cpp-guidelines/cpp-testing-principles/approval-tests.md`
- `/home/msi/repos/cpp-guidelines/cpp-testing-principles/qt-gui.md`

C++ debugging principles:

- `/home/msi/repos/cpp-guidelines/cpp-debugging-principles/README.md`
- `/home/msi/repos/cpp-guidelines/cpp-debugging-principles/root-cause-tracing.md`
- `/home/msi/repos/cpp-guidelines/cpp-debugging-principles/defense-in-depth.md`
- `/home/msi/repos/cpp-guidelines/cpp-debugging-principles/logging.md`
- `/home/msi/repos/cpp-guidelines/cpp-debugging-principles/sanitizers.md`

## Candidate Python guide topics

- Testing behavior and contracts instead of implementation details.
- Choosing pytest test shapes: separate tests, parametrized tests, fixtures, and helper functions.
- Deterministic test data and avoiding hidden randomness, wall-clock dependence, generated identifiers, filesystem ordering, and environment leakage.
- Table-driven tests with `pytest.mark.parametrize`, row labels via `ids=`, and explicit expectation objects when rows become wide.
- Testing success paths, failure paths, and rollback or cleanup behavior.
- Helper and fixture design that keeps scenario-specific details visible.
- Snapshot or approval-style tests for large structured output.
- Predicate-based waiting for async, multiprocessing, file, subprocess, and externally visible conditions.
- Test isolation for process-wide providers: environment variables, logging configuration, global registries, clocks, feature flags, caches, and temporary paths.
- Root-cause debugging loop before code changes.
- Debugging flaky tests by reproducing, isolating order dependence, and identifying leaked state.
- Defense in depth with typed boundary parsing, business validation, environment guards, and observability.
- Logging and observability conventions that work in tests and production.
- Python-specific static checks: type checker, ruff, pytest suite, and targeted commands through uv.

## Transferable testing principles

### Behavior-first tests

- A test should encode an independent statement of expected behavior, based on the domain contract, not on the implementation.
- Avoid tautological tests that recompute the expected value using the same formula or helper logic as production code.
- Expected values should come from specifications, documented examples, domain rules, or clearly stated invariants.
- Read order before writing tests:
  - Interface and type signatures first.
  - Surrounding callers and domain context next.
  - Implementation last, as the thing under test rather than the source of truth.
- Useful bug targets transfer directly to Python:
  - Boundary conditions: empty input, zero, min and max values, inclusive and exclusive limits.
  - State transitions: invalid transitions, double transitions, missing guards.
  - Error paths: raised exceptions, returned error values, partial mutation, cleanup.
  - Implicit assumptions: sorted input, non-empty sequences, valid encoding, present environment variables.
  - Combinatorial interactions: mode by option by input type.
  - Semantic correctness: whether a method leaves state in the promised shape.

### Component and integration scope

- Component tests should isolate a function, class, or module contract.
- Integration tests should assert behavior that no single component test can prove:
  - Data crosses a real boundary without corruption.
  - State remains consistent across components.
  - Errors propagate into the expected rollback, rejection, or notification.
  - Side effects are triggered and observable at the receiving boundary.
  - Unexpected input is absorbed or rejected without corrupting state.
- Avoid integration tests that simply duplicate assertions already covered by component tests.

### Deterministic tests

- Tests should be reproducible under uv, locally and in CI.
- Test data should be deterministic. Avoid uncontrolled randomness; when randomized or property-based testing is intentional, control and report the seed.
- Approval and snapshot tests require deterministic rendering:
  - Sort unordered collections before rendering.
  - Replace wall-clock times, UUIDs, generated IDs, absolute paths, and platform-specific separators with stable placeholders.
  - Normalize line endings and path separators.
- Prefer deterministic clocks, injected ID factories, temporary directories, and explicit environment setup over relying on process state.
- Use pytest fixtures such as `tmp_path` and `monkeypatch` to localize filesystem and environment effects.

### Error-path tests

- Cover both success and failure behavior for fallible operations.
- In Python, failure behavior usually means:
  - `pytest.raises` for exception-throwing APIs.
  - Explicit assertions against result objects, `None`, sentinel values, or domain error types where the project uses them.
- For exception tests, assert the exception type and, when useful, the stable part of the message or attributes. Avoid overfitting on incidental formatting.
- When code mutates state and then fails, assert direct post-call state:
  - The partially inserted item is absent.
  - The temporary file was removed.
  - The transaction was rolled back.
  - The in-memory registry was restored.
- Do not infer cleanup by making a later successful call. That tests the later success path, not the failed operation's cleanup.

### Helper and fixture design

- Test bodies should show only the data that distinguishes the scenario.
- Put repetitive construction behind factories only when the factory provides stable defaults, composes meaningful presets, or derives related objects from a source.
- Do not wrap simple constructors or dataclass initializers with helpers that add no meaning.
- In Python, good factory shapes include:
  - Dataclasses or typed dictionaries for override parameters.
  - Keyword-only factory functions with stable defaults.
  - Project-specific builders where object graphs are large.
  - Small file-local wrappers for one test module's setup variation.
- Use domain vocabulary in helper names and fixtures. Avoid generic names like `obj`, `data`, `helper`, or `manager` when the domain has better terms.
- Keep pytest fixtures focused:
  - Use fixtures for expensive or shared setup and teardown.
  - Prefer explicit fixture names that reveal the state being provided.
  - Avoid autouse fixtures unless they enforce a broad invariant for the whole test module or suite.
- If a test needs internal state to assert hygiene, prefer an intentional test-only observation point over contorting the public workflow. In Python this may be a module-local helper, a protected test hook, or inspection of documented internal state when the project accepts that convention.

### Approval and snapshot tests

- Approval-style tests transfer well to Python for large structured output:
  - Pretty-printed reports.
  - JSON, XML, or YAML serialization.
  - CLI output.
  - Generated code or configuration.
  - Log formatting.
  - Legacy characterization before refactoring.
- Use direct assertions for scalar values, short strings, and narrow behavior.
- Approval files should be small enough for reviewers to read. If reviewers cannot understand every changed line, split the output or replace it with narrower assertions.
- The approved file is the expected value. Treat changes to it as deliberate, reviewable behavior changes.
- Prefer canonical serialization and scrubbing before comparison instead of editing approved files manually.
- Python translation options:
  - Snapshot plugins for pytest where already adopted by the project.
  - Plain expected files loaded from test resources.
  - Inline expected strings only when short and readable.
- Caveat: avoid making the guide depend on a specific snapshot plugin unless the Python guidelines standardize on one.

### Condition-based waiting

- Wait for the observable condition under test, not an guessed duration.
- A reusable Python wait helper should poll a predicate until it becomes true or a timeout expires, and the timeout error should name the condition.
- Use this for:
  - Async tasks finishing.
  - Files appearing.
  - Subprocesses becoming ready.
  - Queues reaching a count.
  - Messages or callbacks arriving.
- For `asyncio`, prefer event, queue, or task synchronization with `asyncio.wait_for` over sleep loops.
- For threaded tests, prefer `threading.Event`, `Condition`, or queue operations with timeouts when the producer can signal completion.
- Fixed sleeps are appropriate only when the timing behavior itself is under test. In that case, first wait for the timer or worker to start, then sleep for a duration derived from the documented timing.
- Common Python caveats:
  - Always set per-test timeouts to avoid hanging CI.
  - Re-read state inside the predicate instead of capturing a stale value.
  - Do not poll too fast; short intervals can burn CPU and perturb timing.

### Test readability

- Use the simplest pytest structure that avoids duplication:
  - Separate tests when setup or assertions differ meaningfully.
  - Parametrize when the same logic covers several rows.
  - Fixtures when setup and teardown are nontrivial and shared.
  - Helper functions when construction details distract from behavior.
- Parametrized tests should identify failing rows clearly:
  - Use `ids=` or row objects with descriptive labels.
  - Include scenario and expected outcome fields when both matter.
  - Split wide tables into named input and expectation structures rather than long tuples.
- Assertions should fail at the most useful point:
  - Use precondition assertions before dereferencing or indexing a value needed by later checks.
  - Use multiple focused assertions when several independent observations should be reported.
  - Use assertion messages sparingly; pytest's assertion rewriting already explains many failures.

## Transferable debugging principles

### Root-cause tracing

- No fix before root-cause investigation. Making a test green is not proof that the underlying bug was fixed.
- Use a four-phase debugging loop:
  - Investigate the root cause.
  - Analyze patterns and working examples.
  - State and test one specific hypothesis.
  - Implement the root-cause fix and verify it.
- If a fix attempt fails, return to investigation with the new evidence instead of stacking another guess on top.
- Trace bad values backward through the call chain:
  - Capture the exact symptom: file, line, exception, value, and operation.
  - Read the failing code and decide whether it behaved correctly given the bad input.
  - Walk to the caller and identify where the bad value came from.
  - Continue until the origin is found.
  - Fix the origin and reinforce intermediate boundaries as needed.
- Python translation:
  - Read the full traceback, including chained exceptions and earlier frames.
  - Use logging, breakpoints, `pytest -k`, `pytest -x`, and focused reproductions to narrow the path.
  - Use `git log`, diffs, and bisecting recent changes when the bug appeared recently.
  - For flaky order dependence, use pytest ordering or randomization plugins only if available; otherwise bisect by running subsets and checking leaked state.

### Defense in depth

- A one-site fix removes a symptom. A defense-in-depth fix prevents the bug class from returning through another path.
- General validation layers transfer to Python:
  - Parse at the boundary: convert raw JSON, CLI arguments, environment values, request bodies, and file contents into typed domain objects early.
  - Business validation: check operation-specific rules that field shape alone cannot express.
  - Environment guards: protect production resources from tests, dry runs, local scripts, and maintenance modes.
  - Debug instrumentation: log enough context at boundaries and failure points for the next investigation.
- Python type hints cannot enforce all invariants at runtime. Use static types to make correct code easier to write, but keep runtime parsing and validation for untrusted input.
- Use assertions for programmer errors and explicit exceptions or error results for invalid external input.
- Test each validation layer directly. For example, construct a typed-but-business-invalid object in a unit test and verify business validation rejects it even though boundary parsing is bypassed.

### Logging and observability

- Portable logging principles:
  - Use a small, consistent severity family: debug, info, warning, error, critical; use lower-level trace only if the project standardizes it.
  - Log domain values directly through meaningful formatting instead of forcing each call site to unwrap or stringify internals.
  - Put debug logs before operations likely to fail so the log captures the decision state, not just the failure.
  - Tests should be able to capture, silence, or assert logs without mutating global logging state for later tests.
- Python translation:
  - Prefer the standard `logging` module unless the project has a chosen structured logger.
  - Use pytest's `caplog` for log assertions.
  - Restore logging configuration after tests that mutate handlers, levels, or filters.
  - Avoid noisy hot-path logging; use throttling, sampling, counters, or structured events where high-volume observability is needed.
- Caveat: C++ compile-time log-level elimination does not translate directly to Python. Python guidance should focus on level checks, lazy formatting, structured context, and avoiding expensive argument construction at disabled levels.

## C++-specific material to omit or adapt heavily

- Catch2 constructs, CTest registration, `catch_discover_tests`, `RUN_SERIAL`, Catch2 tags, `REQUIRE` and `CHECK` mechanics.
  - Python replacement: pytest test discovery, markers, parametrization, fixtures, and command-line selection.
- C++ compile-time tests with `static_assert`, concepts, variant visitor exhaustiveness, and template test cases.
  - Python replacement: type-checker expectations, protocol checks, targeted runtime tests, and exhaustive tests over enums or literals where useful.
- Friend-based `test_probe<T>` and private member access patterns.
  - Python replacement: intentional test hooks or direct inspection only when project conventions allow it; do not encourage casual coupling to private implementation.
- RAII restore guards and destructors.
  - Python replacement: pytest fixtures with `yield`, context managers, and `monkeypatch`.
- Scope guards for rollback.
  - Python replacement: context managers, `try`/`except` cleanup, database transactions, temporary paths, and commit-at-end patterns.
- Qt-specific GUI testing, `QApplication`, `QSignalSpy`, offscreen platform behavior, model/view tester, and Qt event-loop pumping.
  - Omit unless a future Python GUI guide has a target framework such as Qt for Python.
- C++ sanitizer preset guidance.
  - Python replacement should not mention ASan, TSan, MSan as a general Python default. Adapt the principle as "specialized diagnostic runs belong in reproducible project commands" for tools such as type checking, ruff, pytest, coverage, dependency audit, and framework-specific debug checks.
- C++ build-directory and suppression-file rules.
  - Omit from Python testing/debugging guide unless discussing compiled extensions.
- Variant-backed global services.
  - Python replacement: dependency injection, context managers, fixtures, and explicit app startup wiring.

## Python-specific translation notes and caveats

- Use uv commands in examples and verification snippets:
  - `uv run pytest`
  - `uv run pytest path/to/test_file.py -k "case name"`
  - `uv run ruff check`
  - `uv run mypy` or the project's selected static type checker command.
- The root monorepo pattern suggests tooling should target source directories deliberately, not infrastructure or generated artifacts.
- Prefer pytest-native tools:
  - `pytest.mark.parametrize` for tables.
  - `tmp_path` for filesystem isolation.
  - `monkeypatch` for environment and attribute replacement.
  - `caplog` for logging.
  - `capsys` or `capfd` for stdout and stderr.
  - `pytest.raises` for exception paths.
  - `pytest.mark.slow`, `pytest.mark.integration`, or project-specific markers for selection.
- Static type checking is useful for API contracts but is not a substitute for runtime validation of external input.
- Python dictionaries preserve insertion order, but tests should still sort when order is not semantically meaningful. Otherwise tests encode incidental construction order.
- Hash randomization can affect sets, unordered renderings, and process-to-process output. Canonicalize before comparing text.
- Wall-clock time, time zones, locale, random seeds, current working directory, user home directory, network availability, and absolute paths are common sources of nondeterminism in Python tests.
- For async code, do not mix arbitrary sleeps with event-loop scheduling assumptions. Prefer explicit synchronization and timeouts.
- For subprocess readiness, wait on a real readiness signal: port open, health endpoint, file created, output line observed, or process exit.
- Tests that mutate process-global state should restore it with fixtures or context managers. This includes environment variables, working directory, logging configuration, warnings filters, timezone settings, random seed, global registries, cache directories, and singleton clients.
- When debugging in pytest, failing-fast and selection are important:
  - Reproduce one failure with the smallest command.
  - Use `-x` to stop at the first failure while investigating.
  - Use `-k` or a direct test node id to shrink the loop.
  - Add temporary logging or breakpoints only to collect evidence, then remove or convert durable observability into intentional logs.
- Approval tests in Python need explicit policy:
  - Where approved files live.
  - How received files are generated.
  - How scrubbers are shared.
  - How reviewers validate approved-file diffs.
  - Which snapshot library, if any, the project standardizes.
- Avoid importing the C++ guide's "compile-time zero cost" claims into Python. Python costs should be discussed in terms of disabled log levels, lazy formatting, test runtime, fixture scope, and keeping hot-path diagnostics bounded.

## Possible structure for later writing phase

- Testing philosophy: behavior, contracts, invariants, and domain-derived expected values.
- Pytest structure: tests, parametrization, fixtures, markers, and assertions.
- Test data and helpers: deterministic factories, presets, fixtures, and isolation.
- Failure and rollback tests: exceptions, result objects, cleanup, and state inspection.
- Approval and snapshot tests: fit, determinism, review discipline.
- Async and external waiting: predicate waits, timeouts, and fixed-sleep exceptions.
- Debugging loop: reproduce, trace, hypothesize, fix, verify.
- Defense in depth: typed parsing, runtime validation, environment guards, observability.
- Logging in tests and production: capture, restore, context, and volume control.
