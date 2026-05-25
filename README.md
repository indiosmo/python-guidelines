# Python Guidelines

Reusable guidelines for modern Python codebases. The guides help teams shape
projects, design components, test behavior, and investigate failures without
burying the domain model under incidental framework or tooling mechanics.

The guidance is generic by design. A consuming repository should add a local
mapping layer that names its packages, commands, helpers, test fixtures,
runtime boundaries, and domain examples.

## Reading Order

The guides are siblings, not a strict sequence. Start where your task starts:
project setup, component design, testing, or debugging. When reading broadly,
start with the project and design guides because the testing and debugging
guides reuse that vocabulary.

- [python-projects-and-tooling/](python-projects-and-tooling/) covers
  repository layout, uv workflow, Ruff, Pyright, version policy, and typing
  stack.
- [python-design-principles/](python-design-principles/) covers architecture,
  types, error handling, concurrency, pipelines, performance, and dataframes.
- [python-testing-principles/](python-testing-principles/) covers pytest
  conventions, test intent, helpers, error-path coverage, waits, and approval
  tests.
- [python-debugging-principles/](python-debugging-principles/) covers
  root-cause tracing, defense in depth, logging, static analysis, and runtime
  diagnostics.

## Agent Context

The [agent/](agent/) folder contains an always-loaded Python agent context and
an example companion:

- [python-agent-context.md](agent/python-agent-context.md) gives coding agents
  the concise rules they should apply during edits.
- [python-agent-examples.md](agent/python-agent-examples.md) gives good and
  bad Python pairs keyed to those rules.

Humans should read the topical guides first. Agents should load the context
unconditionally and load examples when shaping or reviewing concrete code.

## Themes

These ideas recur across the guide set:

- Domain packages own vocabulary, invariants, errors, and public contracts.
- Adapters translate external shapes into domain values at boundaries.
- Runtime shells own files, sockets, settings, event loops, workers, schedulers,
  logging configuration, and framework entry points.
- Static typing names stable contracts; runtime validation protects hostile or
  shape-rich boundaries.
- Python code is exception-first. Boundaries translate failures into HTTP
  responses, CLI exit codes, job states, queue behavior, or UI messages.
- Tests assert behavior through independent oracles and deterministic data.
- Debugging starts with the exact symptom and traces bad values back to their
  origin before changing production code.

## Conventions

Examples target Python 3.14 for new projects. Existing projects that support
Python 3.13 should keep that support visible in package metadata, CI, and
examples.

Code fences use `python`, `toml`, `yaml`, `sh`, or `text` as appropriate.
Examples use practitioner vocabulary and explicit names rather than generic
helpers.
