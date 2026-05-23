# Design guides: detailed outlines

For each proposed file: scope, h2 outline, cpp source mapping, factors2
citations to anchor each section, and the research notes that back the
opinions. Section ordering inside each file is opinionated, not negotiable.

---

## `python-design-principles/architecture.md`

Scope: how Python code keeps domain logic separate from frameworks, vendors,
and runtime concerns. The cpp principle ports almost verbatim; the enforcement
shifts from header/CMake mechanics to package/import mechanics.

Disposition: PORT_AS_IS.

cpp source: `cpp-design-principles/architecture.md` (all sections).
Claude analysis: `cpp-claude/design/architecture.md`. Codex analysis:
`cpp-codex/design/architecture.md`.

### Section outline

1. **Theory of the domain.** Source code is the working theory; legibility is
   the survival mechanism. PORT_AS_IS from
   `cpp-claude/design/architecture.md`.
2. **Domain separation with adapters.** Components own their domains;
   cross-domain talk goes through dedicated adapter modules. Cite
   `factors-claude/patterns/module-layout.md`: the five sub-packages
   (`pyfactors`, `etl_factors`, `dagster_factors`, `dbt_factors`,
   `dashboards`) are exactly this pattern.
3. **Domain-owned types.** Each domain package ships its own `types.py`. Cite
   `factors-claude/patterns/typing-discipline.md` for the pandera + Pydantic +
   dataclass split per layer.
4. **Internal shape of an adapter module.** `protocols.py` + `vendors/`
   subpackage + `runtime.py`. Cite `factors-claude/patterns/data-pipeline.md`
   for the dlt + ibis + Dagster split.
5. **Forward-only dependencies.** Module-level imports form a DAG. Tools:
   `pydeps`, `import-linter`, ruff `TID252`. Cite
   `factors-claude/patterns/module-layout.md`'s observation that pydeps is
   declared but unused -- recommend wiring it into CI with a committed SVG.
6. **No implicit contracts.** Functions take what they need; return what they
   produce. Cite the cpp running example, render with `@dataclass` and pandas
   `pipe` chains. Reference
   `factors-claude/anti-patterns.md` entry H (defensive `.copy()` workaround
   for mutating indicators) as the failure mode.
7. **Functional core, imperative shell.** Inner layers stay sync and take
   plain values; `asyncio.run` / `anyio.run` is the shell entry point. Cite
   `factors-claude/patterns/data-pipeline.md` for Dagster assets as one-line
   delegations to library functions.
8. **Testability without mocks.** If a domain function needs
   `unittest.mock.patch` against a third-party library, the design is bent.
   FastAPI's `Depends` and dependency-injector belong in the shell.
9. **A note on the monorepo shape.** Cross-link to MONOREPO.md and to the
   `python-projects-and-tooling/project-layout.md` guide.

Backing research: `codex-research/architecture-and-patterns.md`,
`codex-research/generalizable-cpp-design.md`.

---

## `python-design-principles/types-and-correctness.md`

Scope: the Python answer to "compile-time correctness". Types name contracts
and catch mistakes at type-check time; they do not enforce at runtime.
Pydantic and pandera are the runtime-enforcement layers; pyright/pyrefly are
the static layer. The gradual-typing escape valves get their own section.

Disposition: PYTHON_SPECIFIC.

cpp source: `cpp-design-principles/compile-time-correctness.md`.
Claude analysis: `cpp-claude/design/compile-time-correctness.md`. Codex
analysis: `cpp-codex/design/compile-time-correctness.md`.

### Section outline

1. **What "correctness at the type layer" means in Python.** No compile step,
   so the goal shifts to "pyright/pyrefly strict catches the class of bug at
   CI time". Cite `cpp-claude/research/typecheckers-2026.md` for the choice
   of pyright as the default.
2. **Strong typing tier 1: NewType for nominal identity.**
   `AccountId = NewType("AccountId", str)`. Zero runtime cost. Cite
   `factors-claude/patterns/typing-discipline.md` (factors2 has zero
   `NewType` uses; the guideline argues for using it when the primitive
   alone loses meaning).
3. **Strong typing tier 2: frozen dataclass with a `parse` factory.** Use
   when the type carries validation. Cite the `Symbol.parse` recipe from
   `cpp-claude/design/compile-time-correctness.md`.
4. **Strong typing tier 3: pydantic at the boundary.** Use when the value
   crosses HTTP / config / JSON / DB. Cite
   `factors-claude/patterns/pydantic-pandera-dataclasses.md` for the
   four-boundary mapping (config, cross-process records, HTTP, dataframes).
5. **Parse, don't validate.** Single-site parsing; downstream code trusts
   the refined type. Cite
   `factors-claude/patterns/typing-discipline.md` for `Field(gt=0)` style
   precondition encoding.
