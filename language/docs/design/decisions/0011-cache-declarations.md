# ADR 0011: `cache` declarations

Status: accepted.

## Decision

`cache name[: T] = init` declares reconstructible, function-level runtime data.
The initializer runs on every start. Initialization and any rebuilding must
finish before evaluation uses the cache. Given restored inputs and recordable
state, rebuilding must preserve values, validity, ticks, deltas and effects
([ADR 0008](0008-temporal-contracts-and-target-mappings.md)). A historical counter
or pending event cannot be made a cache merely to match existing native storage.

Cache objects are constructed before `start` and initialized on every start.
Recordable state is restored before cache initialization; its initializer
must not overwrite a restored value. Initializers run in declaration order
across state and cache, so each can read earlier declarations. An implementation
could group cache fields in one storage object, provided that these rules hold.
Non-scalar cache construction and generic state without an initializer remain
separate design questions.

## Scheduler recovery contract

Pending scheduler events have an authoritative checkpoint independent of
user state and caches. It records deadlines and tags. Restore replaces startup
scheduler data after `start` and rebuilds scheduling indices. An empty saved
schedule clears startup events too. Events at the restart time are delivered
in that cycle; restarting after a saved deadline is refused. This recovery
contract covers simulation; wall-clock recovery remains open.

`SingleShotScheduler` is best effort. Its schedules are neither stored nor
recovered; its normal `start` behaviour is unchanged.

A finite schedule preserves its emission count and deadlines across recovery.
A three-tick schedule restored after one emission has two remaining, at the
original deadlines. This holds for immediate and delayed first emissions,
restoration before the first emission and exhausted budgets. A completed
schedule does not restart when the graph starts again.

The native time-series-delay overloads also retain their progress. A fresh
`start` input still resets their emission budget and re-bases the grid. Scalar configuration participates in
checkpoint compatibility, so changing a delay or budget requires a new run.

Generated HGL scheduler sources receive an explicit checkpoint contract only
when their runtime state consists of endpoints and pending alarms. Sources
with caches, clock injection, or logger injection remain refused. This does
not implement general native-resource recovery. Wall-clock recovery remains
outside the simulation-only component contract; `SingleShotScheduler` retains
its best-effort exception.
