# Runtime And Concurrency

Concurrency belongs at the runtime boundary. Domain code should read as
ordinary Python: inputs arrive, values change under one owner, and outputs
leave through explicit calls, queues, callbacks, or async functions. The
runtime layer owns event loops, threads, worker pools, rate limits, timeouts,
framework entry points, and cross-thread marshalling.

Use structured concurrency for related async work. Use queues and ownership
boundaries for shared work. Reach for locks only when the state is genuinely
shared and the shared surface is small.

## Runtime Ownership

Application and orchestration code own concurrency policy. A CLI `main`, API
app factory, queue consumer, Dagster asset, worker initializer, or service
startup function decides which event loop, thread, process, client, limiter,
and executor the work uses.

Domain services should not need to know whether a caller runs in a coroutine,
thread, process, actor, or scheduled job. Pass data and dependencies inward,
and return values or domain events outward.

```python
from collections.abc import Iterable
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class Quote:
    symbol: str
    bid: float
    ask: float


class TopOfBook:
    def __init__(self) -> None:
        self._quotes: dict[str, Quote] = {}

    def update(self, quote: Quote) -> None:
        self._quotes[quote.symbol] = quote

    def snapshot(self) -> Iterable[Quote]:
        return tuple(self._quotes.values())
```

`TopOfBook` owns its state. The runtime decides how updates reach it. If the
service later runs inside a worker thread, the runtime serializes calls into
that owner.

## Structured Async Work

Use `asyncio.TaskGroup` as the standard-library baseline for related fan-out
and join. It gives sibling tasks one lifetime, propagates cancellation, and
raises grouped failures through `ExceptionGroup`.

```python
import asyncio
from collections.abc import Iterable

import httpx


async def fetch_quote(client: httpx.AsyncClient, symbol: str) -> Quote:
    response = await client.get(f"/quotes/{symbol}")
    response.raise_for_status()
    payload = response.json()
    return Quote(symbol=symbol, bid=payload["bid"], ask=payload["ask"])


async def fetch_quotes(symbols: Iterable[str]) -> list[Quote]:
    quotes: list[Quote] = []
    async with httpx.AsyncClient(timeout=httpx.Timeout(10.0)) as client:
        async with asyncio.TaskGroup() as task_group:
            tasks = [
                task_group.create_task(fetch_quote(client, symbol))
                for symbol in symbols
            ]

    for task in tasks:
        quotes.append(task.result())
    return quotes
```

Use anyio when the program benefits from cancel scopes, timeout composition,
or trio compatibility. Keep the dependency decision at the project level. A
library should not force anyio into callers merely to run one simple fan-out.

Cancellation is part of the contract. Let `CancelledError` propagate. When an
exception handler translates failures at a boundary, keep cancellation outside
the translation path.

```python
class MarketDataUnavailable(RuntimeError):
    def __init__(self, *, symbols: tuple[str, ...]) -> None:
        super().__init__("market data unavailable")
        self.symbols = symbols


async def run_job(symbols: Iterable[str]) -> list[Quote]:
    try:
        return await fetch_quotes(symbols)
    except* httpx.HTTPStatusError as group:
        raise MarketDataUnavailable(symbols=tuple(symbols)) from group
```

## Clients, Timeouts, And Rate Limits

Network clients should have a clear lifetime and explicit timeouts. Use one
client per resource scope: one HTTP client for a batch, one database connection
or session for a transaction scope, one SDK client for a worker lifetime when
startup is expensive.

Use semaphores for in-flight concurrency and rate limiters for request budgets.
Keep retry, timeout, rate-limit, and cancellation policy together at the
adapter boundary.

```python
import asyncio

from aiolimiter import AsyncLimiter


class QuoteClient:
    def __init__(self, client: httpx.AsyncClient) -> None:
        self._client = client
        self._in_flight = asyncio.Semaphore(20)
        self._rate_limit = AsyncLimiter(100, time_period=60)

    async def fetch_quote(self, symbol: str) -> Quote:
        async with self._in_flight, self._rate_limit:
            response = await self._client.get(f"/quotes/{symbol}")
            response.raise_for_status()
            payload = response.json()
            return Quote(symbol=symbol, bid=payload["bid"], ask=payload["ask"])
```

