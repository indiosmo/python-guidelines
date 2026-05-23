# Testing guides: detailed outlines

For each proposed file: scope, h2 outline, cpp source mapping, factors2
patterns it cites, and the research notes that back it.

---

## `python-testing-principles/philosophy.md`

Scope: behaviour-first testing, independent oracle, component vs integration.
Language-agnostic; ports almost verbatim.

Disposition: PORT_AS_IS.

cpp source: `cpp-testing-principles/philosophy.md`. Claude analysis:
`cpp-claude/testing/philosophy.md`. Codex analysis:
`cpp-codex/testing/philosophy.md`.

### Section outline

1. **Test behaviour, not implementation.** Rectangle and URL examples in
   Python.
2. **Understanding intent before writing tests.** Reading order: public
   API and docstrings first, callers second, implementation last. Python
   sources: API docs, OpenAPI schemas, pydantic models, CLI help, database
   constraints, public fixtures.
3. **Component vs integration.** pytest fixture scopes (`function` /
   `class` / `module` / `session`) map to the distinction. Default to
   `function`; expand only when justified.
4. **What an integration test should assert.** FastAPI handler translates
   a domain exception into the documented HTTP response; queue consumer
   commits an offset only after the side effect succeeds; repository and
   service agree on transaction rollback. Cite
   `cpp-codex/testing/philosophy.md` for the list.
5. **What component tests should not assert.** Already-covered scalar
   calculations; vendor SDK behaviour; framework internals.

Backing research: `codex-research/generalizable-cpp-testing-debugging.md`.

---

## `python-testing-principles/pytest-conventions.md`

Scope: collection rules, nodeids, marks, fixtures and scopes, pytest-xdist
and serial-test policy, plugin inventory, `uv run pytest` as the canonical
invocation.

Disposition: PYTHON_SPECIFIC.

cpp source: `cpp-testing-principles/catch2-conventions.md`. Claude
analysis: `cpp-claude/testing/catch2-conventions.md`. Codex analysis:
`cpp-codex/testing/catch2-conventions.md`.

### Section outline

1. **Test discovery.** pytest collects `test_*.py` under `testpaths`.
   `pythonpath = ["src"]` in `pyproject.toml` so imports read as
   `from package.x import y`. Cite `factors-claude/tooling-baseline.md`.
2. **Test selection by nodeid.** Examples:
   - `pytest tests/order_routing/` (by directory).
   - `pytest -k "router and not slow"` (by keyword).
   - `pytest -m smoke` (by mark).
   - `pytest tests/order_routing/test_router.py::test_route_new_order`.
3. **Marks.** All marks registered in `pyproject.toml`'s
   `[tool.pytest.ini_options].markers`. Otherwise pytest warns. Standard
   marks: `slow`, `serial`, `integration`, `smoke`.
4. **pytest-xdist and serialisation.** `@pytest.mark.serial` plus
   conftest mapping to `xdist_group("serial")` so serial tests pin to one
   worker.
5. **Assertions.** Plain `assert`. `pytest.raises(SomeError, match=...)`
   for exceptions. `pytest.approx(0.1)` for floating-point. No
   `unittest.TestCase` (the assertion rewriter does not work on it).
6. **One logical fact per test.** pytest reports the first failure;
   multiple `assert`s after the first are unreached. Use multiple short
   tests or `pytest-subtests` when "continue after failure" is needed.
7. **Parametrize ids.** `pytest.param(..., id=...)` per row for readable
   reports. Cite `cpp-codex/testing/test-helpers.md` for the recipe.
8. **Fixtures.** Default scope `function`. `yield` separates setup and
   teardown. `tmp_path`, `monkeypatch`, `caplog` are the stdlib-equivalent
   built-ins.
9. **Plugin inventory.** Recommend:
   - `pytest-benchmark` for hot-path correctness + regression.
   - `pytest-approvaltests` for snapshot tests.
   - `pyfakefs` for filesystem-touching code.
   - `pytest-cov` for coverage.
   - `pytest-asyncio` or `pytest-anyio` for async tests.
   - `pytest-randomly` for order-dependent flake detection.
   - `pytest-xdist` for parallelism.
10. **`uv run pytest` is the canonical invocation.** Same command in the
    Makefile, in pre-commit (if enabled), in CI. Cite
    `factors-claude/tooling-baseline.md`.
11. **What does not port.** Drop the cpp `RUN_SERIAL` / `TEST_PREFIX` /
    `CAPTURE` discussions; the Python equivalents are listed above.

Backing research: `factors-claude/tooling-baseline.md`,
`cpp-claude/research/ruff-rule-sets.md`.

