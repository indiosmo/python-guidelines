# Analysis: cpp-design-principles/pipelines.md

Disposition: **PORTS_AS_IS** for the pattern (inbound methods + outbound
callback fields, wiring near `main` as a combinator); **PORTS_WITH_ADAPTATION**
for the concrete callable-storage spelling and the threading-bridge
discussion.

## Section-by-section

### Pattern (inbound member functions, outbound callback fields)

- **Classification:** PORTS_AS_IS.
- **C++ principle:** A stage has inbound methods (`send`, `submit`) and
  outbound callback fields (`on_<event>` holding type-erased callables);
  no stage knows who calls it or who consumes its output.
- **Python rendering:** Identical. Replace `lib::inplace_function<Sig,
  Capacity>` with `Callable[[Args], None] | None` (default `None`) or
  with a list of callables for explicit fan-out:

  ```python
  class OrderRouter:
      def __init__(self) -> None:
          # outbound: filled in by the wiring layer.
          self.on_routed: Callable[[RoutedOrder], None] | None = None
          self.on_rejected: Callable[[Rejection], None] | None = None

      def send(self, request: SubmitRequest | CancelRequest) -> None:
          ...
  ```

  An assertion at startup (`assert self.on_routed is not None`) catches
  the "wiring layer forgot a callback" failure mode -- the equivalent of
  the C++ "default-construct to a no-op" / "assert at startup" rule.

  Convention: name callbacks `on_<event>`; do not default-initialise them
  to `lambda *_: None` unless the absence of a subscriber is truly
  optional. Otherwise leave them `None` and assert.

### Source, sink, stage vocabulary

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same. Use as documentation vocabulary, not as
  ABCs.

### Wiring near `main`

- **Classification:** PORTS_AS_IS.
- **C++ principle:** The wiring layer is the only place that holds the
  full topology; outbound callbacks become lambdas posting to the next
  consumer.
- **Python rendering:** Same. In an async program the wiring lambda
  becomes:

  ```python
  router.on_routed = lambda order: asyncio.create_task(session.send(order))
  ```

  For a fully-sync pipeline, the lambda is a direct call:
  `router.on_routed = session.send`.

### Interception at the wiring point

- **Classification:** PORTS_AS_IS.
- **C++ principle:** The callback assignment is itself a combinator;
  cross-cutting concerns (threading, telemetry, filtering, tee)
  compose at the wiring site.
- **Python rendering:** Identical. The thread-marshalling example
  becomes a `loop.call_soon_threadsafe(...)` or `nursery.start_soon`
  call. The tee, filter, and capture-into-list examples render
  unchanged.

### Defer re-entrant callbacks instead of recursing

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Queue the re-entrant callback now, dispatch when
  the call stack unwinds.
- **Python rendering:** In an async context, `asyncio.create_task(...)`
  defers by itself -- the new task runs after the current coroutine
  yields. For sync code, use `collections.deque` and a drain pass,
  exactly as the C++ `serialized_stage` example shows.

### External integrations

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same. A vendor wrapper class implements the
  vendor's listener interface and re-emits through domain-shaped
  `on_*` callbacks. For an SDK that uses `asyncio` event sources
  internally, the wrapper subscribes to those and forwards to the
  wiring lambda; for a thread-callback SDK, the wrapper marshals
  through `loop.call_soon_threadsafe`.

### Why this shape pays off / Trade-offs

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same prose; drop the "type erasure has a cost"
  bullet (no equivalent concern). Keep the "wiring grows quadratically
  if every component talks to every other", "synchronous unwind is a
  property of direct-call wiring", and "unassigned callbacks are a
  runtime hazard" bullets verbatim.

## Suggested Python file: `python-design-principles/pipelines.md`

Verbatim port. The concrete `lib::inplace_function<Sig, N>` snippets
become `Callable[..., None] | None` fields; the threading-bridge
snippets become `asyncio.create_task` / `loop.call_soon_threadsafe`
snippets; the C++ ABC discussion is dropped (Python's structural
typing already covers it).
