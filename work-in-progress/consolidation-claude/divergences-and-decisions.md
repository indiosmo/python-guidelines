# Divergences between Claude and Codex, with recommended resolutions

Each entry: the divergence, what each side said, the evidence each cites,
and the recommended resolution. The orchestrator should treat these as the
list of items to reconcile in the final plan.

---

## D1. Guide granularity: many narrow files vs few broad files

**Claude:** 22 narrower files in 4 folders mirroring the cpp shape:
`python-design-principles/`, `python-testing-principles/`,
`python-debugging-principles/`, `agent/`. Cite `cpp-claude/00-summary.md`
recommended structure (14 design + 8 testing + 4 debugging + 2 agent + a
new tooling guide).

**Codex:** 11 broader files in 6 folders:
`python-projects/`, `python-design/`, `python-runtime/`, `python-testing/`,
`python-debugging/`, `python-style/`. Cite
`plan-codex/python-guidelines-port-plan.md` "Recommended Durable Guide
Set" -- single files like
`types-data-models-and-validation.md`,
`error-handling-invariants-and-reliability.md`,
`debugging-and-observability.md` that fold 2-3 cpp guides into one.

**Recommended resolution:** Adopt Claude's narrower-file shape. Three
reasons:

1. The cpp guides are scannable at this granularity; the team has
   already proved this works for a six-month period of active use.
2. The always-loaded agent-context maps one-to-one with the topical
   guides; collapsing topical guides into broader files breaks the
   index.
3. Each section is independently citable from CI failures, PR
   comments, and review tools.

The trade-off: more files to maintain. Codex's worry is real but the
sections inside Codex's broader files are largely the same outline; the
incremental cost of splitting them by topic is small.

If the owner prefers Codex's broader shape, the section outlines in
`design-guides.md`, `testing-guides.md`, `debugging-guides.md` still
apply -- they would just be folded into fewer files. The mechanics work
either way.

---

## D2. Cross-cutting services: contextvars singleton vs explicit composition

**Claude:** `contextvars`-backed module singletons are the default for
cross-cutting services (logger, clock, timer, settings, tracing). Cite
`cpp-claude/design/cross-cutting.md`: "module-level singleton with
contextvars override" is the recommended shape; the C++ `std::variant`-
backed global is dropped because Python's module-level globals are
already a single address. The argument: contextvars composes with
asyncio (per-task), threads (per-thread), and pytest fixtures
(per-test) automatically; the alternatives (constructor injection,
DI framework, `unittest.mock.patch`) each have failure modes the
singleton avoids.

**Codex:** Provider modules are an option, but the default leans on
explicit composition at the edge plus `ContextVar` for request-scoped
context only. Cite `cpp-codex/design/cross-cutting.md`: "Use explicit
parameters for domain dependencies. Use module-level provider functions
only for genuinely cross-cutting services. Use ContextVar for
request-scoped context." Warns against "import-time configuration" and
"global mutable state".

**Recommended resolution:** Both opinions are correct; they apply at
different scales. Recommend the Claude shape (contextvars singleton) for
the three services that genuinely cut across the program (logger, clock,
settings) and the Codex shape (explicit constructor parameters) for
everything else. Document both in `cross-cutting-services.md` with a
decision matrix:

| Service shape                              | Use                                                   |
|--------------------------------------------|-------------------------------------------------------|
| Module-level singleton + contextvars       | Logger, clock, timer, settings, tracing.              |
| Explicit constructor parameter             | Domain-specific dependencies (a repository, a vendor session). |
| `ContextVar` only (no singleton facade)    | Request-scoped context (request_id, user, tenant).    |
| pytest fixture + `monkeypatch.setattr`     | Test-time substitution for either of the above.       |

The shared rule that both sides agree on: configure at the application
entry point, never at module load.

---

## D3. Concurrency library: anyio default vs asyncio.TaskGroup default

**Claude:** anyio recommended as the default once the program does
meaningful concurrency; `asyncio.TaskGroup` for the simplest fan-out
cases. Cite `cpp-claude/research/structured-concurrency.md`: anyio's
TaskGroup carries its own cancel scope, supports nested cancel scopes,
treats cancellation as first-class -- which lines up with the
"runtime owns the threads, inner code is single-threaded" discipline
from the cpp guides.

**Codex:** Conservative: stdlib `asyncio.TaskGroup` first; recommend
anyio "only for async-heavy projects". Cite
`consolidation-codex/research-consolidation.md` "Decisions and Tradeoffs
Requiring Explicit Policy" section: "Decide whether to recommend anyio
as the default structured-concurrency layer or keep the guide centered
on stdlib asyncio."

