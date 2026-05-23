# Analysis: cpp-testing-principles/test-helpers.md

Disposition: **PORTS_WITH_ADAPTATION**. The principles (factories with
overrideable parameter structs, presets, fixtures with RAII setup /
teardown, probes for internal-state access, providers for swappable
globals) all port. The Python primitives differ.

## Section-by-section

### Factory functions

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Factories take a parameter struct of
  `std::optional<...>` fields, merged left-to-right; presets are
  file-scoped constants; tests state only the fields that matter.
- **Python rendering:** Two Python idioms:

  - **Keyword-only dataclass with sentinel defaults:**

    ```python
    from dataclasses import dataclass, replace
    from typing import Sentinel  # PEP 661 if available; else `object()` constant

    _UNSET = object()

    @dataclass(frozen=True, kw_only=True, slots=True)
    class RequestParams:
        method: str = "GET"
        scheme: str = "https"
        host: str = "api.example.com"
        path: str = "/"
        headers: tuple[Header, ...] = ()
        body: str = ""
        timeout: float = 5.0

    def make_http_request(*overrides: RequestParams | HttpRequest) -> HttpRequest:
        merged = RequestParams()
        for override in overrides:
            merged = merge_params(merged, override)
        return _build_http_request(merged)
    ```

    File-scoped presets:
    ```python
    JSON_POST = RequestParams(
        method="POST",
        path="/v1/items",
        headers=(Header("content-type", "application/json"),),
        body="{}",
    )

    def test_post_with_path():
        req = make_http_request(JSON_POST, RequestParams(path="/v1/users"))
        assert req.path == "/v1/users"
    ```

  - **pytest fixtures with factory return value:**

    ```python
    @pytest.fixture
    def make_request():
        def _make(**overrides):
            return HttpRequest(**(DEFAULT_REQUEST_KWARGS | overrides))
        return _make

    def test_post(make_request):
        req = make_request(method="POST", path="/v1/users")
    ```

  Recommend the first form for its discoverability and the second for
  cases where the factory must close over fixture state (a database
  session, a temporary directory).

### When not to create a factory

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same rule. A trivial wrapper around a dataclass
  constructor adds noise; just use the dataclass.

### File-local wrappers

- **Classification:** PORTS_AS_IS.
- **Python rendering:** A module-level helper function inside the test
  file (no `namespace { ... }` ceremony in Python; just a private
  function with a leading underscore if it should not leak).

### Test data rules (deterministic, descriptively named, boundary-aware)

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same three rules. Python idioms:
  - `Final[int]` module-level constants for named values.
  - `@pytest.mark.parametrize` rows for boundary triples (below, at,
    above).
  - Avoid `random.*` in tests; if randomness is needed, seed
    explicitly and document the seed.

### Fixtures

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ `TEST_CASE_METHOD(fixture, ...)`:** ctor and dtor run around
  the body.
- **Python equivalent:** pytest fixtures, with `yield` separating
  setup and teardown:

  ```python
  @pytest.fixture
  def library():
      settings = Settings(config_dir=os.environ["LIBRARY_CONFIG_DIR"])
      library.init(settings)
      yield library
      library.shutdown()
  ```

  Scope (`function` / `class` / `module` / `session`) chooses the
  lifetime. Default to `function`; expand only when justified.

### Test probes (private-state access via friend declarations)

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** `test_probe<T>` is a primary template + per-class
  specialization; the class names the specialization as a friend.
- **Python rendering:** Python has no friend mechanism but does have
  weaker private (underscore prefix); the test can reach into
  `_private` attributes without ceremony, although ruff and
  pyright/pyrefly will warn (`reportPrivateUsage`).

  Recommended pattern: write a small **probe module** per class that
  needs it:

  ```python
  # tests/probes/lru_cache_probe.py
  from cache.lru_cache import LruCache

  class LruCacheProbe[K, V]:
      def __init__(self, cache: LruCache[K, V]) -> None:
          self._cache = cache

      @property
      def entries(self) -> dict[K, object]:
          return self._cache._entries  # pyright: ignore[reportPrivateUsage]

      @property
      def lru_list(self) -> list[object]:
          return self._cache._lru_list  # pyright: ignore[reportPrivateUsage]
  ```

  The probe holds a reference to the instance and exposes the private
  attributes with explicit per-line `# pyright: ignore` comments so the
  intent is visible. The C++ "decltype on a probe accessor" technique
  ports as a type alias for the nested types.

### Direct state injection

- **Classification:** PORTS_AS_IS.
- **Python rendering:** Same principle. Use the probe (or just direct
  underscored attribute access in a test) to inject the prerequisite
  state and exercise only the branch under test, instead of 80 lines
  of setup ceremony.

### Test providers (RAII guard pattern)

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** A guard struct installs the test provider on
  construction and restores the previous provider on destruction.
- **Python equivalent:** pytest fixture + `contextvars.Token.reset`
  for `contextvars`-based providers, or the standard "save / yield /
  restore" pattern:

  ```python
  @pytest.fixture
  def test_config():
      saved = config.provider
      config.provider = TestProvider({"retries.max": 3})
      yield config.provider
      config.provider = saved
  ```

  For codebases using the `contextvars`-backed cross-cutting pattern
  from `design/cross-cutting.md`, the fixture is even cleaner:

  ```python
  @pytest.fixture
  def mock_clock():
      token = clock._clock.set(MockClock())
      yield clock._clock.get()
      clock._clock.reset(token)
  ```

## Suggested Python file: `python-testing-principles/test-helpers.md`

Verbatim port with the C++ specifics rewritten in pytest/Python idioms:
keyword-only dataclasses for parameter structs, pytest fixtures with
`yield`, contextvars-based providers, and the probe-module pattern for
internal-state access.
