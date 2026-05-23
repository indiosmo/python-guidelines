# Preprocessor macros

Overall disposition: PYTHON_SPECIFIC_VARIANT.

The C++ macro chapter should become a Python metaprogramming chapter.
Preprocessor mechanics do not port, but the warning does: reach for
metaprogramming only when ordinary functions, classes, and composition do not
remove the repetition.

## Narrow sub-headers, not umbrellas

Disposition: DROP.

No Python analogue. Python import guidance belongs elsewhere: import concrete
modules, avoid import cycles, and do not hide heavy imports in broadly used
module initializers.

## No project-local Boost.PP rename header

Disposition: DROP.

No direct analogue. A weak Python cousin is "do not wrap standard library
tools in project-specific aliases that add no domain meaning", but that fits
the comments or abstraction guidance better than this chapter.

## Indirect three-token paste for argument expansion

Disposition: DROP.

No Python analogue.

## Variadic dispatch via `BOOST_PP_CAT(name, __VA_OPT__(_SUFFIX))`

Disposition: DROP.

No Python analogue.

## Domain macros are not preprocessor utilities

Disposition: PORTS_WITH_ADAPTATION.

The underlying principle ports: if a decorator, descriptor, metaclass, or code
generator expresses a domain concept, keep it with the domain it shapes.

```python
@register_error_code(RoutingErrorCode.UNKNOWN_ORDER)
class UnknownOrder(RoutingError):
    order_id: OrderId
```

Do not move that decorator to a generic metaprogramming module unless it truly
serves many domains.

## Python replacement chapter

Disposition: PYTHON_SPECIFIC_VARIANT.

Recommended sections:

- Prefer functions and classes first.
- Use decorators for call wrapping, registration, instrumentation, and policy.
- Use context managers for scoped setup, cleanup, rollback, and temporary
  provider replacement.
- Use descriptors only for reusable attribute behavior.
- Use metaclasses rarely, when class creation itself is the domain surface.
- Use code generation only when generated code is reviewed, deterministic, and
  easier to maintain than runtime reflection.

Decorator example:

```python
def timed(operation: str) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorate(function: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(function)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            start = time.perf_counter()
            try:
                return function(*args, **kwargs)
            finally:
                metrics.observe(operation, time.perf_counter() - start)

        return wrapper

    return decorate
```

The decorator's type shape needs `ParamSpec`; otherwise it erases the wrapped
function's signature.
