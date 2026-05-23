# Analysis: cpp-design-principles/declarative-style.md

Disposition: **PORTS_AS_IS** for almost every section; the C++ "ranges /
views / `ranges::to`" subsection becomes the Python "comprehensions /
generators / itertools / pandas pipe" subsection.

## Section-by-section

### Decompose

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Cut a function until each piece does one nameable
  thing.
- **Python rendering:** Same advice. The `mean` / `variance` /
  `stats.compute` example renders cleanly:

  ```python
  import statistics
  from dataclasses import dataclass
  from collections.abc import Sequence

  @dataclass(frozen=True, slots=True)
  class Stats:
      mean: float
      variance: float

  def mean(xs: Sequence[float]) -> float:
      return sum(xs) / len(xs)

  def variance(xs: Sequence[float], mu: float) -> float:
      return sum((x - mu) ** 2 for x in xs) / len(xs)

  def compute(xs: Sequence[float]) -> Stats:
      mu = mean(xs)
      return Stats(mean=mu, variance=variance(xs, mu))
  ```

  The "widget render = paint + flush" example renders verbatim (Python's
  `Protocol` for the `Canvas` interface; `Optional` for `text`). The
  "Sean Parent slide" example becomes a function over `list.insert` /
  `list.pop` slices.

### Work on simpler types

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Pass a helper only what it actually uses.
- **Python rendering:** Same rule. The `ack_ratio(sent, acked)` example
  becomes a plain two-arg function. The "find min by a field, not by
  points" example becomes:

  ```python
  # bad - lambda over the whole point
  p = min(points, key=lambda point: point.y)

  # good - operator.attrgetter names the projection
  from operator import attrgetter
  p = min(points, key=attrgetter("y"))
  ```

### Stage variables upfront

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Compute derived data first; assemble after.
- **Python rendering:** Identical. The pagination example renders almost
  unchanged; the dataclass keyword-argument syntax already reads as
  the designated-initializer it would in C++.

### Named predicates

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Lift inline conditions into named predicates.
- **Python rendering:** Identical. Python's idiomatic spelling is a
  module-level function (`def is_active_user(u: User, now: datetime) ->
  bool`) or a `functools.partial` for the factory case. Avoid `lambda`
  for anything past `lambda x: x.field` -- ruff's `E731` already flags
  named-lambda assignments.

### Use ranges and algorithms

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Prefer `std::ranges::any_of`, `find_if`, `count_if`
  over hand-rolled flag-and-break loops.
- **Python rendering:** Use `any()`, `all()`, `next(... default=None)`,
  `sum()`, `min(..., key=...)`. The "has expired order" example becomes:

  ```python
  if any(order.expiry < now for order in orders):
      return Reject.EXPIRED_ORDER_PRESENT
  ```

  The "first missing required field" example becomes
  `next((f for f in fields if f.required and f.value is None), None)`.
- **Open questions:** None.

### Compose lazily; materialize once

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** A pipeline of M operations applied eagerly traverses
  the data M times and allocates M-1 intermediate containers; views fuse
  them.
- **Python rendering:** Generators (`x for x in ...`), `itertools` (chain,
  islice, takewhile, dropwhile), and `more_itertools` are the equivalent.
  The "filter buys, format CSV, take 10" example renders as:

  ```python
  rows = (
      to_csv_row(order)
      for order in orders
      if is_buy(order)
  )
  for row in itertools.islice(rows, 10):
      write(row)
  ```

  **Important Python-specific note:** generators are consumed once. The
  C++ caveat "a view re-runs the predicate per traversal" inverts: a
  Python generator cannot be iterated twice at all. Materialize with
  `list(...)` if a second pass is needed.

  For **pandas** code (the factors2 default), the analogue is `df.pipe(...)`
  chains and lazy evaluation through `dask` or `ibis`. The dataframe
  itself is eager, but pipe chains keep the readability win. Recommend
  adding a short "pandas / DataFrames" subsection.
- **Open questions:** Whether to give pandas its own short subsection or
  let the pandas team's skill (factors2 already uses heavy pandas) cover
  it. Recommendation: a short paragraph in this file plus a cross-link
  to a pandas-specific chapter if the team wants one.

### Cost model

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Lazy view: O(N), zero intermediates; eager: O(N*M),
  M-1 intermediates.
- **Python rendering:** Same table, with generators replacing views. Python
  comprehensions are eager (build a list) so the rule "use a generator
  expression `(... for ...)` not a list comprehension `[... for ...]` when
  the next step is a one-shot consumer" is the load-bearing piece.

## Suggested Python file: `python-design-principles/declarative-style.md`

Verbatim port with the Python idiom substitutions and an added short
pandas section. The "ranges and algorithms" -> "comprehensions, itertools,
and the built-ins" mapping is the largest change.
