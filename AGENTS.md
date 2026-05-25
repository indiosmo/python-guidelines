# Agent Instructions

## Mandatory Context

Load `GOAL.md` unconditionally at the start of every task in this repository.
Use it as the governing objective when making tradeoffs. Do not restate the
goal in prompts, plans, or durable docs merely because it is mandatory context.

For planning, consolidation, or guide-writing tasks, also read
`porting-plan.md` before editing. It is the active working plan.

## Source Material

Use these paths as the main inputs:

- `/home/msi/repos/cpp-guidelines/`: upstream C++ guideline source.
- `/home/msi/python_workspace/factors2/`: concrete Python reference codebase.
- `GOAL.md`: stable objective and success criteria.
- `porting-plan.md`: active orchestration plan.
- `MONOREPO.md`: local Python monorepo shape and boundary philosophy.
- `work-in-progress/`: temporary research, exploration, consolidation, and
  planning artifacts.

Treat `work-in-progress/` as ephemeral feature-work space. It is valid source
material for the porting effort, but durable guides, READMEs, and agent context
files should not link to it or depend on it.

## Porting Principles

- Prefer Python vocabulary, Python libraries, and Pythonic idioms.
- Use static typing where it clarifies contracts and catches real mistakes.
- Keep dynamic Python where it is the clearer Pythonic design, especially at
  validated boundaries, in dataframe-heavy code, or around deliberate adapter
  points.
- Separate boundary parsing, domain invariants, and failure translation.
- Prefer examples that are concrete enough for code review, but not tied to
  one codebase unless the guide is explicitly discussing that codebase.

## Error Handling

Error-handling guidance must be idiomatic Python. Do not mimic C++ LEAF,
`std::expected`, or Result-heavy patterns by default.

Prefer existing Python exception types when they communicate the failure
clearly. Introduce custom exception types only when they carry real domain
meaning, help callers distinguish a meaningful failure class, or support useful
boundary translation. Consider Python-native tools such as exception chaining,
`ExceptionGroup`, `contextlib`, warnings, and sentinel values where they fit
the API.

## Documentation Rules

- Write durable docs in present tense.
- Avoid negative documentation: do not describe old behavior, moved
  responsibilities, or things a module does not do.
- Do not transcribe temporary research into durable docs. Extract the stable
  reasoning and cite durable sources or the final guide structure instead.
- Use practitioner vocabulary. Do not abbreviate names merely to shorten them.
- Avoid glyphs and icons in code, comments, docs, and CLI output.

## Workflow

- Use the `$documentation` skill for README, guide, ADR, runbook, and other
  durable documentation work.
- Use the `$elements-of-style:writing-clearly-and-concisely` skill when writing
  or editing prose for humans.
- Before writing a guide, read the relevant source slice from
  `porting-plan.md` and the applicable consolidated exploration artifacts.
- When several viable approaches exist, build a decision matrix with criteria
  such as complexity, maintenance cost, reversibility, blast radius,
  ergonomics, testability, and coupling.
- Use parallel agents for independent read-only exploration or disjoint write
  scopes. Consolidate their outputs before producing durable docs.
- Keep edits scoped. Do not reorganize research artifacts unless the task asks
  for consolidation.
- Do not delete or overwrite work in `work-in-progress/` unless the user asks.
