# Cross-cutting themes

Six recurring themes bind the proposed guides together. They are the cpp
themes renamed and refocused for Python. Repeat them in
`python-agent-context.md`'s Core Lens and in each guide's opening
paragraph so a reader who only loads one file still encounters the larger
discipline.

## 1. Domain ownership

Every concept has one package that owns it. The owner exports types,
errors, and operations; everyone else imports from there. Same-named
types in two domains stay distinct (separate `NewType` or separate frozen
dataclass classes); cross-domain construction is the explicit job of an
adapter.

The Python-specific tells when this theme is violated:

- Importing a sibling domain's `types.py` deep inside a function body.
- Reaching for `unittest.mock.patch` on a third-party library inside a
  domain unit test.
- Calling `vendor_sdk.Order(...)` inside a domain function.

References: `python-design-principles/architecture.md`,
`factors-claude/patterns/module-layout.md`, cpp `architecture.md`.

## 2. DAG-shaped imports

Module-level imports form a directed acyclic graph. The reader follows the
arrows; cycles either rot the design or hide a refactor opportunity. The
unit of separation in Python is the package + the explicit module-level
import; enforcement comes from `pydeps` (visualisation), `import-linter`
(declarative contracts), and ruff `TID252` (no parent-relative imports).

The Python-specific tells:

- `from .. import x` inside a leaf module.
- `if TYPE_CHECKING:` everywhere because direct imports cause cycles.
- Shared types living in `pipeline/models.py` when `database/` and
  `pipeline/` both need them (the right move is a separate `domain/`
  module that both depend on; cite
  `factors-claude/patterns/module-layout.md`).

References: `python-design-principles/architecture.md`,
`python-projects-and-tooling/project-layout.md`.

## 3. Push effects to the edge

Domain logic stays sync, takes plain values, returns plain values.
Threads, event loops, HTTP clients, database sessions, file I/O,
environment reads, logging configuration, and framework dependency
injection live in the imperative shell near `main`. Tests for the core do
not require mocks for I/O, sockets, clocks, or vendor SDKs.

The Python-specific tells:

- An `async def` deep in a domain module that does no `await`.
- A module-level `httpx.Client(...)` constructed at import time.
- A `logging.basicConfig(...)` inside a domain module.
- A `_TEST_MODE = os.environ.get("TEST_MODE")` constant at module top.
- A pytest fixture that has to patch six different modules to isolate one
  function.

References: `python-design-principles/architecture.md`,
`python-design-principles/runtime-and-concurrency.md`,
`python-design-principles/cross-cutting-services.md`,
`factors-claude/patterns/data-pipeline.md`.

## 4. Types carry the proof, with gradual-typing caveats

Types and static analysis name contracts and catch mistakes at
type-check time. They are not a runtime enforcement layer; pydantic and
pandera fill that role. The Python guide takes a position that the cpp
guide does not need: it documents the typing escape hatches. `Any`,
`cast`, and `# type: ignore` are the explicit ways to opt out, and there
are rules for when each is correct.

The Python-specific tells when this theme is over-applied:

- Phantom-type wrapper classes whose only purpose is to call
  `.value()` everywhere downstream.
- `cast(...)` chains threaded through tests because the type checker
  cannot follow the dataframe shape.
- A `TYPE_CHECKING` block on every other import.

The tells when this theme is under-applied:

- `hasattr` + `inspect.signature` inside a typed union to recover the
  structure the type system already had (cite
  `factors-claude/anti-patterns.md` entry F).
- Trailing `raise ValueError("Unknown ...")` on a closed enum (entry E).
- `cast(...)` to fake a typed empty DataFrame in tests (entry G).
- A 40-arm union with no discriminator.

References: `python-design-principles/types-and-correctness.md`,
`factors-claude/patterns/typing-discipline.md`,
`codex-research/modern-python-tooling-and-typing.md`.

## 5. Idempotency and rollback are first-class

State changes either commit at the end (build new state in locals; mutate
once everything fallible has succeeded) or carry a rollback callable that
fires on failure (`contextlib.ExitStack.callback`). Operations that may
be retried are designed to be safe on retry: dlt's `merge` write
disposition + primary key, `dict.setdefault` for atomic
absent-then-insert, `with conn.transaction():` for multi-statement SQL
writes.

Recoverability is classified explicitly: schedule only the irrecoverable
steps; document the rest as "rerun on demand" so missed runs are not
silent failures. The
`doc/adr/0001-schedule-only-economatica-downloads.md` ADR in factors2 is
the model.

The Python-specific tells:

- A simulation that mutates shared state and never validates the
  snapshot before persisting it.
- Stateful caches that expose both the source and the derived dict as
  public attributes (cite `factors-claude/anti-patterns.md` entry I).
- Silent staleness: `if value is not None: position.mtm_price = value`
  with no comment about what carrying yesterday's price means (entry K
  context).

References: `python-design-principles/invariants-and-rollback.md`,
`factors-claude/patterns/state-invariants.md`,
`python-design-principles/data-pipeline-and-dataframes.md`.

## 6. Root cause over symptom

Debugging traces the bad value backward to the layer that produced it
and fixes the root, then reinforces the intermediate layers. Logging
sits before the likely failure, not after. Validation messages are part
of the contract, tested via `re.escape(...)`. `logger.exception(...)`
inside `except` preserves the traceback; logging-and-re-raising is the
shape that loses both audiences (cite
`factors-claude/anti-patterns.md` entry M). One hypothesis at a time;
multiple simultaneous fixes hide which one worked.

The Python-specific tells:

- `except Exception as e: logging.error(f"...{e}")` then `raise` (entry
  M).
- `except Exception: st.toast(str(e))` swallowing eight failure classes
  into one user-visible toast (entry B).
- A `# raise RuntimeError(...)` commented-out alternative branch (entry
  K).
- A `print(timeit.default_timer() - t1)` left in production (entries C
  and D).

References: `python-debugging-principles/root-cause-tracing.md`,
`python-debugging-principles/defense-in-depth.md`,
`python-debugging-principles/logging.md`,
`factors-claude/patterns/error-handling.md`,
`factors-claude/patterns/logging-observability.md`.

---

## A note on what we did not lift from cpp

The cpp themes are: domain ownership; DAG-shaped headers; functional
core, imperative shell; types-carry-the-proof; idempotency; root-cause
debugging. Five of the six map almost one-to-one. The one we modify is
the typing theme. cpp's "compile-time correctness" carries the implicit
claim that the property holds at runtime. The Python theme has to be
explicit: the property holds at type-check time (CI) and at boundary-parse
time (runtime); inner Python remains dynamic. Documenting the escape
hatches is therefore not optional in the Python guide -- it is the only
way the theme survives contact with realistic Python code.
