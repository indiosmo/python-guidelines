# Modern Python tooling notes

Sources checked on 2026-05-23:

- Python 3.13 `typing` docs: https://docs.python.org/3.13/library/typing.html
- Python 3.14 `asyncio` task docs: https://docs.python.org/3.14/library/asyncio-task.html
- Python typing specification: https://typing.python.org/
- Pyrefly FAQ: https://pyrefly.org/en/docs/pyrefly-faq/
- Ruff linter docs: https://docs.astral.sh/ruff/linter/
- Ruff formatter docs: https://docs.astral.sh/ruff/formatter/
- uv package-management docs: https://docs.astral.sh/uv/pip/packages/
- pytest docs: https://docs.pytest.org/en/stable/

## Findings

Python 3.13 and 3.14 are a reasonable baseline for a modern guide. PEP 695
type parameter syntax is in normal use: generic functions and classes can
declare type parameters inline, and `type X = ...` is the preferred spelling
for aliases in new code. Python 3.13 adds defaults for type variables, type
variable tuples, and parameter specs. The guide should teach `Protocol`,
`TypeVar`, `ParamSpec`, `TypeVarTuple`, `Literal`, `NewType`, `Annotated`,
and `typing.assert_never`, but it should treat them as tools for clear
contracts rather than as a mandate to model every runtime fact in types.

`asyncio.TaskGroup` is the structured-concurrency primitive to prefer over
loose `create_task` ownership. The Python 3.14 docs specify cancellation of
sibling tasks on failure and aggregation of non-cancellation failures into
exception groups. A runtime guide should therefore talk about task ownership,
cancellation, and `except*`, not only about thread queues.

Pyright remains a strong conservative default for a mature type-checking
guide. Pyrefly is now relevant enough to mention as an alternative type
checker and language server; its own FAQ positions it as both a CLI checker
and LSP. The Python guide should avoid checker-specific tricks unless they
serve a project-wide policy.

Ruff should be the default linter and formatter. The linter docs position
`ruff check` as the primary lint command and describe Ruff as replacing
Flake8 plus many common plugins. The formatter docs include docstring and
Markdown code-block formatting, which matters for a guidelines repository
whose examples should be machine-formatted.

uv should be the default package manager and command runner in examples. The
docs cover installing into virtual environments, reading project metadata
from `pyproject.toml`, and using dependency groups.

pytest should replace Catch2 as the testing vocabulary. Use fixtures,
`pytest.mark.parametrize`, markers, `pytest.raises`, `monkeypatch`, `tmp_path`,
and `pytest.approx`. Approval tests can be rendered through `approvaltests`
or syrupy-style snapshot tests, but the guide should emphasize deterministic
rendering and reviewable diffs over a specific package.