**Recommended resolution:** Codex's conservative default is the right
starting point. `asyncio.TaskGroup` lands in 3.11 and is enough for the
"fan out and join" case; anyio earns its dependency when cancel scopes
become load-bearing. Document anyio as the swap-in when:

- The program uses timeouts and the timeout source needs to compose
  through library boundaries (`anyio.fail_after`).
- The program needs to cancel a single child without cancelling the
  whole group (anyio's named cancel scopes do this; TaskGroup does not).
- The library needs to support both asyncio and trio (anyio's job).

For most projects, the stdlib TaskGroup is enough; document anyio as a
cross-link from the `runtime-and-concurrency.md` guide.

---

## D4. Default data carrier: frozen+slots dataclass vs "by layer"

**Claude:** `@dataclass(frozen=True, kw_only=True, slots=True)` as the
default in-domain data carrier; pydantic at trust boundaries. Cite
`cpp-claude/00-summary.md` "Use frozen slotted dataclass as the default
data-carrying shape". The argument: frozen avoids accidental mutation;
slots cut memory and improve attribute access; kw_only bans positional
construction for adjacent same-shaped fields.

**Codex:** Choose per layer; do not "freeze everything by reflex". Cite
`cpp-codex/design/compile-time-correctness.md`: "`frozen=True` is useful
for value objects, but mutable domain state should be plainly mutable
behind methods. Do not freeze everything by reflex."

**Recommended resolution:** Codex is correct in principle; Claude is
correct as a default. Recommend the framing:

- **Value objects**: frozen, slots, kw_only. Default for in-domain data
  passed through a pipeline. Use `dataclasses.replace` to evolve.
- **Stateful aggregates**: mutable; methods carry the mutation contract;
  name the methods so the mutation is visible (`add_order`,
  `mark_to_market`). Do not freeze.
- **Trust-boundary objects**: pydantic. `Field(...)` for constraints;
  `model_validator(mode="after")` for cross-field invariants.
- **DataFrame contracts**: pandera. Cross-row validators behind it.

This is the factors2 split documented in
`factors-claude/patterns/pydantic-pandera-dataclasses.md`, which has
already proved itself in production.

---

## D5. Result library: in-domain pattern vs boundary-only option

**Claude:** Exceptions are the default in-domain; document Result
libraries as the boundary form for codebases that want type-checker-
visible failure branches. Cite `cpp-claude/design/error-handling.md`:
"Recommend [Result] as the boundary form for code that needs to be
exhaustive on errors, not as the in-domain form."

**Codex:** Same default (exceptions in-domain) but more skeptical of
Result libraries even at the boundary. Cite
`cpp-codex/design/error-handling.md`: "Drop the two-result-types frame";
no in-text guidance on when Result is useful.
`consolidation-codex/research-consolidation.md` lists it as an open
question: "Failure style: decide whether exceptions are the default
inside domains, whether result objects get a first-class pattern, and
how boundary translation should look."

**Recommended resolution:** Both agree on exceptions in-domain.
Recommend Claude's framing for the boundary: document Result as an
option, with a short example showing where it pays off (typically: a
boundary that processes a batch of N items and needs to report per-item
success/failure without an exception per item). Do not require Result;
make it discoverable so codebases that want it know where to start.

---

## D6. Approval-test plugin choice

**Claude:** Recommend `approvaltests` + `pytest-approvaltests` with
`PythonNativeReporter`. Cite `factors-claude/tooling-baseline.md`:
factors2 already wires `--approvaltests-use-reporter=PythonNative` in
`pyproject.toml`.

**Codex:** Open question. Cite
`consolidation-codex/research-consolidation.md`: "Decide whether to
standardize a pytest snapshot or approval plugin, use plain expected
files, or describe the pattern without naming a library."

**Recommended resolution:** Use Claude's recommendation. factors2 already
proves the choice works in production; `pytest-approvaltests-geo` is the
adjacent option for geospatial snapshots and shares the same reporter
shape. The alternatives (`syrupy`, `snapshottest`) are credible but the
team has shipped with approvaltests; switching for a new repo is a
solution looking for a problem.

---

## D7. Ruff rule set: explicit baseline vs principles-only

**Claude:** Explicit baseline rule set. Cite
`cpp-claude/research/ruff-rule-sets.md`: a documented selection with the
factors2 codebase as the test case (broad `E F B I N Q UP C4 ARG NPY PD
PERF PL` plus `T201`, `S`, `RUF`, `SIM`).

**Codex:** Open question. Cite
`consolidation-codex/research-consolidation.md`: "Decide whether the
guide gives a baseline rule set, a strict rule set, or principles for
selection only."

**Recommended resolution:** Use Claude's explicit baseline. A
principles-only guide forces every new project to repeat the same
research; a documented baseline is a starting point that the project can
deviate from with a comment. Recommend the baseline live in
`python-projects-and-tooling/tooling-baseline.md` with a per-rule
justification table; recommend the strict mode (additional `ANN`, `PLE`
families) live as a documented strict variant for new code.

---

## D8. Type checker default: pyright vs "choose one"

**Claude:** pyright is the default; pyrefly is the swap-in once
conformance stabilises; ty is not yet recommended; mypy is the legacy
fallback. Cite `cpp-claude/research/typecheckers-2026.md` with explicit
conformance numbers.

**Codex:** "Choose one type checker policy" -- pyright, pyrefly, or
mypy as a policy decision. Cite
`consolidation-codex/research-consolidation.md` and
`plan-codex/python-guidelines-port-plan.md` "Policy Decisions Before
Writing": "Primary type checker: choose Pyright, Pyrefly, mypy, or a
documented combination."

**Recommended resolution:** Make pyright the default in the guide;
document pyrefly explicitly as the credible alternative for large
codebases that want faster whole-project feedback; document mypy as the
legacy fallback for projects that already have it in CI. The decision is
small enough that "pick one and stick with it" is more useful than
"here's a comparison". The strict-vs-standard policy is a separate
question covered in D9.

---

## D9. Strictness policy: global strict vs directory-scoped

**Claude:** `pyright --strict` as the default CI gate. Cite
`cpp-claude/00-summary.md` "Make `pyright --strict` (or `pyrefly --strict`)
the default CI gate."

**Codex:** Directory-scoped strict, or staged adoption. Cite
`cpp-codex/design/compile-time-correctness.md`: "use strictness by
directory" and `consolidation-codex/research-consolidation.md`'s
follow-up question: "Decide whether strict mode is global,
directory-scoped, or aspirational."

**Recommended resolution:** Recommend global strict in the guide as the
target state. For codebases adopting the guidelines, recommend the
ratchet pattern:

1. Start with `typeCheckingMode = "standard"` globally.
2. Enable strict on a directory at a time as it gets cleaned up,
   committed via `[tool.pyright]` per-directory overrides.
3. Final state: global strict with per-file `# pyright: standard`
   overrides on directories that still need a pass.

The same pattern works for pyrefly's strictness levels.

---

## D10. Cross-link to MONOREPO.md vs supersede it

**Claude:** Reference MONOREPO.md from
`python-projects-and-tooling/project-layout.md`; treat it as the
durable local convention for the monorepo shape.

**Codex:** Treat MONOREPO.md as the seed for the architecture guide; the
new architecture guide builds on its boundary-first view. Cite
`consolidation-codex/research-consolidation.md`: "Future guides should
build from that boundary-first view instead of copying the C++
repository structure mechanically."

**Recommended resolution:** Both. Reference MONOREPO.md from
`project-layout.md` as the canonical local source for monorepo shape;
have `architecture.md` quote the boundary-first framing as the design
motivation. The two files are not in conflict; MONOREPO.md is about the
filesystem shape and `architecture.md` is about the conceptual shape that
shape encodes.

---

## D11. Data-engineering scope: dedicated guide vs woven through

**Claude:** Dedicated `data-pipeline-and-dataframes.md` guide,
conditional on the user codebase having dataframes. Cite
`factors-claude/patterns/data-pipeline.md` and
`factors-claude/patterns/pydantic-pandera-dataclasses.md` for the
factors2 patterns that this guide would lift.

**Codex:** Open question. Cite
`consolidation-codex/research-consolidation.md`: "Decide whether
pandas, Polars, Pandera, SQL, dbt, and Dagster deserve a separate guide"
and "Add a focused data-engineering slice for pandas, Polars, Pandera,
SQL generation, dbt, and Dagster if the final guidelines target
analytics codebases."

**Recommended resolution:** Yes, dedicated guide. The factors2 codebase
is dataframe-heavy and the patterns are distinctive (pandera + Pydantic
+ dlt + ibis + dbt + Dagster). The principles thread through other
guides (pandera fits `types-and-correctness.md`, Dagster fits
`runtime-and-concurrency.md`), but the layer-choice decision matrix and
the indicator-purity rule deserve a focused home.

If the first target codebase is not dataframe-heavy, defer
`data-pipeline-and-dataframes.md` to a second pass; the principles
inside other guides do not require it.

---

## D12. Agent docs scope in first writing pass

**Claude:** In scope. Cite `cpp-claude/agent-context/cpp-agent-context.md`
and `cpp-claude/agent-context/cpp-agent-examples.md`: the always-loaded
file is the most-used artifact in cpp-side production and the Python
equivalent should be the same shape.

**Codex:** Open question. Cite
`consolidation-codex/research-consolidation.md`: "Decide whether
agent-context docs are in scope for the first durable guide set."

**Recommended resolution:** In scope. The agent-context file is what
agents load on every turn; defer it and the agents work from training
data that may not reflect the project's conventions. The skill itself
(`agent/skills/repo-specific-python-guidelines/`) is fine to defer.

---

## D13. Logger choice: loguru recommended vs library-neutral

**Claude:** Recommend loguru as the default for new projects; structlog
as the alternative for structured-logging-first stacks; stdlib `logging`
for libraries that should not impose a logger on consumers. Cite
`cpp-claude/debugging/logging.md` and `factors-claude/anti-patterns.md`
entry L (mixing stdlib `logging` and loguru is a real failure mode).

**Codex:** Library-neutral. Cite `cpp-codex/debugging/logging.md`: "Python
should use the stdlib logging stack by default unless a project has
chosen structlog, loguru, or another structured logger."

**Recommended resolution:** loguru recommended for applications; stdlib
`logging` for libraries; structlog as the alternative when the project
needs strict key/value semantics. Library-neutral guidance is too weak in
practice -- a project that does not pick will end up with the
factors2-style failure mode (some modules use stdlib, some use loguru,
configuration drifts).

For libraries that get installed in someone else's runtime, the right
choice is stdlib `logging` because the consumer's logger configuration
should win; loguru's
`logger.add(logging.StreamHandler())`-equivalent interception lets a
loguru-using application capture stdlib log records emitted by libraries.

---

## D14. Pyfakefs / pytest-benchmark: declared-and-used vs declared-then-pruned

**Claude:** Cite `factors-claude/anti-patterns.md` entry Q: every dev dep
has a documented invocation (a Makefile target, a doc file, a test that
uses it). Pyfakefs, pyinstrument, and pydeps are declared in factors2 but
not invoked anywhere; the guideline rule should either require usage or
require removal.

**Codex:** Less opinionated; recommends pyfakefs and pytest-benchmark in
the testing toolkit without addressing the "declared-but-unused" failure
mode.

**Recommended resolution:** Adopt Claude's rule. Every dev dep has a
documented invocation. Specifically:

- pyfakefs: used in tests for filesystem-touching code (ETL loaders, CSV
  writers, atomic-rename patterns).
- pytest-benchmark: used in `benchmarks/`, with a Makefile target.
- pydeps: used to render and commit an architecture SVG, with a Makefile
  target.
- pyinstrument: used in a documented profiling cookbook in the
  performance guide.

If a dep does not have an invocation, prune it.

---

## D15. Comments rule: negative-documentation phrasing

**Claude:** Negative-documentation rule is verbatim from the user's
global CLAUDE.md, with Python-specific examples added (the
"non-responsibility" framing inside a docstring is the most common
Python instance, e.g. `"""Stateless converters... integration of the
resulting events is the caller's responsibility."""`).

**Codex:** Same rule, less Python-specific renumbering. Cite
`cpp-codex/design/comments.md`.

**Recommended resolution:** Adopt Claude's framing with the Python
docstring examples; the rule is identical, the Python-specific tells
make it more actionable in code review.

---

## D16. "Drop these from cpp" lists

**Claude:** Drop only `qt-gui.md` (unless the user codebase has Qt).
Reframe `sanitizers.md` as `static-analysis-and-runtime-checks.md`. The
rest port (most with adaptation).

**Codex:** Drops more aggressively. Cite
`consolidation-codex/research-consolidation.md` "Material to Avoid
Porting from C++": std::expected, LEAF, BOOST_LEAF_ASSIGN, templates,
concepts, constexpr, dependent-false, type-erasure terminology,
std::variant visitor mechanics, RAII / destructor cleanup, friend-based
test probes, Catch2 / CTest mechanics, Qt, ASan / TSan / MSan, container
choice based on allocation, "no allocations on hot path", header /
translation unit / move semantics / undefined behavior framing,
std::function / function_ref / inplace_function callable cost.

**Recommended resolution:** Both lists are right but they answer
different questions. Claude's list is "which files to drop"; Codex's
list is "which content not to port inside the files that survive".
Combine them:

- Drop these files: `qt-gui.md` (unless the codebase has Qt).
- Inside surviving files, do not port: LEAF / std::expected / template
  metaprogramming syntax / RAII destructor terminology / Catch2
  REQUIRE-CHECK distinction / hot-path allocation advice / friend
  declarations. These are mentioned in the Claude per-file analyses as
  "DROP" classifications at the section level.

The combined list should appear in `divergences-and-decisions.md` as
this entry, and inform the writing pass.