6. **Discriminated unions of pydantic models.** The factors2 standard
   pattern: each variant pins
   `function_name: Literal["x"] = Field(default="x", frozen=True)`, the
   parent uses `Annotated[Union, Field(discriminator="function_name")]`.
   Cite `factors-claude/patterns/typing-discipline.md`. Cite
   `factors-claude/anti-patterns.md` entry F as the failure mode (40-arm
   union without discriminator -> `hasattr` + `inspect.signature` +
   `# type: ignore` to recover the structure).
7. **Exhaustiveness with `match` + `assert_never`.** The canonical
   dispatcher shape on a closed Enum or Literal. Cite
   `cpp-claude/research/exhaustive-matching.md` and
   `factors-claude/patterns/typing-discipline.md`. Cite
   `factors-claude/anti-patterns.md` entry E (trailing
   `raise ValueError("Unknown ...")` on closed enum) as the failure mode.
8. **DataFrame contracts with pandera.** Schema enforces columns/dtypes;
   hand-written cross-row validators sit behind it. Cite
   `factors-claude/patterns/pydantic-pandera-dataclasses.md` and
   `factors-claude/patterns/error-handling.md` (named single-purpose
   predicates).
9. **The typing escape hatches: when `Any`, `cast`, and `# type: ignore` are
   correct.** Three rules:
   - `# type: ignore` is acceptable over upstream library calls whose stubs
     are weak (ibis, pandera fluent calls). Banned over your own model.
   - `cast(...)` is acceptable at SDK boundaries to recover dynamism you
     know is correct. Banned in tests; write a constructor instead.
   - `Any` is acceptable at hostile boundaries (JSON, untyped third-party).
     Normalise to a typed domain value within one frame.
   Cite `factors-claude/patterns/typing-discipline.md` for the escape-valve
   audit (26 `# type: ignore`, 9 `Any`, 3 `cast`, 0 `NewType`, 0 `Protocol`,
   0 `TYPE_CHECKING`); cite `codex-research/modern-python-tooling-and-typing.md`
   for the boundary heuristics.
10. **pyright strict configuration template.** Reference
    `cpp-claude/research/typecheckers-2026.md` and
    `cpp-claude/research/ruff-rule-sets.md`. Include a `[tool.pyright]` block
    with `typeCheckingMode = "strict"` and the per-directory override pattern
    for `tests/` (looser) and `scripts/` (looser still).
11. **Designated initialization.** `@dataclass(kw_only=True, frozen=True,
    slots=True)`. Banned positional construction by convention; ruff `COM812`
    for trailing commas.
12. **Free-threaded interpreter caveats.** Mention 3.13t / 3.14t briefly;
    point to `runtime-and-concurrency.md` for detail.

Backing research: `cpp-claude/research/strong-types-and-data-classes.md`,
`cpp-claude/research/pep695-and-generics.md`,
`codex-research/modern-python-tooling-and-typing.md`.

---

## `python-design-principles/comments-and-docstrings.md`

Scope: present-tense comments, docstrings on public API, the
negative-documentation anti-pattern verbatim from the user's global CLAUDE.md.

Disposition: PORT_AS_IS.

cpp source: `cpp-design-principles/comments.md`. Claude analysis:
`cpp-claude/design/comments.md`. Codex analysis: `cpp-codex/design/comments.md`.

### Section outline

1. **Guiding comments.** `# ` for in-body, `"""..."""` for module/class/function.
2. **Non-obvious lines.** Same examples, ported to Python idioms (dict pop
   with a tuple key, chained method call, generator lazy materialisation,
   PEP 695 type-alias gotcha).
3. **Blocks and flow.** Consecutive block comments as the function's table
   of contents.
4. **What not to comment.** Repeat the negative-documentation rule from
   the user's global CLAUDE.md verbatim. Cite the three tells (past-tense,
   delta, non-responsibility framing). Cite
   `factors-claude/anti-patterns.md` entry K (commented-out alternative
   branches) as a Python instance.
5. **Precondition phrasing.** `# precondition: ...` for documentation;
   `assert` only when the program should crash if the precondition fails
   (and only when `python -O` is not in play -- spell this out).
6. **Docstring style.** Recommend Google style for new code; configure
   ruff's `D` rules to match. Document Numpy style as the alternative for
   Sphinx autodoc users.
7. **What docstrings should not repeat.** Cite
   `factors-claude/patterns/documentation.md`: `Args:` blocks that
   re-list the signature are noise; only the parameters whose meaning is
   not in the type get a line.

Backing research: `cpp-claude/design/comments.md`,
`codex-research/generalizable-cpp-design.md`.

