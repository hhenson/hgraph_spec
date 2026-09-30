# ADR 0016: typed scalar buffer capabilities for replay and record

Status: proposed source extension, 2026-09-30. This proposal uses
node-scoped typed injectables rather than general resource types. The
behavioral evidence below concerns reference graph execution and storage;
it is not a claim that either Python/C++ HGL or another HGL implementation
already supports this source surface.

This builds on [eval operator composition](../eval-operator-composition.md).
Its admitted profile is a fresh, finite, dense eval of `bool`, `i64`, `f64`,
`str`, `date`, `time`, `datetime`, or `duration`. In this profile a TS delta
is its scalar value. Collection and structured types, `atomic` wrappers,
references, signals, windows, other scalars, persistence, checkpointing and
restarting a stopped instance are not admitted by this extension.

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
operator configuration argument. Normal operator resolution and HGL bodies
remain in use; the compiler must not substitute a special native temporal
implementation for these HGL bodies when claiming this profile.

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
  slice. Direct capability methods are the only storage-access primitives.
- A borrowed capability cannot be assigned, returned, passed as a value,
  put in a closure, or kept in state/cache. A native implementation must not
  retain a capability reference outside its call. Its owned scalar results
  follow the ordinary value rules instead.

While constructing the graph, eval binds each replay node instance to its
input buffer and each record node instance to its capture buffer. A binding
identifies the run and node, its role and exact scalar type. Validate all
required bindings before starting any node. Missing binding, wrong run,
wrong role/type, or two writers bound to one capture fails graph construction.
A body using these capabilities outside a configured graph therefore fails
construction; it never falls back to ambient or process-global storage.
Another graph builder may supply equivalent bindings under this contract.
The provider ABI and its representation are not language syntax.

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

## Exact capability methods

These are receiver methods of the injected capability, not ordinary HGL
resource-type declarations. T below is the concrete contextual payload type.
Names and argument types are fixed by this extension; ordinary positional
and named argument binding applies.

| Signature | Allowed phase | Result or effect |
|---|---|---|
| `replay_input.length() -> i64` | start, evaluation | Number of slots, including absent slots. |
| `replay_input.has_tick(index: i64) -> bool` | start, evaluation | Presence at an in-bounds position, independent of payload truthiness or equality. |
| `replay_input.delta_at(index: i64) -> T` | evaluation | Owned scalar delta at an in-bounds present position. |
| `capture.begin()` | start | Mark the configured empty recording begun. No output, scheduling or publication. |
| `capture.append(time: datetime, delta: T)` | evaluation | Append one time and independent owned scalar delta to a begun recording. |

No capability method is admitted in stop. No method schedules, publishes an
endpoint, deduplicates values, applies a delta to an output, advances a
cursor, inserts no-tick cells, or performs dense-result padding. Native
implementations provide only these storage operations. HGL determines when
to call them. `delta_at` uses that name because it returns the input slot's
delta; in this admitted profile the return type is the ordinary scalar T.
It does not settle a future collection delta type.

`delta_at` returns an owned scalar result, including for str (ADR 0009).
`append` copies the scalar and time before returning. Later output changes,
input destruction, another run and graph teardown cannot change an earlier
successful capture (VAL-17). This is independence, not mandated allocation.
The run owner keeps configured storage alive through graph stop and result
extraction. The returned eval sequence is owned independently of disposed
graph storage. Extraction may copy or transfer owned storage; no borrowed
hook view escapes in either case.

## Failure and validation

Unsupported host types, phases, or capability escapes are checking errors.
Invalid provider configuration is a graph-construction error, before any
node starts. Diagnostics name the capability and the rejected requirement.
No failure is converted into a default scalar, an absent slot, or successful
empty output.

Within an admitted hook, fallible capability methods use the **translated
native error policy** of [ADR 0009](0009-native-errors-and-the-node-error-model.md).
They may raise an ordinary native exception; they are not noexcept calls.
During evaluation the raise ends that evaluation and follows the existing
node error output/enclosing graph propagation rules. Start failures follow
NOD-11/NOD-20: the failing node never becomes started and is not stopped;
previously started nodes are stopped by the graph. This introduces no HGL
try/catch or error-valued return.

