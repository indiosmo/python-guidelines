# Python Guidelines Porting Plan

This is the active working plan for turning the existing research artifacts
into the first durable Python guideline set. `GOAL.md` is the stable anchor;
this file coordinates the remaining consolidation and writing work.

## Objective

Port the generalizable C++ guidelines into idiomatic Python guidance. Keep the
C++ principles where they transfer, rewrite the mechanisms where Python has a
different idiom, and use factors2 plus current Python tooling research to keep
the guidance practical.

The guidelines must be Pythonic, not a direct mapping of C++ concepts into
Python syntax. When the C++ source guidance and idiomatic Python practice
diverge, the Pythonic approach wins.

The plan should not carry all guide content. It points writing agents at the
right research inputs, resolves the major structure choices, and defines the
order of work.

## Current Inputs

Use these as source material while the port is in progress:

- `GOAL.md`: stable project anchor.
- `/home/msi/repos/cpp-guidelines/`: upstream C++ guideline source.
- `MONOREPO.md`: local Python monorepo shape and boundary philosophy.
- `work-in-progress/exploration/cpp-claude/`: Claude's per-C++-guide port
  analysis and Python research.
- `work-in-progress/exploration/cpp-codex/`: Codex's parallel per-C++-guide
  port analysis.
- `work-in-progress/exploration/codex-research/`: modern Python tooling,
  typing, architecture, testing, and debugging research.
- `work-in-progress/exploration/factors-claude/`: factors2 patterns,
  anti-patterns, and tooling baseline.
- `work-in-progress/claude-factors-antipatterns.md` and
  `work-in-progress/codex-factors-antipatterns.md`: additional factors2
  anti-pattern passes.
- `work-in-progress/consolidation-claude/`: detailed guide outlines,
  cross-cutting themes, divergences, and decisions.
- `work-in-progress/consolidation-codex/research-consolidation.md`: broader
  Codex consolidation.
- `work-in-progress/plan/final-plan.md` and
  `work-in-progress/plan-codex/python-guidelines-port-plan.md`: existing plan
  drafts.

When durable guides are written, do not link to `work-in-progress/` from those
guides. Pull the needed reasoning into the durable guide, or link only to
durable sources.

## Consolidation Work

### 1. Consolidate duplicated C++ exploration

Create `work-in-progress/exploration/cpp-source-python-port.md`.

Inputs:

- `work-in-progress/exploration/cpp-claude/`
- `work-in-progress/exploration/cpp-codex/`

Shape:

- Purpose and source coverage.
- Disposition map for the C++ guide topics.
- Common topic map for design, testing, debugging, and agent context.
- Python replacements for C++-specific mechanisms.
- Places where idiomatic Python deliberately deviates from the C++ source.
- Research notes to preserve.
- Dropped or conditional topics.
- Open questions that still affect writing.

Use `cpp-codex/00-summary.md` as the narrative spine. Fold in the richer
`cpp-claude/research/*.md` files and the stricter file mapping from
`cpp-claude/00-summary.md`.

### 2. Consolidate Python research and factors2 exploration

Create `work-in-progress/exploration/exploration.md`.

Inputs:

- `work-in-progress/exploration/cpp-source-python-port.md`
- `work-in-progress/exploration/codex-research/`
- `work-in-progress/exploration/factors-claude/`

Shape:

- Purpose and baseline.
- Modern Python baseline: Python version, typing syntax, uv, Ruff, type
  checker policy, pytest, CI, and documentation drift checks.
- Architecture and boundaries: package layers, adapters, functional core,
  explicit pipelines, Dagster, dbt, dlt, and dataframe boundaries.
- Typing and models: Pydantic unions, Pandera schemas, dataclasses, Protocol,
  `Any`, `cast`, and type-ignore policy.
- Testing and debugging: pytest-native patterns, approval tests, benchmarks,
  deterministic waits, and the root-cause loop.
- Operations and reliability: recoverability classes, transactions,
  idempotency, logging, and failure translation.
- Promote and avoid table distilled from factors2.
- Open decisions that remain after consolidation.

### 3. Consolidate anti-patterns

Create `work-in-progress/exploration/antipatterns.md`.

Inputs:

- `work-in-progress/claude-factors-antipatterns.md`
- `work-in-progress/codex-factors-antipatterns.md`
- `work-in-progress/exploration/factors-claude/anti-patterns.md`