---

## `python-design-principles/declarative-style.md`

Scope: decomposition, named predicates, comprehensions, generators, itertools,
pandas pipe chains. The cpp content ports almost word-for-word with
substitutions in the algorithm names.

Disposition: PORT_AS_IS.

cpp source: `cpp-design-principles/declarative-style.md`. Claude analysis:
`cpp-claude/design/declarative-style.md`. Codex analysis:
`cpp-codex/design/declarative-style.md`.

### Section outline

1. **Decompose.** Cut a function until each piece does one nameable thing.
2. **Work on simpler types.** `min(points, key=attrgetter("y"))` over
   `min(points, key=lambda p: p.y)`.
3. **Stage variables upfront.** Compute derived data first; assemble after.
4. **Named predicates.** Module-level `def is_active_user(...)` over a
   lambda in a filter. Cite ruff `E731` (no named-lambda assignments).
5. **Use the built-ins.** `any`, `all`, `next(..., default=None)`,
   `min(..., key=...)`. Map the cpp `std::ranges::*` examples one-to-one.
6. **Compose lazily, materialise once.** Generators for the one-shot
   consumer; `list(...)` only when a second pass is required. The
   "iterating a `filter(...)` twice raises `StopIteration`" trap.
7. **Pandas pipe chains.** `df.pipe(untie).pipe(rank)` as the dataframe
   equivalent. Cite
   `factors-claude/patterns/data-pipeline.md` for the ibis-at-the-boundary
   pattern that keeps the chain readable.
8. **Cost model.** Generator vs list-comprehension table; recommend
   generator-expression spelling when the next step is a one-shot consumer.

Backing research: `cpp-claude/design/declarative-style.md`,
`codex-research/generalizable-cpp-design.md`,
`factors-claude/patterns/data-pipeline.md`.

---

## `python-design-principles/functional-programming.md`

Scope: pure functions, sum types as discriminated unions, higher-order
functions, the Python-specific late-binding closure trap.

Disposition: PORT_WITH_ADAPTATION.

cpp source: `cpp-design-principles/functional-programming.md`. Claude
analysis: `cpp-claude/design/functional-programming.md`. Codex analysis:
`cpp-codex/design/functional-programming.md`.

### Section outline

1. **Pure functions and value semantics.** Inputs in, value out, no globals.
   Cite `factors-claude/patterns/data-pipeline.md` indicator-purity rule
   (indicators read input columns, return only the new column; the
   orchestrator joins).
2. **Pattern matching on sum types.** `match` over `type OrderEvent =
   Placed | Canceled | Filled`, with `case _ as never: assert_never(never)`.
   Cite `cpp-claude/research/exhaustive-matching.md`.
3. **Higher-order functions.** `functools.partial`, `operator.attrgetter`,
   `operator.itemgetter`, closure over a captured value.
4. **Storing callables.** `Callable[..., None]` for parameters; `Protocol`
   with `__call__` when the callable carries extra structure;
   `weakref.proxy` to a bound method when the callback should be cleared on
   owner GC.
5. **Type-checker-driven dispatch.** Protocol + `isinstance(x, MyProtocol)`
   with `@runtime_checkable`; PEP 742 `TypeIs` for narrowing. Replaces the
   cpp `if constexpr (requires { ... })` section.
6. **Closures: late binding and lifetime.** `funcs = [lambda i=i: i for i
   in range(3)]` -- the default-argument-binding workaround. Strong
   references in closures can prolong lifetimes; `weakref` if a closure is
   stored long-term.
7. **What does not port.** Drop the C++ type-erasure cost discussion; drop
   parameter packs (use `*args` / `**kwargs` with `TypeVarTuple` only when
   typing pays off).

Backing research: `cpp-claude/research/exhaustive-matching.md`,
`factors-claude/patterns/typing-discipline.md`.

---

## `python-design-principles/generics-and-protocols.md`

Scope: PEP 695 generics, Protocol vs ABC, TypeIs / TypeGuard, assert_never,
ParamSpec. This is the rewrite of `templates.md`.

Disposition: PYTHON_SPECIFIC.

cpp source: `cpp-design-principles/templates.md`. Claude analysis:
`cpp-claude/design/templates.md`. Codex analysis:
`cpp-codex/design/templates.md`.

### Section outline

1. **PEP 695 syntax.** `def first[T](items: Sequence[T]) -> T:`. `type X =
   ...` aliases. Cite `cpp-claude/research/pep695-and-generics.md`.
2. **Protocol for "supports these operations".** Structural typing; the
   Python answer to C++ concepts. Cite the `Serializable` example from
   `cpp-claude/design/templates.md`.