Validate the following preconditions before changing a buffer:

1. `has_tick` and `delta_at` require `0 <= index < length`. Otherwise raise
   with message beginning `replay_input: index out of range` and leave input
   and capture unchanged. An in-bounds absent slot makes has_tick false.
2. After its bounds check, `delta_at` requires a present slot. Otherwise
   raise with message beginning `replay_input: slot has no tick`. Never
   substitute zero, false, empty text or a default temporal scalar.
3. `begin` requires an unbegun recording. A repeated call raises with
   message beginning `capture: already begun`, preserving its state. A
   successful call makes an empty recording present even if append is never
   called. It never clears an existing recording.
4. `append` first requires begin to have succeeded; otherwise raise with
   message beginning `capture: not begun`. Then require time to equal the
   owning record node's current evaluation time, otherwise raise with
   message beginning `capture: timestamp is not evaluation time`. Finally,
   if there is a preceding capture, time must be strictly greater than its
   time; otherwise raise with message beginning
   `capture: timestamp did not advance`. These checks occur in that order.
5. Failed validation or failure to copy/allocate the new capture appends
   nothing and preserves all earlier captures. Successful earlier calls and
   node state/output effects remain, as ADR 0009 requires. The atomicity of
   one buffer append does not roll back an entire node evaluation. Allocation
   errors propagate; their platform-specific message is not standardized.

The standard record body below calls append once per admitted tick and uses
last_modified(ts), which equals evaluation time for that modified scalar
input. Adjacent equal payloads have increasing timestamps and are recorded
separately. A second append at the same time is an error rather than implicit
replacement or deduplication. These invalid-call rules are new capability
contract choices; ordinary reference replay/record traces do not establish
that existing APIs expose the same failure interface.

## HGL implementations

These bodies are executable-intent source under this proposed extension.
They use existing alarm, clock, cache, start and when syntax. They are not
labelled implemented on an existing backend.

```hgl
impl fn replay<T>() -> T
requires T in {bool, i64, f64, str, date, time, datetime, duration}
{
    inject replay_input, alarm, clock
    cache index: i64 = 0

    start {
        if replay_input.length() > 0 {
            alarm.schedule(0s)
        }
    }

    when {
        let current = index
        index += 1
        if index < replay_input.length() {
            alarm.schedule_at(clock.next_cycle_evaluation_time())
        }
        if replay_input.has_tick(current) {
            return replay_input.delta_at(current)
        }
    }
}

impl fn record<T>(ts: T)
requires T in {bool, i64, f64, str, date, time, datetime, duration}
{
    inject capture

    start {
        capture.begin()
    }

    when {
        capture.append(last_modified(ts), ts)
    }
}
```

The replay cursor advances and the next wake-up is requested before return,
because return ends the evaluation. A silent slot causes no publication;
zero length schedules no replay evaluation. The recorder's start still
runs, producing a present empty recording. HGL holds the replay cursor;
the capability cannot silently move it. A more efficient HGL body may skip
silent positions only if it preserves the contract's observable behavior.

## Reasoned acceptance cases

Keep expected traces before execution, separately from observations. For all
eight admitted scalars use two typed values, including false/zero/empty text
where appropriate so presence never depends on truthiness.