---

## `python-testing-principles/test-patterns.md`

Scope: parametrize, fixtures, `assert_type` for type-only tests, table-driven
testing, subtests when needed.

Disposition: PORT_WITH_ADAPTATION.

cpp source: `cpp-testing-principles/test-patterns.md`. Claude analysis:
`cpp-claude/testing/test-patterns.md`. Codex analysis:
`cpp-codex/testing/test-patterns.md`.

### Section outline

1. **Pattern choice.** Catch2 -> pytest vocabulary table:
   `TEST_CASE` -> `def test_`; `SECTION` -> fixture + multiple tests, or
   `pytest-subtests`; `GENERATE` -> `pytest.mark.parametrize`;
   `TEMPLATE_TEST_CASE` -> parametrize over the type.
2. **One scenario per `def test_`.** Default. The pytest report names the
   scenario; the test name becomes the spec line.
3. **Shared setup via fixture.** A `policy` fixture; separate tests for
   each branch.
4. **Parametrize for scalar inputs.** Cite the whitespace-character example.
5. **Stacked parametrize for combinations.** Cite the TCP-connection
   example (two parametrize decorators -> four sub-cases).
6. **Parametrize over types.** Python's dynamic typing makes the
   cross-type test cheap; one parametrize over `[list, collections.deque]`
   beats two near-identical test bodies.
7. **`assert_type` for type-checker-only tests.** PEP 488 /
   `typing.assert_type`; pyright errors if the inferred type does not
   match. Use sparingly; only for stable contracts whose typing matters.
8. **Table-driven testing.** Parametrize with `ids=`; per-row
   `pytest.param(...)`. For wide tables (many expected fields), define a
   typed dataclass per row instead of a positional tuple. Cite
   `cpp-claude/testing/test-patterns.md`.
9. **Markdown-table-in-docstring scenarios.** For mid-complexity tests:
   table in the docstring; `# fmt: off` aligned data; expected values
   computed from the table. Cite
   `factors-claude/patterns/testing.md` for the recipe.
10. **Enumerate meaningful input shapes for pure functions.** Ascending,
    descending, constant, zero, mixed. Cite
    `factors-claude/patterns/testing.md` for the indicator-test pattern.
11. **What does not port.** Drop dependent-false `static_assert` workarounds.
    Drop the `if constexpr` template-specialisation testing pattern; Python
    parametrize over classes already covers it.

Backing research: `cpp-claude/testing/test-patterns.md`,
`factors-claude/patterns/testing.md`.

---

## `python-testing-principles/test-helpers.md`

Scope: factories with overrideable parameters, presets, fixtures with
yield-based teardown, probe modules for internal-state access, contextvars
providers.

Disposition: PORT_WITH_ADAPTATION.

cpp source: `cpp-testing-principles/test-helpers.md`. Claude analysis:
`cpp-claude/testing/test-helpers.md`. Codex analysis:
`cpp-codex/testing/test-helpers.md`.

### Section outline

1. **`_make_valid_X()` factories per test file.** Cite
   `factors-claude/patterns/testing.md`: one factory per type, accepts
   kwargs for what varies, returns a fully-validated object. Tests mutate
   one field and assert one outcome.
2. **Shared factories live in `tests/<package>/util.py`.** Cite
   `factors-claude/patterns/testing.md` for the per-subpackage `util.py`
   shape.
3. **Two factory styles.** Either:
   - Keyword-only dataclass with sentinel defaults + left-to-right merge
     (cpp-claude analysis recipe), OR
   - `def make_request(**overrides): return replace(default, **overrides)`
     (codex recipe; shorter, slightly more dynamic).
   Recommend the second for new code; the first when the param surface
   is wide enough that explicit field discovery pays.
4. **When not to create a factory.** A trivial wrapper around a dataclass
   constructor is noise; just use the dataclass.
5. **File-local wrappers.** Module-level private function (leading
   underscore) inside the test file. No `namespace { ... }` ceremony.
6. **Test data rules.**
   - Deterministic; no `random.*` without explicit seeding.
   - Descriptively named (`Final[int]` module-level constants for named
     values).
   - Boundary-aware: `@pytest.mark.parametrize` for (below, at, above) triples.
7. **Fixtures.** `yield` separates setup and teardown; scope chooses
   lifetime; default `function`; expand only when justified. The
   `library` fixture example from `cpp-codex/testing/test-helpers.md`.