3. **Protocol vs ABC vs duck typing.** Protocol when the consumer side
   needs to type-check the contract; ABC when you also want runtime
   `isinstance` checks plus a default implementation slot; duck typing
   when neither pays off.
4. **Bounded type parameters.** `def foo[T: Serializable](x: T) -> T:` --
   constrain only when the body assumes a shape.
5. **`TypeIs` user-defined narrowing.** PEP 742. The cleanest spelling
   when narrowing a small stable union.
6. **`assert_never` for exhaustiveness.** Combined with `Literal`/`Enum`
   discriminators. Cite `factors-claude/patterns/typing-discipline.md`.
7. **`ParamSpec` and `Concatenate` for decorator typing.** Preserves the
   wrapped function's signature. Cross-link to
   `decorators-and-metaprogramming.md`.
8. **`Self` for fluent APIs and alternate constructors.** Use over the
   forward-string return-type spelling.
9. **Variance.** PEP 695 infers by default; `Protocol` parameters are
   invariant unless declared otherwise. Mention briefly; deep variance
   discussion belongs in a research note, not the guide.
10. **TypedDict at dict-shaped boundaries.** Wire-format JSON, Django
    request bodies, untyped config files. Prefer pydantic models when the
    data lives in memory for more than a frame.
11. **What does not port.** Drop the dependent-false / `static_assert`
    discussion; the equivalent is `assert_never`. Drop the runtime
    "is_specialisation_of" pattern; Python generics do not carry that info.

Backing research: `cpp-claude/research/pep695-and-generics.md`,
`cpp-claude/research/strong-types-and-data-classes.md`.

---

## `python-design-principles/decorators-and-metaprogramming.md`

Scope: when to write a decorator, when to reach for `__init_subclass__`,
when metaclasses earn their place. Includes the ParamSpec-typed decorator
template.

Disposition: PYTHON_SPECIFIC.

cpp source: `cpp-design-principles/preprocessor-macros.md`. Claude
analysis: `cpp-claude/design/preprocessor-macros.md`. Codex analysis:
`cpp-codex/design/preprocessor-macros.md`.

### Section outline

1. **Decorators are the default for "wrap this function".** Logging,
   caching, validation, retry, transaction, registration. Cite
   `functools.cache`, `functools.cached_property`,
   `contextlib.contextmanager`, `dataclass`, `pydantic.validate_call`.
2. **Decorator design rules.** `functools.wraps` on every custom decorator.
   Type with `ParamSpec` so the wrapped function's signature survives.
3. **A decorator template.** The full `def timed(operation: str): ...`
   recipe from `cpp-codex/design/preprocessor-macros.md`, with the
   `ParamSpec` typing made explicit.
4. **`__init_subclass__` for "every subclass of Base must do X".** Runs
   once per subclass declaration; no metaclass required.
5. **Descriptors for reusable attribute behaviour.** Rare. The right tool
   when you need the same get/set logic across many class fields.
6. **Metaclasses are the last resort.** They compose poorly. Use only when
   you need to intercept attribute access at class-creation time.
7. **Code generation.** Generated code is reviewed, deterministic, and
   easier to maintain than runtime reflection. Otherwise prefer reflection.
8. **Domain-shaped decorators live with the domain.** A decorator that
   registers an error category lives in the domain's module, not in a
   project-wide `utilities.py`. Same rule as the cpp file.

Backing research: `codex-research/modern-python-tooling-and-typing.md`.

---

## `python-design-principles/error-handling.md`

Scope: exception-first inside a domain, boundary translation, top-level
catch-all. Optional Result-library subsection for code that wants
type-checker-visible failure branches.

Disposition: PYTHON_SPECIFIC.

cpp source: `cpp-design-principles/error-handling.md`. Claude analysis:
`cpp-claude/design/error-handling.md`. Codex analysis:
`cpp-codex/design/error-handling.md`.

### Section outline

1. **Default: exceptions inside the domain.** Each domain owns a base
   exception (`class RoutingError(Exception):`) and concrete subclasses.
2. **Structured exception classes.** `@dataclass(frozen=True, slots=True)`
   subclass with a `__post_init__` that calls `Exception.__init__` so
   `str(exc)` stays meaningful while typed fields stay available. Cite
   `cpp-claude/design/error-handling.md` for the recipe.
3. **Domain `errors.py` module per package.** The Python analogue of
   `error_code.hpp` + `errors.hpp`. ErrorCode IntEnum only when external
   observers need a stable numeric code (Prometheus, gRPC status).
4. **Handlers match on the structured type.** Specific before broad;
   `except DuplicateOrder | DuplicateRequest as e:` over a chained
   `if`/`elif` on `type(e)`.
