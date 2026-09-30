# ADR 0016: typed scalar buffer capabilities for replay and record

Status: proposed source extension, 2026-09-30. This proposal uses
node-scoped typed injectables rather than general resource types.

This builds on [eval operator composition](../eval-operator-composition.md).
Its admitted profile is a fresh, finite, dense eval of `bool`, `i64`, `f64`,
`str`, `date`, `time`, `datetime`, or `duration`. In this profile a TS delta
is its scalar value. Collection and structured types, `atomic` wrappers,
references, signals, windows, other scalars, persistence, checkpointing and
restarting a stopped instance are not admitted by this extension. The separate
[collection eval extension](../eval-collection-deltas.md) widens the shape
and contextual-payload domain for ordinary nonempty publication traces.

## Operator signatures

The standard library declares these ordinary operators:

```hgl
operator replay<T>() -> T
requires T in {bool, i64, f64, str, date, time, datetime, duration}

operator record<T>(ts: T)
requires T in {bool, i64, f64, str, date, time, datetime, duration}
```

`replay` is a source and `record` a sink. These are the profile's overloads,
not a prohibition on other independently specified library overloads.
The result context of each replay resolves T to the corresponding target
parameter; the record input resolves T to the target result. All such types
are concrete before graph construction. This adds neither explicit generic
function-call syntax nor a first-class delta type.

Eval wires and configures these operators. Its caller still writes
`eval(pass_through, value: [...])`, with no resource, recording key or
operator configuration argument. Normal operator resolution applies. The
HGL bodies below determine publication and capture; the capabilities supply
only the storage operations defined by this contract.

## Construction and typed binding

Two contextual injectable names are added: `replay_input` and `capture`.
They borrow the buffers owned by the enclosing eval run. They are not
ordinary source values, native atomic types, const arguments, time series,
state or cache. They have no constructor, equality, serialization, address,
or process-global lookup operation.

- `replay_input` is admitted only in a runtime source with no temporal
  parameters and one result in the eight-type domain above. Its payload
  type T is exactly that result type. The source still needs `alarm` or
  another already-admitted source mechanism; storage access is not a wake-up.
- `capture` is admitted only in an outputless runtime sink with exactly one
  temporal parameter in that domain. Its payload type T is exactly that
  parameter's type. This profile's operator names that parameter `ts`.
- A node cannot request both capabilities. They are not admitted in value
  functions, native value-helper signatures or composition bodies, and their
  requirements do not propagate transitively through value helpers in this
  slice. Approved indexing and capability functions are the only storage-access primitives.
- A borrowed capability cannot be assigned, returned, passed as an ordinary value,
  put in a closure, or kept in state/cache. No capability reference may be
  retained outside its call. Its owned scalar results
  follow the ordinary value rules instead.

While constructing the graph, eval binds each replay node instance to its
input buffer and each record node instance to its capture buffer. A binding
identifies the run and node, its role and exact scalar type. Validate all
required bindings before starting any node. Missing binding, wrong run,
wrong role/type, or two writers bound to one capture fails graph construction.
A body using these capabilities outside a configured graph therefore fails
construction; it never falls back to ambient or process-global storage.
Another graph builder may supply equivalent bindings under this contract.
A provider is the graph-construction binding that grants a node access to
its run-owned buffer; it is not an ordinary HGL value.

Input buffers are finite immutable sequences of typed present/absent slots.
Every position counts, including absent ones. Their length must fit a
nonnegative i64. For a nonempty buffer the final dense instant, run start
plus `(length - 1)` engine steps, must be representable; reject invalid
configuration at construction instead of wrapping an index or time.
This adds no new general i64 overflow rule.

Each capture is initially unbegun and empty, belongs to one run and has one
writer. Binding it does not begin a recording: the HGL record-node start
hook does that. Retain the sink as part of the evaluated graph even when no
input position contains a tick. Empty input does not bypass graph start/stop.
Physical sharing of immutable input data is allowed; mutable cursor and
capture state cannot leak between nodes or separate eval invocations.

