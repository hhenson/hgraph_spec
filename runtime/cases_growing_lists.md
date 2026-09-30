# Growing list cases

Status: proposed 2026-09-30, reasoned from TS-12, TS-32 and GRF-25 and from
hgraph's RFC 0031; not yet run. Times are cycles from the run start.

## LIST-GROW-SHRINK — TS-12, TS-32

A growing `TSL<TS<i64>>` output, initially invalid. The node writes
positions in the cycles shown; a write to a position past the end grows the
list to include it.

| t | Action | Value | Delta: removed positions | Delta: modified positions | Valid | Modified |
|---|---|---|---|---|---|---|
| 0 | write 0 = 10, 1 = 11 | 10, 11 | — | 0: 10, 1: 11 | true | true |
| 1 | — | 10, 11 | — | — | true | false |
| 2 | write 2 = 12 | 10, 11, 12 | — | 2: 12 | true | true |
| 3 | truncate to 1 | 10 | 1, 2 | — | true | true |
| 4 | truncate to 1 again | 10 | — | — | true | false |
| 5 | write 1 = 20 | 10, 20 | — | 1: 20 | true | true |

A position that joins is a modified position with its whole value as delta.
Removed positions are contiguous from the end. Truncating to the current
size is not a tick.

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
truncate to the lowest removed position, then apply the modified positions —
gives the same four elements. Position 3's new element is fresh, not the
retained 13.

## LIST-INPUT — TS-14, TS-15, TS-32

An input bound to the list above at cycle 2, sampled: it reads valid,
modified, with every element as its delta. Rebound at cycle 5 to an empty
growing list: it reads valid and modified, with positions 0 and 1 removed.
Unbound: it reads not valid, and reports every position it was showing as
removed once (TS-15 for dictionaries and sets is extended here to growing
lists; the ruling is requested).

## Outside these cases

A growing list of collections (a list of bundles) follows the same rules per
element. Inserting or removing in the middle is not a growing-list
operation: a growing list changes only at its end.