5. **Exceptions stop at the domain boundary.** A public function raises
   from its own hierarchy plus whitelisted stdlib types (`ValueError`,
   `KeyError`). It catches and translates sibling-domain exceptions at the
   seam. Cite `factors-claude/patterns/error-handling.md` `simulation.py`
   anti-pattern (two competing classes for the same systemic failure).
6. **Top-level functions catch everything.** FastAPI route handlers
   (`exception_handler`); asyncio tasks; Dagster `@asset`; Click/Typer
   commands. Recommend a `@catch_all_to_log` decorator for the cases where
   a runtime is one frame above the top-level function.
7. **`except Exception` is allowed once per call stack.** Cite
   `factors-claude/anti-patterns.md` entry B (eight `except Exception`
   handlers in one Streamlit file) as the failure mode.
8. **Never log-and-re-raise.** Pick one: re-raise or `logger.exception(...)`
   and stop. Cite `factors-claude/anti-patterns.md` entry M.
9. **Diagnostic context.** Python tracebacks are free; `raise NewError(...)
   from e` for chained context. Loguru's `@logger.catch` for the
   imperative-shell entry point.
10. **Two Python-specific traps.** Bare `except:` (catches
    `KeyboardInterrupt` and `SystemExit`); `raise` inside `finally` (masks
    the original).
11. **Optional: a Result library at the boundary.** For codebases that
    want type-checker-visible failure branches. `from result import Ok,
    Err`; small fixture/helper for tests. Recommend at the boundary form
    only, not inside a domain. Document but do not require.
12. **Validation messages are part of the contract.** Cite
    `factors-claude/patterns/error-handling.md` `_validate_step_result`
    pattern; tests pin the exact message via `re.escape(...)`.

Backing research: `cpp-claude/design/error-handling.md`,
`codex-research/generalizable-cpp-design.md`,
`factors-claude/patterns/error-handling.md`.

---

## `python-design-principles/invariants-and-rollback.md`

Scope: commit-at-end, ExitStack.callback for scope-guard rollback,
idempotency primitives, free-threaded-Python caveats on dict atomicity.

Disposition: PORT_WITH_ADAPTATION.

cpp source: `cpp-design-principles/invariants.md`. Claude analysis:
`cpp-claude/design/invariants.md`. Codex analysis: `cpp-codex/design/invariants.md`.

### Section outline

1. **Exception-safety taxonomy.** Same basic/strong/no-throw vocabulary;
   "no-throw" is a docstring convention in Python.
2. **Commit at the end.** Build new state in locals; mutate only after all
   fallible work succeeds. Cite the `SubscriptionSet.add` example.
3. **Scope-guard rollback with `ExitStack.callback`.** `stack.pop_all()`
   to dismiss; on exception, callbacks fire in reverse order. Cite
   `cpp-claude/design/invariants.md` for the `add_request` example.
4. **`dataclasses.replace` for build-new-copy-then-swap on frozen
   dataclasses.**
5. **Idempotency: pre-compute the rollback amount.** Store
   `rollback_quantity` on the request dataclass at admission. Cite
   `factors-claude/patterns/state-invariants.md` for the dlt
   write-disposition pattern (`merge` with primary key + dedup sort).
6. **Idempotency: check-then-act atomically.** `dict.setdefault` is atomic
   under the GIL; under free-threaded Python it is not. Document both.
7. **Invariants belong on the type that owns them.** Underscore-prefixed
   private attributes + ruff `reportPrivateUsage`; frozen dataclass +
   property + parse-once factory for genuinely-locked-down types.
8. **Validating snapshots after mutation.** Cite
   `factors-claude/patterns/state-invariants.md` for the
   `validate_step_result` pattern: simulation deep-copies the position
   book at each step, runs 30+ named invariants, raises before the next
   step compounds the damage.
9. **`with conn.transaction():` for multi-statement SQL writes.** Cite
   `factors-claude/patterns/state-invariants.md` `TRUNCATE + COPY` example.
10. **Recoverability classification.** Classify each step by what is lost
    on a miss; schedule only irrecoverable steps; everything else is
    "rerun on demand". Cite the
    `doc/adr/0001-schedule-only-economatica-downloads.md` ADR.
11. **Free-threaded sidebar.** 3.13t / 3.14t changes the atomicity
    assumptions; recommend explicit `threading.Lock` for shared mutable
    state or keep the owning-thread discipline.

Backing research: `cpp-claude/design/invariants.md`,
`factors-claude/patterns/state-invariants.md`.

---

## `python-design-principles/state-machines.md`

Scope: when an FSM is the right tool; hand-rolled `match` over discriminated
states for small machines; `transitions` library for large; PlantUML/Mermaid
embedding.

Disposition: PORT_WITH_ADAPTATION.

