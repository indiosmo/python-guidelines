# Invariants, transactions, and rollback

Overall disposition: PORTS_AS_IS.

The exception-safety vocabulary is C++-specific, but the invariant discipline
ports directly. Python mutating methods still need all-or-nothing behavior,
and retries still require idempotency.

## Strategy 1: commit at the end

Disposition: PORTS_AS_IS.

Keep the section. In Python, build replacement state in locals or copies, then
assign once after all fallible steps succeed.

```python
from dataclasses import replace


def add_subscription(state: SubscriptionSet, topic: TopicId, subscriber: Subscriber) -> SubscriptionSet:
    subscriber.connect()

    by_topic = state.by_topic | {topic: subscriber}
    pending = state.pending | {topic}
    return replace(state, by_topic=by_topic, pending=pending)
```

For mutable classes:

```python
def add(self, topic: TopicId, subscriber: Subscriber) -> None:
    subscriber.connect()

    next_by_topic = dict(self._by_topic)
    next_pending = set(self._pending)
    next_by_topic[topic] = subscriber
    next_pending.add(topic)

    self._by_topic = next_by_topic
    self._pending = next_pending
```

## Strategy 2: scope-guard rollback

Disposition: PORTS_WITH_ADAPTATION.

The C++ scope guard becomes a context manager or `contextlib.ExitStack`.

```python
from contextlib import ExitStack


def add_request(self, request: Request) -> None:
    with ExitStack() as rollback:
        self._orders[request.order_id] = build_order_state(request)
        rollback.callback(self._orders.pop, request.order_id, None)

        self._requests[request.request_id] = build_request_state(request)
        rollback.callback(self._requests.pop, request.request_id, None)

        self._limits.reserve(request)
        rollback.pop_all()
```

This is one of the best Python renderings because `ExitStack` gives a standard
library rollback stack with LIFO behavior.

## Commit-at-end vs scope guards

Disposition: PORTS_AS_IS.

Keep the decision rule. Commit-at-end is the default. Use rollback when a
later fallible step must observe earlier mutation or when the mutation touches
external state.

## Idempotency

Disposition: PORTS_AS_IS.

Keep the section. The Python guide should emphasize queues, task retries,
HTTP retries, and message redelivery. Record what was reserved, sent, or
written, then reverse that exact amount.

```python
def on_reject(self, request_id: RequestId) -> None:
    request = self._requests.pop(request_id)
    self._limits.release(
        quantity=request.rollback_quantity,
        notional=request.rollback_notional,
    )
```

For check-then-act, use atomic database constraints, locks, or single-threaded
ownership. In-memory `if key not in dict: dict[key] = value` is not safe
across threads or processes.

```python
order = self._orders.setdefault(order_id, build_order(order_id))
```

Use `setdefault` only when constructing the fallback has no side effect. If
construction is expensive or effectful, keep the operation under the state
owner's lock or event loop.

## Invariants belong on the type that owns them

Disposition: PORTS_AS_IS.

Keep it. Python should use private attributes by convention, frozen
dataclasses for immutable values, public methods for valid transitions, and
module-local helpers where needed.

```python
@dataclass
class RequestBook:
    _by_order_id: dict[OrderId, Request] = field(default_factory=dict)

    def add(self, request: Request) -> None:
        if request.order_id in self._by_order_id:
            raise DuplicateOrder(request.order_id)
        self._by_order_id[request.order_id] = request
```

The guide should not pretend privacy is enforced like C++. The rule is still
valuable because tests and callers should use invariant-preserving methods
unless they are deliberately probing internals.
