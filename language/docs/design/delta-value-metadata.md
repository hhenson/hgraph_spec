# Explicit delta-value metadata

Status: scalar contract clarification, 2026-09-30. The canonical name
`delta_value` is already accepted by
[Native surface completion](native-surface-proposal.md#values-are-values).
This record reconciles older `delta(x)` metadata spelling, specifies the
scalar result and its admission, and uses explicit delta forwarding in the
eval fixture. It does not claim new compiler support or settle structural
delta transport and application.

## Canonical spelling and typing

`delta_value(endpoint)` is the canonical temporal delta accessor. It takes
exactly one endpoint argument. The legacy call `delta(endpoint)` is retained
as a compatibility synonym with identical type, phase, admission and
observation for the domain specified here; it is not deprecated by this
record. `delta<S>(...)` remains the separate contextual struct constructor.
These are prelude intrinsic calls, not new reserved words.

The result is derived from the endpoint's **temporal shape**, not from the
implementation's storage representation. In generic checking, retain that
derived relationship symbolically. Writing `DeltaOf(T)` in this specification
names that relationship only: it is not a source type, an annotation, or a
new first-class storable delta value. A runtime return/output context must
check the derived delta against the same temporal output shape.

For the scalar profile of
[ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md), T is one of
`bool`, `i64`, `f64`, `str`, `date`, `time`, `datetime`, or `duration`.
DeltaOf(T) is the ordinary scalar T because a TS's delta is the scalar
published in this cycle. The call reads that delta explicitly; it does not
read the held value as a substitute when there was no publication. Ordinary
scalar ownership/copy rules apply, and a retained scalar is independent of
subsequent input changes (VAL-17, TS-22).

A structural T requires its recursive sparse delta, not a complete snapshot
of T. The signature and generic body below retain that relationship, but
structural instantiations require the separate contextual-delta/application
contract. This record admits only the eight scalar instantiations above.
It does not admit collections by erasing them to a scalar or assuming
DeltaOf(T) equals T. Existing struct delta constructors retain their own
specified contexts independently of this metadata extension.

## Phase and presence

This specified scalar call is evaluation-local metadata in a runtime
function. Its argument must retain the identity of a temporal input endpoint
of an admitted type; passing an ordinary scalar expression, a const
parameter or a copied payload is a type error. Calls in start/stop or a
composition/value-function body are outside this admitted slice and are
phase errors. This adds no graph-level delta operator or native value-helper
signature.

A scalar delta read requires the particular endpoint to be both valid and
modified. The checker must establish both at the call site. A default when
with one temporal parameter establishes both for that parameter. Explicit
short-circuit guards `valid(x) && modified(x)` also establish both. In a
multi-input handler, the implicit any-input-modified selector does not prove
that every individual input is modified; guard the actual endpoint before
reading its delta. A call without these guarantees is diagnosed instead of
returning a held value, default scalar or invented nullable scalar type.
This does not change TS-2: an unmodified endpoint's runtime delta is nil;
the guarded scalar operation avoids trying to represent that absence as T.

## Generic pass-through fixture

The required compute body is generic and explicitly forwards its input delta:

```hgl
fn pass_through<T>(value: T) -> T {
    when { return delta_value(value) }
}
```

The declaration retains the relationship between input delta and owned output
shape. Each concrete instantiation must be admitted by the relevant delta
contract; a missing structural contract is diagnosed for that instantiation,
not replaced with whole-value copying. The same body is intended for scalar
and structural instantiations as those contracts become specified.

This is a runtime compute node with its own output. It does not return the
wired input port. In the admitted scalar slice its return writes the scalar
delta and ticks its own output even when the scalar equals the previous one.
A silent cycle does not activate this single-input handler.

An exact typed wrapper fixes T for eval, including all-silent/empty sequences
that cannot infer a generic type from a payload:

```hgl
fn pass_through_i64(value: i64) -> i64 => pass_through(value)

test explicit_scalar_deltas {
    assert eval(pass_through_i64, value: [7, _, 7, 9]) == [7, _, 7, 9]
    assert eval(pass_through_i64, value: [_, _, _]) == [_, _, _]
    assert eval(pass_through_i64, value: []) == []
}
```

The wrapper composes the generic compute node; it does not replace that
node with identity wiring. No explicit generic-call syntax is introduced.

For the same reason the ADR 0016 record body calls
`capture.append(last_modified(ts), delta_value(ts))`. Its scalar delta is an
ordinary T, so the capability signature is unchanged. Replay already obtains
explicit deltas through `replay_input.delta_at(index)`.

## Reasoning and evidence

The expected scalar traces are unchanged: an admitted publication is copied
once; a repeated equal scalar is still a tick; silence has no delta; dense
horizon materialization retains silent cells. This follows TS-2, the scalar
row of the runtime delta table, the runtime return contract and EVAL-3/5.

The existing [reference audit](https://github.com/hhenson/hgraph_spec_audit/blob/codex/delta-eval-foundation/runtime/validation/delta_eval/README.md)
uses a compute returning `ts.delta_value`, and its native scalar supplement
applies `ts.delta_value()` to its own output. Its frozen scalar traces already
exercise explicit delta forwarding. No new reference observation is needed
to rename the HGL call; the alias, checking and generic-instantiation rules
still require compiler acceptance tests. Collection/REF observations do not
establish their missing HGL contextual delta types or output application.