cpp source: `cpp-design-principles/state-machines.md`. Claude analysis:
`cpp-claude/design/state-machines.md`. Codex analysis:
`cpp-codex/design/state-machines.md`.

### Section outline

1. **When to reach for an FSM.** Same decision criteria as the cpp file:
   behaviour depends on a graph of legal moves. Skip for single-bit state,
   linear pipelines, pure data transforms.
2. **Library choice decision matrix.** `transitions` library, `python-statemachine`,
   hand-rolled `match`. Recommend hand-rolled for small (less than five
   states, simple actions); `transitions` for large or for "the diagram is
   the spec" cases.
3. **Hand-rolled recipe.** Discriminated union of frozen dataclass states;
   driver class with `state: SessionState`; `match` on `self.state` in
   each handler; explicit no-op for illegal moves.
4. **`transitions` library recipe.** Brief; for "five-plus states with
   entry/exit actions" cases.
5. **Embed the diagram.** PlantUML / Mermaid in the module docstring; review
   diagram and code as one artifact.
6. **Deferred events.** Async: `await event.wait()` on a per-state
   `asyncio.Event`. Sync: `collections.deque` drained after each transition.
7. **Testing.** Drive the FSM with crafted events; assert on the recorded
   action list; one test per legal transition; one test per illegal
   transition (explicit no-op assertion).
8. **Anti-patterns.** Calling `handle_*` from inside an action; mutating
   state from outside the handler; relying on `str` state names for
   dispatch (use `match` instead).

Backing research: `cpp-claude/design/state-machines.md`.

---

## `python-design-principles/pipelines.md`

Scope: stage components with inbound methods and outbound callback fields;
wiring at the entry point; defer re-entrance; interception at the wiring
site.

Disposition: PORT_AS_IS.

cpp source: `cpp-design-principles/pipelines.md`. Claude analysis:
`cpp-claude/design/pipelines.md`. Codex analysis: `cpp-codex/design/pipelines.md`.

### Section outline

1. **Pattern.** Stage has inbound methods (`send`, `submit`) and outbound
   `on_<event>: Callable[..., None] | None` fields. No stage knows who
   calls it or who consumes its output. Startup assertion catches
   "wiring forgot a callback".
2. **Source, sink, stage vocabulary.** Documentation only; not ABCs.
3. **Wiring near `main`.** Combinators at the wiring site: filtering, tee,
   threading bridge, telemetry. Sync wiring: direct assignment. Async
   wiring: `asyncio.create_task` from the wired lambda.
4. **Defer re-entrant callbacks.** Async: `asyncio.create_task` defers by
   itself. Sync: `collections.deque` + drain.
5. **External integrations.** Vendor wrapper implements vendor's listener
   interface; re-emits through domain-shaped `on_*` callbacks.
6. **Pandas pipeline equivalent.** `df.pipe(stage_a).pipe(stage_b)` --
   cross-link to `declarative-style.md`.
7. **Trade-offs.** Wiring grows quadratically; synchronous unwind is a
   property of direct-call wiring; unassigned callbacks are a runtime
   hazard.

Backing research: `cpp-claude/design/pipelines.md`,
`factors-claude/patterns/data-pipeline.md`.

---

## `python-design-principles/runtime-and-concurrency.md`

Scope: asyncio.TaskGroup vs anyio decision, cross-thread/cross-process
marshalling, free-threaded interpreter caveats.

Disposition: PORT_WITH_ADAPTATION.

cpp source: `cpp-design-principles/runtime.md`. Claude analysis:
`cpp-claude/design/runtime.md`. Codex analysis: `cpp-codex/design/runtime.md`.

### Section outline

1. **Single-threaded internals.** A component owns state on one execution
   context (event loop, worker thread, or process). Methods are not
   synchronised; fields are plain.
2. **Marshalling between threads.** `loop.call_soon_threadsafe(b.handle,
   event)` for thread -> loop. `asyncio.Queue` for in-loop;
   `queue.Queue` for thread-to-thread; `multiprocessing.Queue` for
   process-to-process.
3. **TaskGroup and anyio: pick one.** Decision matrix:
   - `asyncio.TaskGroup`: stdlib; fire-and-forget sibling tasks; cancel
     siblings on first failure; no cancel-scope handle.
   - `anyio`: cancel scopes are first-class; `fail_after` / `move_on_after`;
     extra dependency.
   Default to TaskGroup; reach for anyio when cancel scopes pay for the
   extra dependency. Cite
   `cpp-claude/research/structured-concurrency.md`.
4. **Cancellation discipline.** Re-raise `CancelledError`; use
   `move_on_after` / `fail_after` for timeouts, not `asyncio.wait_for`
   (which interacts badly with task groups).
