# Declarative Style

Declarative Python names the work before it performs the work. It separates
parsing, validation, policy, transformation, and dispatch so each step has a
clear input and output. The result reads as a recipe instead of a transcript
of state changes.

Use Python's built-ins, comprehensions, generator expressions, `itertools`,
dataframe methods, and named helpers where they make the intent visible.
Materialize values when the result must be stored, traversed more than once,
or passed to an API that requires a container.

## Decompose By Concept

Split a complex function into steps the domain can name. Keep helpers focused
on one concept, and pass each helper the values it uses.

```python
from statistics import fmean


def variance(values: list[float], mean_value: float) -> float:
    squared_deviations = [(value - mean_value) ** 2 for value in values]
    return fmean(squared_deviations)


def compute_stats(values: list[float]) -> Stats:
    mean_value = fmean(values)
    return Stats(mean=mean_value, variance=variance(values, mean_value))
```

The assembler reads as the recipe. The helper is testable with plain numbers.

Decomposition should reduce real complexity. Do not split a short linear
function into helpers that merely rename individual lines.

## Work On Simpler Inputs

Pass helpers the smallest meaningful input. A ratio needs two counters, not
the session object that happens to store them:

```python
def acknowledgement_ratio(bytes_sent: int, bytes_acknowledged: int) -> float:
    if bytes_sent == 0:
        return 0.0
    return bytes_acknowledged / bytes_sent
```

Small inputs reduce coupling and make tests simpler. Use a domain object when
the helper needs the object's invariant or when the object is the concept being
operated on.

## Stage Values Up Front

Compute derived values before the logic that uses them:

```python
def build_page_response(query: PageQuery, total_items: int) -> PageResponse:
    page_count = ceil(total_items / query.page_size)

    return PageResponse(
        total_items=total_items,
        page=query.page,
        page_size=query.page_size,
        page_count=page_count,
        has_next=query.page < page_count,
        has_previous=query.page > 1,
    )
```

The function first states what it knows, then builds the response. It does not
mix derivation with assignment.

## Name Predicates

Lift multi-clause conditions into named predicates:

```python
def is_active_user(user: User, today: date) -> bool:
    return user.deleted_at is None and user.last_login >= today - timedelta(days=30)


active_users = [user for user in users if is_active_user(user, today)]
```

Use inline lambdas for simple projections such as `key=lambda order:
order.price`. Give a name to predicates, sort keys, or filters that carry
domain meaning or contain several clauses.

Predicate factories are useful when the same comparison recurs with different
values:

```python
def score_at_least(minimum_score: float) -> Callable[[Ranking], bool]:
    return lambda ranking: ranking.score >= minimum_score


selected = [ranking for ranking in rankings if score_at_least(0.8)(ranking)]
```

For a predicate used once, a local named function is often clearer:

```python
def is_selected(ranking: Ranking) -> bool:
    return ranking.score >= minimum_score


selected = [ranking for ranking in rankings if is_selected(ranking)]
```

## Prefer Built-In Operations

Use built-ins and standard-library algorithms that name the operation:

```python
has_expired_order = any(order.expires_at < now for order in orders)
first_missing_field = next(
    (field for field in fields if field.required and field.value is None),
    None,
)
symbols_by_market = {
    market: sorted(order.symbol for order in market_orders)
    for market, market_orders in group_orders_by_market(orders).items()
}
```

The operation name should match the question: `any`, `all`, `sum`, `min`,
`max`, `next`, `sorted`, `groupby`, `Counter`, `defaultdict`, `deque`, and
`heapq` often remove hand-written control flow.

## Compose Lazily

Use generators when the pipeline can stream and the result is consumed once:

```python
def csv_rows(orders: Iterable[Order]) -> Iterator[str]:
    buy_orders = (order for order in orders if order.side is Side.BUY)
    rows = (format_order_row(order) for order in buy_orders)
    yield from itertools.islice(rows, 10)
```

The function filters, formats, and stops after ten rows without allocating
intermediate lists.

Materialize intentionally when the caller needs stable storage, repeated
iteration, random access, sorting, or an API that expects a collection:

```python
active_symbols = tuple(
    order.symbol
    for order in orders
    if order.status is OrderStatus.ACTIVE
)
```

Remember that Python iterators are single-use. If two consumers need the same
sequence, materialize it or create two independent iterators.

## Keep Dataframe Code Native

For pandas, NumPy, Polars, SQL, ibis, and dataframe-heavy work, use the
library's native declarative surface instead of Python-level loops:

```python
def compute_returns(prices: pd.DataFrame) -> pd.DataFrame:
    return (
        prices.sort_values(["symbol", "trade_date"])
        .assign(
            return_pct=lambda frame: frame.groupby("symbol")["close_price"].pct_change(),
        )
        .dropna(subset=["return_pct"])
    )
```

Use `.pipe(...)` when it names meaningful stages:

```python
weighted_signals = (
    universe
    .pipe(compute_scores, scoring_strategy)
    .pipe(select_signals, signal_strategy)
    .pipe(compute_weights, weighting_strategy)
    .pipe(validate_weighted_signals)
)
```

Keep dataframe mutation contracts explicit. A pure transform reads input
columns and returns new columns or a new frame. An in-place transform should
own the frame and say so in its name.

## Explicit Pipelines

Keep pipeline order visible until the graph itself becomes a domain concept:

```python
def run_portfolio_pipeline(request: PortfolioRequest) -> PortfolioResult:
    universe = fetch_universe(request.universe_id, request.trade_date)
    indicators = compute_indicators(universe, request.indicators)
    rankings = rank_universe(indicators, request.ranking_model)
    signals = select_signals(rankings, request.signal_strategy)
    weights = compute_weights(signals, request.weighting_strategy)
    return build_portfolio_result(weights)
```

Reach for a graph or orchestration tool when the system needs asset lineage,
partitions, scheduling, backfills, retries, materialization, or cycle
detection. Use explicit functions for an in-process sequence whose order is
the main thing a reader needs to understand.

## Boundaries

Declarative inner code still needs explicit boundaries:

- Parse external data into named models.
- Validate dataframes where their shape becomes a contract.
- Translate domain exceptions at UI, HTTP, CLI, worker, job, or queue
  boundaries.
- Log and measure at the shell, not inside every transformation.

The core describes the computation. The boundary owns effects.
