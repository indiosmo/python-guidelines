# Comments

Overall disposition: PORTS_AS_IS.

The comment guidance ports directly. Python adds docstrings, but the same rule
holds: comments describe present intent and non-obvious reasons near the code.
They do not narrate syntax, preserve migration history, or list absent
responsibilities.

## Guiding comments

Disposition: PORTS_AS_IS.

Keep it. Python rendering:

```python
# Reject duplicates and resolve the destination book up front.
check_duplicate(incoming_key)
book = find_book(request.instrument)

# Ack precedes trades so receivers see the order id before fills on it.
emit(OrderAck(user=request.user, order_id=request.order_id))
```

## Non-obvious lines

Disposition: PORTS_AS_IS.

Keep it. Good Python examples include lock preconditions, pandas mutation
rules, iterator exhaustion, protocol ordering, performance-sensitive
shortcuts, and lifetime of async tasks.

```python
# precondition: session lock is held; next_sequence is incremented without locking here.
sequence = self._next_sequence
self._next_sequence += 1
```

## Blocks and flow

Disposition: PORTS_AS_IS.

Keep it. A long Python function still benefits from phase comments when
extraction would obscure the local algorithm.

```python
# Rest the residual: allocate a node, link it into the book, and register it for cancels.
node = self._allocate_node(order)
book.place(node)
self._resting_index[incoming_key] = node
```

## What not to comment

Disposition: PORTS_AS_IS.

Keep the anti-patterns. Python-specific bad comments:

```python
# Increment the counter.
count += 1

# This used to call the upstream deduper.
process(order)

# No database access happens here.
return score(order)
```

The last example is negative documentation. A contributor rule such as
"domain modules must not import SQLAlchemy" is fine when it constrains future
changes.

## Shape

Disposition: PORTS_WITH_ADAPTATION.

Use docstrings for modules, public classes, public functions, and complex test
fixtures when they explain contract or intent. Use inline comments for local
flow. Avoid docstrings that restate the signature.

```python
def parse_symbol(raw: str) -> Symbol:
    """Return a domain symbol parsed from a wire value."""
    ...
```

Prefer no docstring over a useless one:

```python
def total(self) -> Money:
    return self.quantity * self.price
```
