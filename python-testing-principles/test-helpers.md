# Test Helpers

Test helpers keep scenario-specific facts visible. A helper earns its place
when it provides stable defaults, composes meaningful presets, derives
related objects, isolates process state, or gives tests a typed fake for a
real collaborator.

Prefer helpers that speak the domain vocabulary. Names such as
`make_order_request`, `approved_shipment`, `triage_encounter`,
`weighted_signals`, and `portfolio_definition` tell the reader what the test
needs.

## Domain Factories

Use factories for objects with many fields or invariants. Give every field a
stable default and make overrides keyword-only.

```python
def make_order_request(
    *,
    account_id: AccountId = AccountId("ACC-001"),
    symbol: str = "PETR4",
    side: OrderSide = OrderSide.BUY,
    quantity: int = 100,
    limit_price: Decimal = Decimal("28.10"),
) -> OrderRequest:
    return OrderRequest(
        account_id=account_id,
        symbol=symbol,
        side=side,
        quantity=quantity,
        limit_price=limit_price,
    )
```

Tests then vary one field and leave unrelated setup out of sight:

```python
def test_rejects_negative_quantity() -> None:
    request = make_order_request(quantity=-1)

    with pytest.raises(ValueError, match="quantity must be positive"):
        validate_order_request(request)
```

A factory that only forwards every argument to a constructor adds ceremony.
Use the constructor directly when the object is small and the call site reads
clearly.

## Presets And Derived Objects

Use presets when several tests share a named state.

```python
ACTIVE_ACCOUNT = AccountPreset(status=AccountStatus.ACTIVE, cash=Decimal("10000.00"))
SUSPENDED_ACCOUNT = AccountPreset(status=AccountStatus.SUSPENDED, cash=Decimal("10000.00"))


def make_account(*, preset: AccountPreset = ACTIVE_ACCOUNT, cash: Decimal | None = None) -> Account:
    return Account(status=preset.status, cash=preset.cash if cash is None else cash)
```

When one object derives from another, accept the source object so related
fields stay coherent.

```python
def make_fill(order: OrderRequest, *, quantity: int | None = None) -> Fill:
    return Fill(
        account_id=order.account_id,
        symbol=order.symbol,
        side=order.side,
        quantity=order.quantity if quantity is None else quantity,
        price=order.limit_price,
    )
```

This prevents tests from accidentally combining unrelated account IDs,
symbols, dates, or units.

## Fixture Factories

Fixture factories work well when a test needs several related instances and
the fixture owns cleanup or shared state.

```python
@pytest.fixture
def account_factory() -> Callable[..., Account]:
    created: list[Account] = []

    def create_account(**overrides: object) -> Account:
        account = make_account(**overrides)
        created.append(account)
        return account

    return create_account
```

Keep the inner factory typed when it is reused across files. A small protocol
or dataclass can make override shapes clearer than `dict[str, object]` once
the factory grows.

## Dataframe Builders

Dataframe tests need builders that construct valid frames from domain-shaped
inputs. Put column names, dtypes, index conventions, and ordering in one
builder or schema-aware constructor.

```python
def make_weighted_signals(rows: Iterable[WeightedSignalRow]) -> pd.DataFrame:
    frame = pd.DataFrame([asdict(row) for row in rows])
    return (
        WeightedSignals.validate(frame)
        .sort_values(["trade_date", "symbol"])
        .reset_index(drop=True)
    )
```

Use schema-aware empty constructors for empty frames:

```python
def empty_weighted_signals() -> pd.DataFrame:
    return WeightedSignals.empty()
```

A cast in a reusable helper hides schema drift. Builders and schemas make the
shape executable.

## Fakes And Protocols

Use fakes for collaborators whose behavior matters to the test. Type the fake
against the same protocol the production code consumes.

```python
class Clock(Protocol):
    def now(self) -> datetime: ...


@dataclass(slots=True)
class FrozenClock:
    current_time: datetime

    def now(self) -> datetime:
        return self.current_time
```

Prefer fakes over deep mock chains for domain collaborators. Use autospecced
mocks for third-party objects when a mock is the clearest tool, so signature
drift becomes visible.

```python
client = create_autospec(StorageClient, instance=True)
```

## Context-Local Providers

When production code uses context-local providers for clock, logger, settings,
or tracing, tests should override them with a context manager or fixture that
restores the previous token.

```python
@pytest.fixture
def frozen_time() -> Iterator[None]:
    token = current_clock.set(FrozenClock(datetime(2026, 1, 2, tzinfo=UTC)))
    try:
        yield
    finally:
        current_clock.reset(token)
```

This keeps process-global state from leaking into later tests.

## Integration Probes

Some tests need a small observation point for hygiene: queued messages,
registered callbacks, cache contents, or rollback state. Add a deliberate
test-facing probe near the component when the public workflow cannot express
the observation cleanly.

Keep probes narrow and named for the invariant they observe:

```python
def assert_registry_empty(registry: Registry) -> None:
    assert registry.snapshot() == {}
```

Use probes to observe state after driving the public behavior. Keep normal
behavior tests on returned values, emitted events, stored records, or public
errors.

## Helper Placement

Put helpers close to their domain:

- File-local helpers for one test file.
- Package-level test utility modules for repeated domain builders.
- Shared fixtures in `conftest.py` when many tests in that subtree use them.
- Project-level helper packages only when several packages share the same
  testing vocabulary.

Keep `conftest.py` focused. A large root `conftest.py` makes test behavior
implicit across the whole suite.
