# Performance

Overall disposition: PORTS_WITH_ADAPTATION.

The philosophy ports; the low-level advice needs Python-specific replacement.
The Python guide should be explicit that CPython performance work usually
starts with algorithmic shape, I/O boundaries, allocation volume, vectorized
libraries, caching, concurrency model, and profiler evidence.

## Philosophy

Disposition: PORTS_AS_IS.

Keep the subsections: default to clarity, require a target, measure rather
than guess, keep costs in mind, evaluate in full context, and leave
declarative style unless a measured hot path says otherwise.

Python rendering:

```text
Requirement: process 50,000 messages per second with p99 under 20 ms.
Measurement: py-spy and pyperf show 45 percent in parse_symbol.
Change: replace regex parse with table lookup and benchmark again.
```

## Type-erased callables on hot paths

Disposition: PYTHON_SPECIFIC_VARIANT.

Replace with Python call overhead guidance. Python functions, bound methods,
closures, and `functools.partial` have real overhead in tight loops. Do not
optimize this by default; measure first. On hot loops, move invariant lookups
out of the loop, avoid per-item closure construction, or use vectorized/native
code.

```python
append = output.append
for item in items:
    append(transform(item))
```

This is justified only in hot code with a benchmark.

## Hot-path allocations

Disposition: PORTS_WITH_ADAPTATION.

Rewrite around Python allocations:

- Avoid constructing temporary dicts, dataclasses, strings, and lists per item
  in measured loops.
- Use generators to avoid intermediate lists when streaming once.
- Use lists intentionally when repeated traversal or indexing is needed.
- Prefer arrays, NumPy, pandas, Polars, or native extensions for numeric bulk
  operations.
- Precompile regexes and parse tables at module load or startup.
- Use object pools only with strong evidence; they often fight CPython's
  allocator and make code worse.

```python
SYMBOL_PATTERN = re.compile(r"^[A-Z0-9_.-]{1,16}$")
```

## Map and ordered container choice

Disposition: PORTS_WITH_ADAPTATION.

Python `dict` is the default hash map and preserves insertion order. The
chapter should discuss:

- `dict` and `set` for ordinary lookup.
- `list` plus `bisect` for small sorted collections.
- `heapq` for priority queues.
- `collections.deque` for queue ends.
- `functools.lru_cache` or `cache` for pure repeated computations.
- Third-party sorted containers when ordered mutation dominates.

Reference stability is a different question in Python because names hold
object references. The guide should instead warn about mutating keys,
invalidating iterators by changing dict size during iteration, and retaining
references to mutable values whose owner still mutates them.
