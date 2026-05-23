# Modern Python tooling and typing research notes

Research date: 2026-05-23.

This is a research slice for later guideline writing. It records current primary-source guidance and candidate heuristics; it is not a full guide draft.

## Sources consulted

- Python 3.14 "What's New": https://docs.python.org/3/whatsnew/3.14.html
- Python 3.14 `typing` module docs: https://docs.python.org/3/library/typing.html
- Python typing specification, type system concepts: https://typing.python.org/en/latest/spec/concepts.html
- Python typing docs, protocols and structural subtyping: https://typing.python.org/en/latest/reference/protocols.html
- PEP 483, theory of type hints: https://peps.python.org/pep-0483/
- PEP 484, type hints: https://peps.python.org/pep-0484/
- PEP 649, deferred evaluation of annotations: https://peps.python.org/pep-0649/
- PEP 695, type parameter syntax: https://peps.python.org/pep-0695/
- PEP 696, type defaults for type parameters: https://peps.python.org/pep-0696/
- PEP 749, implementing PEP 649: https://peps.python.org/pep-0749/
- PEP 723, inline script metadata: https://peps.python.org/pep-0723/
- PEP 735, dependency groups in `pyproject.toml`: https://peps.python.org/pep-0735/
- PyPA dependency groups specification: https://packaging.python.org/en/latest/specifications/dependency-groups/
- uv project guide: https://docs.astral.sh/uv/guides/projects/
- Ruff linter docs: https://docs.astral.sh/ruff/linter/
- Ruff formatter docs: https://docs.astral.sh/ruff/formatter/
- Pyright homepage: https://microsoft.github.io/pyright/
- Pyright configuration docs: https://github.com/microsoft/pyright/blob/main/docs/configuration.md
- Pyrefly FAQ: https://pyrefly.org/en/docs/pyrefly-faq/
- Pyrefly installation and configuration docs: https://pyrefly.org/en/docs/installation/
- Pyrefly v1.0 announcement: https://pyrefly.org/blog/v1.0/

## Python 3.14 notes relevant to style guidance

- Python 3.14 changes annotation evaluation. Function, class, and module annotations use deferred evaluation by default under PEP 649 and PEP 749. Forward references generally no longer need string annotations solely to avoid eager name lookup.
- Runtime annotation consumers should use supported inspection APIs rather than reading `__annotations__` casually. The new `annotationlib` module can evaluate annotations as runtime values, forward references, or strings.
- This reduces the style pressure to write quoted annotations or blanket `from __future__ import annotations` in code that targets Python 3.14 only. For libraries supporting older Python versions, compatibility still governs the annotation style.
- Type parameter syntax from Python 3.12 is now established enough to use in Python 3.14-targeted guidance: `class Repository[T]:`, `def first[T](items: Sequence[T]) -> T:`, and `type JsonScalar = str | int | float | bool | None`.
- Type parameter defaults from Python 3.13 are useful for library APIs where one generic argument has a dominant default. They are not a reason to make local code generic.
- PEP pages for typing features increasingly point to the typing specification as canonical. A style guide should cite PEPs for provenance, but rely on current typing docs/specs for normative behavior.

## Typing principles from official guidance

- Python's type system is gradual. Annotated and unannotated code can coexist, and `Any` is the explicit dynamic escape hatch.
- `Any` is not the same as `object`. `object` says only universally valid operations are allowed; `Any` says the checker cannot know the static type and should allow operations that may be valid.
- Annotations are for static tools and readers. PEP 484 does not make Python a runtime type-enforced language.
- Keep annotations simple. PEP 484 explicitly warns that dynamically computed type expressions are unlikely to be understood by static analysis tools.
- Prefer structural typing where it captures Python's duck typing idiom. Protocols are the static equivalent of "supports these operations", and are often clearer than forcing inheritance.

## Tooling notes

### uv

- uv treats `pyproject.toml` as the project metadata and dependency source, `.python-version` as the default interpreter selector, `.venv` as the project environment, and `uv.lock` as the exact cross-platform lockfile.
- Candidate recommendation: for new projects, use `uv init`, `uv add`, `uv remove`, `uv sync`, and `uv run` as the default workflow. Commit `uv.lock` for applications and tool-managed projects where reproducibility matters.
- Use PEP 735 `[dependency-groups]` for development-only groups such as `test`, `lint`, `typing`, and `docs`. These groups are not published as package metadata and also fit non-package script collections.
- Use PEP 723 inline script metadata for durable single-file scripts with dependencies. This is better than undocumented comments, ad hoc requirements files beside scripts, or assuming global packages.
- Caveat: uv is a project tool, installer, resolver, runner, and lockfile manager. It does not replace the need to understand package metadata boundaries: runtime dependencies belong in `[project.dependencies]`; development tools belong in dependency groups.