5. **`ExceptionGroup` and `except*`.** At task group boundaries; catch a
   subset of group members and re-raise the rest.
6. **High-volume streams.** Python is rarely the right tool past tens of
   thousands of messages per second. Options: Cython, Rust extension,
   separate process, message broker. Cite
   `cpp-claude/design/runtime.md`.
7. **GIL is not a data-invariant guarantee.** Under the GIL, single Python
   bytecodes are atomic but compound invariants are not. Under
   free-threaded 3.13t / 3.14t, single bytecodes are not even atomic for
   compound types.
8. **Document the deviation, not the default.** Document threading on
   runtime wrappers, classes deliberately thread-safe, methods that break
   the surrounding class's contract, components on a non-default loop.
9. **Failure mode: undocumented runtime wrappers.** Misuse of
   `asyncio.get_event_loop()` outside `asyncio.run`. Cite
   `factors-claude/patterns/async-concurrency.md`.
10. **HTTP client recipes.** `httpx.AsyncClient` over `aiohttp` for new
    code; one client per scope; explicit `httpx.Timeout`; document
    `raise_for_status` in the docstring. Cite
    `factors-claude/patterns/async-concurrency.md`.
11. **Rate limiting.** `aiolimiter.AsyncLimiter` for request budgets;
    `asyncio.Semaphore` for in-flight concurrency. Cite
    `factors-claude/anti-patterns.md` entry S (do not write `time.sleep`
    loops when the right shape exists in the repo).
12. **Retries.** `tenacity` wrapped around the single-attempt function;
    the wrapper's `try/except` is the post-retry boundary.
13. **Blocking in async.** Short pandas/pyarrow work is fine inline;
    anything past a few milliseconds goes through `asyncio.to_thread(...)`.
14. **Heavy parallelism.** Ray actors for stateful initialisation +
    stateless per-job work. Cite
    `factors-claude/patterns/async-concurrency.md`.

Backing research: `cpp-claude/research/structured-concurrency.md`,
`factors-claude/patterns/async-concurrency.md`.

---

## `python-design-principles/cross-cutting-services.md`

Scope: clock, logger, timer, settings, tracing. contextvars-backed module
singletons; pytest fixtures for substitution.

Disposition: PYTHON_SPECIFIC.

cpp source: `cpp-design-principles/cross-cutting.md`. Claude analysis:
`cpp-claude/design/cross-cutting.md`. Codex analysis:
`cpp-codex/design/cross-cutting.md`.

### Section outline

1. **The problem.** Threading dependencies through every constructor
   distorts every signature; mocking via `unittest.mock.patch` at the
   module level depends on the consumer's import shape and is fragile.
2. **The pattern: contextvars-backed module singleton.** A `_clock:
   ContextVar[Clock]` plus `configure(impl)` and `now()` free functions.
   Composes with asyncio (per-task), threads (per-thread), and pytest
   fixtures (per-test).
3. **Test substitution.** `pytest.fixture` that sets a ContextVar token
   and resets on teardown. Cite `cpp-claude/design/cross-cutting.md`
   `mock_clock` example.
4. **Catalog: logger, timer, clock.** Three sections; one recipe each.
   Cross-link the logger section to `python-debugging-principles/logging.md`.
5. **Settings.** `pydantic-settings.BaseSettings` for env / CLI / file.
   Cite `factors-claude/anti-patterns.md` entry J (hand-rolled `argparse
   + getenv` is the failure mode).
6. **Tracing and metrics.** OpenTelemetry SDK for tracing; `prometheus_client`
   for metrics; both configured once at startup, exposed via free
   functions inside the domain.
7. **When constructor injection beats the contextvars singleton.** A
   short list: when the implementation needs to be different in two
   simultaneously-running contexts that share a process (e.g. two FastAPI
   apps in one Uvicorn process). Otherwise the singleton wins on
   readability.
8. **No import-time configuration.** Configure at the application entry
   point; never on module load. Cite `factors-claude/anti-patterns.md`
   entry N (setting global pandas mode in `conftest.py`).

Backing research: `cpp-claude/design/cross-cutting.md`,
`factors-claude/patterns/logging-observability.md`.

---

## `python-design-principles/performance.md`

Scope: Python's actual cost model. Clarity first, measure first, vectorise,
`__slots__`, profiler choice, when to leave Python.

Disposition: PYTHON_SPECIFIC.

cpp source: `cpp-design-principles/performance.md`. Claude analysis:
`cpp-claude/design/performance.md`. Codex analysis:
`cpp-codex/design/performance.md`.

### Section outline

