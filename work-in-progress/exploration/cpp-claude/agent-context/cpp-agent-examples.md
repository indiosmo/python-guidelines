# Analysis: agent/cpp-agent-examples.md -> Python equivalent

The C++ examples file is the **on-demand companion** to the context
file: good/bad code pairs keyed to the context sections. The Python
equivalent should follow the same shape: one block per context-file
section, each block showing a good Python rendering and a bad
counterpart that the rule warns against.

The list below indicates, per C++ example section, the disposition and
sketch of the Python good/bad pair. Full code rendering belongs in the
final examples file; this analysis just shows the shape and the open
choices.

## Per-section sketches

### Tests

- **Good** (Python):
  ```python
  def test_rectangle_area():
      r = Rectangle(width=6, height=4)
      assert r.area() == 24  # 6 * 4, from the definition of area
  ```
- **Bad** (Python -- expected value derived from implementation):
  ```python
  def test_rectangle_area():
      r = Rectangle(width=6, height=4)
      assert r.area() == r.width * r.height
  ```

### Debugging

- Same text blocks port verbatim; no code.

### Domain Ownership

- **Good:**
  ```python
  # order_routing/types.py
  from typing import NewType
  from enum import Enum

  UserId = NewType("UserId", int)
  UserOrderId = NewType("UserOrderId", int)
  Symbol = NewType("Symbol", str)
  Quantity = NewType("Quantity", int)

  class Side(Enum):
      BUY = "buy"
      SELL = "sell"

  # order_routing/__init__.py
  from dataclasses import dataclass
  from . import types

  @dataclass(frozen=True, kw_only=True, slots=True)
  class NewOrder:
      user: types.UserId
      order_id: types.UserOrderId
      instrument: types.Symbol
      order_side: types.Side
      order_quantity: types.Quantity
  ```
- **Bad** (raw primitives leaking into the domain):
  ```python
  @dataclass
  class NewOrder:
      user: int      # which kind of id?
      order_id: int
      instrument: str
      order_side: str
      order_quantity: int
  ```

- **Bad** (cross-domain construction without an adapter / helper):
  - direct: `risk.types.OrderId(routing.types.OrderId(value))` -- OK
    when the adapter sits at the boundary.
  - bad: a helper `def to_risk_order_id(id): return RiskOrderId(id)`
    that adds no domain knowledge.

### Forward Dependencies

- **Good:**
  ```python
  def untie(data: Table) -> Column: ...
  def rank(data: Table, tiebreaker: Column) -> Table: ...

  def score(data: Table) -> Table:
      return rank(data, untie(data))
  ```
- **Bad:** mutating a `WorkingSet` dataclass where `untie` writes a
  field that `rank` later reads (the C++ residue antipattern).

### Types Carry Proof

- **Good:**
  ```python
  def parse_symbol(raw: str) -> Symbol: ...  # raises InvalidSymbol

  def place_order(account: AccountId, symbol: Symbol, qty: Quantity) -> None: ...
  ```
- **Bad:**
  ```python
  def validate_symbol(raw: str) -> bool: ...
  def place_order(account: str, symbol: str, qty: int) -> None: ...
  ```

### Strong-Type Ergonomics

- The C++ `.get()` / `operator->` discussion has no Python analogue.
  Replace with the Python version: "`NewType` is identity; use the
  wrapped value directly (`OrderId("X")`). For dataclass-wrapped
  strong types, access the field through the dataclass attribute (no
  `.get()` call -- the field has a name)."

- **Good:**
  ```python
  remaining_quantity = OrderQty(remaining_quantity - fill_quantity)  # identity wrapper
  log.info("order %s", order_id)  # str-based NewType formats directly
  ```
- **Bad:**
  ```python
  remaining_quantity = OrderQty(int(remaining_quantity) - int(fill_quantity))
  log.info("order %s", str(order_id))  # NewType doesn't need str()
  ```

### Designated Initializers

