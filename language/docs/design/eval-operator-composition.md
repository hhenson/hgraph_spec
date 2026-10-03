# Eval composes replay, the target, and record

Status: proposed foundation extension, 2026-09-30. This specifies the graph
behind `eval` and its ownership and dense-result boundary. It does not extend
the set of delta values expressible in HGL.

## Operator composition

`eval` is a language form whose arguments are checked against the target's
parameters. For each temporal input it configures and wires a **replay source
operator**, calls the target with those outputs and the supplied constant
arguments, and wires a **record sink operator** to the target's output. An
outputless target has no output recorder. These are graph operators executed
by the engine, with the target between them in graph order:

```text
typed input buffers → replay operator(s) → target → record operator → capture buffer
```

The test author calls `eval(target, ...)`; eval constructs and configures the
operators and owns the ordinary input data and recording for that run.
Replay receives its input sequence as an ordinary const argument. Record
receives a const string key and uses the run-wide `global_state` store;
eval retrieves the recording after graph stop. For eight scalar types these
source contracts use the [ordinary scalar data contract](../../../library/ordinary_replay_record.md).
The [ordinary delta type extension](ordinary-delta-types.md) generalizes that
data path to the admitted structural profile. Remaining dependencies are in
[ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md#unresolved-source-contracts).
Replay and record remain normal operators with their own callable contracts. They may be
declared in the standard library and used independently. This arrangement
places no visibility restriction on them and does not require the eval
caller to select recording keys, seed storage, or configure a recorder.

This composition introduces no ordinary HGL type for a harness sequence.
For the eight scalar types, eval converts present harness positions to
ordinary `TimedValue<T>` entries and retains the dense horizon separately.
The replay list contains no absent elements. Its slot-derived entry times
are increasing; independent callers of replay retain the existing generator
timing behavior rather than acquiring a new ordering validator.

Eval calls the target as declared. A composition function may wire an existing
input through. A test that claims to exercise delta application by a compute
node must instead use a runtime function: it consumes the replayed input and
writes a distinct output owned by that node. A wiring identity does not test
that operation. The generic runtime fixture explicitly obtains its input delta. Its admitted
scalar instantiations are specified by [delta-value metadata](delta-value-metadata.md):

```hgl
fn pass_through<T>(value: T) -> T {
    when { return delta_value(value) }
}

fn pass_through_i64(value: i64) -> i64 => pass_through(value)

test owns_and_publishes_output {
    assert eval(pass_through_i64, value: [7, _, 7, 9]) == [7, _, 7, 9]
}
```

The `when` makes this a runtime function; its return publishes its own output
on each admitted evaluation. Equal repeated publications remain ticks. This
example fixes the scalar delta type. Structural contextual delta types and
application remain separate. The exact wrapper also fixes T when a sequence
has no present payload from which to infer it.

## Typed buffers and ownership


A **replay input buffer** is a typed sequence of positions, each containing a
delta or no tick. A **capture** is a pair containing an output tick's
evaluation time and an owned copy of its delta. A **recording** is the ordered
sequence of captures made by one recorder. An empty recording has zero
captures; it is distinct from having no recording.

Each temporal parameter determines the type of its replay input buffer; the
target result determines the type of the capture buffer. The buffers contain
the deltas admitted for those types, with timing/no-tick information as
specified by the harness sequence. A scalar's delta is its scalar value. A
structural endpoint's value must not silently replace its sparse delta.

Eval owns the configured input data and recorded result for the duration
needed by that run and its result. Separate eval invocations do not share or
accumulate recordings.

Borrowed endpoint/store access cannot escape its supplying hook. Ordinary
input data follows its value ownership contract. The replay operator publishes into its own output; a compute target publishes into its
own output; the record operator retains an **owned copy** of each captured
output delta. A retained capture cannot be a borrowed view whose contents
change when a later cycle updates or invalidates the source. “Copy” is the
independence guarantee of VAL-17, not a required allocation or layout.

The recorder exists from its start even if the target never ticks (OP-11).
No ticks leave an empty recording, not a missing recording. Engine execution
and result extraction remain separate: an error or failure to obtain a
recording is not silently converted into successful empty output.

## Dense results and silent runs

Let the dense input horizon be the longest supplied temporal sequence length.
An empty supplied sequence contributes zero; an all-silent sequence still
contributes its full length. Materialize one result cell per cycle through
the later of that horizon and the last output tick. A cycle without an output
tick contains `_`. This materialization does not publish a value or cause an
otherwise idle compute node to evaluate.

Consequently, for the scalar fixture above:

```hgl
test silent_horizon {
    assert eval(pass_through_i64, value: [_, _, _]) == [_, _, _]
    assert eval(pass_through_i64, value: []) == []
    assert eval(pass_through_i64, value: [_, 7, _, _]) == [_, 7, _, _]
}
```

The empty case supplies a temporal argument with zero samples; it is not a
new source-only eval form or an end bound for a source. These examples have
no independent scheduled work. Explicit source end bounds remain open under
[ADR 0015](decisions/0015-pull-sources.md#consequences).


## Rules

- **EVAL-1** Eval wires replay source operators, the selected target, and an
  output record sink operator into the evaluated graph. An outputless target
  has no output recorder. Eval owns their run-specific configuration.
- **EVAL-2** Replay input buffers and the capture buffer are typed by the
  target's parameters and result. Their delta shapes are the ones admitted
  by the language; this rule introduces no delta syntax or type.
- **EVAL-3** Every retained output delta is an independent owned value.
  Later cycles cannot change earlier captured deltas (VAL-17, TS-22).
- **EVAL-4** A recording exists at recorder start, including for a run with
  no output ticks (OP-11). Each eval invocation owns its recording.
- **EVAL-5** A dense result preserves the supplied input horizon, including
  silent cells, and any later output ticks. With no output ticks and an input
  horizon of zero, its result is empty. Silent materialization is not a tick.
- **EVAL-6** A failed run is not a successful empty result. An eval with an
  output must obtain its recording; a missing recording is an error, not an
  empty recording.

## Examples

These cases follow the rules above:

| Case | Input | Output | Derivation |
|---|---|---|---|
| first, equal, distinct scalar publications | `[7, _, 7, 9]` | `[7, _, 7, 9]` | Runtime return publishes; TS delta equals scalar; TS-2 makes idle delta nil. |
| all silent | `[_, _, _]` | `[_, _, _]` | No compute publication; OP-11 recording is empty; EVAL-5 retains input horizon. |
| empty | `[]` | `[]` | No publication and zero input horizon; no scheduled work in this fixture. |
| leading and trailing silence | `[_, 7, _, _]` | `[_, 7, _, _]` | Only one publication; input horizon includes both trailing silent cells. |
| retained capture | publish A, then B through a scalar/atomic output | first recorded value stays A | VAL-17 requires independence from later output writes; the first capture is independent of the second publication. |

## Deferred extensions

Beyond the scalar [delta-value contract](delta-value-metadata.md), this
foundation's structural publication shapes and harness literals use the
[collection contract](contextual-collection-deltas.md), with ordinary storage
and derived type matching supplied by [ordinary delta types](ordinary-delta-types.md).
A general apply-delta operation, explicit child invalidation, invalid-child
membership encoding and reference fixtures remain separate. TS-5 still requires
child-validity and membership information beyond published-value deltas when
reconstructing full collection state. These require separate specification
extensions.

The proposed [run-wide keyed-state foundation](decisions/0016-eval-scalar-buffer-capabilities.md)
configures replay with ordinary const data and record with a const key for
reusable shared storage. The [ordinary scalar data contract](../../../library/ordinary_replay_record.md)
completes this data representation and gives source bodies for the eight
scalar types. The ordinary delta extension supplies the structural storage
profile without introducing general nullable ordinary sequences.
