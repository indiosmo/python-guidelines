# factors2 Antipatterns for Python Guidelines

These notes come from a read-only pass over `/home/msi/python_workspace/factors2` plus a side research pass on current Python practice. They are candidates for durable guideline material, not a final guide.

The filter here is deliberate: this file focuses on design, packaging, runtime, testing, and data-contract issues that Ruff does not normally solve. Ruff may catch nearby symptoms in some cases, but the guideline value is in the engineering rule, not the lint rule.

## Source Package Boundary Collapse

Antipattern: one Python project carries the core library, ETL sources, Dagster components, dbt project, dashboards, portfolio management app, backtest scripts, and every runtime dependency in one `[project.dependencies]` list.

Factors2 evidence:

- `pyproject.toml` defines a single project named `factors` with dependencies for Dagster, dbt, Streamlit, FastAPI, Ray, dlt, pandas, DuckDB, HTTP clients, and AWS clients.
- `src/` contains several import roots with different deployability profiles: `pyfactors`, `etl_factors`, `dagster_factors`, `dashboards`, and `dbt_factors`.
- This conflicts with the local repo philosophy in `MONOREPO.md`, which prefers isolated workspace packages and child `pyproject.toml` files for deployable units.

Why it matters: applications, libraries, dashboards, jobs, and orchestration code change and deploy at different rates. A collapsed dependency graph makes every environment heavier, makes security and upgrade reviews noisier, and hides which dependency belongs to which product surface.

Ruff coverage: no. Ruff can report unused imports or import style, but it does not reason about deployable units or dependency ownership.

Guideline candidate: split Python systems by deployable boundary. Core domain packages declare only domain runtime dependencies. ETL, orchestration, dashboards, and operational apps get their own package metadata or workspace member with their own dependency set.

Sources:

- Python Packaging User Guide, src layout: https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/
- PEP 735 dependency groups: https://peps.python.org/pep-0735/
- Dependency groups specification: https://packaging.python.org/en/latest/specifications/dependency-groups/

## Runtime Metadata Drift

Antipattern: project metadata, tool config, CI, and pre-commit each describe a slightly different project.

Factors2 evidence:

- `pyproject.toml` keeps a Black config while Ruff format appears to be the formatter.
- Runtime dependencies include `scipy-stubs`, which belongs with typing tools rather than application runtime.
- The Ruff configured include list covers `src`, `tests`, and `examples`, while `make lint` runs `ruff check .` and benchmarks live outside the configured include list.
- `.pre-commit-config.yaml` has two top-level `repos:` keys, so a normal YAML parser keeps only one of them.
- `.pre-commit-config.yaml` says `rev: 0.6.14  # Use the latest version`; the pinned revision is not latest by construction.
- GitHub Actions pins Ruff at `0.14.6`, while `pyproject.toml` allows `ruff>=0.13.0`.

Why it matters: developers stop trusting tool output when local commands, CI, pre-commit, and metadata disagree. The failure mode is slow: checks appear to pass while one environment silently skips files or runs a different tool version.

Ruff coverage: no. Ruff can lint Python files, but it does not validate the consistency of project metadata across CI, pre-commit, Makefiles, and dependency groups.

Guideline candidate: make one command surface authoritative. CI, pre-commit, docs, and Makefile targets should run the same commands against the same file scopes and tool versions. Keep tool-only packages in dependency groups, not runtime dependencies.

Sources:

- Ruff configuration docs: https://docs.astral.sh/ruff/configuration/
- PEP 735 dependency groups: https://peps.python.org/pep-0735/
- PEP 751 lock files: https://peps.python.org/pep-0751/

## Generated Artifacts Inside Source Trees

Antipattern: generated caches, build metadata, and tool outputs live under source-like directories where readers and tools can mistake them for project inputs.

Factors2 evidence:

- `src/` contains many `__pycache__` directories.
- `src/factors.egg-info` is present.
- `src/dbt_factors/target` and `src/dbt_factors/logs` contain dbt generated outputs.

Why it matters: generated files increase search noise, confuse packaging and code-review diffs, and make it harder to tell which files are source of truth. In data projects, generated manifests can also make stale pipeline state look durable.

Ruff coverage: no. Ruff respects ignore rules, but it does not decide what should be generated, committed, or durable.

Guideline candidate: keep generated artifacts out of source trees unless the artifact is intentionally versioned input. Cache directories, package metadata, dbt targets, compiled Python files, and logs belong in ignored output locations.

Sources:

- Python Packaging User Guide, src layout: https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/
- dbt docs on target directory artifacts: https://docs.getdbt.com/reference/artifacts/dbt-artifacts

## Environment Reads in Business Paths

Antipattern: functions that perform domain or data work read environment variables, load `.env`, attach databases, or inspect proxy settings internally.

