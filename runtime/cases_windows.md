# Window cases

Times are microsecond offsets from a legal run start. A sample written
`value@time` carries its original arrival time. `—` means no publication.
The traces cover live graph outputs, standalone mutations and recorded publications.

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

## WINDOW-DURATION — TS-30, TS-31, TS-36

A live duration window has a maximum span of 3 µs. Its producer writes at
0, 2, 3 and 7. A reader records each publication, including the final
departure after the producer goes silent. Both minimum settings see the
same publications.

| t | Written | Held samples | Added | Removed samples | All valid, zero | All valid, omitted |
|---|---|---|---|---|---|---|
| 0 | 10 | [10@0] | [10] | [] | true | false |
| 2 | 20 | [10@0, 20@2] | [20] | [] | true | false |
| 3 | 30 | [10@0, 20@2, 30@3] | [30] | [] | true | true |
| 4 | — | [20@2, 30@3] | [] | [10@0] | true | false |
| 6 | — | [30@3] | [] | [20@2] | true | false |
| 7 | 40 | [40@7] | [40] | [30@3] | true | false |
| 11 | — | [] | [] | [40@7] | false | false |

At 3 the oldest sample is exactly 3 µs old. It leaves at 4. At 7 the
reader runs once and sees both the departure and arrival. The empty window
at 11 stays valid. With no more writes, the run has no further window event.

## WINDOW-SHARED-EXPIRY — TS-36

An owned window in a root bundle field and an owned window in a nested
graph each receive an arrival at 0. Both spans are 2 µs. A separate reader
observes the nested window through a forwarding input.

| t | Root producer | Nested producer | Root window delta | Nested window delta |
|---|---|---|---|---|
| 0 | 5 | 6 | add 5 | add 6 |
| 2 | — | — | — | — |
| 3 | — | 7 | remove 5@0 | remove 6@0, add 7 |
| 6 | — | — | — | remove 7@3 |

The row at 2 is a passive observation of the closed boundary. At 3 both
readers see their final window state in the same root cycle. The forwarding
input observes the same nested change once. If the nested graph stops at
2, only the root reader sees a departure at 3.

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

For a delayed application, seed two capacity-one windows with [10@1].
At t=3 the sender pushes 20, capturing `added=[20]`, `removed=[10@1]`.
Apply that delta to the receiver at t=8. The sender holds [20@3], the
receiver holds [20@8], and the removed sample still reads 10@1. In a
separate run, applying a whole-state snapshot of the sender at t=8 gives
[20@3].

## WINDOW-DELAYED-EVENTS — TS-30, TS-36

A live window of span 2 µs receives 7 at t=0. A second output delays each
of its events by 1 µs.

| t | Live window | Delayed output |
|---|---|---|
| 0 | add 7 | — |
| 1 | — | add 7 |
| 3 | remove 7@0 | — |
| 4 | — | remove 7@0 |

The delayed addition arrives at 1. Its departure still identifies the
source sample at 0. Each output publishes exactly twice. Similarly, an
operator taking the first two window events publishes exactly twice even
if its source later expires. An independent reader of that operator sees
the last published state.

## WINDOW-SAMPLED-HISTORY — TS-30, TS-36

A source window with a span of 3 µs receives 2 at t=0 and 3 at t=1.
A separate signal samples its complete state at t=2 and t=4.

| t | Source history | Sample pulse | Sampled replacement |
|---|---|---|---|
| 0 | [2@0] | — | — |
| 1 | [2@0, 3@1] | — | — |
| 2 | [2@0, 3@1] | yes | [2@0, 3@1] |
| 4 | [3@1] | yes | [3@1] |
| 5 | [] | — | — |

At 4 the source expires 2 before the sampler reads it. The sample retains
the original time of 3. At 5 the source expires 3. The sampler remains
silent and its last published state is still [3@1].

## WINDOW-GROUPED-RELEASE — TS-30, TS-36

A live window has a span of 2 µs. An operator buffers its publications
until t=5, then releases them together. The source admits 3 at t=0 and
7 at t=4.

| t | Source event | Buffered output |
|---|---|---|
| 0 | add 3 | — |
| 3 | remove 3@0 | — |
| 4 | add 7 | — |
| 5 | — | replace with [7@4] |

The replacement contains only the surviving observation, at its source time.
If the second arrival is omitted, the release publishes a replacement with
an empty sample list. A reader still observes the release.

## WINDOW-EQUAL-VALUE — TS-30

A tick window of size 2. Writing 4 then 4 gives values `4, 4`, two ticks and
two incremental deltas each with `added=[4]`.

## Scope

The scheduled cases cover owned outputs at fixed paths, including fixed
bundle fields and list positions inside nested graph instances. Registration
of window children inserted into a growing list or dictionary and durable
checkpoint recovery have their own implementation stages.
