# Cross-cutting services

Overall disposition: PORTS_WITH_ADAPTATION.

The problem ports; the C++ variant-backed global does not. Python should keep
the edge-selection idea but prefer explicit composition, provider modules,
`ContextVar`, and pytest fixtures over broad mutable globals.

## The problem

Disposition: PORTS_AS_IS.

Keep it. Clocks, loggers, timers, metrics, configuration, tracing, and feature
flags cut across layers. Threading them through every function can distort
signatures; hiding them as arbitrary globals can destroy test isolation.

## Constructor injection through every layer

Disposition: PORTS_AS_IS.

Keep the critique. Passing a dependency through layers that do not use it lies
about the layer's contract.

## Polymorphic singleton

Disposition: PORTS_WITH_ADAPTATION.

Rewrite. Python singleton modules are easy and dangerous. They avoid
constructor plumbing, but they create test leaks, import-time configuration,
and hidden dependencies.

## The pattern

Disposition: PYTHON_SPECIFIC_VARIANT.

Recommended Python pattern:

- Configure cross-cutting services at the application edge.
- Use explicit parameters for domain dependencies.
- Use module-level provider functions only for genuinely cross-cutting
  services.
- Use `ContextVar` for request-scoped context.
- Use context managers or pytest fixtures to restore provider state in tests.

```python
_clock: Clock = SystemClock()


def configure_clock(clock: Clock) -> None:
    global _clock
    _clock = clock


def now() -> datetime:
    return _clock.now()
```

For request context:

```python
request_id_var: ContextVar[RequestId | None] = ContextVar("request_id", default=None)
```

## Why a variant, not an inline template variable

Disposition: DROP.

No Python analogue.

## Trade-offs

Disposition: PORTS_WITH_ADAPTATION.

Keep the trade-offs but rewrite:

- Module-level providers are easy to overuse.
- Tests must restore global providers.
- Import-time configuration is fragile.
- `ContextVar` works for async context but is not a substitute for explicit
  domain parameters.

## Test substitution

Disposition: PORTS_WITH_ADAPTATION.

Use context managers or pytest fixtures.

```python
@contextmanager
def installed_clock(clock: Clock) -> Iterator[Clock]:
    previous = services.clock
    services.clock = clock
    try:
        yield clock
    finally:
        services.clock = previous
```

pytest fixture:

```python
@pytest.fixture
def manual_clock(monkeypatch: pytest.MonkeyPatch) -> ManualClock:
    clock = ManualClock()
    monkeypatch.setattr(services, "clock", clock)
    return clock
```

## A small catalog

Disposition: PORTS_WITH_ADAPTATION.

Keep as examples, but use Python facilities:

- Logger: stdlib `logging` configured at the edge, or a chosen structured
  logging package by project policy.
- Timer: `asyncio` loop scheduling, test manual clock, or scheduler wrapper.
- Clock: protocol with `now()`, real implementation, manual implementation.

## The match helper

Disposition: DROP.

No Python helper needed. Use protocol dispatch, plain method calls, or pattern
matching when matching is the domain operation.
