# Window cases

Status: proposed 2026-09-30, reasoned from TS-13, TS-30 and TS-31 and from
hgraph's window storage; not yet run. Times are cycles from the run start.
`—` means no publication.

## WINDOW-TICK — TS-13, TS-30, TS-31

A tick window of size 3 with minimum 2 over `TS<i64>`. The node writes one
value in the cycles shown.

| t | Written | Value, oldest first | Delta | Removed value | Valid | All valid | Modified |
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
they are 2, 4, 5.

## WINDOW-DURATION — TS-30, TS-31

A duration window with span 3 cycles and minimum span 2 over `TS<i64>`.

| t | Written | Value, oldest first | Delta | Removed value | All valid |
|---|---|---|---|---|---|
| 0 | 1 | 1 | 1 | — | false |
| 1 | — | 1 | nil | — | false |
| 2 | 2 | 1, 2 | 2 | — | true |
| 6 | — | 1, 2 | nil | — | true |
| 7 | 3 | 3 | 3 | 1, 2 | true |

Eviction happens only on arrival: at 6 the values are older than the span
but nothing has arrived, so the window neither ticks nor shrinks. At 7 the
arrival drops every value older than the span measured from 7; the removed
values are both readable in that cycle. All valid, once reached, is not lost
by eviction.

## WINDOW-EQUAL-VALUE — TS-30

A tick window of size 2. Writing 4 then 4 gives values `4, 4`, two ticks and
two deltas of 4; an equal arrival is an arrival.

## Outside these cases

Clearing a window and timer-driven eviction (hgraph RFCs 0006 and 0007) are
deferred and optional. A window inside a collection, and a window bound
through a reference, follow the collection and reference rules and are not
repeated here.
