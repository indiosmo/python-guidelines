# Analysis: cpp-testing-principles/condition-based-waiting.md

Disposition: **PORTS_AS_IS**. The pattern is universal -- wait for the
condition, not for a guess about how long it takes. The Python
primitives differ in shape but the rule is identical.

## Section-by-section

### When to use this pattern

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same list. Add a pytest-specific case:
  waiting for a fixture's background task to publish a message.

### The core `wait_for` pattern

- **Classification:** PORTS_WITH_ADAPTATION (spelling).
- **Python rendering:** Two forms, sync and async:

  Sync:

  ```python
  import time
  from collections.abc import Callable

  def wait_for(
      condition: Callable[[], bool],
      description: str = "condition",
      timeout: float = 5.0,
      poll_interval: float = 0.01,
  ) -> None:
      start = time.monotonic()
      while not condition():
          if time.monotonic() - start > timeout:
              raise TimeoutError(
                  f"timeout waiting for {description} after {timeout}s"
              )
          time.sleep(poll_interval)
  ```

  Async (asyncio / anyio):

  ```python
  import anyio
  from collections.abc import Callable

  async def wait_for_async(
      condition: Callable[[], bool],
      description: str = "condition",
      timeout: float = 5.0,
      poll_interval: float = 0.01,
  ) -> None:
      with anyio.fail_after(timeout):
          while not condition():
              await anyio.sleep(poll_interval)
  ```

  Prefer the anyio `fail_after` form -- it composes with cancel
  scopes and produces a clear `TimeoutError`.

### Condition variables for producer/consumer

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ `std::condition_variable`:** wakes the moment the producer
  notifies.
- **Python equivalents:**
  - `threading.Event` for sync producer/consumer (the simple flag case).
  - `threading.Condition` for the predicate-wait case.
  - `asyncio.Event` for in-loop async producer/consumer.
  - `anyio.Event` for cross-backend.

  The pattern translates directly: producer sets, consumer
  `wait_for(predicate)`. Spurious wakeups are not a concern under
  `threading.Condition.wait_for` (it re-checks the predicate); they
  are not a thing under `asyncio.Event` (single-threaded loop).

### When an arbitrary sleep is actually correct

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. The Python form is `time.sleep`
  (sync) or `anyio.sleep` (async), with the same "wait for the
  trigger, then sleep the documented duration" recipe.

### Common mistakes

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same list. Add a Python-specific one:
  **mixing `time.sleep` and `asyncio.sleep`**. `time.sleep` inside a
  coroutine blocks the event loop and breaks every other task; ruff's
  `ASYNC101` flags this.

## Suggested Python file: `python-testing-principles/condition-based-waiting.md`

Verbatim port with sync and async helpers, recommendation to default
to the anyio cancel-scope form, and a callout for the
`time.sleep`-in-coroutine trap.
