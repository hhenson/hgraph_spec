# Eval composes replay, the target, and record

Status: proposed foundation extension, 2026-09-30. This specifies the graph
behind `eval` and its ownership and dense-result boundary. It does not extend
the set of delta values expressible in HGL. The existing
[testing guide](../user-guide/testing-and-running.md#first-pass-limits)
continues to state the implemented subset.

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
operators and owns the buffers for that run. Replay and record remain normal
operators with their own callable contracts and implementations. They may be
declared in the standard library and used independently. This arrangement
places no visibility restriction on them and does not require the eval
caller to select recording keys, seed storage, or configure a recorder.

The operator contract is independent of its implementation language. Replay
and record behavior can be authored in HGL where the language admits it;
engine integration and buffer access may require native capabilities. This
foundation specifies neither those capabilities' signatures nor a new
ordinary HGL type for a harness sequence.

Eval calls the target as declared. A composition function may wire an existing
input through. A test that claims to exercise delta application by a compute
node must instead use a runtime function: it consumes the replayed input and
writes a distinct output owned by that node. A wiring identity does not test
that operation. For scalar inputs, where delta equals the published scalar,
the following is a runtime compute fixture:

```hgl
fn pass_through(value: i64) -> i64 {
    when { return value }
}

test owns_and_publishes_output {
    assert eval(pass_through, value: [7, _, 7, 9]) == [7, _, 7, 9]
}
```

The `when` makes this a runtime function; its return publishes its own output
on each admitted evaluation. Equal repeated publications remain ticks. This
example makes no assumption about the still-open result type of `delta(x)`.

## Typed buffers and ownership

Each temporal parameter determines the type of its replay input buffer; the
target result determines the type of the capture buffer. The buffers contain
the deltas admitted for those types, with timing/no-tick information as
specified by the harness sequence. A scalar's delta is its scalar value. A
structural endpoint's value must not silently replace its sparse delta.

Eval owns the configured input data and recorded result for the duration
needed by that run and its result. Separate eval invocations do not share or
accumulate recordings. How the buffers are represented, named, or passed to
the operators is not specified here.

A node reads a borrowed view that is stable for the current cycle. The replay
operator publishes into its own output; a compute target publishes into its
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
    assert eval(pass_through, value: [_, _, _]) == [_, _, _]
    assert eval(pass_through, value: []) == []
    assert eval(pass_through, value: [_, 7, _, _]) == [_, 7, _, _]
}
```

The empty case supplies a temporal argument with zero samples; it is not a
new source-only eval form or an end bound for a source. These examples have
no independent scheduled work. Explicit source end bounds remain open under
[ADR 0015](decisions/0015-pull-sources.md#consequences).

A reference harness may return a raw no-output sentinel rather than an empty
sequence when it records no ticks. At the HGL boundary, a **successful** run
with no recorded ticks is materialized using the input horizon: three silent
input cells give three `_` output cells, and zero cells give `[]`. This is
result normalization, not a new output tick and not evidence that raw
reference return objects are identical. Audit records must preserve both the
raw result and the normalized HGL result, and identify this conversion.

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
- **EVAL-6** Reference comparison preserves raw no-output results and names
  any normalization to the HGL sequence. A failed run or absent observation
  is never a successful empty trace.

## Reasoned cases and evidence boundary

These expectations were derived before reference execution:

| Case | Input | Output | Derivation |
|---|---|---|---|
| first, equal, distinct scalar publications | `[7, _, 7, 9]` | `[7, _, 7, 9]` | Runtime return publishes; TS delta equals scalar; TS-2 makes idle delta nil. |
| all silent | `[_, _, _]` | `[_, _, _]` | No compute publication; OP-11 recording is empty; EVAL-5 retains input horizon. |
| empty | `[]` | `[]` | No publication and zero input horizon; no scheduled work in this fixture. |
| leading and trailing silence | `[_, 7, _, _]` | `[_, 7, _, _]` | Only one publication; input horizon includes both trailing silent cells. |
| retained capture | publish A, then B through a scalar/atomic output | first recorded value stays A | VAL-17 requires independence from later output writes; an unchanged final value alone does not verify this. |

Reference observations and implementation identities are recorded in the
[delta-eval audit](https://github.com/hhenson/hgraph_spec_audit/blob/codex/delta-eval-foundation/runtime/validation/delta_eval/README.md),
including its frozen expectations and
[raw observations with normalization](https://github.com/hhenson/hgraph_spec_audit/blob/codex/delta-eval-foundation/runtime/validation/delta_eval/observed.json). The
bounded reference comparison uses genuine Python 0.5.41 and a C++ runtime
accessed through Python authoring; the latter is not a third independent
engine. Both return raw `None` for the audited empty and all-silent scalar
runs. The adapter explicitly materializes the dense input horizon, retaining
that raw result as evidence.

This comparison supports the stated no-output normalization; it does not by
itself observe whether an empty recorder was allocated or prove independent
ownership of captures. Document status does not assert implementation
conformance. The audit's bounded family cases do not settle HGL collection
delta syntax or imply reference-designation coverage.

## Deferred extensions

This foundation does not decide `delta(x)`'s result type, first-class delta
storage, set/map/list harness literals, a generic apply-delta operation,
explicit child invalidation or invalid-child membership encoding, reference
fixtures, or the bridge API used by the operators. TS-5 still requires
child-validity and membership information beyond published-value deltas when
reconstructing full collection state. These require separate specification
extensions and pre-observation traces.
