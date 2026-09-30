# Explicit delta-value metadata

Status: scalar contract clarification, 2026-09-30. This specifies the
canonical delta accessor, its scalar type and admission, and the generic
pass-through function. Structural delta transport and application require
separate contracts.

## Canonical spelling and typing

`delta_value(endpoint)` is the temporal delta accessor. It takes exactly one
endpoint argument. There is no `delta(endpoint)` accessor intrinsic.
`delta<S>(...)` is the distinct contextual struct delta constructor and is
retained. `delta_value` is a prelude intrinsic, not a new reserved word.
The identifier `delta` outside its constructor form follows ordinary name
resolution; a call to an undeclared `delta` is a name error, not a metadata
accessor. An unrelated user-declared callable named delta follows the ordinary
call rules.

| Form | Meaning |
|---|---|
| `delta_value(endpoint)` | Read the delta published by this endpoint in the current cycle. |
| `delta<S>(field: value, ...)` | Construct a sparse update for the nominal struct S; omitted fields do not change. |

The result is derived from the endpoint's **temporal shape**, not from the
stored representation. In generic checking, retain that
derived relationship symbolically. Writing `delta_of(T)` in this specification
names that relationship only: it is not an HGL function, type constructor,
source type, annotation, or new first-class storable delta value. It is
distinct from the `delta_value(endpoint)` accessor and `delta<S>(...)`
constructor. A runtime return/output context must
check the derived delta against the same temporal output shape.

For the scalar profile of
[ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md), T is one of
`bool`, `i64`, `f64`, `str`, `date`, `time`, `datetime`, or `duration`.
`delta_of(T)` is the ordinary scalar T because a TS's delta is the scalar
published in this cycle. The call reads that delta explicitly; it does not
read the held value as a substitute when there was no publication. Ordinary
scalar ownership/copy rules apply, and a retained scalar is independent of
subsequent input changes (VAL-17, TS-22).

A structural T requires its recursive sparse delta, not a complete snapshot
of T. The signature and generic body below retain that relationship, but
structural instantiations require the separate contextual-delta/application
contract. This record admits only the eight scalar instantiations above.
It does not admit collections by erasing them to a scalar or assuming
`delta_of(T)` equals T. Existing struct delta constructors retain their own
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

## Consequences

For a scalar pass-through, an admitted publication is copied
once; a repeated equal scalar is still a tick; silence has no delta; dense
horizon materialization retains silent cells. This follows TS-2, the scalar
row of the runtime delta table, the runtime return contract and EVAL-3/5.
