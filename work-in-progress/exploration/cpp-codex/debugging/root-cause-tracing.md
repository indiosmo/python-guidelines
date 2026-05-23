# Root-cause tracing

Overall disposition: PORTS_AS_IS.

The chapter ports directly. Python stack traces are often richer than C++
stack traces, but the debugging discipline is the same: trace the bad value or
state backward to where it originated.

## When to trace backward

Disposition: PORTS_AS_IS.

Keep the list. Python examples include a `KeyError` deep in a handler, a
Pydantic validation error after several transforms, a task failure reported by
an event loop, or a database integrity error far from the request parser.

## Tracing procedure

Disposition: PORTS_AS_IS.

Keep all six steps. Render examples in Python:

```python
def lookup(self, key: str) -> Destination:
    if not key:
        raise InvalidRoutingKey(key)
```

The function is correct; the debugging question is who passed the empty key.

```python
def parse(buffer: bytes) -> Message:
    payload = json.loads(buffer)
    return Message(payload=payload["payload"], routing_key="")
```

Fix the parser, not the lookup.

## Instrumenting when manual tracing runs out

Disposition: PORTS_WITH_ADAPTATION.

Use `logger.debug(..., stack_info=True)`, `traceback.format_stack`,
`faulthandler`, `pdb`, `breakpoint()`, or debugger breakpoints.

```python
if not key:
    logger.debug("empty routing key", extra={"message_id": message_id}, stack_info=True)
    raise InvalidRoutingKey(key)
```

For async code, log task names, request ids, and queue offsets; stack alone may
not identify the logical request.

## Log before the failure, not after

Disposition: PORTS_AS_IS.

Keep it. Python rendering:

```python
logger.debug(
    "dispatching message",
    extra={"message_id": message.id, "routing_key": message.routing_key},
)
destination = router.lookup(message.routing_key)
```

## Finding the test that pollutes state

Disposition: PORTS_WITH_ADAPTATION.

Keep the bisection idea. Use pytest node ids, markers, and `--cache-clear` or
temporary directories where appropriate.

```bash
uv run pytest tests/path/test_file.py::test_name
```

For process-wide pollution, inspect module globals, environment variables,
registered logging handlers, event loops, monkeypatches, caches, and provider
objects.

## Bisecting flaky tests with randomized order

Disposition: PORTS_WITH_ADAPTATION.

Use pytest randomization plugins or `pytest-randomly` when available. Record
the seed and replay with the same seed before changing code.

## Reusing the mock clock in a debug harness

Disposition: PORTS_AS_IS.

Keep it. A manual clock provider can be installed in the real application
factory and driven by a harness to reproduce timing bugs deterministically.

## Low-perturbation observability

Disposition: PORTS_AS_IS.

Keep it. In Python, detailed logging can hide async scheduling bugs or overload
the GIL. Use counters, sampling, tracing spans, metrics, py-spy, or event
streams where hot-path logging would perturb behavior.
