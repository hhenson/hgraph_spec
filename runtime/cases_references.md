# Reference cases

Status: expected reference observations.

Times below are cycles from `MIN_ST`. `—` means no publication, not an empty
reference. Observe after producers and routing have evaluated. A clock input
makes the observer run even when the data input is passive or invalid.

## REF-SAMPLE — TS-1, TS-14, TS-16 and TS-17

A and B are scalar outputs. A selector publishes a reference to A, B or empty.
A REF input observes designation; a scalar input follows it, and so does a
SIGNAL input (TS-18). Both start invalid.

| t | A publishes | B publishes | Select | Followed value | Data modified | REF modified |
|---|---|---|---|---|---|---|
| 0 | 7 | 20 | empty | nil | false | true |
| 1 | — | — | A | 7 | true | true |
| 2 | 8 | — | — | 8 | true | false |
| 3 | — | 21 | B | 21 | true | true |
| 4 | — | — | — | 21 | false | false |
| 5 | — | — | empty | nil | false | true |
| 6 | 9 | — | A | 9 | true | true |
| 7 | — | — | — | 9 | false | false |

At 1 the input samples A: its delta is 7 and its observed last time is 1;
A's own last time remains 0. At 5 the scalar input is invalid, unmodified,
with last time `never`. The REF itself is valid: empty is a reference value.
A target publication does not evaluate the reference selector. Passive data
inputs still follow and sample; passivity only suppresses their wake.

## REF-DICTIONARY — TS-14, TS-15 and TS-19

A publishes `{"X":1, "Z":9}` at 0. B publishes `{"Y":2, "Z":3}` at 0, then `{"Y":4}`
at 1. No other producer publications occur. The input selects A at 0, B at 2,
empty at 3, and A at 4. R denotes a removal in the delta.

| t | Current value | Added | Removed | Delta | Valid / modified |
|---|---|---|---|---|---|
| 0 | "X":1, "Z":9 | "X","Z" | — | "X":1, "Z":9 | true / true |
| 1 | "X":1, "Z":9 | — | — | nil | true / false |
| 2 | "Y":4, "Z":3 | "Y" | "X" | "X":R, "Y":4, "Z":3 | true / true |
| 3 | nil | — | "Y","Z" | "Y":R, "Z":R | false / true |
| 4 | "X":1, "Z":9 | "X","Z" | — | "X":1, "Z":9 | true / true |
| 5 | "X":1, "Z":9 | — | — | nil | true / false |

At 2 both new children read modified with their whole value as delta,
including overlapping "Z". This samples their input views, not their producers.
At 3 the withdrawal delta remains readable despite invalidity. The input has
no current children; removed children remain inspectable for that cycle.
Do not gate a withdrawal delta on `valid`.

## REF-DICTIONARY-RETARGET — TS-14 and TS-17

A selector designates a dictionary. A nested consumer follows the current
children at keys 0 and 1. Both dictionaries publish at cycle 0. A holds
`{0:1, 1:2}` and B holds `{0:10, 1:20, 2:30}`.

| Cycle | Select | Producer changes after initialization | Current dictionary delta | Consumer deltas at keys 0, 1 |
|---|---|---|---|---|
| 0 | A | — | 0:1, 1:2 | 1, 2 |
| 1 | B | — | 0:10, 1:20, 2:30 | 10, 20 |
| 2 | A | — | 0:1, 1:2, 2:R | 1, 2 |
| 3 | — | B removes 2 | — | — |
| 4 | A | — | — | — |
| 5 | B | — | 0:10, 1:20 | 10, 20 |
| 6 | — | B writes 1:21 | 1:21 | —, 21 |
| 7 | — | A writes 1:3 | — | — |

R denotes removal. At cycle 5 the selected dictionary has the same key set
as the previous one. Its current children are sampled in that cycle.

## REF-EXPIRES — TS-11 and TS-23

Publish X=7 at 0. Save a reference to X's child in a separate node. Remove X at
2. At 3 write that saved reference to a REF output; this does **not** insert X.
Insert a new X=9 at 4. Write the saved reference again at 5.

A data input following the saved reference sees 7 through cycle 2, then nil
from 3 onwards. It never sees 9. The removed-child view is readable at 2 and
absent at 3. Keeping the reference does not extend the endpoint's lifetime.

The expiry rule was confirmed by the user on 2026-09-21.

Allocator slot reuse and generation wrap handling are outside this case's scope.

## REF-MAPPED-SHARED — TS-1, TS-6, TS-14 and TS-19

A map has children `a`, `b` and `c`. Each selects an element from one shared
dictionary and counts evaluations of a consumer of that element. Initially
`a` and `c` select `x`, while `b` selects `y`. The dictionary is passed whole
at the map boundary; the selection happens while wiring each child graph.

| t | Lookup or map change | Shared publication | Consumer count deltas |
|---|---|---|---|
| 0 | create a:x, b:y, c:x | x:1, y:2, z:3 | a:1, b:1, c:1 |
| 1 | — | x:4 | a:2, c:2 |
| 2 | — | z:5 | — |
| 3 | a selects y | — | a:3 |
| 4 | remove c | — | c:R |
| 5 | — | x:6 | — |
| 6 | create c:x | — | c:1 |

At 3 the changed selection samples the current value of `y`. At 6 the new
child samples `x` and starts a fresh counter. At 2 and 5 no selected consumer
evaluates. For value-only updates of existing keys, increasing the number
of unrelated mapped consumers must not add notification or child-selection
work. A consumer of the whole dictionary remains affected by every update;
source lifetime and membership notifications remain observable.


