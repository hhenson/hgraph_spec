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
Every non-keyed reader gives the same trace: a window, a fixed or growing
list, or a bundle, whatever its children are. A SIGNAL reader follows TS-18
and is not traced here. At cycle 1 each wakes
once and reads invalid and unmodified, with nil value and no delta: no removed
samples or positions and no empty replacement. Cycles 2 and 3 are silent.
At cycle 4 each samples its target (TS-14). A set or dictionary reader
reports its removals at cycle 1 instead (REF-DICTIONARY): it reads invalid
and modified, with last modified time never. One that was showing a valid,
empty set or dictionary reads the same, with no removed members. Only a set
or dictionary that is itself the reader does this. A bundle reader whose
field `d` followed `{1:10}` and whose field `x` followed a scalar reads
invalid and unmodified at cycle 1, with no delta: `d` reports no removal.
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

A selector designates a dictionary. An inner input follows the selector. A
forwarding output shows what the inner input follows, as a nested graph's
output does for one of the graph's inputs. An outer input is bound to that
forwarding output; it has no binding of its own to a producer.
A holds `{9:90}`. B holds `{1:10}` and adds `2:20` at cycle 1. C holds
`{1:11, 3:30}`. D first publishes at cycle 7. A second selector designates B
throughout. A second inner input follows it and has its own forwarding
output. At cycle 10 the outer input is rebound from the first forwarding
output to the second. The table gives the outer input. R denotes a removal in the delta.

| Cycle | Select | Producer change | Delta | Added | Removed | Valid / modified |
|---|---|---|---|---|---|---|
| 0 | A | — | 9:90 | 9 | — | true / true |
| 1 | B | B adds 2:20 | 1:10, 2:20, 9:R | 1, 2 | 9 | true / true |
| 2 | A, then C | — | 1:11, 2:R, 3:30 | 3 | 2 | true / true |
| 3 | empty | — | 1:R, 3:R | — | 1, 3 | false / true |
| 4 | — | C writes 1:12 | — | — | — | false / false |
| 5 | B | — | 1:10, 2:20 | 1, 2 | — | true / true |
| 6 | D | — | 1:R, 2:R | — | 1, 2 | false / true |
| 7 | — | D publishes 4:40 | 4:40 | 4 | — | true / true |
| 8 | B | — | 1:10, 2:20, 4:R | 1, 2 | 4 | true / true |
| 9 | C | — | 1:12, 2:R, 3:30 | 3 | 2 | true / true |
| 10 | A; outer input rebound | — | 1:10, 2:20, 3:R | 2 | 3 | true / true |

At each of cycles 0 to 9 the first inner input reads the same row as the
outer input. So does a follower any number of forwarding outputs further
from the first selector. At cycle 10 the first inner input stays with the
first selector and moves from C to A: its delta is `9:90, 1:R, 3:R`.
Removals are the keys a follower showed before the cycle and does not show
after it. Additions are the keys it did not show before the cycle and shows
after it (TS-15). In this trace every key's child has published, so removed
keys, values and items agree with the delta. At cycle 1 key 9
comes from A, not from B's own delta. At cycle 2 the follower compares B
with the final designation (TS-16): key 9 is neither added nor removed, and
overlapping key 1 is sampled. At cycle 6 the input is bound to D, not
unbound, so D's first publication at cycle 7 reaches it as an ordinary tick.
At cycle 9 one designation moves from B to C, whose key sets overlap. The
follower showed keys 1 and 2 before the cycle and shows 1 and 3 after it,
so 2 is removed and 3 is added. Key 1 is on both sides: it is neither added
nor removed, and is sampled at 12, the value C wrote at cycle 4 while the
follower showed nothing. At
cycle 10 the selector and the outer input both move. The outer input compares
C, which it showed before the cycle, with B, which it shows after it. The
outer input shows A at neither end of cycle 10, so A contributes nothing. A set input following the key set reports the
same added and removed keys.

With any non-keyed producers in place of the dictionaries (scalars, windows,
fixed or growing lists, or bundles, whatever their children are), the outer
input follows REF-WITHDRAWAL: at cycles 3 and 6 an active consumer wakes once
and reads invalid and unmodified, with no delta.

## REF-ROUND-TRIP — TS-14, TS-15, TS-16 and TS-17

The chain holds no REF. A, B, D1 and D2 are dictionary outputs. A holds
`{1:10}` and B holds `{2:20}`. D1 and D2 have never published. F1 and F2 are
forwarding outputs. An active dictionary input is bound to F2 throughout, by
a sampled bind at cycle 0. Each cell under F1 and F2 gives what that output
forwards at the end of the cycle; `X, then Y` means two moves in one cycle; `—` means unchanged.

| Cycle | F1 forwards | F2 forwards | Producer change | Delta | Added | Removed | Valid / modified |
|---|---|---|---|---|---|---|---|
| 0 | A | F1 | — | 1:10 | 1 | — | true / true |
| 1 | B | A | — | — | — | — | true / false |
| 2 | — | F1, then A | A adds 3:30 | 3:30 | 3 | — | true / true |
| 3 | D1 | F1 | — | 1:R, 3:R | — | 1, 3 | false / true |
| 4 | D2 | — | — | — | — | — | false / false |
| 5 | A, then D1 | — | — | — | — | — | false / false |
| 6 | — | — | D1 publishes 4:40 | 4:40 | 4 | — | true / true |
| 7 | nothing | — | — | 4:R | — | 4 | false / true |

At cycle 1 F1 moves to B and F2 is rebound from F1 to A. The input showed A
before the cycle and shows A after it, so it has not moved: key 1 is not
sampled and nothing is added or removed. F1's move may already have
scheduled the input's node, which then reads the input unmodified. At cycle
2 the input leaves A for B and returns while A ticks. It reads A's own
delta, as an input bound to A alone would: key 1 is not sampled. At cycle 3
it moves from A to D1 and reports a keyed withdrawal. At cycle 4 it moves
from D1 to D2, from no value to no value: it is not notified and reads
nothing. At cycle 5 it passes through A and ends on D1. It showed no value
before the cycle and shows none after it, so it reports nothing; the move to
A may have scheduled its node, which reads the input unmodified. At cycle 6
it is bound through F2 and F1 to D1, so D1's first publication reaches it as
an ordinary tick. At cycle 7 F1 stops forwarding and the input reports its
removals, as it would for an empty REF.

With a selector REF in place of F1, designating the same outputs and empty
at cycle 7, every row is the same.

With scalar outputs A = 1 and B = 2 in place of the dictionaries, where A
publishes 3 at cycle 2 and D1 publishes 4 at cycle 6, a scalar input reads 1
at cycle 0, unmodified at cycle 1, and 3 at cycle 2. At cycles 3 and 7 it
wakes once and reads invalid and unmodified (REF-WITHDRAWAL). It is
unmodified at cycles 4 and 5 and reads 4 at cycle 6.

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
