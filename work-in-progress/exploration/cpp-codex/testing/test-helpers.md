# Test helpers

Overall disposition: PORTS_WITH_ADAPTATION.

The helper philosophy ports: keep test bodies focused on behavior, not
construction noise. Python gives stronger fixture and data-construction tools,
so the implementation should change.

## Factory functions

Disposition: PORTS_WITH_ADAPTATION.

Use small factories, dataclass defaults, and `dataclasses.replace`.

```python
def make_http_request(**overrides: object) -> HttpRequest:
    request = HttpRequest(
        method="POST",
        scheme="https",
        host="api.example.com",
        path="/v1/items",
        headers={"content-type": "application/json"},
        body="{}",
        timeout=5.0,
    )
    return replace(request, **overrides)
```

For many optional overrides, a typed params dataclass can be clearer than
`**overrides`.

## When not to create a factory

Disposition: PORTS_AS_IS.

Keep it. A Python helper that only forwards arguments to a dataclass
constructor adds noise.

```python
endpoint = Endpoint(scheme="https", host="example.com", port=443)
```

## File-local wrappers

Disposition: PORTS_AS_IS.

Keep it. Define local helpers in the test module when a small customization is
used only there.

## Test data

Disposition: PORTS_AS_IS.

Keep deterministic, named, boundary-aware data. Add `pytest.param(..., id=...)`
for row labels.

```python
@pytest.mark.parametrize(
    ("payload_size", "expected"),
    [
        pytest.param(1023, True, id="below limit"),
        pytest.param(1024, True, id="at limit"),
        pytest.param(1025, False, id="above limit"),
    ],
)
def test_payload_limit(payload_size: int, expected: bool) -> None:
    assert accepts_payload(payload_size) is expected
```

## Fixtures

Disposition: PORTS_WITH_ADAPTATION.

Use pytest fixtures for setup and teardown. Prefer function scope unless a
broader scope is necessary and safe.

```python
@pytest.fixture
def library() -> Iterator[Library]:
    library = Library(settings=TestSettings())
    library.start()
    try:
        yield library
    finally:
        library.stop()
```

## Test probes

Disposition: PORTS_WITH_ADAPTATION.

Python does not need friend declarations, but the intent ports. Use explicit
test-only helpers, module-private attributes by convention, or probe classes
when the test checks internal hygiene.

```python
class CacheProbe:
    def __init__(self, cache: LruCache) -> None:
        self._cache = cache

    @property
    def entries(self) -> dict[Key, Node]:
        return self._cache._entries
```

Use probes for rollback, leak, and branch-setup tests. Do not use them to
avoid testing the public behavior.

## Direct state injection

Disposition: PORTS_AS_IS.

Keep it. Inject state when the branch is the behavior under test and the long
setup path is not.

```python
probe.sessions[session.id] = make_session_entry(session)

with pytest.raises(DuplicateToken):
    manager.refresh(duplicate_refresh())
```

## Test providers

Disposition: PORTS_WITH_ADAPTATION.

Use `monkeypatch`, context managers, and fixtures. Always restore global or
process-wide state.

```python
@pytest.fixture
def provider(monkeypatch: pytest.MonkeyPatch) -> TestProvider:
    provider = TestProvider({"retries.max": 3})
    monkeypatch.setattr(config, "provider", provider)
    return provider
```