| Case | Expected observation | Basis |
|---|---|---|
| `[a, _, a, b]` | Three output ticks at positions 0, 2, 3; equal a retained twice. | Scalar delta and own-output return; no deduplication. |
| `[_, _, _, _]` | Recorder starts and stops; zero appends; four dense no-tick cells. | OP-11, NOD-11, EVAL-5. |
| `[]` | Recorder starts and stops; zero replay evaluations or appends; empty dense result. | Empty scheduling branch, OP-11, EVAL-5. |
| Two fresh runs using the same operator definitions | Each observes only its own configured data and captures. | Per-node/run binding and fresh ownership. |
| Capture a then publish b; dispose graph | Earlier capture remains a and is readable in the owned result. | VAL-17, EVAL-3. |
| Missing/wrong-type/wrong-run provider; two capture writers | Graph construction fails before any start effect. | The new binding contract. |
| Index -1 or length; absent delta_at slot | Specified translated error and no buffer mutation. | Bounds/presence rules. |
| Repeated begin; append before begin | Specified failure; earlier state preserved. | The new lifecycle preconditions. |
| Wrong time, repeated current time, or decreasing time | Validation order above selects the error; no extra capture. | The new timestamp contract. |
| Unsupported type/phase or escaped capability | Checking fails; no graph runs. | The new admission/borrowing rules. |

Native binding tests must verify method signatures, phases, contextual type,
fallibility and escape restrictions. Those are not proven by scalar parity.
HGL test syntax cannot yet assert errors (ADR 0009); do not invent such syntax
here. Backend/native acceptance tests may cover these failures until that
separate harness feature is specified.

## Evidence and remaining scope

The foundation [reference audit](https://github.com/hhenson/hgraph_spec_audit/blob/codex/delta-eval-foundation/runtime/validation/delta_eval/README.md)
records frozen scalar cases, raw results, normalization and engine identity.
The [direct native scalar supplement](https://github.com/hhenson/hgraph_spec_audit/blob/codex/delta-eval-foundation/runtime/validation/delta_eval/native_observed.json)
corroborates the scalar traces through native C++ authoring. It uses the same
C++ runtime as the Python authoring facade, not a third independent engine.
Native eval already returns dense empty/silent vectors; the two Python-facing
harnesses return raw None in those cases and require explicit normalization.

The separate [lifecycle expectations](https://github.com/hhenson/hgraph_spec_audit/blob/codex/delta-eval-foundation/runtime/validation/delta_eval/lifecycle_reasoned.json)
precede the [lifecycle observations](https://github.com/hhenson/hgraph_spec_audit/blob/codex/delta-eval-foundation/runtime/validation/delta_eval/lifecycle_observed.json).
Both engines directly invoke recorder start and stop on empty and all-silent
runs. Graph-stop callbacks precede eval return. Repeated runs in one scope
using the same recording key do not append to earlier results. Captured TSD
deltas survive reuse and mutation of the producer dictionary, later updates,
eval return, further producer mutation and garbage collection. That bounded
retention probe supports VAL-17; it does not prove arbitrary atomic-payload
copying or admit collection types under this new source capability.

Python exposes a present empty external recording after recorder stop. The
separate [positive control](https://github.com/hhenson/hgraph_spec_audit/blob/codex/delta-eval-foundation/runtime/validation/delta_eval/lifecycle_control_observed.json)
confirms that both engines expose two external entries for input [1,2]. A
subsequent all-silent run exposes a present empty recording in Python and an
absent recording key through the C++ facade, repeated consistently in three
fresh processes. This preserves a specific disagreement with the expected
accessible empty recording; it is not normalized into an observed empty
buffer. External lookup is not the native graph-local buffer, so it does
not prove missing internal allocation at start or a failure to run recorder
lifecycle. Before-first-tick storage accessibility and the exact destruction
point remain unestablished by these probes. OP-11 remains the independent
empty-recording requirement; the proposed capture.begin contract satisfies it.

These are behavioral observations of existing operators, not tests of this
new injectable surface. The missing-provider, scalar-type, phase, bounds,
begin and timestamp rejections above are explicit proposed contract choices;
they require compiler/provider acceptance tests. No existing reference API
is claimed to expose those exact methods or diagnostics.

No persistence/checkpoint/restart contract, timed input interface, general
resource ownership language, shared capture writer, untyped global registry,
or collection delta API is introduced. The source/capture methods are a
bounded language facility borrowing an already-owned run resource; they do
not settle the broader resource concept left open by runtime state/cache.