## REF-KEYED-LOOKUP — TS-6, TS-14, TS-19 and TS-35

Two lookups select `d["x"]` and `d["y"]`; both keys initially are absent.
The counts below are membership notifications to the lookup nodes. Their
consumers follow the selected children for value ticks.

| t | Dictionary change | x lookup notifications | y lookup notifications |
|---|---|---|---|
| 0 | insert z:1 | 0 | 0 |
| 1 | insert x:2, then y:3 in the same evaluation | 1 | 1 |
| 2 | update x:4 | 0 | 0 |
| 3 | remove x | 1 | 0 |
| 4 | reinsert x:5 | 1 | 0 |
| 5 | remove z | 0 | 0 |

The consumer of `x` receives 2, 4, withdrawal, then 5 at times 1–4. Increasing
unrelated lookups must not increase notification work for a single key change.
Changing a selection detaches the old membership subscription. Replacing the
whole dictionary can notify all selected keys; stopping a lookup detaches it.

## REF-WITHDRAWAL — TS-8, TS-16 and TS-17

A scalar reader is active and has no validity guard. At cycle 0 it follows A.
At cycle 1 the selector either publishes an empty reference, invalidates its
REF output, or points to B, which has never published. Each choice produces
the same scalar-reader trace.

| Cycle | Selector | A publishes | Reader observes |
|---|---|---|---|
| 0 | A | 7 | 7 |
| 1 | Withdraw | — | Invalid, unmodified, last modified time never |
| 2 | Repeat withdrawal | — | — |
| 3 | — | 8 | — |
| 4 | A | — | 8 |

A passive reader sees the same state if another input wakes its node.
A window reader and a list reader give the same trace. At cycle 1 each wakes
once and reads invalid and unmodified, with nil value and no delta: no removed
samples or positions and no empty replacement. Cycles 2 and 3 are silent.
At cycle 4 each samples its target (TS-14).
The three selector actions remain distinguishable on the REF itself. An empty
reference and a reference to B are valid designations. Invalidating the REF
clears its validity.

If A holds 1 and B holds 10, writing A and then B in one evaluation gives
the scalar reader 10 once. A later tick from A is silent and a later tick
of 11 from B reaches the reader.

A Python producer returns the same reference to A on four consecutive
evaluations. A publishes 1 at cycle 0 and 2 at cycle 2. On hgraph 0.9.8
the active REF reader observes four publications. The ordinary reader
observes 1 at cycle 0 and 2 at cycle 2. With the same producer assigning
`_output.value = a.value` instead, the REF reader observes only cycle 0 and
the ordinary reader observes the same two values.

A relay copies a selector's current REF value once. At cycle 0 the selector
holds A, whose value is 1. At cycle 1 the selector selects B, whose value is
10, while A publishes 2. At cycle 2 A publishes 3 and B publishes 11. The
relay stays idle after cycle 0 and its scalar reader observes 1, 2, 3.

## REF-CHAINED — TS-15, TS-16 and TS-17

A selector designates a dictionary. An inner input follows the selector. An
outer input is bound to the inner input, as when a nested graph's output
forwards what its input follows; it has no binding of its own to a producer.
A holds `{9:90}`. B holds `{1:10}` and adds `2:20` at cycle 1. C holds
`{1:11, 3:30}`. D has never published. The table gives the outer input.
R denotes a removal in the delta.

| Cycle | Select | Producer change | Delta | Removed | Valid / modified |
|---|---|---|---|---|---|
| 0 | A | — | 9:90 | — | true / true |
| 1 | B | B adds 2:20 | 1:10, 2:20, 9:R | 9 | true / true |
| 2 | A, then C | — | 1:11, 2:R, 3:30 | 2 | true / true |
| 3 | empty | — | 1:R, 3:R | 1, 3 | false / true |
| 4 | — | C writes 1:12 | — | — | false / false |
| 5 | B | — | 1:10, 2:20 | — | true / true |
| 6 | D | — | 1:R, 2:R | 1, 2 | false / true |

The inner input reads the same rows. So does a follower any number of
bindings further from the selector. Removals are the keys shown before the cycle and absent after
it; removed keys, values and items agree with the delta. At cycle 1 key 9
comes from A, not from B's own delta. At cycle 2 the follower compares B
with the final designation (TS-16): key 9 is neither added nor removed, and
overlapping key 1 is sampled. A set input following the key set reports the
same added and removed keys.

With scalar, window or list producers in place of the dictionaries, the outer
input follows REF-WITHDRAWAL: at cycles 3 and 6 an active consumer wakes once
and reads invalid and unmodified, with no delta.

## REF-LIST-POSITION — TS-16, TS-17 and TS-25

A holds the pair (3, −3) and B holds (10, −10). A selector initially
designates B and switches to A at cycle 2. A router receives the whole-list
reference as a list of reference inputs. Its index input selects position 1
once, at cycle 0. A second observer counts changes to the designation at
position 0, either projected directly or inside a graph taking a list parameter.

| Cycle | Selected list | Other publication | Value from saved position 1 | Designation count, direct or through parameter |
|---|---|---|---|---|
| 0 | B | A and B publish their initial pairs | −10 | 1 |
| 1 | — | — | — | — |
| 2 | A | — | −3 | 2 |
| 3 | A again | B publishes (11, −11) | — | — |
| 4 | — | A publishes (4, −4) | −4 | — |

The router stays idle after selecting its position. The designation observer
wakes when the position changes target. Value ticks of either list leave its
count unchanged.