Shape:

- Selection rules for what belongs in the anti-pattern briefing.
- Package boundaries and tool metadata.
- Configuration, environment, and process-global state.
- Data and type contracts.
- Dynamic dispatch and extension points.
- Mutability, caches, and dataframe ownership.
- SQL, text-to-code, and generated maintenance workflows.
- Error handling, logging, and operational scripts.
- Concurrency, multiprocessing, async fan-out, and rate limiting.
- Testing numeric, tabular, and domain behavior.
- Documentation and repository hygiene.

Each entry should state whether it is observed, research-derived, or
enforcement-only. Keep concrete factors2 examples as evidence, but write the
guideline candidate in domain-neutral Python terms.

### 4. Reconcile the final plan

After the three consolidated exploration artifacts exist, revise this file if
needed and treat it as the active plan. Use
`work-in-progress/plan/final-plan.md` as the detailed base, with
`work-in-progress/plan-codex/python-guidelines-port-plan.md` as a check against
over-fragmentation.

Before writing `python-design-principles/error-handling.md`, do a focused
Python error-handling pass. The output should describe current idiomatic Python
practice: built-in exceptions, when a custom exception type earns its keep,
exception chaining, `ExceptionGroup` where concurrent work can fail in several
places, `contextlib`, warnings, sentinel values such as `None` where the API
already makes absence normal, and explicit boundary translation. The pass
should actively reject C++ LEAF or Result mimicry unless a Python boundary case
clearly benefits from a result-shaped value.

## Resolved Structure

Use the narrower, C++-parallel guide structure from the Claude consolidation.
The Codex broad-file plan remains useful as a thematic cross-check, but the
narrow structure is better for citation, review comments, and agent context.
This structure mirrors the C++ repo for navigation only. It must not force
direct conceptual mappings where Python has a more idiomatic answer.

Durable output folders:

- `python-projects-and-tooling/`
- `python-design-principles/`
- `python-testing-principles/`
- `python-debugging-principles/`
- `agent/`

Conditional outputs:

- `python-design-principles/data-pipeline-and-dataframes.md` earns its place
  when the target audience includes dataframe-heavy systems.
- `python-testing-principles/web-ui.md` earns its place only for web UI test
  guidance.
- Qt guidance stays dropped unless a Python Qt surface appears.
- `agent/skills/repo-specific-python-guidelines/` is deferred until the first
  guide set exists.

## Resolved Policy Defaults

These defaults come from the consolidation and should guide the first writing
pass:

| Topic | Default |
|---|---|
| Python version | Python 3.13 baseline, Python 3.14 forward target. |
| Type checker | Pyright by default; pyrefly as the fast alternative; mypy as legacy fallback. |
| Strict typing | Global strict as target; ratchet from standard directory by directory. |
| Package workflow | uv workspaces, locked installs, and `uv run` command surface. |
| Lint and format | Ruff for linting and formatting. |
| Data carriers | Frozen slotted dataclasses for value objects, mutable aggregates when mutation is the domain operation, Pydantic v2 at trust boundaries, Pandera at dataframe boundaries. |
| Error handling | Idiomatic Python exceptions inside a domain; use existing exception types when they communicate the failure clearly; introduce custom exception types only for meaningful domain failures or boundary translation. |
| Boundary failures | Translate failures explicitly at HTTP, CLI, job, queue, and UI boundaries. |
| Result style | Research and mention only where it is idiomatic for a Python boundary problem; do not port C++ Result or LEAF patterns as a default. |
| Concurrency | `asyncio.TaskGroup` baseline; anyio when cancel scopes or trio compatibility pay for the dependency. |
| Cross-cutting services | Context-local facades for logger, clock, timer, settings, and tracing; explicit parameters for domain-specific dependencies. |
| Logging | One logging stack per project; loguru for applications, stdlib logging for libraries, structlog when strict key/value logs are required. |
| Approval tests | approvaltests plus pytest-approvaltests unless a project has a stronger existing convention. |
| Import boundaries | import-linter for CI contracts, pydeps for visualization, Ruff import rules for cheap checks. |
| Comments | Present-tense, positive comments and docstrings; no past-tense refactor notes or non-responsibility narration. |

## Durable Guide Set

Create these guides in the first writing pass:

