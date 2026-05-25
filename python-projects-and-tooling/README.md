# Python Projects And Tooling

Python project structure should make runtime boundaries, dependency ownership,
and developer commands visible. The project root owns workspace policy and
command surfaces. Packages own their runtime dependencies and public APIs.
Tooling gives fast feedback without becoming a second implementation.

## Navigation

- [project-layout.md](project-layout.md) covers `src/` layout, uv workspaces,
  child `pyproject.toml` files, package layering, tests, generated files, and
  operational files.
- [tooling-baseline.md](tooling-baseline.md) covers uv workflow, Ruff, Pyright,
  optional mypy, pytest, pre-commit, CI, security checks, and review
  scaffolding.
- [python-version-and-typing-stack.md](python-version-and-typing-stack.md)
  covers Python 3.14 baseline, Python 3.13 migration, PEP 695, deferred
  annotations, type-checker policy, gradual typing, and dataframe-heavy code.

Start with `project-layout.md` when creating or reshaping a repository. Use
`tooling-baseline.md` when adding commands or CI jobs. Use
`python-version-and-typing-stack.md` when choosing interpreter support or
typing policy.
