# Analysis: cpp-design-principles/architecture.md

Disposition: **PORTS_AS_IS** for the principles; **PORTS_WITH_ADAPTATION** for the
section that uses C++ headers/CMake as the unit of dependency direction.

## Section-by-section

### Theory of the domain (Naur / Brooks / Reeves / Evans)

- **Classification:** PORTS_AS_IS.
- **C++ principle (1 sentence):** Source code is the working theory of the
  domain, and the team's understanding survives only as long as the code stays
  legible and malleable.
- **Python rendering:** Same prose, same references. No Python-specific
  changes; the section is language-agnostic. Replace one C++ helper name in
  the running example (`std::string` -> `str`).
- **Open questions:** None.

### Domain separation with adapters

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Components own their domains; cross-domain talk happens
  through dedicated adapter modules.
- **Python rendering:** Identical principle. The example becomes "an order
  routing package depending on a fix_routing package depending on a
  fix_vendor adapter package". The unit of separation in Python is the
  **package** (a directory with `__init__.py`) and the explicit import
  graph; ruff's `TID252` (no relative parent imports) and `import-linter` (a
  third-party tool) enforce the arrows.

### Domain-owned types

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Each domain has its own `types` namespace; a same-named
  type in two domains stays distinct.
- **Python rendering:** Each domain package ships a `types.py` (or a
  `_types.py` for the internal-only case) module. Same-shape types in two
  domains become two `NewType`s or two frozen dataclasses with the same
  field shape but distinct classes -- pyright/pyrefly will keep them apart.
  Cross-domain construction reads as
  `risk.types.OrderId(routing.types.OrderId(value))` -- the adapter that
  does the translation is the only place that builds the target type from
  the source.
- **Open questions:** Naming: `types.py` works in Python, but the file does
  not have to nest in a `types` subnamespace -- `from domain import types`
  already qualifies it. Python's lack of nested namespaces inside a single
  module makes the `domain::types::` flavour of the C++ rule unnecessary;
  the `types.OrderId` qualifier comes for free.

### Internal shape of an adapter module

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Adapter modules split into an abstract interface, per-
  vendor implementations, and a runtime layer.
- **Python rendering:** Same shape: `protocols.py` for `Protocol`-typed
  interfaces, a `vendors/` subpackage with one module per vendor, and a
  `runtime.py` or `runtime/` subpackage that composes them with whatever
  Python equivalent of "vendor SDK session" the integration provides.

### Forward-only dependencies

- **Classification:** PORTS_AS_IS for the principle; PORTS_WITH_ADAPTATION
  for the enforcement tooling.
- **C++ principle:** Dependencies form a DAG; reading code follows the
  arrows.
- **Python rendering:** Same rule. Python lacks header files, so the unit
  is the **module-level import** and the **package-level import**.
  Cycles are detected by `python -X importtime ...`, `pydeps`,
  `import-linter` (declarative contracts), and ruff's `TID252`. Suggest
  picking one of pydeps or import-linter and wiring it into CI.

### No implicit contracts

- **Classification:** PORTS_AS_IS.
- **C++ principle:** A function takes everything it needs and returns
  everything it produces; do not leave residue for the next call to read.
- **Python rendering:** Same example, swap `struct working_set` for a
  `@dataclass`, swap `column`/`table` for whatever stands in (the
  `pandas.DataFrame` example fits the natural Python idiom). The bad
  version mutates the dataclass; the good version returns new values.
  Special call-out: in pandas-heavy Python code, the **chained-method
  style** (`df.pipe(untie).pipe(rank)`) is the idiomatic equivalent of the
  good C++ version.

### Functional core, imperative shell

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Pure functional logic at the core; threads, sockets,
  timers, and I/O live in a thin shell near `main`.
- **Python rendering:** Identical. Concrete tools change: the shell is
  where `asyncio.run(...)` / `anyio.run(...)` lives, where
  `httpx.AsyncClient()` is constructed, where the database connection
  pool is built. The functional core stays sync (it does not need to be
  async unless it is doing I/O), takes plain values, returns plain
  values. The "library deep in the graph should not open a socket"
  becomes "no `requests.get(...)` in a domain function".
- **Open questions:** Whether to mandate that domain-layer functions are
  sync-only. Recommendation: yes by default; introduce `async def` only
  when the function genuinely awaits I/O. Sync domain functions are
  trivially callable from both sync test runners and from async shells
  via `asyncio.to_thread`.

### Testability without mocks

- **Classification:** PORTS_AS_IS.
- **C++ principle:** If a unit test needs a mock vendor SDK or a fake
  thread pool to exercise the inner layer, the design is bent.
- **Python rendering:** Same rule. The Python idiom equivalents: if a test
  needs `unittest.mock.patch` against a third-party library to exercise a
  domain function, the function is doing too much. The dependency-
  injection escape hatches (FastAPI's `Depends`, `dependency-injector`)
  are a smell at the unit-test level -- they belong in the shell.

## Suggested Python file: `python-design-principles/architecture.md`

Verbatim port with the changes called out above and a short Python-tooling
appendix at the bottom (import-linter / pydeps for arrow enforcement).
