# Publication data and separately observed state

Status: proposed, 2026-10-08. These decisions are for review; no new mutation
syntax or application semantics are adopted. References remain excluded.

## Decision 1: keep `delta<T>` as publication data

Retain [delta type identity](ordinary-delta-types.md) and the valid+modified
accessor guard. Omission means no child publication; `_` means no supplied
harness publication. Neither omission, `_`, nor `null` encodes invalidation or
creation of an invalid child.

[Endpoint validity and membership](../../../runtime/time_series.md) require
separate observation. Invalidating a map child keeps its key; creating an
initially invalid child changes membership without a valid child publication.
Publication recording/replay therefore cannot promise full-state reconstruction.

## Decision 2: test state through public observers

Drive observations with an independent clock-step input:

```hgl
fn observe_valid(step: i64, watched: map<i64, i64>) -> bool {
    when modified(step) && valid(step) {
        return valid(watched)
    }
}
```

The explicit [validity selector](../user-guide/types-and-expressions.md#temporal-metadata)
admits invalid `watched`; omitting it implicitly requires all inputs valid.
The watched dependency orders observation after its producer, including silent
cycles. For a valid map, [existing collection access](../user-guide/types-and-expressions.md#collection-views-and-iteration)
provides `contains(watched, 9)` for membership and `items(watched)` with
`valid(child)` for child validity.

Existing [map/list child invalidation](../user-guide/functions.md#output-access)
preserves membership/length. Isolate four missing mutation extensions:

- whole-output invalidation;
- struct/tuple child invalidation;
- creating initially invalid map membership;
- appending an initially invalid list child.

Each needs selected syntax, typed checking, preconditions, validity/membership
and notification/accumulation rules, plus producer/observer conformance cases.
Initialization followed by invalidation is a separate experiment.

## Preserve empty application; leave forwarding open

Preserve shape-specific application: initial empty set/map data can establish
validity; repeats can be silent; empty fixed/list/struct data need not publish.
The [application profile](contextual-collection-deltas.md#deliberate-boundaries)
retains its exclusions. Empty delta storage does not admit an event; atomic
empty-list snapshots remain admitted complete publications.

Real ticks notify (TS-6). Set add/remove cancellation can tick with an empty net
delta (TS-10). Preserving that tick when applying its payload downstream remains
open. Selecting this guarantee requires event admission, parent validity,
zero-child behavior and modified/time rules; empty data alone supplies none.

Measured outcomes remain in the [publication-boundary audit](https://github.com/hhenson/hgraph_spec_audit/blob/5aa30e8bda922708d251e122d7d546e87f5aec0b/runtime/validation/publication_boundaries/README.md).