- **Good:**
  ```python
  config = ServerConfig(
      listen_address="0.0.0.0",
      listen_port=8080,
      reuse_address=True,
      backlog=1024,
  )
  ```
- **Bad** (positional construction):
  ```python
  config = ServerConfig("0.0.0.0", 8080, True, 1024)
  ```

### Functional Core, Imperative Shell

- **Good:** pure function returning a new state.
  ```python
  def apply_fill(order: OrderState, fill: Fill) -> OrderState: ...
  ```
- **Bad:** I/O inside the domain function.
  ```python
  def apply_fill(order: OrderState, fill: Fill) -> None:
      log.info("fill %s", fill.id)
      background_executor.submit(persist, order)
  ```

### Pipelines

- **Good:** stage with `on_<event>` callable fields, wiring elsewhere.
- **Bad:** stage knows its consumer (`self.engine = engine`).

### Runtime And Threads

- **Good:** capture by value into the posted closure.
  ```python
  loop.call_soon_threadsafe(functools.partial(publisher.send, event))
  ```
- **Bad:** post a closure that captures a soon-to-be-dead reference.

### Error Handling

- **Good:**
  ```python
  class RoutingError(Exception): ...

  @dataclass(frozen=True, slots=True)
  class UnknownOrder(RoutingError):
      order_id: OrderId
      def __post_init__(self):
          super().__init__(f"unknown order {self.order_id}")

  def route(request: NewOrder) -> None:
      try:
          sink = select_sink(request.route)
          orders.process(request)
          on_routed(RoutedOrder(sink=sink, request=request))
      except UnknownRoute:
          raise
      except SinkError as exc:
          raise RoutingError("sink failed") from exc
  ```
- **Bad:** bool returns + side-channel logging in place of exceptions.

### Invariants And Rollback

- **Good:**
  ```python
  with ExitStack() as stack:
      orders[req.order_id] = build_order_state(req)
      stack.callback(orders.pop, req.order_id, None)
      requests[req.request_id] = build_request_state(req)
      stack.callback(requests.pop, req.request_id, None)
      limits.reserve(req)  # may raise
      stack.pop_all()  # success: cancel rollbacks
  ```

### Declarative Style

- **Good:**
  ```python
  has_residual = order.remaining_quantity > 0
  is_market = order.limit_price == 0
  should_rest = has_residual and not is_market

  if should_rest:
      rest(order)
  ```
- **Bad:** inline composite condition.

### Variants, Protocols, And Generics

- **Good:** `match` over discriminated union + `assert_never`.
- **Bad:** `if isinstance(x, A): ... elif isinstance(x, B): ... else: raise`.

### State Machines

- **Good:** match over dataclass-state union.
- **Bad:** correlated booleans.

### Cross-Cutting Services

- **Good:** `contextvars`-backed module singleton + free `log()` function.
- **Bad:** thread a logger through every constructor.

### Performance Discipline

- **Good:** numpy-vectorised loop; `__slots__` on the per-message
  dataclass.
- **Bad:** Python `for`-loop over a million rows; class without
  `__slots__` used as the per-message struct.

### Comments

- **Good:** block comment naming the step.
- **Bad:** comment narrating the syntax; past-tense / delta-framed /
  non-responsibility comments (the negative-documentation antipattern).

### Imports / Namespace Aliases

- **Good:** absolute imports, no implicit relative imports.
  ```python
  from order_routing import types as routing_types
  from matching_engine import types as me_types
  ```
- **Bad:**
  ```python
  from ..order_routing.types import *
  ```

### Final and Read-Only (replaces "Const Placement")

- **Good:**
  ```python
  MAX_RETRIES: Final = 5
  ```
- **Bad:**
  ```python
  MAX_RETRIES = 5  # plain int; type checker can't prevent mutation
  ```

## Suggested Python file: `python-agent-examples.md`

Same structure as the C++ examples file, with section headings matching
the context file 1:1. Each section has at least one good/bad pair. The
file is loaded on demand by the agent when "the rule alone is not
enough to shape an edit".