### Ruff

- Ruff's linter is designed as a fast replacement for Flake8 plus many common plugins, including import sorting and pyupgrade-style checks.
- Ruff's formatter is designed as a Black replacement, but its docs say it should not be used interchangeably with Black on an ongoing basis because there are intentional differences.
- Candidate recommendation: choose one formatter per repository. If using Ruff format, also use Ruff lint, and run import sorting through `ruff check --select I --fix` before `ruff format`.
- Avoid enabling formatter-conflicting lint rules. The Ruff formatter docs identify some lint rules that can fight formatting, and its default configuration avoids those conflicts.
- Treat Ruff as syntax, import, style, modernization, and simple bug linting. Do not present Ruff as a substitute for a type checker.

### Pyright

- Pyright is a standards-focused static type checker with CLI, language server, and VS Code integration.
- Configuration can live in `pyrightconfig.json` or `[tool.pyright]` in `pyproject.toml`; `pyrightconfig.json` takes precedence if both exist.
- `typeCheckingMode` supports `off`, `basic`, `standard`, and `strict`; the default is `standard`.
- Pyright analyzes unannotated functions by default, which helps incremental adoption.
- Candidate recommendation: use `standard` for initial adoption and move stable application or library code toward `strict` by package or directory. Avoid mandating global strict mode for exploratory, notebook-adjacent, or heavily dynamic code.
- Caveat: do not commit developer-specific virtual environment paths. Pyright docs warn that `venvPath` often differs by developer and is better as a command-line or per-user setting.
- Prefer stubs for untyped libraries where stable interfaces matter. Pyright can inspect library code when stubs are absent, but its docs call that information incomplete.

### Pyrefly

- Pyrefly is Meta's Rust-based static type checker and language server. Its docs describe both terminal/CI type checking and IDE language-server use.
- Pyrefly reached version 1.0 on 2026-05-12. The announcement says Meta considers it production-ready and cites use in large production codebases.
- `pyrefly init` can add `[tool.pyrefly]` configuration or create `pyrefly.toml`, and attempts to migrate existing type checker configuration.
- `pyrefly check --summarize-errors` gives project diagnostics and error summaries. `pyrefly suppress` can add ignores for large adoption or upgrade waves.
- Candidate recommendation: treat Pyrefly as a credible modern option, especially for large repositories and teams that value fast whole-project feedback. In conservative guidelines, frame it as an alternative to Pyright rather than a universal replacement.
- Caveat: Pyrefly has its own ignore comments and configuration surface. Switching type checkers is a policy decision because diagnostics and inference behavior differ.

## Typing escape heuristics

Static typing helps most when it protects stable contracts and high-fanout code. Dynamic Python remains idiomatic when the code is exploratory, highly reflective, or built around runtime-discovered shapes that the type system cannot express clearly.

### Boundaries

- Type public package APIs, service boundaries, CLI argument models after parsing, database rows after validation, message schemas, and long-lived cross-module functions.
- Use `TypedDict`, dataclasses, Pydantic models, attrs classes, or explicit domain classes at boundaries where dictionaries otherwise leak across the codebase.
- Accept `Any` or `object` at hostile boundaries only briefly. Normalize to typed domain values close to the edge.
- For third-party untyped input, isolate the dynamic call in a small adapter. Add a runtime check if the downstream code relies on shape or invariants the type checker cannot see.

### Data models

- Type domain entities and command/result objects. These names carry meaning and are often read more than they are written.
- Prefer precise field types over giant nested aliases when the alias becomes harder to understand than the data model.
- Avoid generic data models until there are multiple real instantiations. A single-use `T` usually adds ceremony without buying safety.
- Use `Protocol` for capabilities such as "has a `read` method" or "can serialize", especially when callers should accept several unrelated implementations.

### Dynamic plugins and reflection

