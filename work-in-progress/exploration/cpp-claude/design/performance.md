# Analysis: cpp-design-principles/performance.md

Disposition: **PORTS_WITH_ADAPTATION** for the philosophy; **PYTHON_SPECIFIC_VARIANT**
for the concrete primitives (no `fixed_string`, no `static_vector`, no
`inplace_function`; the Python performance vocabulary is allocations,
GC pressure, dict layout, numpy/pandas vectorisation, and CPython
bytecode overhead).

## Section-by-section

### Default to clarity / requirements / measure

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Performance follows from good design; optimise
  measured hot paths; have a budget; do not optimise blind.
- **Python rendering:** Identical. Tools change:
  - Benchmarks: `pytest-benchmark` (already in factors2), `pyperf`,
    `timeit` for one-offs.
  - Profilers: `pyinstrument` for everyday, `py-spy` for production,
    `memray` for memory, `scalene` for line-level CPU+memory.
  - Microbenchmarks: `dis.dis(fn)` to inspect bytecode; profile-guided
    optimisation is much rarer in Python than in C++.

### Type-erased callables on hot paths

- **Classification:** DROP.
- **Reason:** Python callables already pay the indirection cost. There
  is no equivalent "inline the call site" lever; the hot-path advice is
  "drop into C / Cython / Rust if the rate budget cannot be met by
  Python".

### Hot-path allocations

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** Use `fixed_string<N>` instead of `std::string`;
  `static_vector` for bounded, `vector` with `reserve` for growing;
  allocate at startup, never during request handling.
- **Python rendering:** Different toolkit:

  | C++ swap                                | Python equivalent                                                            |
  |-----------------------------------------|------------------------------------------------------------------------------|
  | `lib::fixed_string<N>` (stack-resident) | No analogue. Use `sys.intern(s)` for small repeated strings; otherwise eat it.|
  | `static_vector<T, N>`                   | `array.array` for numeric; `numpy.empty(N)` for numeric hot paths; `bytearray` for bytes. |
  | `vector<T>` with `reserve`              | `list` pre-allocated as `[None] * N`; for numeric, `np.empty(N)`.             |
  | `inplace_function` (no heap)            | No analogue; closures are heap-allocated.                                     |

  Python-specific hot-path advice:
  - Vectorise with **numpy / pandas / polars** before reaching for any
    Python-loop optimisation. A pure-Python loop over 10^6 elements is
    one to two orders of magnitude slower than the same operation in
    numpy.
  - Use **`__slots__`** on classes that are instantiated millions of
    times -- it cuts memory ~40% and improves attribute access.
  - For per-message allocations on a real hot path, **`functools.lru_cache`**
    is your friend; for object pooling, write a tiny ring of pre-built
    instances.
  - Free-threaded Python (3.13t/3.14t) trades single-threaded throughput
    for true multi-thread throughput. Benchmark before adopting.

### Map and ordered container choice

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** Flat vs node trade-off (cache locality vs
  reference stability).
- **Python rendering:** dict is hash-based and always node-like (each
  entry's value is a separate allocation); there is no flat-vs-node
  choice in CPython. The Python equivalents:

  - **For numeric / columnar hot paths**: pandas DataFrame or polars
    DataFrame (columnar, vectorised) instead of `dict[str, list]`.
  - **For LRU eviction**: `collections.OrderedDict` plus
    `move_to_end`; `functools.lru_cache` for cached function calls.
  - **For sorted-key access**: `sortedcontainers.SortedDict` /
    `sortedcontainers.SortedList`; the stdlib has no balanced
    BST.

### "Declarative style on the hot path"

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Declarative is the default; depart only on a
  measured hot path with a known budget.
- **Python rendering:** Same rule, stated more strongly: do not
  micro-optimise Python loops; the win is usually in moving the work
  to numpy/pandas/polars (vectorisation), not in handwriting a tighter
  Python loop. Reach for Cython / Numba / Rust extensions when
  vectorisation is not available.

## Suggested Python file: `python-design-principles/performance.md`

A rewritten chapter focused on Python-specific levers: vectorisation,
`__slots__`, interning, `functools.lru_cache`, profiler choice, the
"when to leave Python" decision, and a short free-threaded sidebar.
