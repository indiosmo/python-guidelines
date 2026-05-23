# Conformance tests with approval files

Overall disposition: PORTS_AS_IS.

The approval-test principle ports directly. Python can use `approvaltests`,
snapshot plugins such as syrupy, or plain file comparisons. The core contract
is deterministic render output plus human review of diffs.

## When ApprovalTests fits

Disposition: PORTS_AS_IS.

Keep all cases: large structured output, evolving shape, human oracle, many
parallel scenarios, documentation by example, and legacy characterization.

Python examples:

- Rendered JSON, YAML, TOML, or HTML.
- CLI output.
- generated Python or SQL.
- report text.
- normalized log output.

## When to reach for something else

Disposition: PORTS_AS_IS.

Keep all warnings. Direct assertions are better for scalars, narrow behavior,
error codes, and state transitions.

## Writing the test

Disposition: PORTS_WITH_ADAPTATION.

Render with pytest. The exact package can vary; the guide should show the
concept without locking the repository to one library.

```python
def test_order_codec_encode_new_order(approvals: Approvals) -> None:
    message = OrderCodec().encode(make_new_order())
    approvals.verify(scrub(message))
```

Plain-file alternative:

```python
def test_report_matches_approved(approved_file: Path) -> None:
    actual = render_report(make_report_input())
    assert actual == approved_file.read_text()
```

## Determinism is the contract

Disposition: PORTS_AS_IS.

Keep it. Normalize timestamps, UUIDs, absolute paths, dict ordering, random
seeds, locale, time zone, and platform line endings before comparing.

```python
def scrub(output: str) -> str:
    output = UUID_PATTERN.sub("<uuid>", output)
    return output.replace(os.sep, "/")
```

## Aggregating scenarios into one approval

Disposition: PORTS_AS_IS.

Keep it. Python rendering:

```python
def test_order_codec_roundtrip(approvals: Approvals) -> None:
    blocks = []
    for label, order in order_cases():
        blocks.append(f"# {label}")
        blocks.append(scrub(OrderCodec().encode(order)))
        blocks.append("")

    approvals.verify("\n".join(blocks))
```

## Reviewing approval diffs

Disposition: PORTS_AS_IS.

Keep it. The approved artifact is part of the review. If nobody can read the
diff, split the approval or replace it with focused assertions.
