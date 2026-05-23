# Agent-context outlines: always-loaded sheet and on-demand examples

Two files mirror the cpp agent pair at `/home/msi/repos/cpp-guidelines/agent/`:

- `python-agent-context.md`: the always-loaded rule sheet. Same order as the
  cpp file; condensed to the vocabulary an agent needs to write code without
  loading the topical guides.
- `python-agent-examples.md`: the on-demand pair of good/bad examples per
  section.

Both Claude (`cpp-claude/agent-context/cpp-agent-context.md`) and Codex
(`cpp-codex/agent-context/cpp-agent-context.md`) recommend preserving the cpp
section ordering. We agree; mapping mental models between cpp and Python
agents is easier when both files share the topology.

---

## `python-agent-context.md` -- section-by-section spec

Length target: ~450 lines (the cpp file is ~400). Python adds two sections
(typing escape hatches, import discipline) that the cpp file does not need.

Section ordering and intent per section:

### 1. Opening: placeholder vocabulary

- The cpp file uses `lib::` for in-house utilities. The Python equivalent
  is the codebase's chosen package; common shapes:
  - `<project>` / `<project>.core` for in-house utilities.
  - `<project>.types` for domain types per the Domain Ownership section.
  - `<project>.errors` for per-domain exception hierarchies.
- Document the placeholder mapping at the top.
- Rule: search for existing local helpers before creating abstractions.
  Cite `factors-claude/anti-patterns.md` entry S (two implementations of
  rate limiting in the same repo, because the second author did not
  reuse the first).

### 2. Core Lens (PORT_AS_IS)

The five questions from the cpp file port verbatim:

- Which package owns this concept?
- Which type or parser carries the proof this value is valid?
- Which boundary translates to another domain or external API?
- Which effect can move outward?
- Which invariant must remain true if a middle step fails?