Factors2 evidence:

- `PostgresSettings.from_env` reads `PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`, and `PGDATABASE`.
- `dashboards.db.connect` calls `load_dotenv()` and builds Postgres settings inside the connection factory.
- `fetch_dataset` reads `HTTP_PROXY` and uses the process-wide `duckdb.sql` connection state.
- Several entry points call `date.today()` inside the work path.

Why it matters: environment reads are process-global inputs. When they appear deep in the call graph, tests become order-dependent, command-line tools become harder to compose, and callers cannot tell which inputs affect a result.

Ruff coverage: no. Ruff does not model configuration ownership or process-global input flow.

Guideline candidate: read environment, load dotenv files, choose clocks, and configure external services at process boundaries: CLI `main`, app factory, Dagster resource, test fixture, or worker initializer. Pass settings, connections, and clocks inward as explicit objects.

Sources:

- Python `__main__` docs: https://docs.python.org/3/library/__main__.html
- pytest monkeypatch docs: https://docs.pytest.org/en/stable/how-to/monkeypatch.html
- Python context variables docs: https://docs.python.org/3/library/contextvars.html

## Process-Global Mutable State as Dataflow

Antipattern: work is coordinated through mutable module globals, process-global library state, or implicit singletons instead of explicit inputs and outputs.

Factors2 evidence:

- `generate_one_pagers.py` uses `_shared_reference_data` and `_shared_universe` globals to feed child processes.
- `generate_one_pagers.py` mutates `pd.options.mode.copy_on_write` at module import.
- `dataset.fetch_dataset` and `load_indicator_portfolios` use `duckdb.sql`, which operates on DuckDB's module-level default connection.
- `backtest/example.py` is a notebook-style script with import-time runtime setup and execution cells in a `.py` file.

Why it matters: process-global state hides dependencies and makes concurrent execution fragile. It is especially hard to test because one test can leave state behind for the next test.

Ruff coverage: no. Ruff may flag some global-statement patterns, but it does not identify hidden dataflow through library global state.

Guideline candidate: keep dataflow explicit. Prefer parameters, returned values, context managers, dependency objects, and framework-owned resources. Use process-local or context-local state only when the runtime boundary requires it, and reset it in tests.

Sources:

- Python context variables docs: https://docs.python.org/3/library/contextvars.html
- pytest fixtures docs: https://docs.pytest.org/en/stable/explanation/fixtures.html

## Fork-Dependent Multiprocessing

Antipattern: multiprocessing correctness depends on fork inheriting large preloaded Python objects and mutable module globals.

Factors2 evidence:

- `generate_one_pagers.py` loads reference data and universe into globals, then starts `multiprocessing.Process` workers that read those globals.
- The script comments describe forked children inheriting shared data through OS copy-on-write.

Why it matters: fork inheritance is platform- and version-sensitive. Python 3.14 changed multiprocessing defaults away from `fork` as the default start method on POSIX, so code that assumes inherited globals can fail or become slow when it runs under `spawn` or `forkserver`.

Ruff coverage: no. Ruff does not analyze multiprocessing start-method assumptions.

Guideline candidate: make multiprocessing contracts explicit. Choose and document the start method, pass worker inputs intentionally, use an initializer when workers need shared resources, and test the path under the start method used in production.

Sources:

- Python multiprocessing start methods: https://docs.python.org/3/library/multiprocessing.html#contexts-and-start-methods

## Hidden Mutation of Nested Configuration

Antipattern: constructing or validating one model mutates a nested model supplied by the caller.

Factors2 evidence:

- `SimulationParams.model_post_init` appends a `DelistedSymbols` restriction to `self.signal_strategy.restrictions`.
- `SignalStrategy.restrictions` is a mutable list field.

Why it matters: callers may reuse a strategy object across simulations or tests. Hidden mutation makes object construction order observable and can duplicate derived rules after repeated validation.

Ruff coverage: no. Ruff catches some mutable default mistakes, but it does not know whether nested mutation is part of a domain contract.

Guideline candidate: treat user-supplied configuration as input. Build an effective configuration by copying, deriving, or normalizing explicitly. Prefer immutable models or immutable collections for long-lived strategy configuration.

Sources:

- Pydantic configuration docs: https://docs.pydantic.dev/latest/api/config/
- Pydantic strict mode docs: https://docs.pydantic.dev/latest/concepts/strict_mode/

## Dynamic Dispatch Without an Adapter Contract

Antipattern: runtime reflection decides how to call domain objects, while type ignores mark the dynamic boundary as acceptable.

Factors2 evidence:

