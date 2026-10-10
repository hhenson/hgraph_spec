# Rolling-window publications

Admit exact `rolling<V, Max, Min>` shapes to the publication profile. V is an
admitted finite ordinary payload type: a scalar leaf or complete ordinary
composite under the finite atomic payload grammar, including [bytes](bytes-values.md). A window's V is ordinary
data, not a temporal child. Min is optional: `rolling<V, Max>` is exactly
`rolling<V, Max, Max>`. Existing size-kind and size-bound rules apply.
The resolved kind, Max, Min and V all participate in exact shape identity.
Windows remain inherently temporal and cannot become ordinary atomic payloads.

## Arrival delta

`delta<rolling<V, Max, Min>>` is exactly V, the value arriving this cycle.
It is not the held window, a sparse collection patch, an evicted value, or a
list of values/timestamps. Construct it through ordinary V expressions; no
rolling-delta constructor is introduced. Owning `TimedValue` and recording
entries independently retain V under existing copy rules.

Use the unchanged generic body `when { return delta_value(value) }`. Valid and
modified prove an arrival is present; `all_valid` is not required. A window is
valid from its first arrival even while its minimum is not satisfied. Returning
that arrival publishes it to the pass-through's own same-shaped output, which
independently derives its window using the output publication time. A direct
same-cycle pass-through reproduces the same retained values/times; a delayed
publication uses its new time and does not transfer the input's window state.
Equal arrivals still tick. `_` is silence, with no new value or eviction.

An ordinary V alone does not infer a rolling shape. Preserve scalar inference
for existing scalar-only uses; choose rolling only from an exact endpoint,
wrapper, explicit `TimedValue<rolling<...>>` or another existing type-bearing
context. No size or kind is inferred from data length, payload or timestamp.

## Eviction and readiness

Tick windows retain at most Max arrivals, oldest first, evicting the oldest
on a new arrival when full. Duration windows evict values whose age at the
new arrival time is greater than Max, then retain the new arrival. A value
exactly Max old stays. Eviction only occurs on arrival; idle time alone does
not change the retained window or schedule a tick.

`all_valid` describes the currently retained window, not a sticky historical
milestone or elapsed time since graph start. A tick window is ready when its
current count is at least Min. A duration window is ready when its newest
retained arrival time minus its oldest is at least Min. Min=0 is ready from
the first arrival; positive duration minima require at least two arrivals.
After a long gap, a new arrival can evict all older values and make a positive
minimum unready again while the window remains valid. Readiness does not
suppress that arrival's delta or output publication.

Removed-value observation remains a separate runtime contract and is not
part of the arrival delta.

## Eval, replay and recording

Eval fixes exact rolling shapes before construction, materializes and validates
each present ordinary V before target start, and schedules one arrival for
each present dense slot at its existing evaluation time. It retains the dense
horizon separately. Recording stores arrival V and its actual publication
time, with ordinary ownership; harness equality compares presence and V.
Empty/all-silent input runs the existing lifecycle and produces no arrivals.
Silence never reads a held window to invent output. These rules also apply
when a rolling endpoint is an admitted structural child.

No timed-input syntax, timer-driven eviction, clearing, whole-window
invalidation, reference payload or new window iteration API is introduced.

See [examples](../../examples/rolling-publications.hgl) and
[compiler cases](../../../compiler/cases_rolling_publications.md).
