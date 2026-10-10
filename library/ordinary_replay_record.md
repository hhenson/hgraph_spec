# Ordinary replay and recording

Status: proposed library data contract, 2026-10-03.

Replay and record use ordinary values for the finite publication profile in the
[ordinary delta type extension](../language/docs/design/ordinary-delta-types.md)
and its collection contract. This includes the admitted scalar leaves and the
admitted structural shapes, including [bytes](../language/docs/design/bytes-values.md)
leaves and keys, plus the [finite atomic profile](../language/docs/design/atomic-delta-publications.md).
Both use `list<TimedValue<T>>`, where T is the
temporal shape and the value field carries its publication delta. Signals,
references, windows and the other excluded shapes remain outside that profile.

## Data and callable contracts

The library declares an ordinary generic nominal struct:

```hgl
export struct TimedValue<T> {
    time: datetime
    value: delta<T>
}

operator replay<T>(const values: list<TimedValue<T>>) -> T

operator record<T>(ts: T, const key: str)
```

`TimedValue<T>` is an ordinary complete value with two required fields, not
a runtime category or a temporal struct delta. Its ordinary construction,
retention and nominal identity rules apply. Replay's const parameter is an
unbounded ordinary list, not a temporal list endpoint. Its element type fixes
the temporal result T, including for an empty list. Record's temporal input
fixes T and its ordinary recording type `list<TimedValue<T>>`. These are exact
type relationships; no new inference, conversion or generic-call syntax is
needed. A fixed-size ordinary list is not implicitly converted to this type.

T names the originating temporal shape, not an already-derived payload type.
For example, the value field of `TimedValue<map<i64, i64>>` has exact ordinary
type `delta<map<i64, i64>>`, while that of `TimedValue<i64>` is i64 by scalar
reduction. This replaces the former payload-type parameter convention; there
is no compatibility alias or second TimedValue definition. The field's
`delta<T>` formation requirement restricts T to the admitted profile.

Every entry holds publication-delta data and an absolute time. Applying its
data follows the publication profile, including
[empty sparse application](../language/docs/design/empty-delta-validity.md).
An empty delta entry is present data but may cause no tick on a valid target;
record retains only actual ticks. An admitted atomic empty-list value is a
present complete publication. Omitting an entry
expresses silence. Scalar false, zero, empty text and empty bytes remain present values,
and equal scalar publications at distinct times remain distinct ticks.
Neither `_` nor `null` is an ordinary list element. An empty timed list
publishes nothing; it carries no dense horizon.

`delta_value(ts)` extracts the publication to record. `delta<T>(...)` is
delta construction; `delta<T>` is the ordinary field type, reducing to T
for the admitted scalar leaves and to V for admitted `atomic<V>`.
A complete held structural T value
cannot replace that delta field.

## Replay execution and time boundaries

The ordinary data can drive the existing generator-source contract: traverse
the list in order, yielding each entry's absolute time and delta payload.
The [source example](../language/examples/ordinary-replay-record.hgl) gives
that complete execution body using ordinary length, indexing and an i64
cursor. It does not add a replay-data injectable or a graph execution API.

1. Visit entries in list order. Each time must strictly exceed the preceding
   entry's time, including past entries. Check when the yield is reached,
   after both operands; equal or decreasing times raise a node error.
2. An admitted past entry skips; a due entry publishes; a future entry
   suspends until its time. Replay neither sorts nor merges entries and
   publishes no held values between entries.
3. ENG-3 bounds execution: start is inclusive and end exclusive. An entry at
   or after run end does not publish; it adds no replay-specific error.

List length and indexed access retain their ordinary representability and
bounds rules. The source's cursor never indexes at length: it tests before
each read and increments only after processing an existing entry. Engine
time representation and sentinel bounds retain ENG-16 for scheduled and
published times. A past absolute entry that passes ordering is skipped before
scheduling; for
example, the absolute 1969 instant in the existing pull-source example is
skipped, not rejected as a replay input.

## Recording lifecycle and ownership

Record requests ordinary `global_state` and `clock`. Before start, bind its
const key to exact ordinary type `list<TimedValue<T>>` under
[ADR 0016](../language/docs/design/decisions/0016-eval-scalar-buffer-capabilities.md).
Binding alone does not initialize the entry. At recorder start, set a newly
constructed typed empty list into that entry. Thus a started recorder with
no admitted input ticks leaves an empty recording, not an absent entry.

On each admitted input publication, construct
`TimedValue<T>(time: clock.evaluation_time, value: delta_value(ts))` and push
it onto an exclusive writable borrow of the typed list. The single-input
default handler establishes valid and modified for that delta read. The
timestamp is evaluation time, not wall time. Push independently retains the
entry; later endpoint updates cannot change any earlier capture. Every
admitted tick is appended, including equal scalar publications. An unmodified
input appends nothing.

Each borrow ends with its hook. Record needs no stop write or flush: completed
pushes already changed the ordinary entry. After all graph stop hooks, the
run owner may read the exact typed entry and retain an independently owned
result before destroying run storage. Missing data or run failure is not a
successful empty recording. Failed set, construction or push follows its
ordinary failure contract; a failed push leaves prior captures unchanged
and earlier successful effects are not rolled back.

This describes one recorder's lifecycle without competing writes to its
entry. It neither reserves keys nor imposes a store-wide single-writer rule.
Eval protects its generated keys through pre-start
[fresh-key selection](../language/docs/design/eval-recorder-keys.md).
Independent record callers still use their supplied keys under ordinary
global-state semantics. Checkpoint/restart and accumulation
across separate runs are outside this profile; generator restart behavior
does not establish recording recovery.

## Eval adaptation

For each scalar temporal parameter, eval maps present dense slot i to an
ordinary entry whose time is the run start plus i minimum engine steps and
whose value is that slot's payload. It passes the resulting typed list as
replay's const argument. A silent slot produces no entry, including when
surrounded by false, zero or empty text.

Eval keeps each original dense sequence length separately. The longest
length is the input horizon even if no entry was produced. It obtains the
ordinary recording after stop and materializes the dense result under
EVAL-5, preserving trailing silent positions and any later output ticks.
An empty input and an all-silent input can therefore use identical empty
timed data while producing different dense result lengths. Materialization
does not make replay publish silence or make the recorder append absent
entries. Unrepresentable dense lengths or slot-time arithmetic are not valid
configurations; the phase and diagnostic for rejecting them remain part of
the separate eval admission extension.

Eval configures these normal operators without exposing configuration work
to its caller. They remain independently callable library operators. This
mapping settles the scalar ordinary storage dependency without introducing
nullable ordinary values. Structural delta storage uses the separate
ordinary delta type extension.

[Cases](../runtime/cases_ordinary_replay_record.md) state expected observations.
[Timed-value evidence](https://github.com/hhenson/hgraph_spec_audit/blob/78e44d6876dd4cbb7c805bca5f03c949fbcd4198/runtime/validation/eval_timed_values/README.md)
supports the valid scalar traces and separate horizon; it does not establish
malformed-input admission. The
[ordinary-value evidence](https://github.com/hhenson/hgraph_spec_audit/blob/78e44d6876dd4cbb7c805bca5f03c949fbcd4198/runtime/validation/value_sequences/README.md)
supports the retention direction. The normative ownership rules remain the
ordinary value contracts, independent of any particular representation.