- Dynamic plugin registries, import hooks, decorators that rewrite call signatures, and runtime attribute injection are natural pressure points against static typing.
- Put static types around the registry contract: plugin input, plugin output, lifecycle hooks, and error behavior.
- Let the discovery mechanism remain dynamic when typing it would require inaccurate casts or duplicated metadata.
- If a decorator preserves a function signature, use `ParamSpec` or `Concatenate`. If it changes a signature in ways the checker cannot express, document the runtime contract at the boundary and keep the dynamic area small.

### pandas, numpy, and dataframe-heavy code

- Static typing is usually useful for file paths, configuration, domain identifiers, function boundaries, and scalar results around dataframe pipelines.
- Full dataframe schemas are often better enforced with runtime validation, tests, or domain-specific schema tools than with complex Python type annotations.
- Avoid contorting code into elaborate type aliases for column-level transformations when the checker cannot verify the actual dataframe shape.
- Prefer small typed functions at pipeline edges and clear runtime assertions at shape-changing steps.

### Tests

- Type test helpers, fixtures with non-obvious shape, builders, fake implementations, and assertion utilities reused across many tests.
- Do not require exhaustive annotations inside short test bodies where literals and assertions make intent obvious.
- Avoid weakening production APIs just to satisfy test mocks. Instead, type the fake with the same protocol or interface the production code consumes.
- Use checker ignores in tests more readily than production code when the test deliberately exercises invalid input.

### Prototypes and notebooks

- For prototypes, type the parts likely to survive: parsed inputs, external API responses after validation, and functions copied into production modules.
- Leave throwaway exploration dynamic. Requiring complete typing too early can freeze bad abstractions.
- Convert useful prototype code by first extracting named functions and data models, then adding types.

### Performance-sensitive code

- Type annotations normally do not optimize CPython execution. They help readers and tools, not runtime speed.
- In hot paths, avoid runtime annotation introspection, validation decorators, or `typing.get_type_hints()` calls unless measured and justified.
- Python 3.14's deferred annotations reduce definition-time costs, but runtime consumers can still pay evaluation costs.
- Prefer simple, concrete annotations in hot modules. Complex generic machinery can slow tools and distract from measured performance work.

### Third-party untyped interfaces

- Prefer upstream `py.typed` packages or maintained stubs when available.
- For important untyped libraries, write narrow local stubs or typed adapters for the calls the project actually uses.
- Use `cast` only after a nearby runtime check, documented invariant, or trusted library contract. Repeated casts around the same library signal the need for an adapter.
- Use `Any` deliberately at the edge, not as a silent flow through core code.

## Candidate guide topics

- "Use types to name contracts, not to simulate another language": gradual typing, `Any` discipline, and when dynamic code is still Pythonic.
- "Project tooling baseline": `pyproject.toml`, `uv.lock`, `.python-version`, dependency groups, `uv run`, `uv sync`.
- "Ruff owns lint and format": rule selection, formatter compatibility, import sorting, preview rules, and CI commands.
- "Choose one type checker policy": Pyright as conservative default, Pyrefly as high-performance alternative, with adoption levels by directory.
- "Typing boundaries": public APIs, adapters, data models, protocols, and untyped third-party packages.
- "Modern annotation syntax": built-in generics, `|` unions, `Self`, `LiteralString` where security-sensitive, `type` aliases, PEP 695 generics, PEP 696 defaults only when they simplify a real API.
- "Pragmatic escape hatches": `Any`, `object`, `cast`, ignore comments, stubs, and runtime validation.
- "Data science exception paths": dataframe-heavy code, notebook migration, shape validation, and scalar/domain typing around pipelines.
- "Tests and typing": fixture/helper typing, invalid-input tests, and mock/fake protocols.

## Recommended constraints for the later guide

- Do not mandate full strict typing across every Python file.
- Do require types at stable module boundaries and in reusable helpers.
- Do not allow `Any` to spread silently from adapters into core domain code.
- Prefer `Protocol` over inheritance when the code depends on behavior rather than class identity.
- Prefer modern syntax for Python 3.12 and newer code: `list[str]`, `str | None`, `type Alias = ...`, and PEP 695 type parameters.
- For Python 3.14-only projects, avoid quoted annotations unless the quote is part of a compatibility or runtime-inspection decision.
- Pick one formatter and one primary type checker per repository.
- Keep developer-specific environment paths out of committed configuration.
- Keep tool configuration small at first. Add stricter rules because they prevent real defects or preserve maintainability, not because the tool exposes them.
- Prefer local adapters, stubs, or runtime validation over widespread ignores for untyped third-party libraries.
