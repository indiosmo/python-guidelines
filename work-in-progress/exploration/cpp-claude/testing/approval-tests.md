# Analysis: cpp-testing-principles/approval-tests.md

Disposition: **PORTS_AS_IS**. ApprovalTests has an actively-maintained
Python port (`approvaltests` on PyPI, used in factors2), so the chapter
ports almost word-for-word.

## Section-by-section

### When ApprovalTests fits / when not

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Identical guidance. The "many parallel
  scenarios" use case is the most common in factors2-style codebases:
  ETL output, snapshot of a normalized DataFrame, a rendered HTML
  report. The "single scalar" and "narrow assertion" cases stay better
  served by a plain `assert`.

### Writing the test

- **Classification:** PORTS_WITH_ADAPTATION (spelling only).
- **C++ snippet:**
  ```cpp
  ApprovalTests::Approvals::verify(message);
  ```
- **Python equivalent:**
  ```python
  from approvaltests import verify, Options
  from approvaltests.reporters import PythonNativeReporter

  def test_order_codec_encode_new_order():
      codec = OrderCodec()
      message = codec.encode(make_new_order())
      verify(message, options=Options().with_reporter(PythonNativeReporter()))
  ```

  factors2 already wires `--approvaltests-use-reporter=PythonNative` in
  `pyproject.toml`; document this as the recommended default for new
  projects.

### Determinism is the contract

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. Scrubbing tools:

  ```python
  from approvaltests.scrubbers import (
      create_regex_scrubber, scrub_all_dates, scrub_all_guids,
  )

  scrubber = combine_scrubbers(
      scrub_all_dates,
      scrub_all_guids,
      create_regex_scrubber(r"order_id=[A-Z0-9]+", "order_id=<scrubbed>"),
  )

  verify(message, options=Options().with_scrubber(scrubber))
  ```

  For pandas DataFrames: serialize with a stable column order, sort
  rows on a domain key, format floats with a fixed precision before
  passing the string to `verify`.

### Aggregating scenarios into one approval

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same shape; the `blocks: list[str]` and final
  `"\n".join(blocks)` translate directly. For tabular data, prefer
  `verify_as_json(...)` over a hand-built string; the JSON formatter is
  stable across runs.

### Reviewing approval diffs

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rules. factors2 uses `pytest-approvaltests`,
  which surfaces received files in the standard pytest report and
  integrates with the chosen reporter; recommend that integration.

## Suggested Python file: `python-testing-principles/approval-tests.md`

Verbatim port with the `ApprovalTests::Approvals::verify` ->
`approvaltests.verify` substitution; add a short paragraph on pandas /
polars DataFrame stabilisation (sort, fixed precision, JSON
serialisation).
