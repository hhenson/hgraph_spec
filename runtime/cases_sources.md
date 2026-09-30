# Source cases

Status: proposed 2026-09-30, reasoned from the Node rules for schedulers,
pull sources and push queues; not yet run. Times are cycles from the run
start; the run ends after cycle 9 unless stated.

## SOURCE-ALARM — NOD-4, NOD-25, NOD-26, NOD-27

A pull source on the alarm. In start it asks to be evaluated 2 cycles after
the start time; in each eval it publishes a count and asks again 2 cycles on,
until it has published 3 times.

| t | Evaluated | Publishes |
|---|---|---|
| 0 | no | — |
| 2 | yes | 1 |
| 4 | yes | 2 |
| 6 | yes | 3 |
| 8 | no | — |

The node has no inputs, so it is admitted whenever scheduled (NOD-27) and
is never evaluated otherwise. Asking twice in one eval, for 2 cycles on and
for 5, keeps the earliest: the node is evaluated at the earlier time, and
not at the later one, because the alarm kept nothing (NOD-25). A request
for a past time is refused. A node that injects both the scheduler and the
alarm is refused when the graph is described (NOD-26).

## SOURCE-SCHEDULER-REPLACE — NOD-12, NOD-13, NOD-14

A pull source on the scheduler. In start it requests the start time. In eval
at 0 it requests, with tag `a`, 3 cycles on; at 3 it requests `a` 5 cycles on
and, without a tag, 1 cycle on.

| t | Scheduled now | Due requests removed | Next entry written |
|---|---|---|---|
| 0 | yes | the start request | 3 |
| 3 | yes, by `a` | `a` at 3 | 4 (the untagged one is earliest; `a` at 8 stays pending) |
| 4 | yes, untagged | the untagged request | 8 |
| 8 | yes, by `a` | `a` at 8 | none |

Replacing `a` while it is due does not lose the new request (GRF-14, Graph
"Scheduling a node"). Asking whether the node is scheduled now, and by which
tag, answers as the table says.

## SOURCE-SCHEDULER-CANCEL — NOD-12, Graph point 8

As above, but at 3 the node cancels `a` and requests nothing. The graph's
entry for 3 was already used; nothing further is written. The node is not
evaluated again. This case records the accepted observation of Graph point
8 in its other form: the cancelled request's cycle *is* the current one, so
there is no spurious later wake to observe here; the spurious wake arises
only when the cancelled request was for a later time already written to the
schedule. Second variant: at 3 request `b` 4 cycles on, at 4 cancel `b`. The
schedule entry for 7 stays; at 7 the node is considered, evaluated (it has
no inputs) with nothing due, and the observation is recorded either way
until point 8 is ruled on.

## PUSH-ONE-AT-A-TIME — NOD-17, NOD-23, NOD-24, ENG-8

Real time. A push source whose queue hands over one event at a time.
Before the first cycle a sender on another thread admits 1, 2 and 3. The
first cycle evaluates the source, which publishes 1 and asks for another
cycle; the next two cycles publish 2 and 3, each one smallest step after the
last at the earliest, and before any other scheduled work at those times.
Order of admission is order of publication.

## PUSH-BURST — NOD-24

As above with a queue that hands over everything at once: one cycle
publishes the tuple `(1, 2, 3)`. Events admitted after the queue was
emptied form the next tuple. An empty tuple is never published: a cycle at
which nothing is pending evaluates the source only if something else
scheduled it, and then it publishes nothing.

## PUSH-COMBINED — NOD-24

A queue that folds pending events into one current value (the last value
wins, for a scalar): admitting 1, 2 and 3 before a cycle publishes 3 once.

## PUSH-BOUNDED — NOD-17

A queue bounded at 2. A sender admits 1 and 2; a third non-waiting admission
is refused; a waiting admission is admitted once the source has taken one.
Capacity is released when an event is taken, not when downstream work is
done. Refusal is the only back-pressure: nothing else in the run observes
the full queue.

## PUSH-STOP — NOD-18

When the run stops, the sender is closed before the source's stop is
called. An admission after closing is refused without waiting; a sender that
was waiting for room is released, refused. The source's stop then runs.

## PUSH-REFUSED-IN-SIMULATION — ENG-7, NOD-16

A description containing a push source is refused by a simulation run
before anything starts, and is refused inside a nested graph in any mode.

## Outside these cases

Protocol acknowledgements after downstream processing are the protocol's
business, wired as a sink; they do not change queue capacity. A push
source's placement at the lowest ranks is a description rule (GRF-5) checked
at instantiation.