## Exact capability operations

These use [receiver-first capability functions](../capability-function-syntax.md),
not methods or ordinary HGL resource-type declarations. T below is the
concrete contextual payload type.
Names and argument types are fixed by this extension; ordinary positional
and named argument binding applies.

| Signature | Allowed phase | Result or effect |
|---|---|---|
| `len(replay_input) -> i64` | start, evaluation | Number of slots, including absent slots. |
| `replay_input[index]`, with i64 index | evaluation | Owned scalar delta when present; `null` for an absent in-bounds slot. |
| `begin(capture)` | start | Mark the configured empty recording begun. No output, scheduling or publication. |
| `append(capture, time: datetime, delta: T)` | evaluation | Append one time and independent owned scalar delta to a begun recording. |

No replay/capture operation is admitted in stop. No operation schedules, publishes an
endpoint, deduplicates values, applies a delta to an output, advances a
cursor, inserts no-tick cells, or performs dense-result padding. HGL determines
when to call these storage operations.

A present indexed read returns an owned scalar result, including for str (ADR 0009).
`append` copies the scalar and time before returning. Later output changes,
input destruction, another run and graph teardown cannot change an earlier
successful capture (VAL-17). This is independence, not mandated allocation.
The run owner keeps configured storage alive through graph stop and result
extraction. The returned eval sequence is owned independently of disposed
graph storage. Extraction may copy or transfer owned storage; no borrowed
hook view escapes in either case.

## Indexed replay reads

`replay_input[index]` reads one configured slot. The zero-based i64 index
counts every position, including `_` slots. For `[10, _, 12]`,
`len(replay_input)` is 3 and reads yield present 10, `null`, and present 12.
An in-range absent read succeeds; it does not return a held or default value.

The result is contextually nullable. Bind it with `let item = replay_input[index]`
and establish `item != null` before using its scalar payload. The
[nullable indexing contract](../nullable-replay-indexing.md) defines branch
facts, short-circuit propagation, alias rules and rejected unproven uses.
It introduces no ordinary nullable source type and does not redefine null
field clearing or general runtime returns.

The read is evaluation-only and does not publish, schedule, consume or move
a cursor. Repeated reads preserve slot contents with independent ownership.
The index need not be the current cycle number; the replay body determines
how positions map to evaluation times. `delta_value(endpoint)` instead reads
a live temporal endpoint's current publication. The old separate replay
presence/read functions are not admitted or retained as aliases.

## Failure and validation

Unsupported host types, phases, or capability escapes are checking errors.
Invalid provider configuration is a graph-construction error, before any
node starts. Diagnostics name the capability and the rejected requirement.
No failure is converted into a default scalar, an absent slot, or successful
empty output.

Within an admitted hook, fallible capability operations use the `translated`
error policy and [node error model](0009-native-errors-and-the-node-error-model.md). During
evaluation a failure ends that evaluation and follows the node error
output/enclosing graph propagation rules. Start failures follow
NOD-11/NOD-20: the failing node never becomes started and is not stopped;
previously started nodes are stopped by the graph. This introduces no HGL
try/catch or error-valued return.

Validate the following preconditions before changing a buffer:

1. Indexing requires `0 <= index < len(replay_input)`. Otherwise raise
   with message beginning `replay_input: index out of range` and leave input
   and capture unchanged. An in-bounds absent slot returns `null` successfully.
2. `begin` requires an unbegun recording. A repeated call raises with
   message beginning `capture: already begun`, preserving its state. A
   successful call makes an empty recording present even if append is never
   called. It never clears an existing recording.
3. `append` first requires begin to have succeeded; otherwise raise with
   message beginning `capture: not begun`. Then require time to equal the
   owning record node's current evaluation time, otherwise raise with
   message beginning `capture: timestamp is not evaluation time`. Finally,
   if there is a preceding capture, time must be strictly greater than its
   time; otherwise raise with message beginning
   `capture: timestamp did not advance`. These checks occur in that order.
