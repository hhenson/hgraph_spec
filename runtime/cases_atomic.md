# Atomic cases

Status: the retention and zero cases are proposed for TS-1, TS-2 and TS-3.
The scalar invalidation case records the agreed TS-7 observation.

## ATOMIC-RETENTION

One owned `TS<i64>`, initially invalid; at most one admitted publication per
cycle. Binding, invalidation, concurrency, allocation failure and batching
are excluded. Begin advances time, retaining value and stamp. Publish sets
both before observation. Observe changes nothing.

Each row is read after its action. `—` means absent, not zero.

| Action | Value | Valid | Modified | Last modified | Delta |
|---|---|---|---|---|---|
| begin(10) | — | false | false | never | — |
| publish(7) | 7 | true | true | 10 | 7 |
| observe again | 7 | true | true | 10 | 7 |
| begin(20) | 7 | true | false | 10 | — |
| begin(30) | 7 | true | false | 10 | — |
| publish(7) | 7 | true | true | 30 | 7 |
| begin(40) | 7 | true | false | 30 | — |
| publish(9) | 9 | true | true | 40 | 9 |

Idle cycles keep the value. Reads do not consume the delta. The equal
publication at 30 still ticks. [Conformance](conformance.md) distinguishes
publication from a setter that may suppress equal results.

## ATOMIC-ZERO

Start fresh at offset 0, publish 0, then begin at offset 1:

| Observation | Value | Valid | Modified | Last modified | Delta |
|---|---|---|---|---|---|
| before publication | — | false | false | never | — |
| after publication | 0 | true | true | 0 | 0 |
| next cycle | 0 | true | false | 0 | — |

Neither zero is `never`. The [layout](layout_example.md) has its own encoding.

## ATOMIC-INVALIDATION

TS-7. Start with an owned scalar and a directly bound input. Observe after
the producer finishes each cycle. An independent probe makes every row
readable even if the input is passive.

| Cycle | Producer action | Valid | Modified | Invalidated | Last modified |
| --- | --- | --- | --- | --- | --- |
| 1 | Clear before the first value | false | false | false | never |
| 2 | Publish 7 | true | true | false | 2 |
| 3 | Clear | false | false | true | never |
| 4 | Idle | false | false | false | never |
| 5 | Clear again | false | false | false | never |
| 6 | Publish 8 then clear | false | false | true | never |
| 7 | Publish 9 | true | true | false | 7 |
| 8 | Clear then publish 10 | true | true | false | 8 |

At cycle 3 an active input can wake its node on the clear. A passive input
can read the same transition on an independent wake that cycle. A probe
that first wakes at cycle 4 sees an invalid value and `invalidated == false`.
