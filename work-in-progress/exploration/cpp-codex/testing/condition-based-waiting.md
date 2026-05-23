# Condition-based waiting

Overall disposition: PORTS_AS_IS.

The core principle ports exactly: wait for the condition the test cares about,
not for a guessed duration.

## When to use this pattern

Disposition: PORTS_AS_IS.

Keep it. Python cases include worker threads, async callbacks, files,
subprocess readiness, queue messages, web sockets, and GUI events.

## The core pattern

Disposition: PORTS_WITH_ADAPTATION.

Python polling helper:

```python
def wait_for(
    condition: Callable[[], bool],
    description: str = "condition",
    timeout: float = 5.0,
    interval: float = 0.01,
) -> None:
    deadline = time.monotonic() + timeout
    while not condition():
        if time.monotonic() >= deadline:
            raise TimeoutError(f"timeout waiting for {description}")
        time.sleep(interval)
```

Async version:

```python
async def wait_for_async(
    condition: Callable[[], bool],
    description: str = "condition",
    timeout: float = 5.0,
) -> None:
    async with asyncio.timeout(timeout):
        while not condition():
            await asyncio.sleep(0.01)
```

## Common scenarios

Disposition: PORTS_AS_IS.

Keep the table with Python predicates:

```python
wait_for(lambda: ready.is_set(), "worker ready")
wait_for(lambda: queue.qsize() >= 5, "queue reached threshold")
```

## Condition variables for producer/consumer

Disposition: PORTS_WITH_ADAPTATION.

Use `threading.Condition` or `asyncio.Condition`.

```python
with condition:
    signaled = condition.wait_for(lambda: ready, timeout=5.0)
assert signaled
```

Prefer `threading.Event` for one-bit readiness.

## When an arbitrary sleep is actually correct

Disposition: PORTS_AS_IS.

Keep it. A fixed sleep is appropriate when the timed duration is itself under
test, and only after waiting for the trigger to start.

## Common mistakes

Disposition: PORTS_AS_IS.

Keep all mistakes and adapt memory ordering to Python:

- Polling too fast burns CPU.
- Missing timeouts hang pytest runs.
- Capturing stale state makes the predicate useless.
- Plain shared mutable state still needs a lock, event, condition, queue, or
  owner-thread discipline.
