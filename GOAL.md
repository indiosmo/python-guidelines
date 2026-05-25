# Goal

This repository ports the generalizable engineering guidance from the C++
guidelines into idiomatic Python guidelines.

The port preserves the parts that transfer across languages: domain ownership,
forward-only dependency graphs, declarative dataflow, invariant ownership,
idempotency, deterministic testing, root-cause debugging, and tool-backed
feedback. It rewrites the parts that are C++-specific into idiomatic modern
Python: type annotations and runtime validation instead of compile-time
mechanics, pytest instead of Catch2, context managers instead of RAII,
idiomatic Python exception handling and boundary translation instead of C++
result machinery, and Python static analysis plus runtime diagnostics instead
of sanitizers.

The intended output is a set of Python guidelines that a practitioner can use
directly when designing, testing, debugging, reviewing, and maintaining Python
systems. The guidance should be Pythonic: when a C++ pattern and idiomatic
Python practice diverge, prefer the Pythonic way. Do not produce a direct
mapping of C++ concepts into Python syntax.

## Scope

The first pass produces durable guidance for:

- Python project layout, tooling, and version policy.
- Design principles for architecture, typing, data modeling, error handling,
  invariants, concurrency, pipelines, performance, comments, and extension
  points.
- Testing principles expressed through pytest, fixtures, parametrization,
  approval tests, failure-path tests, and condition-based waits.
- Debugging principles expressed through Python tracebacks, logging,
  reproducible diagnostics, static analysis, warnings, and profilers.
- Agent context and examples that make the guidelines usable by coding agents.

The work uses the existing C++ guidelines as the conceptual source, the
factors2 codebase as a concrete Python reference, and current Python tooling
research as a calibration point.

## Porting Rules

- Preserve the engineering intent, not the C++ mechanism.
- Prefer the Pythonic approach when it conflicts with a direct C++ mapping.
- Prefer Python vocabulary and Python libraries where the ecosystem has a
  clear idiom.
- Use static typing where it clarifies contracts and catches real mistakes.
- Keep dynamic Python where static typing would add ceremony without useful
  proof, especially behind validated boundaries or in dataframe-heavy code.
- Treat error handling as Python guidance first. Prefer existing exception
  types and Python-native control flow where they communicate the failure
  clearly; introduce custom exception types only when they carry real domain
  meaning or support useful boundary translation.
- Put boundary parsing, domain invariants, and failure translation in separate
  concepts.
- Make examples concrete enough to guide code review, but not so specific that
  the guidelines become a transcription of one codebase.
- Avoid negative documentation and past-tense refactor notes in the durable
  guides.

## Success Criteria

The port is successful when the repository has a coherent Python guideline set
that:

- Covers the transferable C++ principles.
- Names the Python-specific replacements for non-transferable C++ mechanisms.
- Gives writing agents a stable plan and source map.
- Gives reviewers and contributors concise, citable guidance.
- Avoids duplicating temporary research artifacts in durable documentation.
