# Approval Tests

Approval tests compare rendered output with an approved file checked into the
repository. The approved file is the expected value. It works best for
structured output that humans can review more easily as a diff than as a long
inline assertion.

Use `approvaltests` with `pytest-approvaltests` unless the project already has
a stronger snapshot convention.

## Fit

Approval tests fit:

- Generated SQL, configuration, code, reports, and CLI output.
- JSON, YAML, XML, CSV, markdown, and other structured text.
- Pretty-printer and serializer output.
- Log formatting when the format itself is the contract.
- Legacy characterization before a refactor.
- Scenario catalogs where one file with labeled sections is easier to review
  than many inline literals.

Use direct assertions for scalar values, short strings, and narrow behavior.
Approval tests should carry output shape, not hide a one-line fact behind a
file diff.

## Pytest Shape

Configure a deterministic reporter in pytest configuration when approval
tests are part of the project command surface.

```toml
[tool.pytest.ini_options]
addopts = "--approvaltests-use-reporter=PythonNative"
```

A typical test verifies the large artifact and asserts narrow facts directly:

```python
from approvaltests import verify


def test_universe_query() -> None:
    query = make_universe_query(
        filters=[SectorFilter(["Technology", "Healthcare"])],
        start_date=date(2026, 1, 1),
        end_date=date(2026, 4, 1),
        benchmark_rate="CDI",
        benchmark_index="IBOV",
    )

    assert query.params == ["CDI", "IBOV", "Technology", "Healthcare"]
    verify(query.sql)
```

The direct assertion protects small security-sensitive or behavior-critical
facts. The approval file carries the multi-line SQL shape.

## Deterministic Rendering

Approval comparisons are byte comparisons. Stabilize output before calling
`verify`.

Normalize:

- Ordering of dictionaries, sets, rows, and grouped sections.
- Wall-clock times, UUIDs, generated IDs, random values, and absolute paths.
- Platform path separators and line endings.
- Locale-sensitive number, date, and currency formatting.
- Floating-point precision when exact binary output is irrelevant to the
  contract.

```python
def render_approved_json(payload: Mapping[str, object]) -> str:
    return json.dumps(payload, indent=2, sort_keys=True) + "\n"
```

For dataframe output, sort rows and columns according to the domain contract
before rendering. Use direct dataframe assertions for dtypes, tolerances, and
index rules when those are the main behavior.

## Scrubbers

Use scrubbers for nondeterministic fields that are present and meaningful but
whose exact value varies by run.

```python
UUID_PATTERN = re.compile(r"\b[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}\b")
TIMESTAMP_PATTERN = re.compile(r"\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?Z")


def scrub_report(report: str) -> str:
    report = UUID_PATTERN.sub("<uuid>", report)
    return TIMESTAMP_PATTERN.sub("<timestamp>", report)
```

Keep shared scrubbers near the tests that use the same artifact type. A
stable placeholder still shows that the field exists.

## Scenario Aggregation

Aggregate related scenarios when one approved file reads like a catalog.
Label each section with stable headings.

```python
def test_order_event_rendering_catalog() -> None:
    blocks = [
        render_block("new order", render_event(make_new_order_event())),
        render_block("cancel order", render_event(make_cancel_order_event())),
        render_block("reject order", render_event(make_reject_order_event())),
    ]

    verify("\n".join(blocks))
```

Use separate approval files when scenarios are unrelated, one scenario is much
larger than the others, or reviewers need to inspect one artifact in
isolation.

## Review Discipline

An approval update is a behavior change. The author and reviewer should read
the received diff and confirm every changed line is intended.

Keep approved files small enough to review. If the diff is too large to
understand, split the artifact, scrub noise, or replace the broad approval
with focused assertions.

Pair approval changes with a short explanation in the pull request: what
changed in the output and which source change caused it. This makes the
approved file useful as a review artifact instead of a generated blob.

## File Placement

Commit approved files with the tests that own them. Ignore received files and
other temporary diff artifacts through the project's normal ignore policy.

Use stable names derived from the test and scenario. Avoid names that include
timestamps, random values, local paths, or developer-specific settings.

## Combining With Direct Assertions

Approval tests and direct assertions complement each other.

Use direct assertions for:

- Query parameters and bind variables.
- Record counts and boundary conditions.
- Error codes, status codes, and warnings.
- Security-sensitive escaping and redaction.
- Numeric tolerances and dtype contracts.

Use approval tests for the large rendered artifact once those narrow facts are
already explicit.
