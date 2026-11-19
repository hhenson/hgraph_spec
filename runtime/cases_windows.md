# Window cases

Times are microsecond offsets from a legal run start. A sample written
`value@time` carries its original arrival time. `—` means no publication.
The traces cover incremental mutations and whole-window replacement.
Scheduled expiry is a later stage.

## WINDOW-TICK — TS-13, TS-30, TS-31

A tick window of size 3 with minimum 2 over `TS<i64>`. The node writes one
value in the cycles shown.

| t | Written | Value, oldest first | Arrival | Singular removal | Valid | All valid | Modified |
|---|---|---|---|---|---|---|---|
| 0 | — | nil | nil | — | false | false | false |
| 1 | 5 | 5 | 5 | — | true | false | true |
| 2 | 6 | 5, 6 | 6 | — | true | true | true |
| 3 | — | 5, 6 | nil | — | true | true | false |
| 4 | 7 | 5, 6, 7 | 7 | — | true | true | true |
| 5 | 8 | 6, 7, 8 | 8 | 5 | true | true | true |
| 6 | — | 6, 7, 8 | nil | — | true | true | false |

The window is valid from its first value and all valid from its second
(the minimum). The fourth arrival evicts the oldest; the evicted value is
readable at 5 and absent at 6. Each value carries the time it arrived: at 5
they are 2, 4, 5. The delta at 5 has `cleared=false`, `added=[8]` and
`removed=[5@1]`.

## WINDOW-DURATION — TS-30, TS-31

A duration window of 3 µs receives 10 at t=0, 20 at t=2, 30 at t=3 and
40 at t=7. The input is silent at t=6. Compare an explicit minimum of zero
with an omitted minimum of 3 µs.

| t | Written | Held samples | Added | Removed samples | All valid, zero | All valid, omitted |
|---|---|---|---|---|---|---|
| 0 | 10 | [10@0] | [10] | [] | true | false |
| 2 | 20 | [10@0, 20@2] | [20] | [] | true | false |
| 3 | 30 | [10@0, 20@2, 30@3] | [30] | [] | true | true |
| 6 | — | [10@0, 20@2, 30@3] | — | — | true | true |
| 7 | 40 | [40@7] | [40] | [10@0, 20@2, 30@3] | true | false |

At t=3 the oldest sample lies exactly on the cutoff. At t=6 an independent
probe sees the retained window unmodified. At t=7 the window ticks once
and the singular removal is 30. Each publication in this trace has
`cleared=false`.

## WINDOW-CLEAR — TS-13, TS-30, TS-31

Start with held samples [4@1, 4@2]. Observe the window after each action.
The Python compatibility columns show only their respective scalar observations.

| t | Action | Held samples | Cleared | Added | Removed samples | Python arrival | Singular removal |
|---|---|---|---|---|---|---|---|
| 3 | clear then push 9 | [9@3] | true | [9] | [4@1, 4@2] | 9 | — |
| 4 | clear | [] | true | [] | [9@3] | None | — |
| 5 | clear again | [] | true | [] | [] | None | — |
| 6 | idle | [] | — | — | — | None | — |

A reader active on the window runs once at 3, 4 and 5. Its validity guard
admits an empty window. At 4 and 5 the window is valid and not all valid.
At 6 the window produces no delta. A copy captured at 3 still contains both
4s and their original times.

## WINDOW-REMOVAL — TS-30

Start with held samples [5@10, 5@10, 7@11] in a 3 µs duration window.
At t=14 explicitly remove the expired prefix. These two variants start
from the same state and do not rely on a timer.

| Other action at 14 | Held samples | Cleared | Added | Removed samples |
|---|---|---|---|---|
| none | [7@11] | false | [] | [5@10, 5@10] |
| clear then push 9 | [9@14] | true | [9] | [5@10, 5@10, 7@11] |

The subscriber runs once in either variant. The removal-only tick has no
arrival. The two identical removals are separate samples.

## WINDOW-REPLACE — TS-30

A tick window of capacity 3 holds [8@1]. At t=2 assign the complete Python
sequence [4, 4, 6]. Capture the whole-window replacement and apply it
to another window seeded with [99@1]. Both now hold [4@2, 4@2, 6@2]. At
t=3 push the scalar 7 to the source. The incremental delta has `cleared=false`, `added=[7]`
and `removed=[4@2]`. Applying this delta to the second window makes both
hold [4@2, 6@2, 7@3]. The previously captured replacement still holds
[4@2, 4@2, 6@2].

## WINDOW-REPLAY — TS-30

A legacy recording contains scalar arrivals 10 at t=0 and 20 at t=7 for a
3 µs duration window. An observer of replay receives publications at 0 and
7. No observation was recorded at 4, so replay introduces no departure tick
there. A recording containing a clear-only or removal-only delta reproduces
that modification when replayed. A removed sample keeps its original
arrival time.

## WINDOW-EQUAL-VALUE — TS-30

A tick window of size 2. Writing 4 then 4 gives values `4, 4`, two ticks and
two incremental deltas each with `added=[4]`.

## Outside these cases

Simulation and real-time scheduling of departures without a producer
mutation belong to the subsequent expiry stage. These cases exercise the
window state and delta on each supplied evaluation.
