# Nullable scalar time series

Agreed 2026-11-17. Rules: TS-1, TS-7, TS-29, TS-33, TS-34.

## NULLABLE-LIFECYCLE

Wiring: `commands -> produce -> sample -> record`; `commands -> sample.clock`.
Each command ticks the clock, so sampling continues through silence and
invalidation. `sample` has no validity requirement on `produce`.

`produce` publishes a positive command, publishes null for `0`, writes nothing
for `-1`, and invalidates for `-2`. Its output is `TS<Optional<i64>>`.
`sample` records validity, modification and current value after `produce` runs.

| Tick | Command | Current value | Valid | Modified | Producer delta |
|---|---:|---|---|---|---|
| 1 | 7 | 7 | true | true | 7 |
| 2 | 0 | null | true | true | null |
| 3 | -1 | null | true | false | absent |
| 4 | -2 | absent | false | false | absent |
| 5 | -1 | absent | false | false | absent |
| 6 | 9 | 9 | true | true | 9 |
| 7 | 0 | null | true | true | null |
| 8 | 0 | null | true | true | null |

Null is a value in a valid publication. Absent means no publication payload.
Invalidation notifies even though modified is false.

## NULLABLE-WIDENING

Use the same wiring with a plain `TS<i64>` producer. It publishes non-negative
commands, stays silent for `-1`, and invalidates for `-2`. `sample` still accepts
`TS<Optional<i64>>`. For commands `[7, -1, -2, -1, 9]`, it observes:

| Tick | Value | Valid | Modified |
|---|---|---|---|
| 1 | 7 | true | true |
| 2 | 7 | true | false |
| 3 | absent | false | false |
| 4 | absent | false | false |
| 5 | 9 | true | true |

No null is synthesized. An overload accepting only the nullable input is
eligible; an exact plain-input overload wins when both are available.
A nullable producer cannot bind to a plain input. Widening cannot change the
payload type. Reference retargeting to a valid plain source samples its current
value, and subsequent source ticks remain visible.

## NULLABLE-RETENTION

Wire `commands -> produce -> record`. Let command position `i` occur at
`t_i = t_0 + i * step`. The recorder appends only publications;
[dense eval materialization](../library/ordinary_replay_record.md) supplies
silent positions through the stated final timestamp.

| Producer | Commands | Recorded entries | Dense eval result | Final timestamp |
|---|---|---|---|---|
| Nullable | `[5, 0, -1, -2, 7]` | `[(t_0, 5), (t_1, null), (t_4, 7)]` | `[5, null, absent, absent, 7]` | `t_4` |
| Plain, widened to nullable | `[5, -1, 7, 11]` | `[(t_0, 5), (t_2, 7), (t_3, 11)]` | `[5, absent, 7, 11]` | `t_3` |

Earlier recorded payloads and presence states survive the later writes and
graph teardown. Repeat with a composite payload: later writes must not alter
earlier fields. An empty recording remains empty; no null publication is
invented.

Physical layout and timestamp packing are not specified here.
