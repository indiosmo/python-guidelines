# C++ to Python guidelines exploration

The C++ guides mostly transfer as design pressure, not as mechanics. The
highest-value Python guide should keep the C++ repository's architecture,
declarative style, invariant, testing, and debugging disciplines while
rewriting type-level, error-handling, concurrency, template, macro, and tool
chapters into Python idiom.

The Python edition should target Python 3.13 or 3.14, use pyright or pyrefly
for static checking, Ruff for linting and formatting, uv for project and tool
execution, and pytest for tests. It should not teach "C++ with Python syntax".
The guide needs an explicit escape hatch from typing: use annotations to make
interfaces, domain ownership, and refactors safer, but avoid modeling dynamic
dispatch or untyped I/O so hard that the type layer becomes the design.

## Disposition by guide

| C++ guide | Disposition | Python direction |
|---|---|---|
| `architecture.md` | PORTS_AS_IS | Keep domain ownership, adapters, DAG dependencies, and functional core / imperative shell. Render modules, packages, and call graphs in Python terms. |
| `declarative-style.md` | PORTS_AS_IS | Keep decomposition, simpler inputs, staged variables, named predicates, and lazy composition. Use comprehensions, generator expressions, `itertools`, and small functions. |
| `invariants.md` | PORTS_AS_IS | Keep commit-at-end, rollback, idempotency, and owner-enforced invariants. Use context managers and copied dataclasses. |
| `error-handling.md` | PYTHON_SPECIFIC_VARIANT | Rewrite around exceptions internally and explicit translation at trust boundaries. |
| `functional-programming.md` | PORTS_WITH_ADAPTATION | Keep pure functions and higher-order style. Adapt sum types to dataclasses, `Literal` discriminators, `match`, and `assert_never`. |
| `pipelines.md` | PORTS_AS_IS | Keep typed stages, explicit wiring, edge interception, and re-entrance deferral. Use callbacks, queues, async tasks, and protocols. |
| `state-machines.md` | PORTS_AS_IS | Keep the FSM decision criteria, transition-table discipline, diagrams, and edge tests. Pick Python libraries or small tables by complexity. |
| `performance.md` | PORTS_WITH_ADAPTATION | Keep clarity-first and measure-first. Rewrite hot-path advice for CPython, allocation, vectorization, profiling, and native extensions. |
| `compile-time-correctness.md` | PORTS_WITH_ADAPTATION | Recast as "types and static analysis carry useful proof". Use `NewType`, frozen dataclasses, `Annotated`, `Literal`, `Protocol`, and parser functions. |
| `templates.md` | PORTS_WITH_ADAPTATION | Rewrite as generics and structural protocols: PEP 695, `TypeVar`, `ParamSpec`, variance, bounds, and overloads. |
| `preprocessor-macros.md` | PYTHON_SPECIFIC_VARIANT | Replace macro guidance with decorators, context managers, descriptors, metaclasses, and code generation used sparingly. |
| `runtime.md` | PORTS_WITH_ADAPTATION | Keep threading as an edge effect. Add `asyncio.TaskGroup`, queue ownership, cancellation, and GIL-aware thread/process guidance. |
| `comments.md` | PORTS_AS_IS | Same present-tense, intent-focused comment guidance. Adjust examples for Python docstrings and inline comments. |
| `cross-cutting.md` | PORTS_WITH_ADAPTATION | Keep edge-selected services, but prefer explicit composition, `ContextVar`, fixtures, and small provider modules over variant-backed globals. |
| `philosophy.md` | PORTS_AS_IS | Same independent-oracle testing principle. |
| `approval-tests.md` | PORTS_AS_IS | Same approval-test contract. Use Python snapshot/approval tooling and deterministic renderers. |
| `catch2-conventions.md` | PYTHON_SPECIFIC_VARIANT | Rewrite as pytest conventions, markers, fixtures, assertion style, and `uv run pytest`. |
| `condition-based-waiting.md` | PORTS_AS_IS | Same predicate wait principle. Render with polling helpers, `threading.Condition`, and async waits. |
| `error-path-testing.md` | PORTS_WITH_ADAPTATION | Rewrite result-return checks toward exceptions plus boundary translation tests. |
| `test-helpers.md` | PORTS_WITH_ADAPTATION | Keep factories, fixtures, probes, and provider guards. Use dataclass replace, pytest fixtures, monkeypatch, and context managers. |
| `test-patterns.md` | PORTS_WITH_ADAPTATION | Map `TEST_CASE`, `SECTION`, generators, and typed tests to pytest functions, parametrization, fixtures, and optional property tests. |
| `qt-gui.md` | PYTHON_SPECIFIC_VARIANT | Rewrite as Python GUI testing, likely PySide/PyQt plus pytest-qt, and include Playwright for web UIs if the repo needs it. |
| `root-cause-tracing.md` | PORTS_AS_IS | Same backwards tracing discipline. Use Python stack traces, `pdb`, logging, and `faulthandler`. |
| `defense-in-depth.md` | PORTS_AS_IS | Same layered validation, with runtime parsing through dataclasses, Pydantic/msgspec/attrs when appropriate. |
| `logging.md` | PORTS_WITH_ADAPTATION | Use stdlib logging or structlog/loguru by policy, structured context, lazy formatting, and throttling. |
| `sanitizers.md` | PYTHON_SPECIFIC_VARIANT | Rewrite as static/dynamic tooling: pyright/pyrefly, Ruff, asyncio debug mode, faulthandler, py-spy, memray, tracemalloc, coverage, and fuzzing. |
| `agent/cpp-agent-context.md` | PORTS_WITH_ADAPTATION | Preserve the rules, rewrite C++ mechanics into Python agent defaults. |
| `agent/cpp-agent-examples.md` | PORTS_WITH_ADAPTATION | Re-render examples in Python with pytest, dataclasses, protocols, exceptions, and async. |

