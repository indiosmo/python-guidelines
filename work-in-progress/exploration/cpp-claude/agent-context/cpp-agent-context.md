# Analysis: agent/cpp-agent-context.md -> Python equivalent

The C++ agent-context file is the **always-loaded rule sheet**: the
condensed vocabulary an agent uses to write code without going
through the full topical guides. The Python equivalent should be the
same shape: condensed, opinionated, vocabulary-loaded.

Disposition by section:

| C++ section                          | Python disposition  | Notes                                                                                                 |
|--------------------------------------|----------------------|-------------------------------------------------------------------------------------------------------|
| Core Lens                            | PORTS_AS_IS         | "Code is the working theory of the domain" travels verbatim.                                          |
| Layout                               | PORTS_WITH_ADAPTATION | Replace `src/<comp>/<comp>/` with `src/<package>/`; runtime wrappers under `<package>/runtime/`.        |
| Tests                                | PORTS_AS_IS         | Same rules. Catch2 -> pytest mechanics.                                                               |
| Debugging                            | PORTS_AS_IS         | Same rules. Tools change (pdb, py-spy, memray).                                                       |
| Domain Ownership                     | PORTS_AS_IS         | `types.hpp` -> `types.py`; nested-namespace caveat dropped (Python's module qualifier is enough).      |
| Forward Dependencies                 | PORTS_AS_IS         | DAG rule travels; enforcement via `import-linter` / `pydeps`.                                          |
| Types Carry Proof                    | PORTS_WITH_ADAPTATION | `NewType` + frozen dataclass / pydantic factory; pyright/pyrefly --strict; PEP 695 syntax.            |
| Strong-Type Ergonomics                | PYTHON_SPECIFIC_VARIANT | No `.get()` / `operator->` distinction. Two flavours: `NewType` (identity) and dataclass wrapper (carries fields).      |
| Designated Initializers              | PORTS_AS_IS         | `@dataclass(kw_only=True)` + ruff `COM812` + omit defaulted args.                                     |
| Functional Core, Imperative Shell    | PORTS_AS_IS         | Same rule; `asyncio.run` / `anyio.run` is the shell entry point.                                       |
| Pipelines                            | PORTS_AS_IS         | `on_<event>: Callable[..., None] \| None`; wiring at the entry point.                                  |
| Runtime And Threads                  | PORTS_WITH_ADAPTATION | `loop.call_soon_threadsafe`; anyio `task_group.start_soon`; free-threaded sidebar.                    |
| Error Handling                       | PYTHON_SPECIFIC_VARIANT | Exceptions in-domain; `except` clauses replace LEAF handlers; top-level catch-all decorator.        |
| Invariants And Rollback              | PORTS_WITH_ADAPTATION | `contextlib.ExitStack.callback`; commit-at-end via `dataclasses.replace`.                              |
| Declarative Style                    | PORTS_AS_IS         | Same rule; generators / itertools / pandas pipe.                                                       |
| Variants, Concepts, And Templates    | PYTHON_SPECIFIC_VARIANT | `match` + `assert_never`; `Protocol`; PEP 695 generics; `TypeIs` for narrowing.                       |
| State Machines                       | PORTS_WITH_ADAPTATION | Hand-rolled `match` over discriminated-union states for small machines; `transitions` for large.       |
| Cross-Cutting Services               | PYTHON_SPECIFIC_VARIANT | `contextvars`-backed module singletons; no `std::variant` argument.                                   |
| Performance Discipline               | PORTS_WITH_ADAPTATION | Vectorise with numpy/pandas; `__slots__`; benchmark, profile, leave Python when budget cannot be met. |
| Comments                              | PORTS_AS_IS         | `//` -> `#`, `/* */` -> `"""..."""`; same content rules.                                              |
| Namespace Aliases                    | PYTHON_SPECIFIC_VARIANT | `import x as y` at module top; ruff `TID253` to forbid project-internal aliasing; no `using` analogue. |
| Const Placement                       | DROP                 | No analogue. Python uses `Final[T]` for read-only module-level constants.                              |

## Recommended Python `agent-context.md` structure

The file should be just as long as the C++ one (~400 lines), with the
same shape. Section ordering should follow the C++ order to make
mapping mental-model-easy for someone fluent in both files.

Concrete section list for `python-agent-context.md`:

1. **Core Lens** (verbatim port)
2. **Layout** (Python package layout: `src/<pkg>/`, tests outside src)
3. **Tests** (verbatim port; pytest mechanics)
4. **Debugging** (verbatim port; Python tooling)
5. **Domain Ownership** (Python `types.py` per package)
6. **Forward Dependencies** (DAG, `import-linter`, no relative imports)
7. **Types Carry Proof** (NewType, frozen dataclass, pydantic at boundary)
8. **Strong-Type Ergonomics** (NewType identity vs dataclass wrappers)
9. **Designated Initializers** (kw-only dataclasses, no positional)
10. **Functional Core, Imperative Shell** (sync inner, async/IO at edge)
11. **Pipelines** (Callable fields, wiring at the entry point)
12. **Runtime And Threads** (loop, call_soon_threadsafe, anyio TaskGroup,
    free-threaded sidebar)
13. **Error Handling** (exceptions, top-level catch-all, no exceptions
    across domain seams without translation)
14. **Invariants And Rollback** (ExitStack.callback, dataclasses.replace)
15. **Declarative Style** (generators, itertools, comprehensions, pipe)
16. **Variants, Protocols, And Generics** (match + assert_never, Protocol,
    PEP 695, TypeIs)
17. **State Machines** (match over discriminated states; library escape
    hatch for large machines)
18. **Cross-Cutting Services** (contextvars-backed module singletons)
19. **Performance Discipline** (vectorise, __slots__, profile, leave
    Python when budget cannot be met)
20. **Comments** (verbatim port with syntax swap)
21. **Imports and Namespace Aliases** (absolute imports, ruff TID rules)
22. **Final and Read-Only** (Final[T], frozen=True, slots=True)

## Placeholder vocabulary

The C++ file uses `lib::` as a placeholder for in-house utilities. The
Python equivalent is the codebase's chosen package; common shapes:

- `<project>` or `<project>.core` for the in-house utilities.
- `<project>.types` for domain types per the "Domain Ownership"
  section.
- `<project>.errors` for the per-domain exception hierarchies.

Document the placeholder mapping at the top of the Python
agent-context file the same way the C++ file documents `lib::`.
