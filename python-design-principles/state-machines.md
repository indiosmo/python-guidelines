# State Machines

A state machine makes legal state transitions explicit. It earns its place when
behavior depends on a lifecycle: connection protocols, retries, approvals,
orders, imports, background jobs, or workflows with entry and exit actions.

Use the smallest representation that makes the graph clear. A few states can
be an enum plus `match`. State that carries data can be a union of dataclasses.
Large graphs with guards, callbacks, diagrams, and tooling can use a library
such as `transitions`.

## When It Fits

Use a state machine when several of these are true:

- The legal next action depends on the current state and the event sequence.
- Transitions have side effects such as scheduling, cancellation, notification,
  or persistence.
- Illegal transitions carry real operational or domain risk.
- Several booleans or mode flags must move in lockstep.
- Reviewers need to read the lifecycle as a graph.

Use a plain function, strategy object, or pipeline when the work is a linear
parse-validate-write sequence or a pure transformation.

## Enum And Match

For small machines, an enum and an explicit transition function are enough.

```python
from enum import StrEnum
from typing import assert_never


class SessionState(StrEnum):
    CLOSED = "closed"
    CONNECTING = "connecting"
    ESTABLISHED = "established"
    CLOSING = "closing"


class SessionEvent(StrEnum):
    CONNECT = "connect"
    ESTABLISHED = "established"
    CLOSE = "close"
    CLOSED = "closed"


def next_session_state(state: SessionState, event: SessionEvent) -> SessionState:
    match state, event:
        case SessionState.CLOSED, SessionEvent.CONNECT:
            return SessionState.CONNECTING
        case SessionState.CONNECTING, SessionEvent.ESTABLISHED:
            return SessionState.ESTABLISHED
        case SessionState.CONNECTING | SessionState.ESTABLISHED, SessionEvent.CLOSE:
            return SessionState.CLOSING
        case SessionState.CLOSING, SessionEvent.CLOSED:
            return SessionState.CLOSED
        case _, _:
            raise InvalidTransition(state=state, event=event)
```

The transition function is pure and easy to table-test. The owning session
performs side effects around successful transitions.

## State Classes

Use dataclass variants when each state carries different data. This removes
optional fields and invalid field combinations from the public shape.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Literal, assert_never


@dataclass(frozen=True, slots=True)
class WaitingForApproval:
    kind: Literal["waiting_for_approval"]
    submitted_at: datetime


@dataclass(frozen=True, slots=True)
class Approved:
    kind: Literal["approved"]
    approved_at: datetime
    approver: str


@dataclass(frozen=True, slots=True)
class Rejected:
    kind: Literal["rejected"]
    rejected_at: datetime
    reason: str


type ApprovalState = WaitingForApproval | Approved | Rejected
```

Consume the union with `match` when an operation covers every state. Use
`assert_never` in the unreachable arm for checker-backed coverage.

## Transition Tables

A transition table makes the graph reviewable when states and events grow:

```python
type TransitionKey = tuple[SessionState, SessionEvent]


TRANSITIONS: dict[TransitionKey, SessionState] = {
    (SessionState.CLOSED, SessionEvent.CONNECT): SessionState.CONNECTING,
    (SessionState.CONNECTING, SessionEvent.ESTABLISHED): SessionState.ESTABLISHED,
    (SessionState.CONNECTING, SessionEvent.CLOSE): SessionState.CLOSING,
    (SessionState.ESTABLISHED, SessionEvent.CLOSE): SessionState.CLOSING,
    (SessionState.CLOSING, SessionEvent.CLOSED): SessionState.CLOSED,
}


def transition(state: SessionState, event: SessionEvent) -> SessionState:
    try:
        return TRANSITIONS[(state, event)]
    except KeyError as exc:
        raise InvalidTransition(state=state, event=event) from exc
```

Keep side effects out of the table unless they are tiny named actions. The
owner class can call the pure transition function, then run the action that
belongs to the edge.

## Owner Boundary

The state machine should sequence states and actions. The owner owns I/O,
storage, timers, locks, and framework callbacks.

```python
class Session:
    def __init__(self, transport: Transport) -> None:
        self._transport = transport
        self._state = SessionState.CLOSED

    def connect(self) -> None:
        self._move(SessionEvent.CONNECT)
        self._transport.open()

    def on_established(self) -> None:
        self._move(SessionEvent.ESTABLISHED)

    def close(self) -> None:
        self._move(SessionEvent.CLOSE)
        self._transport.close()

    def _move(self, event: SessionEvent) -> None:
        self._state = transition(self._state, event)
```

This keeps tests for the graph separate from tests for transport behavior.

## Deferred Events

Re-entrant events should be queued and drained from the outermost call. This
prevents a transition from recursively entering the same owner while it is
still mutating state.

```python
from collections import deque
from collections.abc import Callable


class EventPump:
    def __init__(self) -> None:
        self._pending: deque[Callable[[], None]] = deque()
        self._draining = False

    def defer(self, action: Callable[[], None]) -> None:
        self._pending.append(action)
        if self._draining:
            return

        self._draining = True
        try:
            while self._pending:
                self._pending.popleft()()
        finally:
            self._draining = False
```

Use event-loop scheduling, queues, or task groups for runtime-level deferral.
Use a local pending queue for synchronous state-machine re-entrance.

## Library State Machines

Use a library such as `transitions` when the graph has enough operational
meaning to justify tooling: many states, guard conditions, generated diagrams,
entry and exit callbacks, persistence, or runtime inspection. Keep the model's
actions small and delegate business work to named functions or services.

When the graph is part of an external protocol, keep a diagram or transition
table close to the implementation and test every edge that matters.

## Tests

State-machine tests should cover:

- Every legal edge.
- Representative illegal edges.
- Guards that accept and reject.
- Entry and exit actions.
- Deferred events and ordering.
- Persistence or recovery from each durable state.

Prefer table-driven tests for pure transition functions. Test the owner
separately for side effects and integration behavior.

## Review Checks

- The lifecycle is a graph rather than a linear sequence.
- State representation prevents invalid field combinations where practical.
- Transition logic is visible in `match`, a table, or library configuration.
- Side effects live in named owner actions.
- Re-entrant events are deferred intentionally.
- Tests cover edges, guard behavior, and illegal transitions.
