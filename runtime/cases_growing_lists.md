# Growing list cases

Times are cycles from the run start. LIST-INPUT remains proposed.

## LIST-GROW-SHRINK — TS-12, TS-32

A growing `TSL<TS<i64>>` output, initially invalid. The node writes
positions in the cycles shown; a write to a position past the end grows the
list to include it.

| t | Action | Value | Delta: removed positions | Delta: modified positions | New size | Valid | Modified |
|---|---|---|---|---|---|---|---|
| 0 | write 0 = 10, 1 = 11 | 10, 11 | — | 0: 10, 1: 11 | 2 | true | true |
| 1 | — | 10, 11 | — | — | — | true | false |
| 2 | write 2 = 12 | 10, 11, 12 | — | 2: 12 | 3 | true | true |
| 3 | truncate to 1 | 10 | 1, 2 | — | 1 | true | true |
| 4 | truncate to 1 again | 10 | — | — | — | true | false |
| 5 | write 1 = 20 | 10, 20 | — | 1: 20 | 2 | true | true |

Truncating to the current size is silent. A shrink reports every dropped
index. Clearing a live position preserves the length.

## LIST-RETAIN — TS-32, GRF-25

Continue from a list `10, 11, 12` (three elements, last written at 2).

| t | Action | Value | Removed | Modified | Readable removed elements after the action |
|---|---|---|---|---|---|
| 3 | truncate to 1 | 10 | 1, 2 | — | position 1 holds 11 with time 0; position 2 holds 12 with time 2 |
| 3 | then grow to 3 without writing | 10, 11, 12 | — | — | none: the same elements came back with their values and times |
| 4 | — | 10, 11, 12 | — | — | none |

Growing back over a truncated position in the same cycle restores the same
element; the net delta of cycle 3 is empty and the list did not tick. From
cycle 4 a position truncated at 3 and not restored would be gone: reading it
is reading nothing, and a new element at that position is fresh.

## LIST-SHRINK-THEN-GROW — TS-32

From `10, 11, 12, 13, 14` (five elements): truncate to 3, then write
position 3 = 40 in the same cycle. Value `10, 11, 12, 40`; delta removed
`4`, modified `3: 40`. Applying the delta to a copy of the previous value —
set the length to 4, then apply the modified positions —
gives the same four elements. The write replaces the retained value at position 3.

## LIST-CLEAR-RESIZE — TS-32

| Cycle | Producer writes | Live values | Modified | Removed | New size |
|---|---|---|---|---|---|
| 0 | publish three values | 10, 20, 30 | 0: 10, 1: 20, 2: 30 | — | 3 |
| 1 | clear position 1 | 10, invalid, 30 | — | 1 | — |
| 2 | resize to 1 | 10 | — | 1, 2 | 1 |
| 3 | resize to 3 | 10, invalid, invalid | — | — | 3 |
| 4 | write positions 1 and 2 | 10, 20, 31 | 1: 20, 2: 31 | — | — |
| 5 | resize to 0 | empty | — | 0, 1, 2 | 0 |

Growing from length 1 to 3 and writing both new values in one cycle reports
size 3 and both positions in the modified map. A bounded list of length 3
has the same clear at cycle 1. Its length is fixed.
Replaying each captured delta reproduces the same values, removals, and sizes.

In the Python API, a bounded list exposes cycle 1 as the plain dictionary
`{1: REMOVE}`. A growing list exposes a `TSLDelta` on every tick. At cycle
1 it carries `removed={1}` and `size_modified=None`. At cycle 3 it carries
`size_modified=3` and empty positional changes.

## LIST-INPUT — TS-14, TS-15, TS-32

An input bound to the list above at cycle 2, sampled: it reads valid,
modified, with every element as its delta. Rebound at cycle 5 to an empty
growing list: it reads valid and modified, with positions 0 and 1 removed.
Unbound: it reads not valid, and reports every position it was showing as
removed once (TS-15 for dictionaries and sets is extended here to growing
lists; the ruling is requested).

## LIST-EMPTY-PUBLICATION — TS-9, TS-32

Start with an unpublished growing list. On each step a producer performs
the indicated operation. An independently scheduled observer reads the
parent after the producer, including on silent steps.

| Step | Operation | Length | Valid | All valid | Modified |
|---|---|---:|---|---|---|
| 1 | Apply a sparse update specifying length zero | 0 | true | true | true |
| 2 | Apply the same sparse update | 0 | true | true | false |
| 3 | Publish the whole empty list | 0 | true | true | true |
| 4 | Publish the whole empty list again | 0 | true | true | true |
| 5 | Resize to two without writing a position | 2 | true | false | true |
| 6 | Write 10 and 20 | 2 | true | true | true |
| 7 | Clear both values without resizing | 2 | true | false | true |
| 8 | Publish the whole empty list | 0 | true | true | true |

A whole empty list as the **first** publication has the same validity and
tick as step 1. At steps 3 and 4 there are no changed indices or size change.
Step 7 reports both positions as removed. Step 8 reports both dropped
positions and new size zero. For a fixed, owned list of two, steps 6 and 7
give the same validity and positional changes. TS-26 separately describes
an input assembled from its children.

Repeat steps 3 and 4 with the growing list at position 0 of an outer
fixed list. At the start of each replay, position 0 already holds a valid
empty list. An independent clock observes the parent and position 0.

| Replay input | Step 3 | Step 4 |
|---|---|---|
| In-memory recording that identifies position 0 as having published | Parent and position 0 tick | Parent and position 0 tick |
| Ordinary timed deltas carrying the same empty positional changes | Both stay quiet | Both stay quiet |

In the in-memory recording, each child publication has empty positional
changes and no new length. A parent entry naming only a sibling leaves
position 0 quiet. An explicit parent publication with no child entries
wakes the parent alone.

The [ordinary replay operator](../library/ordinary_replay_record.md) applies
the supplied delta values. Applying an empty sparse edit to a valid
position leaves the child and parent quiet under
[EMPTY-2 and EMPTY-3](../language/docs/design/empty-delta-validity.md).

## LIST-INDEX-EXTENT — TS-12, TS-32

Wire a consumer to position 1 of a growing list and observe it after each
producer step. Start with values 1, 2, 3. The selection reads 2. Truncate
to length 1. The selection becomes invalid and an idle cycle leaves it
invalid. Write 7 at position 1. The selection now reads 7. The removed
position can still be inspected through the parent's removal observation in
the truncation cycle. A live position selection follows the current extent.

## LIST-MAP-SINK — GRF-18, GRF-23

Map a sink over a growing list. Give each started child a distinct identity
and record its position on start, evaluation and stop.

| Step | Action | Lifecycle observation |
|---|---|---|
| 1 | Publish positions 0, 1, 2 | Three children start and observe their respective values. |
| 2 | Truncate to length 1 | Children 1 and 2 stop once. Child 0 stays live. |
| 3 | Append a value at position 1 | A fresh child starts there and observes the new value. |
| 4 | Stop the run | Child 0 and the new child at 1 each stop once. |

The same trace holds for a graph body that contains the sink and returns
no output. Python graph bodies may declare `-> None` or omit the return
annotation. The trace does not prescribe an ordering between siblings.

## Outside these cases

A growing list of collections (a list of bundles) follows the same rules per
element. Inserting or removing in the middle is not a growing-list
operation: a growing list changes only at its end.
