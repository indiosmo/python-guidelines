# Structured concurrency in Python (2026)

## Bottom line

For these guidelines, recommend **anyio** (with trio-style cancel scopes) as
the default when the program does meaningful concurrency. Use
`asyncio.TaskGroup` (3.11+) for the simplest "fan out and join" cases where
the extra anyio dependency is not justified.

## Why anyio over bare asyncio

`asyncio.TaskGroup` provides task spawning and exception aggregation, but
does not expose cancel scopes, does not let you list the contained tasks,
and gives no way to cancel a single child without cancelling the whole
group. anyio's `TaskGroup` carries its own cancel scope, supports nested
cancel scopes, and treats cancellation as first-class -- which lines up
with the "runtime owns the threads, inner code is single-threaded"
discipline from the C++ guides.

## Mapping to the C++ runtime model

| C++ idiom                                            | Python equivalent                                                                |
|------------------------------------------------------|----------------------------------------------------------------------------------|
| `event_loop::post(task)` for cross-thread work       | `loop.call_soon_threadsafe(coro)` for true threads; `nursery.start_soon` (anyio) when staying on the event loop. |
| Inner component single-threaded by default           | Coroutine-only, single-threaded asyncio/anyio loop; no `threading.Lock` in domain code. |
| Threading is an edge effect                          | Push `asyncio.to_thread(...)` and `concurrent.futures.ThreadPoolExecutor` to the runtime layer near `main`. |
| `lib::inplace_function<void(), 2048>` task budget     | No analogue. The Python guidance is "keep task closures small for readability"; runtime cost is dominated by the event loop, not closure size. |
| SPSC queue per high-volume publisher                 | `asyncio.Queue` for in-loop; `janus.Queue` for thread/coroutine bridges; native CPython `queue.SimpleQueue` for thread-to-thread. |
| Read-mostly atomic flag                               | `asyncio.Event` for in-loop; `threading.Event` for thread bridges. For one-word counters, plain `int` plus the GIL is usually safe; for the cross-thread `bool`, use `threading.Event`. |
| `std::shared_mutex` / `std::mutex`                    | `asyncio.Lock`, `anyio.Lock`, `threading.RLock`. Same "last resort" advice. |

## Cancellation discipline

```python
import anyio

async def main():
    async with anyio.create_task_group() as tg:
        tg.start_soon(serve_market_data)
        tg.start_soon(serve_orders)
        # On any child raising, the whole group is cancelled; cleanup
        # happens through normal context managers and `finally` blocks.
```

Two cancellation rules to call out in the Python guide:

1. **Re-raise `BaseException` / `CancelledError`.** Catching and swallowing
   cancellation leaves the task group unable to exit; anyio explicitly
   documents this as "undefined behaviour" risk.
2. **Use `move_on_after` / `fail_after` for timeouts**, not
   `asyncio.wait_for(...)`. The cancel-scope shape composes; the
   coroutine-wrapping shape interacts badly with task groups.

## Debugging

- `PYTHONASYNCIODEBUG=1` (or `asyncio.run(..., debug=True)`) catches slow
  callbacks, unawaited coroutines, and accidental cross-thread access.
- `anyio.run(main, backend_options={"debug": True})` does the same for
  anyio.
- For real "what is each thread doing right now", py-spy is the production
  primitive (sampling, no instrumentation).

## Sources

- https://anyio.readthedocs.io/en/stable/why.html
- https://anyio.readthedocs.io/en/stable/cancellation.html
- https://anyio.readthedocs.io/en/stable/tasks.html
- https://mattwestcott.org/blog/structured-concurrency-in-python-with-anyio
- https://peps.python.org/pep-0789/
