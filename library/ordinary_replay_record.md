# Ordinary scalar replay and recording

Status: proposed library data contract, 2026-10-03.

Replay and record use ordinary values. This profile admits `bool`, `i64`,
`f64`, `str`, `date`, `time`, `datetime` and `duration`; each has an ordinary
scalar publication delta of its own type. Structural deltas, signals,
references and windows require separate storage contracts.

## Data and callable contracts

The library declares an ordinary generic nominal struct:

```hgl
export struct TimedValue<T> {
    time: datetime
    value: T
}

operator replay<T>(const values: list<TimedValue<T>>) -> T
requires T in {bool, i64, f64, str, date, time, datetime, duration}

operator record<T>(ts: T, const key: str)
requires T in {bool, i64, f64, str, date, time, datetime, duration}
```

`TimedValue<T>` is an ordinary complete value with two required fields, not
a runtime category or a temporal struct delta. Its ordinary construction,
retention and nominal identity rules apply. Replay's const parameter is an
unbounded ordinary list, not a temporal list endpoint. Its element type fixes
the scalar result T, including for an empty list. Record's temporal input
fixes T and its ordinary recording type `list<TimedValue<T>>`. These are exact
type relationships; no new inference, conversion or generic-call syntax is
needed. A fixed-size ordinary list is not implicitly converted to this type.

Every entry describes one present scalar publication at an absolute time.
Omitting an entry expresses silence. False, zero and empty text are present
values, and equal payloads at distinct publication times remain distinct
ticks. Neither `_` nor `null` is an element of this ordinary data contract.
An empty timed list publishes nothing; it carries no dense horizon.

`delta_value(ts)` extracts the scalar publication to record. `delta<T>(...)`
remains contextual delta construction; `delta_of(T)` remains specification
notation and cannot be used as the recording's source element type.

## Replay execution and time boundaries

The ordinary data can drive the existing generator-source contract: traverse
the list in order, yielding each entry's absolute time and scalar payload.
The [source example](../language/examples/ordinary-replay-record.hgl) gives
that complete execution body using ordinary length, indexing and an i64
cursor. It does not add a replay-data injectable or a graph execution API.

For increasing times within the run, each entry publishes once at its time;
no intervening instant publishes a held value. The existing generator rules
apply to other orderings: an entry in the past when encountered is skipped,
an entry equal to the current time publishes immediately, and a second
publication at the same time fails. Replay does not sort or merge entries.
The engine's inclusive start and exclusive end bound execution (ENG-3): a
scheduled entry at or after the end is not published. No replay-specific
rejection of such a future entry is added. This does not require a cycle at
every silent instant.

List length and indexed access retain their ordinary representability and
bounds rules. The source's cursor never indexes at length: it tests before
each read and increments only after processing an existing entry. Engine
time representation and sentinel bounds retain ENG-16 for scheduled and
published times. A past absolute entry is skipped before scheduling; for
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
entry; later endpoint updates cannot change any earlier capture. Equal
payloads are appended on each tick. An unmodified input appends nothing.

Each borrow ends with its hook. Record needs no stop write or flush: completed
pushes already changed the ordinary entry. After all graph stop hooks, the
run owner may read the exact typed entry and retain an independently owned
result before destroying run storage. Missing data or run failure is not a
successful empty recording. Failed set, construction or push follows its
ordinary failure contract; a failed push leaves prior captures unchanged
and earlier successful effects are not rolled back.

This describes one recorder's lifecycle without competing writes to its
entry. It neither reserves keys nor imposes a store-wide single-writer rule.
Same-type key collisions, protection from other writers and eval key
selection remain a separate policy. Checkpoint/restart and accumulation
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
nullable ordinary values or structural delta storage.

[Cases](../runtime/cases_ordinary_replay_record.md) state expected observations.
[Timed-value evidence](https://github.com/hhenson/hgraph_spec_audit/blob/78e44d6876dd4cbb7c805bca5f03c949fbcd4198/runtime/validation/eval_timed_values/README.md)
supports the valid scalar traces and separate horizon; it does not establish
malformed-input admission. The
[ordinary-value evidence](https://github.com/hhenson/hgraph_spec_audit/blob/78e44d6876dd4cbb7c805bca5f03c949fbcd4198/runtime/validation/value_sequences/README.md)
supports the retention direction. The normative ownership rules remain the
ordinary value contracts, independent of any particular representation.