- `ComputedIndicator.compute_rolling` inspects the signature of `self.function.compute_rolling` to decide whether to pass `reference_data`.
- `ComputedIndicator.deps` uses `hasattr` and `# type: ignore`.
- `indicator_portfolios_sql_from_md.py` builds an eval context by walking module exports with `dir()` and `getattr()`.

Why it matters: reflection can be useful at integration boundaries, but it makes supported call shapes implicit. New implementations can fail at runtime even though they type-check through the wrapper.

Ruff coverage: no. Ruff can flag some insecure `eval` usage with security rules, but it does not design the adapter contract.

Guideline candidate: put dynamic dispatch behind a named adapter boundary. Prefer a small `Protocol`, an explicit registry, or a normalized wrapper that gives all implementations the same public call shape.

Sources:

- Python typing protocols: https://typing.python.org/en/latest/reference/protocols.html
- Python typing best practices: https://typing.python.org/en/latest/reference/best_practices.html

## Graph Expansion Without Graph Invariants

Antipattern: recursive dependency expansion assumes acyclic, unique dependencies without naming those invariants in code.

Factors2 evidence:

- `extract_computed_indicators` expands computed-indicator dependencies depth-first and returns dependencies before dependents.
- The implementation does not appear to deduplicate shared dependencies or detect dependency cycles.

Why it matters: small dependency trees can tolerate simple recursion. As soon as indicators share dependencies or users can configure new ones, duplicate work and infinite recursion become user-facing failures.

Ruff coverage: no. Ruff cannot infer domain graph invariants.

Guideline candidate: when a dependency list becomes a graph, give nodes stable identities, deduplicate shared dependencies, reject cycles with a clear error, and test ordering behavior.

Sources:

- Python graphlib `TopologicalSorter`: https://docs.python.org/3/library/graphlib.html

## Unstructured Async Fan-Out

Antipattern: related async work is launched as a task list and awaited with `gather`, while cancellation, sibling failure, and lifecycle rules stay implicit.

Factors2 evidence:

- `ProxyDownloader.fetch` creates a task for every query and awaits `asyncio.gather(..., return_exceptions=True)`.
- The downloader has a semaphore and rate limiter, which is good, but structured cancellation and sibling-failure policy are not visible at the fan-out boundary.
- `ProxyDownloader.max_retries` is stored but `_fetch_one` has a hard-coded retry decorator with five attempts.

Why it matters: fan-out code must define what happens when one request fails, when the parent is cancelled, and when a callback fails. `return_exceptions=True` can turn task failures into data unless the caller handles them deliberately.

Ruff coverage: mostly no. Some optional async rules catch blocking calls in async functions, but structured concurrency policy is a design issue.

Guideline candidate: use `asyncio.TaskGroup` for related concurrent work when sibling failure should cancel the group. When partial failure is expected, make the result type explicit and keep retry, timeout, rate-limit, and cancellation policy in one place.

Sources:

- asyncio TaskGroup docs: https://docs.python.org/3/library/asyncio-task.html#task-groups

## Tabular Contracts Living Only in DataFrame Code

Antipattern: a module accepts or returns plain `DataFrame` values whose required columns, dtypes, sort order, index assumptions, and units are known only by convention.

Factors2 evidence:

- Factors2 often uses Pandera models well for pipeline boundaries, such as `Scores`, `Signals`, and `WeightedSignals`.
- Several script and app boundaries still move plain `DataFrame` values through operational code without a named schema.
- Dashboard chart modules and maintenance scripts rely on column conventions that are easy to break during query or transform changes.

Why it matters: pandas is dynamic. Without named boundary schemas, tabular contracts drift into comments, tests, and reviewer memory. Failures then happen after expensive computation or inside plotting/reporting code.

Ruff coverage: no. Ruff can catch some pandas-vet patterns, but it does not know dataframe schemas or business units.

Guideline candidate: validate tabular data at module and pipeline boundaries with a declared schema. Keep pandas internals flexible, but name the boundary contract in domain terms such as `trade_date`, `symbol`, `market`, `close`, `weight`, and `signal`.

Sources:

- Pandera docs: https://pandera.readthedocs.io/
- pandas testing helpers: https://pandas.pydata.org/docs/dev/reference/api/pandas.testing.assert_frame_equal.html

## Operational Scripts That Signal Failure by Printing

Antipattern: scripts print errors or warnings but return success, or continue after per-record failures without a summary failure policy.

Factors2 evidence:

- `etc/update_dataset.py` catches invalid date input, prints an error, and returns from `main()` without a nonzero exit status.
- `indicator_portfolios_sql_from_md.py` catches exceptions per block, prints an error, and still writes SQL for remaining rows.
- `brasilapi/fetch_cnpj.py` catches every exception per CNPJ, records it in output, and exits successfully.