1. **Default to clarity.** Performance follows from good design.
2. **Requirements before optimisation.** Define the budget.
3. **Measure before changing.** Tools: `pytest-benchmark` (correctness +
   regression); `pyinstrument` (everyday CPU); `py-spy` (live process,
   production-safe); `memray` (memory); `scalene` (line-level CPU+mem).
4. **Vectorise.** numpy / pandas / polars before any pure-Python loop
   optimisation. A pure-Python loop over 1M elements is one to two orders
   of magnitude slower.
5. **`__slots__`.** On classes instantiated millions of times. Cuts
   memory and improves attribute access.
6. **String interning.** `sys.intern(s)` for small repeated strings.
7. **Caching.** `functools.lru_cache` and `functools.cache`. Be explicit
   about the cache scope (process-global, per-instance, per-request).
8. **Type annotations do not speed CPython.** They help readers and tools.
   Avoid runtime annotation introspection on hot paths.
9. **Avoid needless materialisation.** Generators over lists; iterator
   chains over intermediate containers; bounded logging.
10. **When to leave Python.** Cython, Numba, Rust extensions, separate
    process. Cite the rate-budget threshold (tens of thousands per second).
11. **Database-side work and batching.** Push computation to the warehouse
    when the data already lives there; batch network round-trips; do not
    re-implement what dbt has already materialised. Cite
    `factors-claude/patterns/data-pipeline.md`.
12. **Free-threaded sidebar.** 3.13t / 3.14t trades single-threaded
    throughput for true multi-threaded throughput. Benchmark before
    adopting.
13. **Benchmark placement.** `benchmarks/` directory, pytest-discoverable,
    one Makefile target. Cite `factors-claude/patterns/testing.md`.
14. **Anti-pattern: timing via `print(timeit.default_timer())`.** Cite
    `factors-claude/anti-patterns.md` entry D. Use a `with timed_block:`
    context manager that emits one structured log line on exit.

Backing research: `cpp-claude/design/performance.md`,
`codex-research/architecture-and-patterns.md`,
`factors-claude/patterns/performance.md`.

---

## `python-design-principles/data-pipeline-and-dataframes.md`

Scope: dataframe-heavy codebases. Layer choice (dlt / ibis / dbt / pandas);
pandera at the boundary; indicator purity; no f-string SQL; lookup
classes vs `set_index` vs `merge_asof`.

Disposition: NEW. Skip if the first target codebase has no dataframes.

cpp source: none. Sources: `factors-claude/patterns/data-pipeline.md`,
`factors-claude/patterns/pydantic-pandera-dataclasses.md`,
`codex-research/architecture-and-patterns.md`.

### Section outline

1. **Choose the layer per stage.** Ingest in dlt; reshape in ibis at the
   ingest boundary; warehouse transforms in dbt; in-memory queries in
   duckdb; per-symbol rolling math in pandas; orchestration in Dagster.
2. **Parse at the boundary.** Pydantic for HTTP / config; pandera for
   DataFrames; dataclasses for "aggregate of already-validated parts".
3. **Pandera schema + hand-written cross-row validator.** Schema enforces
   columns/dtypes; the validator composes named single-purpose predicates,
   each of which raises with `context`, `rule`, `count`, `sample`.
4. **DataFrame contracts in function signatures.** `pa.typing.DataFrame[Scores]`
   carries intent across the call site even if pandera only enforces it
   at `.validate()` time.
5. **Indicator purity.** Read input columns; return only the new column +
   join keys; never mutate the input. The orchestrator joins. Cite
   `factors-claude/anti-patterns.md` entry H.
6. **Never f-string SQL.** Use the connector's composition API
   (`psycopg.sql`, ibis, `sqlalchemy.text` with bindparams). Cite
   `factors-claude/anti-patterns.md` entry A. Enforce via ruff `S608`.
7. **Lookup-class anti-pattern.** "Wrap a DataFrame, build a dict, expose
   `get_X` methods" -- generalise or replace with `set_index` or
   `merge_asof`. Cite `factors-claude/patterns/data-pipeline.md`.
8. **Dagster discipline.** Asset bodies are one-line delegations to a
   library function with no Dagster imports. The Dagster file owns
   wiring; the library file owns logic.
9. **Schedule durability concerns, not pipelines.** Classify each step by
   recoverability; schedule only the irrecoverable steps; document the
   rest as "rerun on demand". Cite the
   `doc/adr/0001-schedule-only-economatica-downloads.md` ADR template.
10. **dbt vs Python.** dbt for warehouse transforms; Python for everything
    else. Do not recompute in pandas what dbt has already materialised.

Backing research: `factors-claude/patterns/data-pipeline.md`,
`factors-claude/patterns/pydantic-pandera-dataclasses.md`,
`codex-research/architecture-and-patterns.md`.
