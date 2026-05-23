# Defense-in-depth validation

Overall disposition: PORTS_AS_IS.

The chapter ports directly. Python needs this discipline because values often
enter as dictionaries, JSON, database rows, or third-party objects before they
become domain types.

## Why one validation is not enough

Disposition: PORTS_AS_IS.

Keep it. One check fixes one path; layered validation closes a bug class.

## Types first: parse, don't validate

Disposition: PORTS_WITH_ADAPTATION.

Keep the principle but use Python tools: parser functions, frozen dataclasses,
Pydantic, msgspec, attrs, `NewType`, `Annotated`, and enums. Pick based on the
boundary and performance needs.

```python
def create_order(raw: RawOrderParams) -> Order:
    return Order(
        symbol=parse_symbol(raw.symbol),
        quantity=parse_quantity(raw.quantity),
        side=parse_side(raw.side),
    )
```

## Four layers

Disposition: PORTS_AS_IS.

Keep the layer model.

### Layer 1: parse at the boundary

Disposition: PORTS_AS_IS.

Use schema or parser code at JSON, CLI, HTTP, database, queue, and file
boundaries. Convert to domain dataclasses before inner logic.

### Layer 2: business-logic validation

Disposition: PORTS_AS_IS.

Keep it. These checks depend on state: account funds, known symbol, user
authorization, workflow phase.

### Layer 3: environment guards

Disposition: PORTS_AS_IS.

Keep it. Python examples: dry-run mode, test mode, production endpoint guards,
feature-flag restrictions, migration mode, or tenant isolation.

### Layer 4: debug instrumentation

Disposition: PORTS_AS_IS.

Keep it. Use structured logging and metrics at decision points.

## A worked example

Disposition: PORTS_WITH_ADAPTATION.

Render with Python domain objects:

```python
@dataclass(frozen=True)
class Entry:
    name: EntryName
    source: Source


def load_entries(path: Path) -> list[Entry]:
    ...
```

The loader either returns entries with parsed names or raises a parse error.
The registry still checks duplicates and environment-specific constraints.

## Applying the pattern

Disposition: PORTS_AS_IS.

Keep the four steps: map data flow, add layer-appropriate validation, add
debug logging where silent, and test each layer.

## Where validation belongs, and where it does not

Disposition: PORTS_AS_IS.

Keep the boundaries. Python-specific note: type hints are not runtime
validation. If external input needs enforcement, use parser code or a runtime
schema library.

Use `assert` for programmer invariants only if optimized mode removing asserts
would not break caller-visible validation. For user input, raise a domain
exception or return the boundary's documented error.
