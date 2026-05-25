# Condition-Based Waiting

Asynchronous work, threads, subprocesses, queues, and external resources make
tests flaky when waits are based on guesses. Wait for the observable condition
under test, and give each wait a timeout with a failure message that names the
condition.

Fixed sleeps belong in tests for timing behavior. Other tests should use
events, queues, timeouts, readiness signals, or a reusable predicate wait.

## Predicate Waits

A synchronous wait helper polls a predicate until it returns true or a
monotonic timeout expires.

```python
from collections.abc import Callable
from time import monotonic, sleep


def wait_for(
    condition: Callable[[], bool],
    *,
    description: str,
    timeout_seconds: float = 5.0,
    interval_seconds: float = 0.01,
) -> None:
    deadline = monotonic() + timeout_seconds
    while True:
        if condition():
            return
        if monotonic() >= deadline:
            raise TimeoutError(f"timeout waiting for {description} after {timeout_seconds:.3f}s")
        sleep(interval_seconds)
```

Read state inside the predicate so each poll observes current state.

```python
wait_for(
    lambda: subscriber.message_count() >= 5,
    description="subscriber received five messages",
)
```

Use a short interval that keeps tests responsive without burning CPU. The
timeout is a per-test diagnostic, not the CI job's last line of defense.

## Threaded Code

When the producer can signal readiness, use `threading.Event`,
`threading.Condition`, or `queue.Queue` with a timeout. These primitives avoid
polling and make missed wakeups less likely.

```python
def test_worker_publishes_result() -> None:
    ready = threading.Event()
    results: queue.Queue[Result] = queue.Queue()

    worker = Worker(on_result=lambda result: (results.put(result), ready.set()))
    worker.start()

    assert ready.wait(timeout=5.0)
    assert results.get_nowait().status is ResultStatus.ACCEPTED
```

For shared mutable state, protect reads and writes with the same lock or use a
queue that owns the synchronization.

## Async Code

For `asyncio`, prefer `asyncio.Event`, `asyncio.Queue`, task completion, and
`asyncio.wait_for` over sleep loops.

```python
async def test_consumer_receives_message() -> None:
    messages: asyncio.Queue[Message] = asyncio.Queue()
    consumer = Consumer(on_message=messages.put_nowait)

    async with consumer.running():
        message = await asyncio.wait_for(messages.get(), timeout=5.0)

    assert message.kind == "portfolio_updated"
```

Use `TaskGroup` when the test owns several concurrent tasks. Keep timeouts at
the boundary where the test waits for an observable result.

```python
async def test_reconciler_finishes_batch() -> None:
    async with asyncio.TaskGroup() as task_group:
        task_group.create_task(reconciler.run_once())
        completed = await asyncio.wait_for(reconciler.completed_batches.get(), timeout=5.0)

    assert completed.batch_id == BatchId("B-001")
```

## Subprocess Readiness

Subprocess tests should wait for a real readiness signal:

- A health endpoint returns success.
- A port accepts connections.
- A known output line appears.
- A pid file or socket file appears.
- The process exits with the expected status.

```python
def server_is_healthy() -> bool:
    try:
        return httpx.get("http://127.0.0.1:8080/health").status_code == 200
    except httpx.TransportError:
        return False


def test_server_reports_ready(server_process: subprocess.Popen[str]) -> None:
    wait_for(
        server_is_healthy,
        description="server health endpoint",
        timeout_seconds=10.0,
    )
```

If the process can fail before readiness, include process state in the
predicate or check it between polls so the test reports early failure.

## Files And External Effects

For file creation, queue depth, log records, callback delivery, and database
visibility, wait for the state the caller cares about.

```python
wait_for(
    lambda: (export_dir / "positions.csv").exists(),
    description="positions export file",
)
```

When asserting file contents, wait for both existence and complete content if
the writer can create the file before flushing all data.

```python
wait_for(
    lambda: (export_dir / "positions.csv").exists()
    and "PETR4" in (export_dir / "positions.csv").read_text(),
    description="positions export content",
)
```

## Timing Behavior

A fixed sleep is appropriate when elapsed time is the behavior under test.
First wait for the timer or worker to start, then sleep for a duration derived
from the documented timing.

```python
def test_heartbeat_emits_twice_per_interval() -> None:
    heartbeat.start()
    wait_for(heartbeat.is_running, description="heartbeat started")

    sleep(0.25)

    assert heartbeat.emitted_count >= 2
```

Keep the derivation near the assertion in the test name, constants, or helper
names so the timing rule is visible during review.

## Failure Messages

Timeout failures should name the awaited condition and include useful
diagnostics. Wrap common waits in domain helpers when that improves failures.

```python
def wait_for_order_status(
    repository: OrderRepository,
    order_id: OrderId,
    status: OrderStatus,
) -> None:
    def has_status() -> bool:
        order = repository.get(order_id)
        return order is not None and order.status is status

    wait_for(
        has_status,
        description=f"order {order_id} reached {status.value}",
        timeout_seconds=5.0,
    )
```

Domain helpers keep repeated waits consistent and keep test bodies focused on
the behavior under test.