Use async-compatible libraries inside `async def`. Blocking I/O moves behind
`asyncio.to_thread`, a bounded executor, a process pool, or a synchronous
boundary that runs outside the event loop. CPU-bound Python work usually needs
vectorization, native extensions, multiprocessing, Ray, or another worker
system; `asyncio.to_thread` keeps the event loop responsive but still runs
Python bytecode under interpreter constraints.

## Queues And Marshalling

Use queues to cross ownership boundaries. An `asyncio.Queue` serializes work
inside one event loop. `queue.Queue` or `queue.SimpleQueue` connects threads.
`multiprocessing.Queue` connects processes. A bridge such as `janus.Queue`
fits thread-to-event-loop handoff.

```python
import asyncio
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class QuoteUpdate:
    quote: Quote


async def consume_quotes(queue: asyncio.Queue[QuoteUpdate]) -> None:
    top_of_book = TopOfBook()
    while True:
        update = await queue.get()
        try:
            top_of_book.update(update.quote)
        finally:
            queue.task_done()
```

For true cross-thread scheduling into an event loop, use
`loop.call_soon_threadsafe` for callbacks or `asyncio.run_coroutine_threadsafe`
for coroutines. Name the loop owner in the runtime wrapper so future callers
know which side owns the state.

## Shared State

Prefer one owner that serializes mutations. Shared mutable objects are harder
to reason about than message passing, even with the GIL. Async code can
interleave at `await` points. Threaded code can interleave around I/O and C
extension calls. Process and distributed workers need durable coordination,
not only in-process locks.

Use synchronization primitives when the shared state is small and the shared
surface is deliberate:

- `asyncio.Event` or `anyio.Event` for in-loop readiness.
- `threading.Event` for thread-visible readiness flags.
- `asyncio.Lock`, `anyio.Lock`, or `threading.Lock` around a short critical
  section.
- A database transaction, uniqueness constraint, lease, or idempotency key for
  cross-process ownership.

Document runtime wrappers that are deliberately thread-safe, tied to a
specific event loop, or callable from several threads. Inner domain classes
inherit the project convention that one owner calls them.

```python
import threading


class WorkerStatus:
    def __init__(self) -> None:
        self._ready = threading.Event()

    def mark_ready(self) -> None:
        self._ready.set()

    def wait_until_ready(self, timeout_seconds: float) -> bool:
        return self._ready.wait(timeout_seconds)
```

## Re-Entrant Work

When a callback would re-enter the same owner while it is mutating state, defer
the follow-up work. Use a local deque in synchronous code, `asyncio.Queue` in
async code, or `asyncio.create_task` when the follow-up becomes independent.
Deferral changes ordering and failure propagation, so use it where it makes a
real ownership hazard visible.

```python
from collections import deque
from collections.abc import Callable


class SerializedDispatcher:
    def __init__(self) -> None:
        self._pending: deque[Callable[[], None]] = deque()
        self._dispatching = False

    def dispatch(self, action: Callable[[], None]) -> None:
        self._pending.append(action)
        if self._dispatching:
            return

        self._dispatching = True
        try:
            while self._pending:
                self._pending.popleft()()
        finally:
            self._dispatching = False
```

## Multiprocessing And Workers

Multiprocessing code should name its start method, worker initializer, input
serialization contract, and shared-resource strategy. Pass worker inputs
explicitly, and build expensive per-process state once in the worker
initializer or actor constructor.

Stateful actors fit workloads with expensive startup and many independent
jobs. The actor owns cached reference data, a model, or a connection pool; each
method call performs one bounded unit of work and returns a serializable
result.

## Review Checklist

- The runtime entry point owns event loops, clients, pools, rate limits,
  settings, logging, and worker setup.
- Related async tasks use `TaskGroup` or anyio task groups.
- Cancellation propagates through broad exception handlers.
- Async functions use async I/O or explicit offload for blocking work.
- Queues, callbacks, or event-loop scheduling cross ownership boundaries.
- Shared mutable state has one owner or a small documented lock surface.
- Worker processes receive explicit inputs and build expensive state once.