Why it matters: operational scripts are often called from Makefiles, cron jobs, CI, or data jobs. A successful exit code means downstream automation can trust the output. If partial failure is valid, the script needs an explicit report and threshold.

Ruff coverage: partial only. Ruff can flag `print` if configured, but it cannot decide exit-code semantics.

Guideline candidate: command-line tools return nonzero on invalid user input and unrecoverable failures. If partial success is a feature, report counts, write machine-readable failure details, and make the acceptable failure threshold explicit.

Sources:

- argparse docs: https://docs.python.org/3/library/argparse.html
- Python `sys.exit` docs: https://docs.python.org/3/library/sys.html#sys.exit

## Unsafe Text-to-Code Maintenance Workflows

Antipattern: maintenance scripts parse semi-structured prose with regex and execute extracted Python expressions to generate SQL or configuration.

Factors2 evidence:

- `indicator_portfolios_sql_from_md.py` parses markdown code blocks with regular expressions.
- It calls `eval` for indicator expressions and numeric bound expressions.
- It builds SQL with string interpolation after escaping only single quotes in the JSON field.

Why it matters: maintenance scripts often graduate into production-adjacent workflows. Regex plus `eval` makes the accepted input language larger than intended, and string-built SQL makes escaping rules easy to miss.

Ruff coverage: partial. Security-oriented Ruff rules can flag `eval`, but they do not provide the replacement architecture.

Guideline candidate: use structured input for generated configuration: YAML, TOML, JSON, or a Python module imported deliberately as trusted code. Generate SQL through parameterized database APIs where values are data, not text.

Sources:

- Python `ast.literal_eval` docs: https://docs.python.org/3/library/ast.html#ast.literal_eval
- Python sqlite parameter substitution example: https://docs.python.org/3/library/sqlite3.html#sqlite3-placeholders
- OWASP SQL injection prevention cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html

## Brittle Numeric and DataFrame Tests

Antipattern: tests assert lists of floats, incidental ordering, or selected scalar cells when the contract is a whole table, tolerance, dtype, index, or invariant.

Factors2 evidence:

- Factors2 has good examples of domain-shaped tabular tests and approval tests for SQL.
- Some tests still assert long `.tolist()` values or selected scalar cells where a whole-output assertion would better document the contract.

Why it matters: numerical and dataframe code fails in shapes, dtypes, indexes, nullability, and ordering, not only scalar values. Tests that assert only selected cells can pass while a consumer breaks.

Ruff coverage: no.

Guideline candidate: for dataframe transforms, prefer `pandas.testing` helpers or schema-plus-invariant assertions. For floating-point results, use explicit tolerances through `pytest.approx`, `numpy.testing`, or pandas testing APIs.

Sources:

- pytest `approx`: https://docs.pytest.org/en/stable/reference/reference.html#pytest.approx
- pandas `assert_frame_equal`: https://pandas.pydata.org/docs/dev/reference/api/pandas.testing.assert_frame_equal.html

## Over-Mocking Domain Tests

Antipattern: tests verify private calls, patches, and plumbing instead of observable behavior.

Factors2 evidence:

- The strongest factors2 tests tend to use realistic domain dataframes, approval tests, and public behavior.
- Keep this as a guideline because code-generation tools often introduce mock-heavy tests when they do not understand the domain model.

Why it matters: over-mocked tests can pass while real integrations fail. They also make refactors expensive because private call structure becomes part of the test contract.

Ruff coverage: no.

Guideline candidate: mock external systems and expensive side effects. Use real domain objects, fakes, fixtures, and assertions on returned values, persisted effects, emitted events, or user-visible errors for core logic.

Sources:

- pytest fixtures docs: https://docs.pytest.org/en/stable/explanation/fixtures.html
- pytest monkeypatch docs: https://docs.pytest.org/en/stable/how-to/monkeypatch.html
- 2026 study on over-mocking by coding agents: https://arxiv.org/abs/2602.00409

## Antipatterns Excluded From the Main List

These are real issues, but they are already well-covered by common or optional Ruff rules and should not drive new prose unless a guide needs local policy around them.

- `eval` itself: Ruff security rules can flag it. The guideline value above is the replacement workflow: structured inputs and explicit trusted-code boundaries.
- `print` in library code: Ruff can flag it. The guideline value above is exit-code and partial-failure policy for operational scripts.
- broad `except Exception`: Ruff can flag broad exception handlers. The guideline value is boundary-specific failure translation and observable failure reporting.
- HTTP calls without timeout: Ruff can catch some `requests` calls when security rules are enabled. The broader design issue is client lifecycle, retry, streaming, rate limiting, and cancellation policy.
- naive UTC datetimes: Ruff's `DTZ` rules can catch many cases when enabled. A datetime guideline may still define project policy for instants, local dates, trading calendars, and timezone-aware storage.
