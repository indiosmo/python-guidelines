# Pipelines

A pipeline is an explicit flow of data through sources, stages, and sinks. The
best Python pipeline shows the domain order directly until the graph has enough
operational meaning to deserve a graph tool.

Use functions, methods, callbacks, queues, async streams, or orchestration
assets according to the boundary. Keep the stage contracts typed and keep
runtime wiring near the application entry point.

## Explicit Stages

Name each stage after the domain operation it performs. A reader should see
the order without chasing generic framework hooks.

```python
def run_rebalance_pipeline(
    request: RebalanceRequest,
    services: RebalanceServices,
) -> RebalanceResult:
    universe = services.universe_loader.fetch(request.trade_date)
    indicator_values = compute_indicators(universe, request.indicators, services.reference_data)
    rankings = rank_universe(indicator_values, request.ranking_policy)
    signals = request.signal_strategy.compute_signals(rankings)
    weighted_signals = request.weighting_strategy.compute_weights(signals)
    orders = request.portfolio_strategy.rebalance(weighted_signals)
    return RebalanceResult(orders=orders, weighted_signals=weighted_signals)
```

This shape is often better than an in-process graph abstraction. It lets each
boundary validate inputs and outputs, and it keeps the business sequence in one
place.

## Sources, Stages, And Sinks

A source produces values, a stage consumes and produces values, and a sink
consumes values. These roles are vocabulary for pipeline shape rather than
base classes every component inherits from.

Use protocols when a boundary needs substitutable implementations:

```python
from typing import Protocol


class PriceSource(Protocol):
    def fetch(self, trade_date: date) -> PriceFrame:
        ...


class OrderSink(Protocol):
    def submit(self, orders: list[Order]) -> None:
        ...
```

Use concrete functions when the pipeline is a simple in-process calculation.

## Callback Pipelines

Event-driven systems often use inbound methods and outbound callbacks. Each
callback should be named, typed, and wired to a specific consumer.

```python
from collections.abc import Callable
from dataclasses import dataclass


@dataclass(slots=True)
class OrderRouter:
    route: RoutingTable
    on_routed: Callable[[RoutedOrder], None] | None = None
    on_rejected: Callable[[OrderRejection], None] | None = None

    def submit(self, order: Order) -> None:
        routed_order = self.route.try_route(order)
        if routed_order is None:
            if self.on_rejected is not None:
                self.on_rejected(OrderRejection(order_id=order.order_id, reason="no route"))
            return

        if self.on_routed is not None:
            self.on_routed(routed_order)
```

The wiring layer decides where callbacks go:

```python
router.on_routed = execution_client.submit
router.on_rejected = rejection_store.save
```

Add logging, metrics, filtering, throttling, and thread or event-loop
marshalling at the wiring point. The stage stays independent of whether its
consumer is direct, queued, async, or remote.

## Queues And Async Boundaries

Use queues when producer and consumer lifetimes differ, when backpressure
matters, or when the runtime crosses task, thread, or process boundaries.

```python
import asyncio


async def route_orders(
    input_queue: asyncio.Queue[Order],
    output_queue: asyncio.Queue[RoutedOrder],
) -> None:
    while True:
        order = await input_queue.get()
        routed_order = route_order(order)
        await output_queue.put(routed_order)
        input_queue.task_done()
```

Use `asyncio.TaskGroup` for related fan-out where sibling failure should cancel
the group. Use an explicit result object or per-item status when partial
failure is expected and useful to the caller.

For thread bridges, use `queue.Queue`, `threading.Event`, event-loop
thread-safe scheduling, or a dedicated bridge library. Keep those primitives in
the runtime layer.

## Data Pipelines

Choose the layer that owns the work:

- Ingest tools own incremental source state, file metadata, and write
  disposition.
- Query builders or SQL engines own relational reshaping and value binding.
- dbt owns warehouse transformations, schema tests, and SQL lineage.
- pandas, Polars, NumPy, or ibis own vectorized in-memory transformations.
- Dagster owns durable assets, partitions, schedules, resources, and
  materializations.

Keep dataframe stages native to the library. Validate tabular contracts at
pipeline boundaries with schemas and named domain invariants.

## Dagster And Graph Tools

Use Dagster, dbt, or another graph tool when the graph has operational meaning:
lineage, materialization, schedules, partitions, retries, backfills, ownership,
or observability. Keep asset bodies thin and delegate domain work to library
functions.

```python
def compute_daily_signals_asset(context: AssetExecutionContext) -> MaterializeResult:
    result = compute_daily_signals(
        trade_date=date.fromisoformat(context.partition_key),
        settings=context.resources.settings,
    )
    return MaterializeResult(metadata={"row_count": result.row_count})
```

The orchestration layer owns resources and scheduling. The library function
owns the domain calculation.

When an in-process dependency list becomes a graph, add graph invariants:
stable node identities, deduplication of shared dependencies, cycle detection,
and tests for ordering.

## Runtime Wiring

Create the topology near the runtime entry point: CLI `main`, web application
factory, worker initializer, Dagster definitions, or test fixture. This is
where settings, clients, queues, logging, metrics, and task groups belong.

The pipeline core should accept typed values and services. Keep environment
reads, process-pool creation, logging configuration, and orchestration
framework imports in runtime code around the core.

## Tests

Test stage behavior with direct inputs and outputs. Test wiring with probes,
fakes, queues, or small integration tests that assert data moves across the
edges. For generated SQL or large structured pipeline outputs, use approval
tests plus direct assertions for security-sensitive parameters and counts.

## Review Checks

- Pipeline order is visible in code or in an orchestration graph.
- Each stage has typed inbound and outbound contracts.
- Runtime primitives live at the wiring boundary.
- Graph tooling appears when graph operations matter operationally.
- Dataframe stages use native vectorized operations and boundary validation.
- Dependency expansion has stable node identity, deduplication, and cycle
  checks when the graph can grow.
