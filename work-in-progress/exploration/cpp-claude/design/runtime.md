# Analysis: cpp-design-principles/runtime.md

Disposition: **PORTS_WITH_ADAPTATION**. The principle (threading is an
edge effect; inner components are single-threaded; marshalling beats
sharing) ports directly. The concrete primitives (event_loop, posted
inplace_function, SPSC queues, atomics, seqlocks) become Python's asyncio /
anyio primitives plus the stdlib `threading` and `multiprocessing` modules,
with a sidebar on the free-threaded interpreter (3.13t/3.14t).

## Section-by-section

### Single-threaded internals

- **Classification:** PORTS_AS_IS.
- **C++ principle:** A component owns its state on one thread; methods
  are not synchronized; fields are plain members.
- **Python rendering:** Same rule. In an asyncio program the
  "thread" is **the event loop's single thread**, and the rule is "all
  mutations to component state happen on the loop". In a sync program
  with no concurrency, the rule is automatic.
- **Open questions:** Under free-threaded Python (3.13t / 3.14t),
  GIL-derived atomicity disappears. The guide should flag this and
  recommend, for free-threaded code, an explicit per-component
  `threading.Lock` when the component is shared across threads -- or,
  better, keep the discipline of "components are single-threaded; cross-
  thread access posts a callable to the owning thread's loop or
  thread-pool".

### Marshalling between threads

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Each thread owns an event loop with a thread-safe
  task queue; `loop.post(closure)` is the one-liner that hands work
  across.
- **Python rendering:** Three concrete shapes the chapter covers:

  | C++ shape                                  | Python equivalent                                                                |
  |--------------------------------------------|----------------------------------------------------------------------------------|
  | `b_loop.post([&b, event]{ b.handle(ev); })` | `loop.call_soon_threadsafe(b.handle, event)` for thread -> loop.                  |
  | "Each thread runs an event loop"            | `loop.run_until_complete(...)` / `asyncio.run(...)` plus an executor for I/O work. |
  | Capture-by-value rule                       | `functools.partial(b.handle, event)` plus "don't capture references to objects that may die before the task runs". |
  | "Don't block the posting thread"            | Same rule. Use `loop.create_task(...)` for replies; never `Future.result()` from the loop thread. |

### The marshalling primitive (`event_loop`)

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** A narrow `event_loop` with a high-quality
  thread-safe queue and fixed-capacity inplace-function tasks.
- **Python rendering:** asyncio's `BaseEventLoop` is the equivalent;
  `call_soon_threadsafe` is the equivalent of `post`. The
  "fixed-capacity inplace function" budget has no Python analogue --
  Python closures allocate freely. Drop the inplace-function discussion
  but keep the "don't block the posting thread" rule.

### Per-publisher SPSC queues for high-volume streams

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Lock-free SPSC queues for firehose fan-out.
- **Python rendering:** Python is not the right tool for kHz fan-out;
  the equivalent advice is "if you need this, use one of (1) Cython,
  (2) a Rust extension, (3) a separate process consuming from a shared
  queue like `multiprocessing.Queue` or a true broker like Redis
  Streams". Within Python: `asyncio.Queue` for in-loop, `queue.Queue`
  for thread-to-thread, `multiprocessing.Queue` for process-to-process.
  Document the rate-budget threshold (typically tens of thousands per
  second) where Python becomes the wrong tool.

### Choosing between marshalling and sharing

- **Classification:** PORTS_AS_IS (the principle), PORTS_WITH_ADAPTATION
  (the primitives).
- **Python rendering:** Same three-tier decision tree:

  1. **Default: post to the owning thread.** `loop.call_soon_threadsafe`
     for thread -> loop, `loop.create_task` for in-loop scheduling.
  2. **Lock-free primitive: small, hot, mostly-read state.**
     - `threading.Event` for a one-bit "is ready" flag.
     - `itertools.count()` for a thread-safe-under-GIL counter (drop
       under free-threaded; use `threading.Lock` instead).
     - `concurrent.futures.Future` for a one-shot result.
  3. **`threading.Lock` / `threading.RLock`: last resort.** Same rule
     as C++: rare, and the right answer is usually "post instead".

### Document the deviation, not the default

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. Document threading on:
  - Runtime-layer wrappers that own threads or loops.
  - Classes that are deliberately thread-safe.
  - Methods that break the surrounding class's contract.
  - Components running on a non-default loop in a multi-loop program.

  Add a Python-specific bullet: **components that run on a thread pool
  executor** -- the rule is "any function passed to
  `loop.run_in_executor(...)` must not touch loop-owned state".

### Failure mode: undocumented runtime wrappers

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same hazards. Add: misuse of
  `asyncio.get_event_loop()` outside `asyncio.run` is the Python
  equivalent of an undocumented wrapper -- the call may return a
  brand-new loop when the author thinks they are reaching the
  application's loop.

## Suggested Python file: `python-design-principles/runtime.md`

Verbatim port with the primitive substitutions called out above, a
sidebar on free-threaded Python, and a short "when Python is the wrong
tool" paragraph for genuine high-rate fan-out.
