# Analysis: cpp-design-principles/state-machines.md

Disposition: **PORTS_WITH_ADAPTATION** for the principles (when to reach
for an FSM, anti-patterns, deferred events) and **PYTHON_SPECIFIC_VARIANT**
for the implementation library (Boost.SML has no Python clone with
equivalent zero-cost properties; the Python options are different
trade-offs).

## Section-by-section

### When a state machine is the right tool / when not

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Reach for an FSM when behaviour depends on a graph
  of legal moves; skip for single-bit state, linear pipelines, pure
  data transformations.
- **Python rendering:** Identical. The "smoke test" (draw the lifecycle
  on paper before writing it) is language-independent.

### Implementing with Boost.SML

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** Boost.SML compiles a transition table into a
  zero-allocation branch-table dispatcher; the table is the source of
  truth, business logic lives in an actions struct, the `sml::testing`
  policy unlocks state injection.
- **Python rendering:** Three viable options. Recommend the third:

  | Option                     | Pros                                                                                  | Cons                                                                                  |
  |----------------------------|---------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
  | `transitions` library      | Mature, named transitions table, callbacks fire on entry/exit, supports nested machines. | Runtime overhead; states are strings (no type-checker help); the DSL grows quickly.   |
  | `python-statemachine`      | Decorator-flavoured; types states as classes; integrates with Django/SQLA.            | Heavier API surface; opinionated.                                                     |
  | **`match` + dataclass states** (hand-rolled) | Type-checker-friendly; no dependency; reads as a discriminated union; exhaustiveness via `assert_never`. | Loses entry/exit-action ceremony; the team writes the dispatcher by hand.            |

  Recommended Python pattern (the hand-rolled option):

  ```python
  from dataclasses import dataclass
  from typing import Literal, assert_never

  @dataclass(frozen=True, slots=True)
  class Closed: kind: Literal["closed"] = "closed"
  @dataclass(frozen=True, slots=True)
  class Connecting: kind: Literal["connecting"] = "connecting"
  @dataclass(frozen=True, slots=True)
  class Established: kind: Literal["established"] = "established"
  @dataclass(frozen=True, slots=True)
  class Closing: kind: Literal["closing"] = "closing"

  type SessionState = Closed | Connecting | Established | Closing

  @dataclass
  class SessionFSM:
      state: SessionState = Closed()
      on_async_connect: Callable[[], None] = lambda: None
      on_async_close: Callable[[], None] = lambda: None

      def handle_connect(self) -> None:
          match self.state:
              case Closed():
                  self.state = Connecting()
                  self.on_async_connect()
              case _:
                  return  # explicit no-op for illegal moves

      def handle_established(self) -> None:
          match self.state:
              case Connecting():
                  self.state = Established()
              case _:
                  return
      # ... etc.
  ```

  For more complex graphs (more than five states, multiple entry/exit
  actions, deferred events), reach for `transitions` and document the
  diagram alongside as `.puml` / `mermaid`.

### Embedded PlantUML

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. Embed a PlantUML or Mermaid block in
  the module docstring above the dispatch function; review the diagram
  and the code as one artifact.

### Deferred events

- **Classification:** PORTS_AS_IS for the principle; PORTS_WITH_ADAPTATION
  for the implementation.
- **C++ principle:** `sml::defer` (wait for next state change) and
  `sml::process` (post a follow-up after the current call unwinds).
- **Python rendering:**
  - In an async FSM, deferral is `await event.wait()` on a per-state
    `asyncio.Event`; follow-up is `asyncio.create_task(...)`.
  - In a sync FSM, deferral is a `collections.deque` of pending events
    drained after each transition (same shape as the C++ recipe).

### Testing a state machine

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Drive the FSM with crafted events, assert on
  the recorded action list, walk every edge. The recorder for actions
  is a plain list of dataclass instances:

  ```python
  @dataclass(frozen=True)
  class ActionAsyncConnect: pass
  @dataclass(frozen=True)
  class ActionAsyncClose: pass

  Action = ActionAsyncConnect | ActionAsyncClose

  @pytest.fixture
  def fsm():
      actions: list[Action] = []
      fsm = SessionFSM(
          on_async_connect=lambda: actions.append(ActionAsyncConnect()),
          on_async_close=lambda: actions.append(ActionAsyncClose()),
      )
      return fsm, actions

  def test_session_fsm_initial_state(fsm):
      sm, actions = fsm
      assert isinstance(sm.state, Closed)
      assert actions == []
  ```

### Anti-patterns

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same list; rephrase the "calling `process_event`
  from inside a transition" warning as "calling `handle_*` from inside
  an action": the Python analogue is the same hazard.

## Suggested Python file: `python-design-principles/state-machines.md`

A largely rewritten chapter: principles port, but the implementation
example becomes the hand-rolled `match` pattern (with `transitions` as
the fallback for larger machines). The PlantUML / Mermaid diagram
embedding rule ports verbatim.