8. **Probe modules for private-state access.** Python has no friend
   mechanism; tests can reach `_private` attributes directly. The probe
   pattern: a `tests/probes/lru_cache_probe.py` with explicit per-line
   `# pyright: ignore[reportPrivateUsage]` so intent is visible. Cite
   `cpp-claude/testing/test-helpers.md` for the recipe.
9. **Direct state injection.** Use the probe to inject prerequisite state
   and exercise only the branch under test, instead of 80 lines of setup
   ceremony.
10. **Test providers.** Two shapes:
    - For contextvars-backed services: `token = svc._var.set(impl); yield
      svc._var.get(); svc._var.reset(token)`.
    - For module-level provider attributes: `monkeypatch.setattr(...)` in
      a fixture.
    Cite `cpp-claude/testing/test-helpers.md` and
    `cpp-codex/testing/test-helpers.md`.
11. **No `cast(...)` in tests.** Banned as a substitute for a constructor.
    Write `WeightedSignals.empty()` instead. Cite
    `factors-claude/anti-patterns.md` entry G.
12. **No `except Exception` in tests.** Anything that needs catching needs
    to be caught by class.

Backing research: `cpp-claude/testing/test-helpers.md`,
`factors-claude/patterns/testing.md`.

---

## `python-testing-principles/error-path-testing.md`

Scope: `pytest.raises`, exact-message via `re.escape`, rollback assertions,
optional Result-style helper.

Disposition: PORT_WITH_ADAPTATION.

cpp source: `cpp-testing-principles/error-path-testing.md`. Claude analysis:
`cpp-claude/testing/error-path-testing.md`. Codex analysis:
`cpp-codex/testing/error-path-testing.md`.

### Section outline

1. **Default: `pytest.raises`.** Cite the `parse_int` example. The
   `match=` argument accepts a regex; the structured-exception payload is
   inspected via `exc_info.value.<field>`.
2. **Validation messages are part of the contract.** Use
   `pytest.raises(ValueError, match=re.escape("StepResult validation
   failed: mark_to_market_nav must be finite"))`. Cite
   `factors-claude/patterns/testing.md` for the
   `test_validate_step_result.py` pattern: one test per rule, exact
   message via `re.escape(...)`.
3. **Test the rollback, not just the raise.** After the `pytest.raises`
   block, assert the state has not been mutated: `assert bad_entry.id not
   in r.entries`. Cite `cpp-claude/testing/error-path-testing.md` for the
   anti-example (asserting cleanup via a follow-up "good" call -- that
   does not prove cleanup ran).
4. **Exception groups via `except*`.** For TaskGroup-style failures;
   `pytest.raises(BaseExceptionGroup) as exc_info` plus `match` on each
   member.
5. **Optional Result-style testing.** Provide an `unwrap_ok(...)` helper
   for the `Ok` path; `match` on the variant for the `Err` path. Document
   but do not require.
6. **Anti-patterns.**
   - `pytest.raises(Exception)`: too broad; catches programmer errors.
   - Asserting on `str(exc)` without `match=`: drifts silently when the
     message changes.
   - Multiple `assert`s after `pytest.raises`: only the first is reached
     on failure.

Backing research: `cpp-claude/testing/error-path-testing.md`,
`factors-claude/patterns/testing.md`,
`factors-claude/patterns/error-handling.md`.

---

## `python-testing-principles/condition-based-waiting.md`

Scope: wait for the condition, not for a guess about how long it takes.
Sync `wait_for`, async `anyio.fail_after`, the time.sleep-in-coroutine trap.

Disposition: PORT_AS_IS.

cpp source: `cpp-testing-principles/condition-based-waiting.md`. Claude
analysis: `cpp-claude/testing/condition-based-waiting.md`. Codex analysis:
`cpp-codex/testing/condition-based-waiting.md`.

### Section outline

1. **When to use this pattern.** Tests for asynchronous events: a worker
   thread, an event loop, a network round-trip, a background task in a
   pytest fixture.
2. **The sync `wait_for` helper.** Predicate, description, timeout,
   poll interval; raise `TimeoutError` on expiry. Cite
   `cpp-claude/testing/condition-based-waiting.md` for the recipe.
3. **The async `wait_for_async` helper.** `anyio.fail_after(timeout)` +
   `await anyio.sleep(poll_interval)`. Composes with cancel scopes.
4. **Condition variables for producer/consumer.** `threading.Event` for
   simple flags; `threading.Condition.wait_for` for predicate waits;
   `asyncio.Event` for in-loop; `anyio.Event` for cross-backend.
5. **When an arbitrary sleep is correct.** Document the
   "wait-for-trigger, then sleep the documented duration" recipe.
