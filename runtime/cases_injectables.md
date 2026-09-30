# Injectable cases

Status: proposed 2026-09-30. These cases follow INJ-1 to INJ-14 and the Node
rules they reference.

## INJECT-SIGNATURE — INJ-1, INJ-2, INJ-4

Two nodes with the same inputs, output and scalars, one of which injects the
clock, logger, scheduler and its output. Wiring treats them identically: the
same calls wire both, and a caller cannot tell which is which. A node whose
type does not request the scheduler and whose implementation uses one fails
when the graph is instantiated, not silently at run time.

## INJECT-CLOCK — INJ-5, INJ-6, INJ-7, INJ-11

See ENGINE-CLOCK: every node of every graph in the run reads one clock;
evaluation time and next cycle time are the same for all of them in a cycle;
nothing injected can change it.

## INJECT-OUTPUT — INJ-8, NOD-22

A node that injects its output. In start it reads its output: not valid,
value nil, and the output's type. In eval at 0 it writes key `a` of a
dictionary output, and at 1 key `b`; each cycle's delta holds that key
only, and the value accumulates both. In stop it reads its output: both
keys present. No other node reaches this output except by binding to it.

## INJECT-STATE — NOD-8, NOD-9, NOD-10, INJ-3

A node keeps a running total in recordable state and a count of evaluations
in state. Evaluated with 1, 2, 3 it publishes 1, 3, 6. Each write to
recordable state ticks it, and a node bound to the recordable state sees 1,
3, 6 on the same cycles. Writing recordable state never schedules the
writing node. A node whose recordable state was restored to 3 before start
publishes 4 when it next sees 1: start did not overwrite the restored value
(this variant needs the optional restore facility, and is recorded for it).

## INJECT-LOGGER — INJ-9, INJ-12

A node logs at debug in every eval; the run's level is info. The node's
ticks are unchanged whether or not the level is enabled, and asking whether
debug is enabled answers false. The message text is never built when it is
not.

## INJECT-ENGINE-CONTROL — INJ-10

See ENGINE-STOP-REQUEST. Engine control also reports the run's mode, start
time and end time as configured.

## INJECT-ALARM — NOD-25, NOD-26, INJ-4

See SOURCE-ALARM. A node that injects the alarm and the scheduler is
refused; a node that injects the alarm has no scheduler to query.

## INJECT-GLOBAL-STATE — INJ-3, INJ-5, INJ-13, INJ-14

The owner seeds an i64 entry before graph start. A node reads it in start,
updates it during evaluation and updates it again in stop. The owner reads
the final i64 after all stop hooks. A nested graph's node sees prior writes
to the same key, while a separate run using the same graph description and
key sees only its own seed and writes. These store writes alone schedule no
node and publish no endpoint.

Reading stored false, zero or empty text with the corresponding expected
scalar type succeeds and preserves that value. Reading a missing key fails;
reading an i64 entry as bool fails. Neither failure yields null. A source
read lacking a concrete expected ordinary type is rejected during checking.
An owned scalar already read remains unchanged when the entry is replaced.
These cases do not specify aggregate aliases or borrowing.