Add one Python-specific question (Codex's recommendation, we agree):

- Which dynamic escape hatch is justified here? Cite the typing escape
  hatches section.

### 3. Layout (PORT_WITH_ADAPTATION)

- `src/<package>/<domain>/`, `tests/<domain>/`.
- Functional domain modules stay importable without runtime clients.
- Runtime wiring lives in application factories, CLI entry points, ASGI
  startup, worker entry points, or adapter packages.
- Top-level packages name the layer they own. No `common/`, `shared/`,
  `utils/` at the top level. Cite
  `factors-claude/patterns/module-layout.md`.
- Subpackages are one concept each. No `helpers/` or `utils/` folders
  inside a subpackage.
- `__init__.py` is the public API: re-export the variants; assemble
  discriminated-union type aliases; set `__all__` as a static literal
  list (not a `get_args(...)` expression -- cite
  `factors-claude/anti-patterns.md` entry R).
- Tests mirror `src/` one-for-one.

### 4. Tests (PORT_WITH_ADAPTATION)

- Behaviour-first.
- Plain `assert`.
- `pytest.mark.parametrize` with `pytest.param(id=...)` rows.
- Fixtures for setup and teardown; default scope `function`.
- `monkeypatch`, `tmp_path`, `caplog` are the built-in seams.
- `pytest.raises(SomeError, match=re.escape("..."))` for exception paths.
- `pytest.approx(0.1)` for floating-point.
- One file per source file; one logical fact per test.
- Validation messages are part of the contract; pin via `re.escape(...)`.
- No `cast(...)` in tests; write a constructor instead.
- No `except Exception` in tests.
- Type-level expectations checked by pyright/pyrefly, not by runtime
  pytest alone.

### 5. Debugging (PORT_AS_IS plus Python additions)

- Read full tracebacks including chained exceptions (`__cause__`,
  `__context__`).
- Use `faulthandler` for hangs.
- Name async tasks; log request ids.
- Treat cancellation as a control path, not a generic failure.
- `breakpoint()` for interactive debug.
- `py-spy dump` for live processes.

### 6. Domain Ownership (PORT_AS_IS)

- Each domain package ships `types.py` (and possibly `errors.py`).
- Same-named types in two domains stay distinct (separate `NewType` or
  separate frozen-dataclass classes).
- Do not pass raw dicts deep into the domain when a dataclass or parsed
  type is available.
- Cross-domain construction is an adapter responsibility; named
  explicitly.

### 7. Forward Dependencies (PORT_AS_IS plus tooling)

- Module-level imports form a DAG.
- Enforce via `import-linter` or `pydeps`; ruff `TID252` forbids
  parent-relative imports.
- When adding a dependency, check the import graph; prefer extracting a
  lower-level module over circular imports.

### 8. Types Carry Proof (PORT_WITH_ADAPTATION)

The exact wording from Codex's recommendation, refined:

> Use type hints to make public contracts, domain ownership, and refactors
> safer. Parse untyped input at boundaries into domain values. Use NewType,
> Literal, Enum, frozen dataclasses, Annotated, Protocol, and assert_never
> where they make the contract clearer. pyright strict (or pyrefly) is the
> CI gate. Escape typing at I/O boundaries, untyped third-party libraries,
> dynamic dispatch that resists a clean Protocol, and prototypes. Do not
> over-type local Python until the type layer becomes harder to read than
> the code it protects.

### 9. Strong-Type Ergonomics (PORT_WITH_ADAPTATION)

- `NewType` for nominal IDs with no runtime behaviour.
- Frozen dataclass with `parse` factory for values with validation or
  behaviour.
- Pydantic at trust boundaries (HTTP, config, JSON, DB).
- Do not unwrap to primitives except at serialization/database/wire/UI
  boundaries.
- Avoid wrapper classes that only create ceremony.

### 10. Typing Escape Hatches (NEW)

- `# type: ignore[<rule>]` is acceptable over upstream library calls with
  weak stubs (ibis fluent calls, pandera, dagster Resolver). Always
  trailing comment naming the reason. Banned over your own model.
- `cast(...)` is acceptable at SDK boundaries to recover dynamism you know
  is correct. Banned in tests; write a constructor instead.
- `Any` is acceptable at hostile boundaries (JSON, untyped third-party).
  Normalise to a typed domain value within one frame.
- `TYPE_CHECKING` is acceptable to break import cycles for type-only
  imports; mark the imported name with a quoted annotation. Reach for it
  when an actual cycle appears, not by reflex.

Cite `factors-claude/patterns/typing-discipline.md` for the escape-valve
audit numbers (26 `# type: ignore`, 9 `Any`, 3 `cast`, 0 `NewType`, 0
`Protocol`, 0 `TYPE_CHECKING`) as evidence that under-typing and
over-typing are both real failure modes.

### 11. Designated Initializers (PORT_WITH_ADAPTATION)

- `@dataclass(kw_only=True, frozen=True, slots=True)` for in-domain values.
- Keyword construction at every call site for dataclasses with adjacent
  same-shaped fields.
- ruff `COM812` for trailing commas.

### 12. Functional Core, Imperative Shell (PORT_AS_IS)

- Inner layers stay sync and take plain values.
- `asyncio.run` / `anyio.run` is the shell entry point.
- Python-specific edge effects: import-time I/O, global clients,
  framework DI, task creation, environment reads, ORM sessions, logging.

### 13. Pipelines (PORT_AS_IS)

- `on_<event>: Callable[..., None] | None` fields on stages.
- Wiring at the entry point.
- An async callback must be awaited or scheduled at the wiring point.
- For dataframe pipelines, `df.pipe(stage_a).pipe(stage_b)` is the
  equivalent shape.

### 14. Runtime And Threads (PORT_WITH_ADAPTATION)

- `asyncio.TaskGroup` for sibling task ownership; anyio for cancel
  scopes.
- Cancellation discipline: re-raise `CancelledError`.
- `asyncio.Queue` and `queue.Queue`.
- `loop.call_soon_threadsafe(...)` for thread -> loop.
- `loop.run_in_executor(...)` work must not touch loop-owned state.
- The GIL does not make compound invariants safe.
- Free-threaded sidebar: 3.13t / 3.14t changes the atomicity assumptions.

### 15. Error Handling (PYTHON_SPECIFIC)

> Inside a trust boundary, use structured domain exceptions. At public
> boundaries, catch and translate to the caller vocabulary: HTTP response,
> CLI exit code, task log-and-swallow, retry, dead-letter event, or typed
> result object. Domain exceptions carry attributes, not only message
> strings. Use `raise NewError(...) from e` when translating lower-level
> exceptions. Never log-and-re-raise. `except Exception` is allowed at
> exactly one place per call stack -- the boundary that maps internal
> failures to the next protocol.

### 16. Invariants And Rollback (PORT_WITH_ADAPTATION)

- `contextlib.ExitStack.callback(undo_fn)` is the equivalent of
  `lib::scope_exit`.
- `stack.pop_all()` to dismiss on success.
- Build new state in locals; mutate the object once everything fallible
  has succeeded.
- `dataclasses.replace` for build-new-copy-then-swap.
- For shared mutable state, validate the snapshot before persisting it
  (cite the factors2 `validate_step_result` pattern).
- `with conn.transaction():` for multi-statement SQL writes.

### 17. Declarative Style (PORT_AS_IS)

- Generators / comprehensions over hand-rolled loops.
- `any`, `all`, `next(..., default=None)`, `min(..., key=...)`.
- `itertools` for chaining, slicing, fan-out.
- Pandas `df.pipe(...)` for dataframe pipelines.
- ruff `E731` flags named lambdas; use `def`.

### 18. Variants, Protocols, And Generics (PYTHON_SPECIFIC)

- Discriminated union of pydantic models for plugin-shaped subsystems.
  Each variant pins `Literal["x"] = Field(default="x", frozen=True)`;
  parent uses `Annotated[Union, Field(discriminator="...")]`.
- `match` + `assert_never` over `Literal` or `Enum` for closed
  exhaustive dispatch.
- `Protocol` for structural typing; `@runtime_checkable` when also using
  `isinstance`.
- PEP 695 syntax (`class Box[T]:`, `def f[T](...)`, `type X = ...`) in
  new code.
- `TypeIs` (PEP 742) for user-defined narrowing.
- `ParamSpec` / `Concatenate` for decorator typing.
- `Self` for fluent APIs and alternate constructors.

### 19. State Machines (PORT_AS_IS for criteria; PORT_WITH_ADAPTATION for mechanics)

- Reach for an FSM when behaviour depends on a graph of legal moves.
- Hand-rolled `match` over discriminated-union states for small machines.
- `transitions` library for large machines.
- Embed PlantUML / Mermaid in the module docstring.

### 20. Cross-Cutting Services (PYTHON_SPECIFIC)

- `contextvars`-backed module singletons for logger, clock, timer,
  settings, tracing.
- `pytest.fixture` with `set` / `reset` token for substitution.
- Configure at the application entry point; never at module load.

### 21. Performance Discipline (PORT_WITH_ADAPTATION)

- Default to clarity; measure before changing.
- Vectorise with numpy / pandas / polars before any pure-Python loop
  optimisation.
- `__slots__` on classes instantiated millions of times.
- `functools.cache` / `functools.cached_property` for memoisation.
- Type annotations do not speed CPython execution.
- Leave Python (Cython / Rust / separate process) when the rate budget
  cannot be met.

### 22. Comments And Docstrings (PORT_AS_IS)

- `# ` for in-body; `"""..."""` for module / class / function.
- Negative-documentation rule from the user's global CLAUDE.md applies
  verbatim.
- Docstring describes intent and non-obvious behaviour; do not repeat the
  type.
- Google-style docstrings as the default; configure ruff `D` rules to
  match.

### 23. Imports and Namespace Aliases (PYTHON_SPECIFIC)

- Absolute imports only; ruff `TID252` forbids parent-relative imports.
- `import x as y` at module top only.
- `np`, `pd`, `pa`, `pl` are accepted conventions.
- No `from module import *`.
- Project-internal aliasing (`import myproject.foo as foo`) is allowed
  when local naming improves readability; otherwise use the full
  qualifier.

### 24. Final and Read-Only (PYTHON_SPECIFIC)

- `Final[T]` for module-level read-only constants.
- `@dataclass(frozen=True, slots=True)` for in-domain value objects.
- `MappingProxyType` for read-only mapping views.
- No equivalent of `const` on a function parameter; convention is
  "callee does not mutate its arguments".

---

## `python-agent-examples.md` -- section-by-section spec

Same section ordering as the context file. For each section, one good
example and one bad example, rendered in the smallest Python that proves
the rule. The cpp version uses 600+ lines; aim for similar density.

For the sections most likely to be cited from PR comments, prefer real
factors2-derived examples (with the citation in a trailing comment) so
the reader can find the surrounding context:

- Error Handling -> the `except Exception: st.toast(str(e))` bad example
  from `factors-claude/anti-patterns.md` entry B; the
  `_validate_step_result` named-rule good example from
  `factors-claude/patterns/error-handling.md`.
- Variants, Protocols, And Generics -> the unstructured `Function = A | B
  | ...` bad example with `hasattr` + `inspect.signature` workaround from
  entry F; the `Annotated[Strategy, Field(discriminator="function_name")]`
  good example from
  `factors-claude/patterns/typing-discipline.md`.
- Cross-Cutting Services -> the
  `services.clock = clock; ... services.clock = previous` good fixture
  from `cpp-codex/design/cross-cutting.md`.
- Logging -> the `logger.info(f"downloaded {screen} to {path}")` bad
  example from entry O; the `logger.info("downloaded file",
  screen=screen, path=path)` good example from
  `factors-claude/patterns/logging-observability.md`.
- Data pipeline (if the file ends up needing one) -> the f-string SQL bad
  example from entry A; the `psycopg.sql.SQL(...).format(...)` good
  example from
  `factors-claude/patterns/data-pipeline.md`.

For sections without a strong factors2 anchor (Declarative Style,
Functional Core, Pipelines), use minimal synthetic examples sized to make
the rule unmistakable.

---

## Skill (deferred)

The cpp side has `agent/skills/repo-specific-cpp-guidelines/`. The Python
equivalent (`agent/skills/repo-specific-python-guidelines/`) is out of
scope for the first writing pass; document the intent and defer to a
follow-up. The skill's job is the same: a single skill file that loads the
agent-context plus the per-topic guides on demand.
