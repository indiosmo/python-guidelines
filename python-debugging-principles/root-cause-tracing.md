# Root-Cause Tracing

Most bugs surface where code finally uses a bad value. A parser rejects an
empty field, a dataframe validator finds a missing column, or a domain method
raises after an invariant is already broken. The useful question is where the
value first became wrong.

Trace the bad value backward through the call chain, fix the origin, and add a
small regression test that fails for the original reason. A green test after a
local workaround is useful only when the root cause is gone.

## Start With The Exact Symptom

Capture the failure as a concrete observation: command, test node, file, line,
exception type, value, and operation. Narrow the loop before changing code.

```sh
uv run pytest tests/orders/test_routing.py::test_order_requires_route -x
```

Read the whole traceback. Python tracebacks often include the important frame
above the final exception, and exception chaining shows the translation path.
For grouped concurrent work, read each branch of the `ExceptionGroup`; sibling
failures often share the same upstream input.

```python
try:
    order = parse_order(payload)
except KeyError as exc:
    raise OrderParseError("order payload is missing a routing key") from exc
```

The final exception explains the boundary contract. The chained cause explains
which lower-level operation failed.

## Walk The Value Backward

When the failing function behaves correctly for the value it received, move to
the caller. Keep moving until you find the first function that creates,
normalizes, mutates, or drops the value incorrectly.

```python
from collections.abc import Mapping
from dataclasses import dataclass


@dataclass(frozen=True, slots=True, kw_only=True)
class OrderRequest:
    order_id: str
    routing_key: str
    quantity: int


def destination_for(order: OrderRequest, router: Router) -> Destination:
    if order.routing_key == "":
        raise ValueError("routing_key must be populated")
    return router.lookup(order.routing_key)


def dispatch_order(payload: Mapping[str, object], router: Router) -> None:
    order = parse_order(payload)
    destination = destination_for(order, router)
    router.dispatch(destination, order)
```

If `destination_for` rejects an empty `routing_key`, inspect `parse_order`.

```python
def parse_order(payload: Mapping[str, object]) -> OrderRequest:
    return OrderRequest(
        order_id=str(payload["order_id"]),
        routing_key=str(payload.get("route", "")),
        quantity=int(payload["quantity"]),
    )
```

The root cause is the parser defaulting a missing route to the empty string.
The fix belongs at the boundary:

```python
def parse_order(payload: Mapping[str, object]) -> OrderRequest:
    try:
        routing_key = str(payload["route"])
    except KeyError as exc:
        raise OrderParseError("order payload is missing a route") from exc

    if routing_key == "":
        raise OrderParseError("order payload route is empty")

    return OrderRequest(
        order_id=str(payload["order_id"]),
        routing_key=routing_key,
        quantity=int(payload["quantity"]),
    )
```

Downstream validation still helps diagnose impossible states, but the boundary
now owns the malformed payload.

## Probe With Focus

Add probes only to answer the next question. Good probes capture the value
before the operation that changes it, include enough context to connect the
event to a request or test, and keep variable data separate from the message.

```python
logger.debug(
    "dispatching order",
    extra={
        "order_id": order.order_id,
        "routing_key": order.routing_key,
        "quantity": order.quantity,
    },
)
```

For stack evidence inside a long callback path, use `traceback.format_stack()`
or a breakpoint at the point where the impossible value appears.

```python
import traceback

logger.debug(
    "empty routing key reached dispatcher",
    extra={"stack": "".join(traceback.format_stack(limit=12))},
)
```

In pytest, keep the loop small:

```sh
uv run pytest tests/orders/test_routing.py -k "missing route" -x
```

Use `breakpoint()`, `pdb`, `caplog`, `capsys`, and focused log output while
investigating. Convert durable diagnostics into intentional logging or remove
the probe before finishing the fix.

## Isolate Flaky Tests

Flakes usually come from leaked process state, time, ordering, randomness, or
shared resources. Treat the failure as an investigation problem before editing
production code.

Check these sources first:

- Environment variables, current working directory, and process-wide settings.
- Logging handlers, warning filters, caches, registries, and singleton clients.
- Random seeds, filesystem ordering, set rendering, time zones, and wall-clock
  reads.
- Threads, tasks, subprocesses, sockets, temporary files, and background
  workers.
- Dataframe global options and library-level configuration.

When the suite order matters, reproduce with a fixed order or a
`pytest-randomly` seed when the project uses one. Then bisect the smallest set
of tests that creates the pollution and the smallest test that observes it. The
fix usually belongs in a fixture, context manager, startup boundary, or
explicit dependency object.

## Keep Observability Low-Perturbation

Detailed logging can change timing, especially around async, threads,
multiprocessing, high-volume data processing, and retry loops. For
load-sensitive defects, prefer counters, sampled events, queue-depth metrics,
bounded debug logs, or profilers over per-item logging in a hot path.

Use a profiler when the symptom is performance or memory pressure, and use the
debugging loop when the symptom is incorrect behavior. The tools answer
different questions.

## Fix The Origin

Once the origin is known, make one root-cause change and verify it through the
smallest failing reproduction. Add a regression test that fails on the broken
path and names the behavior being protected.

After the focused test passes, run the relevant wider checks:

```sh
uv run pytest tests/orders
uv run ruff check .
uv run pyright
```

If the first fix fails, treat that as new evidence. Return to the trace with
the new observation instead of stacking a second guess on top of the first.
