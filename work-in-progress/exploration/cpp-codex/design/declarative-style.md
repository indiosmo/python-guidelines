# Declarative style

Overall disposition: PORTS_AS_IS.

The whole chapter ports. Python has different tools: comprehensions, generator
expressions, `itertools`, `sorted(key=...)`, `min(key=...)`, `any`, `all`, and
small named functions. The principle is the same: describe the transformation
and name the domain steps.

## Decompose

Disposition: PORTS_AS_IS.

Keep the rule. Python makes extraction cheap, so the guide should push small
helpers that take values and return values.

```python
from statistics import fmean


def variance(samples: list[float], mean: float) -> float:
    return fmean((sample - mean) ** 2 for sample in samples)


def compute_stats(samples: list[float]) -> Stats:
    mean = fmean(samples)
    return Stats(mean=mean, variance=variance(samples, mean))
```

For side effects, split pure work from commit:

```python
def paint(widget: Widget, bounds: Rect, canvas: Canvas) -> None:
    canvas.fill(bounds, widget.background)
    canvas.stroke(bounds, widget.border)


def render(widget: Widget, canvas: Canvas) -> None:
    paint(widget, compute_rect(widget), canvas)
    canvas.flush()
```

## Work on simpler types

Disposition: PORTS_AS_IS.

Keep the section. Python code frequently passes whole ORM models, request
objects, pandas frames, or settings objects into helpers that need one field.
Prefer the smaller value.

```python
def ack_ratio(bytes_sent: int, bytes_acked: int) -> float:
    if bytes_sent == 0:
        return 0.0
    return bytes_acked / bytes_sent
```

For projections, use `key`:

```python
lowest = min(points, key=lambda point: point.y)
```

Name the key function when it carries domain meaning.

## Stage variables upfront

Disposition: PORTS_AS_IS.

Keep it. Python readability benefits from separating derived facts from
assembly.

```python
pages = math.ceil(total / query.page_size)

return PageResponse(
    total=total,
    page=query.page,
    size=query.page_size,
    pages=pages,
    has_next=query.page < pages,
    has_prev=query.page > 1,
)
```

For longer transformations, avoid rebinding one name through several meanings.

```python
positioned = calculate_positions(delta_time, shapes)
grid = fill_grid(positioned, width, height)
velocities = solve_collisions(grid)
return apply_velocities(positioned, velocities)
```

## Named predicates

Disposition: PORTS_AS_IS.

Keep it. Inline lambdas are fine for one obvious projection; named predicates
are better for domain conditions.

```python
def is_active_user(user: User, now: datetime) -> bool:
    return user.deleted_at is None and user.last_login > now - timedelta(days=30)


active_count = sum(1 for user in users if is_active_user(user, now))
```

Predicate factories port directly:

```python
def greater_than(threshold: int) -> Callable[[int], bool]:
    return lambda value: value > threshold
```

## Use ranges and algorithms

Disposition: PORTS_WITH_ADAPTATION.

The principle ports, but Python names differ. Prefer `any`, `all`, `next`,
`sum`, `min`, `max`, `sorted`, `itertools`, and comprehensions over manual
flag loops.

```python
if any(order.expiry < now for order in orders):
    return RejectReason.EXPIRED_ORDER_PRESENT
```

First match:

```python
missing = next(
    (field for field in fields if field.required and field.value is None),
    None,
)
if missing is None:
    return None
return f"field {missing.name} is missing"
```

## Compose lazily; materialize once

Disposition: PORTS_AS_IS.

Keep the cost model, with Python-specific wording. Generators are lazy;
lists, sets, and tuples materialize. Materialize when the result must be
stored, iterated repeatedly, sorted, indexed, or passed to an API that needs a
container.

```python
rows = (
    to_csv_row(order)
    for order in orders
    if order.side is Side.BUY
)

for row in itertools.islice(rows, 10):
    writer.write(row)
```

If a pipeline will be traversed twice, materialize intentionally:

```python
active_ids = [user.id for user in users if is_active(user)]
```

Warn that Python generator expressions are single-use. This is the main
Python-specific hazard absent from the C++ view discussion.
