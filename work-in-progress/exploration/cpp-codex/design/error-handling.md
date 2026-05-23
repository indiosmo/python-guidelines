# Error handling

Overall disposition: PYTHON_SPECIFIC_VARIANT.

This chapter needs a rewrite. C++ uses result types internally and translates
at component boundaries. Idiomatic Python uses exceptions internally, then
translates explicitly at trust boundaries: HTTP responses, CLI exit codes,
task top levels, message consumers, retry loops, and plugin callbacks.

## Domain error types

Disposition: PORTS_WITH_ADAPTATION.

Keep domain-owned errors, but make them exception classes with typed context.

```python
class RoutingError(Exception):
    code: RoutingErrorCode


@dataclass(frozen=True)
class InvalidFieldValue(RoutingError):
    field: FieldId
    value: str
    code: ClassVar[RoutingErrorCode] = RoutingErrorCode.INVALID_FIELD_VALUE

    def __str__(self) -> str:
        return f"{self.field}: invalid value {self.value!r}"
```

The producer domain owns the class. Adapters catch producer-domain exceptions
and translate them into the consuming domain's vocabulary.

## Codes integrate with `std::error_code`

Disposition: PYTHON_SPECIFIC_VARIANT.

Replace `std::error_code` with `Enum` values, stable string codes, and
structured exception attributes. Numeric global prefixes are rarely worth it
in Python unless the system persists numeric codes externally.

```python
class RoutingErrorCode(StrEnum):
    INVALID_FIELD_VALUE = "routing.invalid_field_value"
    MISSING_REQUIRED_FIELD = "routing.missing_required_field"
    UNKNOWN_ORDER = "routing.unknown_order"
```

## Structured errors carry typed context

Disposition: PORTS_WITH_ADAPTATION.

Keep this rule. Avoid exceptions that only contain a message string.

```python
raise InvalidFieldValue(field=FieldId("side"), value=raw_side)
```

Loggers, HTTP translators, metrics, and tests can inspect the fields without
parsing `str(error)`.

## A type-erased carrier propagates through LEAF

Disposition: DROP.

There is no direct Python analogue worth preserving. The exception object is
already the carrier. `ExceptionGroup` matters for concurrent failures, but it
is not a LEAF replacement.

## Handlers match on the structured type

Disposition: PORTS_WITH_ADAPTATION.

Use `except` clauses, `except*` for exception groups, or `match` on error
attributes when needed. Keep handler ordering: specific before broad.

```python
try:
    reply = route(request)
except DuplicateOrder as error:
    logger.info("duplicate order", extra={"order_id": error.order_id})
    return Drop()
except InvalidFieldValue as error:
    return Reject(field=error.field, value=error.value)
except RoutingError:
    logger.exception("unhandled routing error")
    return InternalReject()
```

## Unwrapping a result

Disposition: PYTHON_SPECIFIC_VARIANT.

Drop result-unwrapping macros. Replace with guidance against `Optional` as an
implicit error channel. If absence is meaningful, return `T | None`. If failure
has a reason, raise a domain exception or return an explicit boundary result
object.

## Exceptions do not cross component boundaries

Disposition: PORTS_WITH_ADAPTATION.

Keep the boundary rule, but invert the internal default. Exceptions can flow
inside a component; public boundaries catch and translate to the caller's
contract.

```python
def handle_request(request: Request) -> Response:
    try:
        return build_response(route(request))
    except RoutingError as error:
        return translate_routing_error(error)
```

Registered callbacks need policy. If a callback is user-owned, let its
exception belong to the caller or isolate it in a documented boundary.

## Top-level functions catch everything

Disposition: PORTS_AS_IS.

Keep it. Python top-level functions include ASGI handlers, queue workers,
scheduled jobs, CLI commands, thread targets, and task-group roots.

```python
async def consume_message(message: bytes) -> None:
    try:
        request = parse_request(message)
        await process_request(request)
    except KnownReject as error:
        await publish_reject(error)
    except Exception:
        logger.exception("message consumer failed")
```

Do not catch `BaseException` unless the code is deliberately handling
shutdown. Let `KeyboardInterrupt`, `SystemExit`, and cancellation semantics
remain visible unless a framework contract says otherwise.

## Enriching errors with diagnostic context

Disposition: PORTS_WITH_ADAPTATION.

Use exception attributes, exception chaining, structured logging, `logger.exception`,
and `traceback`/`faulthandler` rather than a custom error carrier.

```python
try:
    return parse_order(raw)
except json.JSONDecodeError as error:
    raise InvalidPayload(source="order_api") from error
```

The Python guide should warn against swallowing the original exception without
`from error`; chained tracebacks are often the root-cause trail.
