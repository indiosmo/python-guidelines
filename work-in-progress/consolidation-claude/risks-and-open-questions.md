# Risks and open questions for the owner

Each question gets one paragraph and a recommendation. The orchestrator
should treat the recommendations as defaults and override only where the
owner has a specific reason.

---

## Q1. Python version baseline: 3.12, 3.13, or 3.14

**Question.** Does the guide target Python 3.12+ (PEP 695 syntax
available), 3.13+ (TaskGroup stable, free-threaded interpreter
introduced as an experimental build), or 3.14+ (PEP 649 deferred
annotation evaluation by default)?

**Recommendation.** Target 3.13 as the baseline, with 3.14 as the
forward target. Reasons: factors2 already requires 3.13 (`requires-python
= ">=3.13"` in `pyproject.toml`); TaskGroup is stable; PEP 695 syntax
is universally available; PEP 649 lands in 3.14 and changes
annotation-evaluation semantics enough that a guide that targets 3.13
need not contort itself for backward compatibility. Document one
sidebar in `types-and-correctness.md` about 3.14's deferred
annotations and the implications for `__annotations__` consumers; this
is enough.

---

## Q2. Primary type checker: pyright, pyrefly, or mypy

**Question.** Which type checker is the recommended default? See
`divergences-and-decisions.md` D8 for the divergence.

**Recommendation.** pyright. Strongest IDE integration (Pylance);
leader on typing-spec conformance; mature. Document pyrefly as the
swap-in for large monorepos that hit pyright's cold-start cost. Skip ty
for now; its conformance is too low. Mention mypy only as the legacy
fallback for projects that already have it.

---

## Q3. Strictness adoption pattern

**Question.** Global strict, directory-scoped strict, or staged
adoption? See `divergences-and-decisions.md` D9.

**Recommendation.** Global strict is the target state. For codebases
adopting the guidelines, recommend the ratchet pattern documented in
D9: start at `standard`, enable strict on a directory at a time, final
state is global strict with explicit per-file `# pyright: standard`
overrides where unavoidable.

---

## Q4. Dataframe API: pandas, polars, ibis as default in examples

**Question.** What DataFrame API do the guide's examples use?

**Recommendation.** pandas as the default in examples (largest community,
most code in the wild, matches factors2). Mention polars and ibis as the
alternatives in `data-pipeline-and-dataframes.md`. polars is the right
choice for cold-start performance and SQL-style chaining; ibis is the
right choice when the same code needs to run on duckdb / postgres /
bigquery. Where the guide cites a specific dataframe API, cite pandas
plus a one-liner on the polars equivalent.

---

## Q5. Pydantic version: v2 only, or document v1 migration

**Question.** Does the guide target pydantic v2 only, or also document
v1 migration?

**Recommendation.** v2 only. pydantic v1 reached EOL on 2024-06-30 (per
the upstream announcement); the migration is well-documented; new
projects have no reason to use v1. The guide cites v2 APIs throughout
(`Field`, `model_validator`, `Annotated[..., Field(discriminator="...")]`).
Recommend the project pin `pydantic>=2`. If a target codebase still has
v1 code, the migration is its own task, not a guideline concern.

---

## Q6. Alternative data validators: attrs, msgspec, attrs-cattrs

**Question.** Document pydantic alone, or also document attrs,
msgspec, cattrs as alternatives?

**Recommendation.** Document attrs in `types-and-correctness.md` as
"when pydantic is overkill": attrs is faster, lighter, and gives field
validators without pydantic's validation engine. msgspec is the right
choice when the deserialisation hot path is the bottleneck. Treat both
as documented alternatives, not as defaults; the default is pydantic at
the trust boundary, frozen dataclass in-domain.

---

## Q7. Logging stack: loguru, structlog, or stdlib

**Question.** Which logger does the guide recommend by default? See
`divergences-and-decisions.md` D13.

**Recommendation.** loguru for applications; stdlib `logging` for
libraries; structlog as the alternative when strict key/value semantics
are required. Avoid the factors2-style failure mode of mixing two
loggers in the same codebase (cite
`factors-claude/anti-patterns.md` entry L).

---

## Q8. Concurrency baseline: asyncio.TaskGroup or anyio default

**Question.** Is anyio in baseline guidance or only in async-heavy
projects? See `divergences-and-decisions.md` D3.

**Recommendation.** asyncio.TaskGroup as the baseline; anyio as the
documented swap-in when cancel scopes become load-bearing. Document the
specific triggers in `runtime-and-concurrency.md`.

---

## Q9. Import-boundary enforcement tool

**Question.** import-linter, pydeps, both, or principles only?

**Recommendation.** Both, but for different jobs:

