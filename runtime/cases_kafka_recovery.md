# Kafka initial recovery and reconnect

Status: required behavior, 2026-10-28. These cases describe the Kafka
extension's recovery contract, not a requirement for all push sources.

## KAFKA-RECOVERY-1 — completion is monotonic

The legacy `recovered` output describes completion of initial recovery for
one subscription lifetime. Before `Live` or `BoundedComplete` it is false.
After either state it remains true, including during retries, reassignment,
failure, and shutdown. A newly created subscription has its own marker.
Connection status remains available independently through subscription state.

| Event | Subscription state | recovered value | recovered tick |
|---|---|---|---|
| Start | Starting | false | false |
| Disconnect before completion | Retrying | false | none |
| Read initial history | Recovering | false | none |
| Initial history drained | Live or BoundedComplete | true | true |
| Disconnect after completion | Retrying | true | none |
| Reassignment after initial recovery | Live or BoundedComplete | true | none |
| Resume live input | Live | true | none |
| Failure | Failed | true | none |
| Shutdown | Stopped | true | none |

## KAFKA-RECOVERY-2 — manual assignment survives connection loss

A broker disconnect does not revoke a manual assignment. Once initial
recovery completes, reconnecting preserves that assignment, its generation,
and its fetch positions. The configured startup position is not reapplied.
Both arrival-clock and record-timestamp recovery follow this rule.

Given one partition containing `history` at offset 0, an earliest-start,
unbounded subscription with commits disabled produces this trace:

| Event | Record output | State |
|---|---|---|
| Initial snapshot read | history, offset 0 | Recovering |
| Snapshot drained | none | Live |
| All brokers disconnect | none | Retrying |
| Brokers reconnect | none | Live |
| Append after at offset 1 | after, offset 1 | Live |

The record sequence is exactly `history, after`; recovery does not restart,
and both delivered cursors identify the same assignment generation.
The rule does not change group-rebalance ownership or commit policy.

## KAFKA-RECOVERY-3 — group reassignment preserves the completed phase

Initial recovery completes once per subscription lifetime. Group revocation
may emit `Retrying`, but a subsequent assignment returns to `Live` without
entering `Recovering` again. The new assignment has a fresh generation and
uses the configured offset-resolution and commit rules. Any records delivered
under it are live input, including redelivery required by those rules; they
do not re-enter record-time recovery.

A subscription that has reached `BoundedComplete` remains complete after
reassignment and emits no further records.

| Event | State | Assignment generation | recovered value |
|---|---|---|---|
| Initial assignment to an empty partition | Recovering | A | false |
| Initial snapshot drained | Live | A | true |
| Brokers disconnect until the group session expires | Retrying | A revoked | true |
| Brokers reconnect and the group reassigns the partition | Live | B, distinct from A | true |
| A new record arrives | Live | B | true |

The same state trace applies to arrival-clock and record-timestamp recovery.