- `python-projects-and-tooling/README.md`
- `python-projects-and-tooling/project-layout.md`
- `python-projects-and-tooling/tooling-baseline.md`
- `python-projects-and-tooling/python-version-and-typing-stack.md`
- `python-design-principles/README.md`
- `python-design-principles/architecture.md`
- `python-design-principles/types-and-correctness.md`
- `python-design-principles/comments-and-docstrings.md`
- `python-design-principles/declarative-style.md`
- `python-design-principles/functional-programming.md`
- `python-design-principles/generics-and-protocols.md`
- `python-design-principles/decorators-and-metaprogramming.md`
- `python-design-principles/error-handling.md`
- `python-design-principles/invariants-and-rollback.md`
- `python-design-principles/state-machines.md`
- `python-design-principles/pipelines.md`
- `python-design-principles/runtime-and-concurrency.md`
- `python-design-principles/cross-cutting-services.md`
- `python-design-principles/performance.md`
- `python-testing-principles/README.md`
- `python-testing-principles/philosophy.md`
- `python-testing-principles/pytest-conventions.md`
- `python-testing-principles/test-patterns.md`
- `python-testing-principles/test-helpers.md`
- `python-testing-principles/error-path-testing.md`
- `python-testing-principles/condition-based-waiting.md`
- `python-testing-principles/approval-tests.md`
- `python-debugging-principles/README.md`
- `python-debugging-principles/root-cause-tracing.md`
- `python-debugging-principles/defense-in-depth.md`
- `python-debugging-principles/logging.md`
- `python-debugging-principles/static-analysis-and-runtime-checks.md`
- `agent/python-agent-context.md`
- `agent/python-agent-examples.md`
- Root `README.md`

## Writing Order

1. Project layout and tooling.
2. Architecture and boundaries.
3. Types, correctness, validation, and typing escape hatches.
4. Comments and docstrings.
5. Declarative style, functional programming, generics, and metaprogramming.
6. Error handling, invariants, state machines, and pipelines.
7. Runtime, concurrency, cross-cutting services, and performance.
8. Testing principles.
9. Debugging principles.
10. Agent context and examples.
11. Root and folder READMEs.

## Per-Guide Source Index

Use the detailed guide-by-guide briefs in
`work-in-progress/plan/final-plan.md`. After `exploration.md` and
`antipatterns.md` exist, writing agents should receive only:

- `GOAL.md`
- this plan
- the relevant section of `work-in-progress/plan/final-plan.md`
- `work-in-progress/exploration/exploration.md`
- `work-in-progress/exploration/antipatterns.md`
- the relevant original C++ guide, when exact parity matters

That keeps each writing prompt small while preserving traceability.

## Material Not To Port

Do not port these C++ mechanisms directly:

- Compile-time correctness as a literal Python guarantee.
- `std::expected`, LEAF, templates, concepts, `constexpr`, and type-erasure
  terminology.
- Custom exception hierarchies created merely to imitate C++ typed error
  channels.
- `std::variant` visitor mechanics.
- RAII and destructor cleanup as the primary model.
- Friend-based test probes, Catch2, CTest, Qt mechanics, or C++ sanitizer
  workflows.
- Container-choice, allocation, cache-locality, inlining, move-semantics, and
  undefined-behavior framing.
- Header, translation-unit, include, macro-expansion, and preprocessor framing.

Use Python replacements: type checkers, protocols, tagged unions, context
managers, fixtures, transactions, pytest, runtime warnings, profilers, and
diagnostic tooling.

## Verification

Before calling the first writing pass done:

- Check every durable guide against `GOAL.md`.
- Check that the root README points to durable guides, not temporary research.
- Check that each guide uses Python vocabulary and Python mechanisms.
- Check that code examples are plausible under the selected Python baseline.
- Check that C++-specific terms survive only where they are explicitly being
  rejected or translated.
- Check that comments and documentation follow the positive, present-tense
  rule.
- Check that no durable guide links to `work-in-progress/`.

## Deferred Work

- Re-survey Pyright, pyrefly, ty, and mypy before freezing long-lived tool
  comparisons.
- Validate the Ruff baseline against a representative project.
- Decide whether data-engineering guidance deserves a mandatory guide after the
  first pass.
- Add `web-ui.md` only after there is enough web UI testing material to justify
  it.
- Package the agent skill after the guide set stabilizes.
