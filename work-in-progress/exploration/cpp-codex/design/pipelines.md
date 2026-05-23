# Pipelines

Overall disposition: PORTS_AS_IS.

The pipeline chapter ports almost directly. Python changes callback spelling
and runtime choices, but the topology rule is the same: stages expose inbound
methods and outbound callbacks; wiring near the composition root owns the
graph.

## Pattern

Disposition: PORTS_AS_IS.

Use methods for inbound messages and typed callbacks for outbound events.

```python
@dataclass
class OrderRouter:
    on_routed: Callable[[RoutedOrder], None]
    on_rejected: Callable[[Rejection], None]

    def send(self, request: SubmitRequest | CancelRequest) -> None:
        ...
```

For async pipelines:

```python
type AsyncSink[T] = Callable[[T], Awaitable[None]]


@dataclass
class MarketDataNormalizer:
    on_book_update: AsyncSink[BookUpdate]
```

The stage should not know who consumes its output.

## Source, sink, and stage

Disposition: PORTS_AS_IS.

Keep the vocabulary. In Python, avoid base classes unless runtime polymorphism
or a type-checking `Protocol` earns its place.

```python
class Sink[T](Protocol):
    def send(self, item: T) -> None: ...
```

## Wiring near `main`

Disposition: PORTS_AS_IS.

Keep it. In Python, the wiring point may be an application factory, CLI entry,
ASGI lifespan hook, dependency container, or test fixture.

```python
def build_app(config: Config) -> App:
    session = VendorSession(config.session)
    router = OrderRouter(
        on_routed=session.send,
        on_rejected=lambda rejection: logger.info("reject %s", rejection),
    )
    session.on_execution_report = router.send
    return App(session=session, router=router)
```

## Interception at the wiring point

Disposition: PORTS_AS_IS.

Keep the section. Threading, telemetry, filtering, retries, and capture belong
in the assigned callable.

```python
router.on_routed = lambda order: engine_loop.call_soon_threadsafe(engine.send, order)
```

Async version:

```python
async def routed(order: RoutedOrder) -> None:
    if universe.contains(order.symbol):
        await session.send(order)


router.on_routed = routed
```

## Defer re-entrant callbacks instead of recursing

Disposition: PORTS_AS_IS.

Keep it. Python can use `asyncio.Queue`, `queue.Queue`, `deque`, or event-loop
posting.

```python
class SerializedStage:
    def __init__(self, inner: Stage) -> None:
        self._inner = inner
        self._pending: deque[Message] = deque()
        self._dispatching = False

    def send(self, message: Message) -> None:
        if self._dispatching:
            self._pending.append(message)
            return

        self._dispatching = True
        try:
            self._inner.send(message)
            while self._pending:
                self._inner.send(self._pending.popleft())
        finally:
            self._dispatching = False
```

## External integrations

Disposition: PORTS_AS_IS.

Keep it. Wrappers around SDKs, sockets, webhooks, and queues should translate
foreign messages into domain dataclasses before calling `on_*`.

```python
def on_vendor_execution(self, message: VendorExecution) -> None:
    self.on_execution_report(translate_execution(message))
```

## Why this shape pays off

Disposition: PORTS_AS_IS.

Keep all four arguments: no circular dependencies, easy test capture, visible
topology, and effects at the edge.

## Trade-offs

Disposition: PORTS_WITH_ADAPTATION.

Keep the warnings but add Python-specific hazards:

- Default no-op callbacks can hide missing production wiring.
- Callable fields should be initialized deliberately, often through a factory.
- Async callbacks must be awaited or scheduled explicitly.
- Global event buses are especially tempting in Python and should be treated as
  a coupling smell unless the domain really is pub-sub.