4. Failed validation or failure to copy/allocate the new capture appends
   nothing and preserves all earlier captures. Successful earlier calls and
   node state/output effects remain, as ADR 0009 requires. The atomicity of
   one buffer append does not roll back an entire node evaluation. Allocation
   errors propagate; their diagnostic message is not standardized.

The standard record body below obtains the scalar delta explicitly with
[delta_value(ts)](../delta-value-metadata.md), calls append once per admitted
tick and uses last_modified(ts), which equals evaluation time for that modified scalar
input. Adjacent equal payloads have increasing timestamps and are recorded
separately. A second append at the same time is an error rather than implicit
replacement or deduplication.

## Operator bodies

These bodies define the operators using alarm, clock, cache, start and when
syntax. The capabilities provide storage access; the bodies define cursor
advancement, scheduling, publication and capture.

```hgl
impl fn replay<T>() -> T
requires T in {bool, i64, f64, str, date, time, datetime, duration}
{
    inject replay_input, alarm, clock
    cache index: i64 = 0

    start {
        if len(replay_input) > 0 {
            schedule(alarm, 0s)
        }
    }

    when {
        let current = index
        index += 1
        if index < len(replay_input) {
            schedule_at(alarm, clock.next_cycle_evaluation_time)
        }
        let item = replay_input[current]
        if item != null {
            return item
        }
    }
}

impl fn record<T>(ts: T)
requires T in {bool, i64, f64, str, date, time, datetime, duration}
{
    inject capture

    start {
        begin(capture)
    }

    when {
        append(capture, last_modified(ts), delta_value(ts))
    }
}
```

The replay cursor advances and the next wake-up is requested before return,
because return ends the evaluation. A silent slot causes no publication;
zero length schedules no replay evaluation. The recorder's start still
runs, producing a present empty recording. HGL holds the replay cursor;
the capability cannot silently move it.

## Examples

For each admitted scalar type, a and b below are values of that type.
False, zero and empty text are present values; they do not denote absent
slots. Presence depends only on the slot, not on its payload.

| Case | Result | Basis |
|---|---|---|
| `[a, _, a, b]` | Three output ticks at positions 0, 2, 3; equal a retained twice. | Scalar delta and own-output return; no deduplication. |
| `[_, _, _, _]` | Recorder starts and stops; zero appends; four dense no-tick cells. | OP-11, NOD-11, EVAL-5. |
| `[]` | Recorder starts and stops; zero replay evaluations or appends; empty dense result. | Empty scheduling branch, OP-11, EVAL-5. |
| Two fresh runs using the same operator definitions | Each observes only its own configured data and captures. | Per-node/run binding and fresh ownership. |
| Capture a then publish b; dispose graph | Earlier capture remains a and is readable in the owned result. | VAL-17, EVAL-3. |
| Missing/wrong-type/wrong-run provider; two capture writers | Graph construction fails before any start effect. | The new binding contract. |
| Index -1 or length | Specified translated error and no buffer mutation. | Bounds rule. |
| In-bounds absent slot | Read returns `null`; guarded replay produces no publication. | Nullable indexed read and ordinary no-output path. |
| Repeated begin; append before begin | Specified failure; earlier state preserved. | The new lifecycle preconditions. |
| Wrong time, repeated current time, or decreasing time | Validation order above selects the error; no extra capture. | The new timestamp contract. |
| Unsupported type/phase or escaped capability | Checking fails; no graph runs. | The new admission/borrowing rules. |

## Excluded behaviors

No persistence/checkpoint/restart contract, timed input interface, general
resource ownership language, shared capture writer, untyped global registry,
or collection delta API is introduced. The source/capture operations are a
bounded language facility borrowing an already-owned run resource; they do
not settle the broader resource concept left open by runtime state/cache.
