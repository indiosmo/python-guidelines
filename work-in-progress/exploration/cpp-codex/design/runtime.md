# Runtime and inter-thread communication

Overall disposition: PORTS_WITH_ADAPTATION.

The architectural rule ports: threading and async are edge effects. The Python
chapter should cover both threads and `asyncio`, because modern Python systems
often mix event loops, thread pools, and process pools.

## Single-threaded internals

Disposition: PORTS_AS_IS.

Keep it. A component owns state in one event loop, one worker thread, or one
process. Other execution contexts send messages to it.

```python
class QuoteBook:
    def __init__(self) -> None:
        self._levels: dict[SymbolId, Level] = {}
        self._updates_seen = 0

    def on_quote(self, update: Quote) -> None:
        self._levels[update.symbol] = update.level
        self._updates_seen += 1
```

The class is not thread-safe by default; the runtime decides where it runs.

## Marshalling between threads

Disposition: PORTS_WITH_ADAPTATION.

Use `queue.Queue`, `asyncio.Queue`, `loop.call_soon_threadsafe`, executor
handoff, or actor-style mailboxes.

```python
loop.call_soon_threadsafe(consumer.send, event)
```

For async code:

```python
await queue.put(event)
```

Do not block the sending context waiting for a reply unless the protocol is
explicitly synchronous.

## The marshalling primitive

Disposition: PORTS_WITH_ADAPTATION.

Replace the C++ `event_loop` with standard Python primitives. Recommended
shape:

```python
class Mailbox[T]:
    def __init__(self) -> None:
        self._queue: asyncio.Queue[T] = asyncio.Queue()

    async def send(self, item: T) -> None:
        await self._queue.put(item)

    async def run(self, handle: Callable[[T], Awaitable[None]]) -> None:
        while True:
            await handle(await self._queue.get())
```

Use bounded queues when backpressure is part of the correctness story.

## Per-publisher SPSC queues for high-volume streams

Disposition: PORTS_WITH_ADAPTATION.

The shape ports, but Python's implementation choices differ. For high-volume
streams, first ask whether Python should be in the hot path. Options include
bounded `asyncio.Queue`, `queue.SimpleQueue`, multiprocessing queues,
shared-memory buffers, native extensions, or pushing the stream into a
specialized library.

## Choosing between marshalling and sharing

Disposition: PORTS_AS_IS.

Keep the decision tree, with Python specifics:

- Default: schedule work on the owner loop or worker.
- Small shared state: use immutable snapshots, `ContextVar`, atomics from
  libraries only when needed, or a lock around a tiny surface.
- Last resort: `threading.Lock`, `asyncio.Lock`, or process-safe primitives.

The GIL does not make compound invariants safe. It also does not protect
state shared across processes or native extensions.

## Document the deviation, not the default

Disposition: PORTS_AS_IS.

Keep it. Document runtime wrappers, thread-safe methods, non-default event
loops, and methods callable from other threads. Do not add "not thread-safe"
noise to every ordinary domain class.

## Failure mode: undocumented runtime wrappers

Disposition: PORTS_AS_IS.

Keep it. Python wrappers around SDK callbacks, schedulers, and ASGI lifespan
state are frequent wrong-context sources.

## Python-specific additions

Disposition: PYTHON_SPECIFIC_VARIANT.

Add sections for:

- `asyncio.TaskGroup` as the default owner of sibling tasks.
- Cancellation as a normal control path.
- `ExceptionGroup` and `except*` at task boundaries.
- `contextvars.ContextVar` for request-scoped context.
- When to use threads, processes, native extensions, or async I/O.