Top-level disposition counts for these 28 requested guide and agent files:

- PORTS_AS_IS: 10
- PORTS_WITH_ADAPTATION: 13
- PYTHON_SPECIFIC_VARIANT: 5
- DROP: 0

## Recommended Python guide structure

- `python-design-principles/architecture.md`
- `python-design-principles/types-and-static-analysis.md`
- `python-design-principles/declarative-style.md`
- `python-design-principles/functional-core.md`
- `python-design-principles/error-handling.md`
- `python-design-principles/invariants.md`
- `python-design-principles/pipelines.md`
- `python-design-principles/state-machines.md`
- `python-design-principles/runtime-and-async.md`
- `python-design-principles/cross-cutting-services.md`
- `python-design-principles/generics-and-protocols.md`
- `python-design-principles/metaprogramming.md`
- `python-design-principles/performance.md`
- `python-design-principles/comments-and-docstrings.md`
- `python-testing-principles/philosophy.md`
- `python-testing-principles/pytest-conventions.md`
- `python-testing-principles/test-patterns.md`
- `python-testing-principles/test-helpers.md`
- `python-testing-principles/error-path-testing.md`
- `python-testing-principles/condition-based-waiting.md`
- `python-testing-principles/approval-tests.md`
- `python-testing-principles/gui-testing.md`
- `python-debugging-principles/root-cause-tracing.md`
- `python-debugging-principles/defense-in-depth.md`
- `python-debugging-principles/logging.md`
- `python-debugging-principles/tooling.md`
- `agent/python-agent-context.md`
- `agent/python-agent-examples.md`

## Highest-impact recommendations

1. Preserve the C++ repository's dependency-DAG and domain-ownership rules.
   Python import cycles and ambient module state are just as damaging as C++
   library cycles.
2. Rewrite the type chapter with restraint. The Python guide should teach
   where static types help and where dynamic Python should remain dynamic.
3. Rewrite error handling as exception-first inside a trust boundary, with
   explicit translation at boundaries such as HTTP, CLI, task runners, and
   queue consumers.
4. Make runtime guidance async-aware: `TaskGroup`, cancellation, queues, and
   thread/process boundaries belong in the first-class design vocabulary.
5. Make pytest conventions a full sibling to the testing philosophy. The C++
   test intent ports directly, but the ergonomics live in fixtures,
   parametrization, monkeypatching, and plain `assert`.
