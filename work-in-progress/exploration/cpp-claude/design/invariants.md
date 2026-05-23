# Analysis: cpp-design-principles/invariants.md

Disposition: **PORTS_WITH_ADAPTATION** -- the principles port, the
implementation idioms swap (exceptions instead of `BOOST_LEAF_CHECK`,
`contextlib.ExitStack` / `contextmanager` instead of `lib::scope_exit`).

## Section-by-section

### Exception-safety taxonomy (basic / strong / no-throw)

- **Classification:** PORTS_AS_IS for the vocabulary; PORTS_WITH_ADAPTATION
  for the concrete shape.
- **C++ principle:** Mutating methods with non-trivial invariants target
  the strong exception guarantee.
- **Python rendering:** Keep the same three-tier vocabulary. The Python
  equivalent of "no-throw" is "this function does not raise"; document
  it in the docstring and rely on the type checker to verify (via
  `assert_never` in unreachable arms). Python has no `noexcept`, so the
  contract is conventional.

### Strategy 1: commit at the end

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Build the new state in locals; only mutate the
  object once everything fallible has succeeded.
- **Python rendering:** Same. The `subscription_set.add` example:

  ```python
  from copy import copy
  from dataclasses import dataclass, field

  @dataclass
  class SubscriptionSet:
      by_topic: dict[TopicId, Subscriber] = field(default_factory=dict)
      pending: set[TopicId] = field(default_factory=set)

      def add(self, topic_id: TopicId, subscriber: Subscriber) -> None:
          # fallible work first; if it raises, state is untouched.
          subscriber.connect()
          self.by_topic[topic_id] = subscriber
          self.pending.add(topic_id)
  ```

  For deeper "build new copy, swap" patterns, `dataclasses.replace(...)`
  on a frozen dataclass is the idiomatic spelling.

### Strategy 2: scope-guard rollback

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Pair each mutation with a `lib::scope_exit` that
  undoes it; `dismiss()` only on the success path.
- **Python rendering:** `contextlib.ExitStack` plus
  `stack.callback(undo_fn)` is the direct equivalent:

  ```python
  from contextlib import ExitStack

  def add_request(self, request: Request) -> None:
      with ExitStack() as stack:
          self._orders[request.order_id] = build_order_state(request)
          stack.callback(self._orders.pop, request.order_id, None)

          self._requests[request.request_id] = build_request_state(request)
          stack.callback(self._requests.pop, request.request_id, None)

          self._limits.reserve(request)  # may raise

          # success: cancel the rollback callbacks
          stack.pop_all()
  ```

  The `stack.pop_all()` call is the equivalent of dismissing every guard
  at once. On any exception, every registered callback fires in reverse
  order before the exception propagates.

  For two-step composed helpers, return an `ExitStack` (or a context
  manager that holds it) from the helper and have the caller `enter_context`
  it.
- **Open questions:** ExitStack is the right tool, but the spelling is
  not as light as `lib::scope_exit` lambdas. Document the pattern with a
  small helper `def rollback(stack, fn, *args)` if the team finds the
  raw `stack.callback` ugly.

### Idempotency: pre-compute the rollback amount

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Record exactly what you reserved at admission time;
  rollback applies that, not a recomputed value.
- **Python rendering:** Identical; store the `rollback_quantity` field on
  the request dataclass at admission.

### Idempotency: check-then-act, atomically

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Use a single primitive that folds "absent ->
  insert" into one indivisible step.
- **Python rendering:** Python's dict has `setdefault` for this:

  ```python
  if self._orders.setdefault(order_id, build_order(...)) is build_order(...):
      ...  # newly inserted
  else:
      ...  # already present
  ```

  The cleaner spelling uses `.get(...)` + an explicit branch, or
  `collections.defaultdict`. For thread-safe atomicity, `dict.setdefault`
  is atomic in CPython under the GIL; under free-threaded 3.13+ this no
  longer holds and a lock is required (call this out explicitly in the
  guide).
- **Open questions:** Free-threaded 3.13t/3.14t changes the
  GIL-derived atomicity assumptions. Recommend documenting both modes.

### Invariants belong on the type that owns them

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Keep invariant-maintaining methods on the class with
  the fields they protect private.
- **Python rendering:** Same. Python's "private" is convention
  (underscore prefix); enforce with pyright/pyrefly's
  `reportPrivateUsage`. For genuinely-locked-down types, frozen
  dataclasses + properties + a parse-once factory enforce the same
  property at runtime.

## Suggested Python file: `python-design-principles/invariants.md`

Verbatim port with the `BOOST_LEAF_CHECK` -> `raise` and `lib::scope_exit`
-> `ExitStack.callback` substitutions, and a free-threaded-Python
sidebar on `dict.setdefault` atomicity.