6. **The `time.sleep`-in-coroutine trap.** `time.sleep` blocks the event
   loop and breaks every other task. ruff `ASYNC101` flags this. Use
   `anyio.sleep` / `asyncio.sleep` in async code.
7. **Common mistakes.** Hardcoded sleeps that pass on the dev's laptop
   and fail in CI; unbounded `while True` polling; sharing a poll
   primitive across tests.

Backing research: `cpp-claude/testing/condition-based-waiting.md`.

---

## `python-testing-principles/approval-tests.md`

Scope: when ApprovalTests fits, the `verify(...)` recipe, determinism via
scrubbers, dataframe stabilisation, review workflow.

Disposition: PORT_AS_IS.

cpp source: `cpp-testing-principles/approval-tests.md`. Claude analysis:
`cpp-claude/testing/approval-tests.md`. Codex analysis:
`cpp-codex/testing/approval-tests.md`.

### Section outline

1. **When ApprovalTests fits.** Many parallel scenarios; a generated
   text artifact (SQL, JSON, rendered HTML); a multi-block diff that
   would otherwise be many small asserts. Cite
   `factors-claude/patterns/testing.md` for the SQL-query-builder example
   in factors2.
2. **When it does not fit.** A single scalar; a narrow assertion; data
   that legitimately drifts.
3. **Writing the test.** `from approvaltests import verify, Options`.
   factors2 wires `--approvaltests-use-reporter=PythonNative` as the
   default reporter; cite `factors-claude/tooling-baseline.md`.
4. **Determinism is the contract.** Scrubbers for dates, GUIDs, paths,
   ephemeral IDs. `combine_scrubbers(...)` to layer.
5. **DataFrame stabilisation.** Sort rows on a domain key; format floats
   with a fixed precision; `verify_as_json(...)` for the stable JSON
   formatter.
6. **Aggregating scenarios into one approval.** `"\n".join(blocks)`;
   labels per block; one `.approved.txt` file is the spec.
7. **Reviewing approval diffs.** The `.approved.txt` is part of the
   source tree; review it as part of the PR.
8. **`pytest-approvaltests` integration.** Received files surface in the
   pytest report; recommend the integration.

Backing research: `cpp-claude/testing/approval-tests.md`,
`factors-claude/patterns/testing.md`.

---

## (conditional) `python-testing-principles/web-ui.md`

Scope: Playwright-driven browser tests, page-object pattern, snapshot tests,
headless/headed configuration.

Disposition: NEW (conditional). Add only if the target codebase has a web UI
or HTTP API surface large enough to need it. The factors2 codebase uses
Streamlit and Dagster UIs; this file is not required for the first pass.

cpp source: `cpp-testing-principles/qt-gui.md` (dropped).

### Section outline (if written)

1. **When this file fits.** A FastAPI / Starlette / Django HTTP surface;
   a Streamlit / Dagster UI; a client-rendered frontend.
2. **HTTP API tests via `httpx.AsyncClient` against a `TestClient`-mounted
   app.** No browser required.
3. **Browser tests via Playwright.** Page-object pattern; `expect(locator)`
   waits; headless in CI, headed in dev.
4. **Visual regression.** `pytest-playwright-snapshot`; review as part of
   the PR.
5. **Streamlit-specific.** `streamlit testing` framework; treat the UI
   handler as a thin adapter and test the underlying domain function
   directly.

Backing research: `cpp-claude/testing/qt-gui.md`.

---

## Cross-cutting testing rules

These apply across all of the above files; recommend repeating them in the
`philosophy.md` opener and in the agent-context.

1. **One file per source file.** `tests/<pkg>/<...>/test_<name>.py` matches
   `src/<pkg>/<...>/<name>.py`. Cite
   `factors-claude/patterns/testing.md`.
2. **One assertion per logical fact.** pytest reports the first failure;
   multiple asserts after the first are unreached.
3. **Determinism is the contract.** Seed randomness; freeze time via a
   contextvars-backed clock; control filesystem with `tmp_path` /
   pyfakefs; control environment with `monkeypatch.setenv`.
4. **Validation messages are part of the contract.** Test with
   `re.escape`, not `in str(e)`.
5. **No `except Exception` in tests.** Catch by class.
6. **No `cast(...)` in tests.** Write a constructor instead.
7. **Global library modes go in `__init__.py`, not in `conftest.py`.**
   Cite `factors-claude/anti-patterns.md` entry N.
8. **Benchmarks live in `benchmarks/`.** Pytest-discoverable; one
   Makefile target.
9. **Approval `.approved.txt` files are tracked in git.**
