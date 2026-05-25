# Decorators And Metaprogramming

Metaprogramming changes how code is defined, registered, wrapped, or generated.
Use it where it removes repeated, reviewable boilerplate or creates a clear
extension surface. Prefer plain functions and explicit registration until the
repetition is real.

The practical ladder is:

1. Plain functions, dataclasses, and explicit calls.
2. Decorators for local wrapping or registration.
3. `functools.singledispatch` for type-based function extension.
4. Class decorators or `__init_subclass__` for class families.
5. Descriptors for reusable attribute behavior.
6. Code generation when checked source should contain repeated declarations.
7. Metaclasses for framework-level class creation rules.

## Plain Functions First

Start with the smallest explicit shape:

```python
def normalize_symbol(symbol: str) -> str:
    return symbol.strip().upper()


SYMBOL_NORMALIZERS = {
    "default": normalize_symbol,
}
```

This is easy to search, type check, and test. A decorator earns its place when
many definitions repeat the same registration or wrapping pattern.

## Decorators

A decorator should make a local rule visible at the definition site. Common
uses are registration, timing, retry policy, permission checks, validation, and
framework routing.

Use `functools.wraps` so introspection, tracebacks, documentation, and test
failure output point at the wrapped function.

```python
from collections.abc import Callable
from functools import wraps
from time import perf_counter


def log_duration[**P, R](
    logger: Logger,
) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorate(function: Callable[P, R]) -> Callable[P, R]:
        @wraps(function)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            started_at = perf_counter()
            try:
                return function(*args, **kwargs)
            finally:
                logger.info(
                    "function completed",
                    extra={
                        "function": function.__qualname__,
                        "elapsed_s": perf_counter() - started_at,
                    },
                )

        return wrapper

    return decorate
```

Keep decorators small. A decorator that parses configuration, opens clients,
registers plugins, handles errors, and emits metrics is several policies
hidden behind one line.

## Registration Decorators

Registration decorators fit catalogs of strategies, commands, checks, or
serializers when the catalog is a public extension surface.

```python
from collections.abc import Callable


ScoreBuilder = Callable[[ScoreConfig], ScoreFunction]
SCORE_BUILDERS: dict[str, ScoreBuilder] = {}


def score_builder(name: str) -> Callable[[ScoreBuilder], ScoreBuilder]:
    def register(builder: ScoreBuilder) -> ScoreBuilder:
        SCORE_BUILDERS[name] = builder
        return builder

    return register


@score_builder("z_score")
def build_z_score(config: ScoreConfig) -> ScoreFunction:
    return ZScore(config.window)
```

Use explicit imports at the composition root so registration happens in a
predictable place. When the supported set is part of the public contract, a
literal registry or union type can be better than discovery.

## Single Dispatch

Use `functools.singledispatch` when behavior varies by the runtime type of one
argument and callers should see one function name.

```python
from functools import singledispatch


@singledispatch
def to_wire_value(value: object) -> str:
    raise TypeError(f"unsupported wire value: {type(value).__name__}")


@to_wire_value.register
def _(value: date) -> str:
    return value.isoformat()


@to_wire_value.register
def _(value: Decimal) -> str:
    return format(value, "f")
```

Use protocols, `match`, or methods when dispatch depends on domain state,
multiple arguments, or a closed set that should be handled exhaustively.

## Class Decorators And Init Subclass

Class decorators fit one-time transformations of a class definition. They are
useful for validation, registration, or adding generated methods when a
dataclass, attrs class, or Pydantic model is not the right tool.

Use `__init_subclass__` when every subclass of a base class must participate in
the same class-level rule.

```python
class Strategy:
    _registry: dict[str, type["Strategy"]] = {}

    strategy_name: str

    def __init_subclass__(cls, *, strategy_name: str, **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        cls.strategy_name = strategy_name
        Strategy._registry[strategy_name] = cls


class KeepTopN(Strategy, strategy_name="keep_top_n"):
    ...
```

This pattern is useful when subclassing is already the extension mechanism.
For structural plugins, prefer a protocol plus an explicit registry.

## Descriptors

Descriptors fit reusable attribute behavior: lazy loading, cached conversion,
field validation, or framework-managed attributes. Keep descriptor behavior
focused on attribute access. Use a normal object when the behavior needs
several methods or lifecycle hooks.

```python
class PositiveDecimal:
    def __set_name__(self, owner: type[object], name: str) -> None:
        self.private_name = f"_{name}"

    def __get__(self, instance: object, owner: type[object]) -> Decimal:
        if instance is None:
            return self
        return getattr(instance, self.private_name)

    def __set__(self, instance: object, value: Decimal) -> None:
        if value <= 0:
            raise ValueError("value must be positive")
        setattr(instance, self.private_name, value)
```

Reach for Pydantic fields, dataclass validation, or a property before writing a
descriptor. A descriptor is shared machinery.

## Code Generation

Generate code when the checked source needs repeated declarations from a
structured input: API clients, schema bindings, SQL models, protocol messages,
or static registries. Prefer structured inputs such as TOML, YAML, JSON, OpenAPI
schemas, database metadata, or a trusted Python module.

Generated code should be deterministic, formatted, and reviewed like any other
artifact when it is committed. The command that regenerates it belongs in the
project command surface.

## Metaclasses

Use metaclasses when class creation itself is the abstraction: ORMs, validation
frameworks, plugin frameworks, declarative query builders, or compatibility
with an existing framework. A metaclass affects every subclass and can make
ordinary Python lookup rules harder to follow.

Most project code can use dataclasses, Pydantic, decorators,
`__init_subclass__`, descriptors, or explicit factories instead.

## Review Checks

- A plain function or explicit registry has been considered before a decorator.
- Decorators use `wraps`, preserve signatures, and hide only one policy.
- Registration happens at a predictable composition point.
- Class-level hooks match a real class family.
- Descriptors own reusable attribute behavior rather than broad object
  lifecycle.
- Generated code is deterministic and has a documented regeneration command.
- Metaclasses appear only where class creation is the domain mechanism.