- `import-linter` for declarative contracts in CI ("no module under
  `pipeline/` may import from `database/`").
- `pydeps` for visualisation: commit an SVG of the package graph in
  `doc/architecture.svg`; refresh in CI.
- ruff `TID252` (no parent-relative imports) as the universal cheap
  check.

This is more tooling than Codex recommends but the cost is small and the
benefit is high once the codebase has more than ~20 modules.

---

## Q10. Result library: name one or stay library-neutral

**Question.** Does the error-handling guide name a specific Result
library, or stay library-neutral?

**Recommendation.** Library-neutral with one example, citing the
`returns` library or `result` library by name in the example. The
pattern (Ok / Err with `match` dispatch) is what matters; the specific
library is a detail. A team adopting Result for the boundary form can
pick either; the guideline structure is unchanged.

---

## Q11. Approval-test plugin

**Question.** approvaltests, syrupy, snapshottest, or
library-neutral? See `divergences-and-decisions.md` D6.

**Recommendation.** approvaltests + pytest-approvaltests with
`PythonNativeReporter`. factors2 already uses this; it works; the
alternatives (syrupy especially) have nicer ergonomics for JSON-heavy
snapshots but the gain does not justify a switch.

---

## Q12. Cross-cutting service shape

**Question.** contextvars singleton, explicit composition, or both? See
`divergences-and-decisions.md` D2.

**Recommendation.** Both, with the decision matrix in D2. contextvars
singleton for logger/clock/timer/settings/tracing; explicit constructor
parameters for domain-specific dependencies; ContextVar alone (no
singleton facade) for request-scoped context.

---

## Q13. Settings library: pydantic-settings or hand-rolled

**Question.** Standardise on pydantic-settings for environment-driven
config, or describe principles only?

**Recommendation.** Standardise on pydantic-settings.BaseSettings. The
typed-field + env_prefix + nested model pattern covers the common case
in 10 lines; the alternative (hand-rolled argparse + getenv) is the
documented anti-pattern in `factors-claude/anti-patterns.md` entry J.

---

## Q14. Monorepo layout depth

**Question.** Does the guide describe the monorepo layout from
MONOREPO.md as canonical, or describe principles and let projects pick?

**Recommendation.** Canonical for the patterns that MONOREPO.md
already documents (the `src/libs/`, `src/apps/`, `src/dagster/`,
`src/dbt/` split; centralised docker build context; child
`pyproject.toml` per workspace member; one root `uv.lock`). Principles
for the rest. Cross-link MONOREPO.md as the durable local convention
from `python-projects-and-tooling/project-layout.md`.

---

## Q15. Free-threaded interpreter: in scope or out

**Question.** Does the guide cover the 3.13t / 3.14t free-threaded
interpreter, or treat it as out of scope?

**Recommendation.** In scope, but as a sidebar per relevant guide rather
than a dedicated chapter. The free-threaded interpreter changes
atomicity assumptions in `invariants-and-rollback.md` (`dict.setdefault`
is no longer atomic) and removes the GIL-derived single-thread default
in `runtime-and-concurrency.md`. A sidebar in each of those guides is
enough for now; revisit when the free-threaded build is the default.

---

## Q16. Skill packaging in agent/

**Question.** Is the
`agent/skills/repo-specific-python-guidelines/` skill in scope for the
first writing pass?

**Recommendation.** Out of scope for the first pass. The agent-context
file plus the topical guides are enough to validate the structure; the
skill is mechanical packaging that can wait. Document the intent in
`agent-context.md` so a follow-up pass has the spec.

---

## Q17. Pre-commit hook policy: lint and format only, or also type-check

**Question.** Does the guide recommend running mypy/pyright in
pre-commit, or only in CI?

**Recommendation.** Lint and format in pre-commit (cheap); type-check in
CI only (slow). Cite `factors-claude/tooling-baseline.md`: factors2
runs ruff in pre-commit and mypy/pyright in CI; the split works.
Recommend `default_install_hook_types` includes `pre-commit`,
`post-checkout`, `post-merge`, `post-rewrite` so `uv sync` runs on every
branch switch (factors2's pattern).

---

## Q18. CI coverage gate

**Question.** Should the guide require a coverage threshold in CI?

**Recommendation.** Yes, with a documented threshold (suggest 80% as
the starting point), reducible only via PR. Cite
`factors-claude/tooling-baseline.md`: factors2 has `pytest-cov`
installed but no gate. A coverage gate is the cheapest way to catch
"PR adds 200 lines and 0 tests"; the threshold value is less important
than the gate existing.

---

## Q19. Documentation tooling: mkdocs, sphinx, or no choice

**Question.** Does the guide recommend a docs renderer?

**Recommendation.** mkdocs-material as the default for new projects;
sphinx for projects with extensive autodoc needs or that already use
sphinx. Keep this as a sidebar in `comments-and-docstrings.md`; the
docstring style (Google) is the load-bearing decision, the renderer is
ergonomic.

---

## Q20. CODEOWNERS and PR templates

**Question.** Does the guide cover CODEOWNERS, PR templates, issue
templates?

**Recommendation.** Yes, as a short section in
`python-projects-and-tooling/tooling-baseline.md`. These are
lightweight; the cost of including them is small; the benefit is that
new projects start with the right scaffolding. Cite
`factors-claude/tooling-baseline.md`'s observation that factors2
lacks them.

---

## Q21. Security tooling: pip-audit, safety, gitleaks

**Question.** Does the guide recommend a security-scan tool in CI?

**Recommendation.** Yes, in `python-projects-and-tooling/tooling-baseline.md`:
pip-audit (or safety) for dependency CVEs; gitleaks (or trufflehog) for
secrets in git history. Run in CI; warn-only in PRs initially, gate
after the team has cleaned up the baseline.

---

## Q22. PR review checklist

**Question.** Does the guide ship a PR review checklist that mirrors the
guides?

**Recommendation.** Defer. A checklist is downstream of the guides;
write it once the guides exist and have shipped to a real codebase.
Cross-link to the relevant guide section per checklist item so a
reviewer can read the rule, not just the bullet.
