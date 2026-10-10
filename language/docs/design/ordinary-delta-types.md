# Ordinary publication-delta types

Status: proposed source extension, 2026-10-03.

`delta<T>` names the ordinary value type of a publication delta for the
exact temporal shape T. The same marker followed by arguments,
`delta<T>(...)`, constructs a delta value. `delta_value(endpoint)` remains
the guarded publication accessor.
There are no alternative spellings or compatibility aliases.

## Formation and canonical identity

Add this alternative to both `type` and `value_type`:

```ebnf
delta_type = "delta", "<", type, ">";
```

Recognize this form in type positions, including annotations and type
arguments. `delta` is contextual here, not a hard reserved word. In an
expression, `delta<T>(...)` uses the constructor grammar; `delta<T>` alone
does not produce a runtime type value. There is no `delta(endpoint)` accessor
or new reflection/conversion call. Ordinary expression names retain their
existing resolution rules.
The argument is a type, never an endpoint expression or runtime type value.
Generic argument positions retain their existing type-versus-const checking.
The unified spelling adds no constructor argument forms: scalar deltas remain
ordinary scalar values, while structural constructors keep their admitted
named arguments and sparse entries.

T must have a finite shape admitted by the
[collection publication profile](contextual-collection-deltas.md): the admitted
scalar leaves, including [bytes](bytes-values.md) and [any](any-values.md); sets with admitted scalar members; fixed lists with nonnegative constant sizes;
growing lists under their [net delta contract](growing-list-publications.md);
positional tuples; fully applied concrete nominal structs; and maps with admitted scalar keys,
with recursively admitted children. Exact rolling shapes use their
[arrival delta relationship](rolling-publications.md). Recursive nominal definitions,
references, signals and other scalars are not added.
The [finite atomic extension](atomic-delta-publications.md) also admits
`atomic<V>` and such children: its delta is complete ordinary V.
An unsupported concrete T is a checking error.

For an admitted scalar S, `delta<S>` is exactly S, not a wrapper or a
distinct nominal type. For admitted `atomic<V>`, it is exactly V.
For structural T it is a distinct canonical ordinary
value type retaining T's full originating shape: container kind, fixed size,
tuple positions, scalar/key/member types, module-qualified nominal origin,
all nominal arguments and recursive children. Equal field layouts do not
erase nominal identity. Two fixed sizes do not share a delta type merely
because a particular delta updates an index valid in both.

A structural delta is not T's complete held value or an ordinary map of
lookalike entries. There is no implicit conversion between those types.
Canonical type identity is resolved during checking/specialization and bound
before graph execution. Storage and application need no type-name lookup,
runtime shape registry or source-visible type test.

## Generic checking and matching

In a generic declaration, keep `delta<T>` symbolic until T is bound.
Its use imposes the existing generic requirement that T be admitted by every
place where it occurs. Formation is checked after substitution. No new
recursive `requires` predicate or dynamic shape discovery is introduced.
An unresolved T cannot reach an instantiated ordinary value or graph.
These formation requirements also govern [generic struct shape arguments](generic-struct-shape-arguments.md),
including requirements forwarded through enclosing generic applications.

Matching `delta<T>` against a structural delta type binds T to that
type's exact originating shape. Matching against an admitted scalar S binds
T to S. Matching two symbolic delta applications preserves equality of their
originating shapes. All repeated occurrences and other constraints must
agree; incompatible or unresolved bindings fail checking. This is matching
of the specified type relationship, not inference from payload contents or
from a key's spelling. Composite ordinary payload V alone does not invert
`delta<T>` to an atomic shape; T must already be fixed by another type-bearing
context under the [atomic matching rule](atomic-delta-publications.md#matching-eval-and-recording).

TimedValue uses the originating temporal shape as its parameter and derives
the ordinary payload type in its field:

```hgl
export struct TimedValue<T> {
    time: datetime
    value: delta<T>
}

operator replay<T>(const values: list<TimedValue<T>>) -> T
operator record<T>(ts: T, const key: str)
```

Matching replay's list fixes T directly from the TimedValue argument, even
when the list is empty. When an ordinary TimedValue constructor infers T from
its supplied value field, the `delta<T>` relationship above still applies.
The nominal type is TimedValue applied to the originating shape: a map sample
is `TimedValue<map<i64, i64>>`, with a `delta<map<i64, i64>>` payload.
This changes the former payload-type parameter convention without adding a
compatibility alias. A structural delta type is not itself an admitted T.

Record's T is fixed by its temporal input; its body supplies the explicit
ordinary recording type. This extends the scalar signatures rather than
adding overlapping scalar overloads: scalar reduction leaves their concrete
types unchanged. It requires no explicit generic function-call syntax or
inference solely from an expected result.

## Ordinary values, observations and retention

An admitted delta type may occur in ordinary bindings, const parameters,
value-function parameters/results, ordinary struct fields and type
arguments, list elements and prepared global-state entries. An owned value
may be copied, passed as an ordinary value, returned from a value function,
retained in an ordinary constructor/list/global entry, and installed by
owning replacement, under the existing exact-type and ownership rules.
Each owning result is independent of later changes to its source.
Retention preserves sparse child omissions, additions/removals, scalar types
and nested delta structure. It cannot turn a removal into an ordinary child
value or fill an omitted child from held endpoint state.

For structural T, `delta_value(ts)` remains a read-only observation bounded
to the current evaluation. It requires the existing validity/modification
proofs and retains the exact derived type `delta<T>`. An evaluation-local
immutable `let`, including an explicitly typed one, may preserve that
observation under the existing contextual local rules. An annotation alone
does not create writable access, an owning copy or permission to escape.
Scalar specializations retain their ordinary owned scalar result. Atomic
specializations follow ordinary V access and independent retention as specified
by the [atomic contract](atomic-delta-publications.md).

An admitted owning retention boundary may consume a structural observation
by independently retaining its data. Constructor arguments, ordinary list
push, global set and an already owning destination's replacement are such
boundaries. Runtime publication also must not retain a borrow into its input.
No endpoint view or borrowed entry handle becomes the stored payload.
The [value-mutability contract](value-mutability.md) still governs aliases,
lexical entry borrows and when a borrowed value may be passed onward.

`let` remains recursively read-only. `var` authorizes existing owning
replacement or exclusive entry access; authority alone adds no operation.
This extension supplies no structural-delta field/index inspection, mutation,
iteration, ordinary equality, ordering, hashing or conversion. Existing
harness comparison retains its own exact-shape sparse comparison rules;
scalar reductions keep their ordinary scalar operations. Reading an ordinary
TimedValue's time/value fields or indexing its containing list is still an
ordinary struct/list operation, not inspection inside the delta.

Storage admission does not admit new state/cache forms, escaping borrows,
general nullable values, or temporal endpoints whose payload is a structural
delta object. A structural delta type is an ordinary type in this extension;
using it as a new temporal shape, including inside an atomic boundary, is
outside this profile. The [any extension](any-values.md) separately permits
boxing an owning delta as ordinary data; this adds no direct delta endpoint,
inspection or missing value capability.

## Construction order and failure

In an ordinary delta-value context, `delta<S>(...)` constructs an independent
value of exact type `delta<S>`. The explicit S determines its derived
type; an expected type must agree. Existing contextual publication uses and
constructor spellings continue to work. No scalar or atomic constructor form is added: an ordinary complete value
already inhabits its reduced delta type.

Before evaluating payload expressions, check the whole constructor: resolve
S, validate argument/field names, resolve constant members/keys/indices, check
duplicates, set/map overlap, index bounds and child delta types.
Growing-list removed/modified overlap retains its state-dependent publication
admission check; it is not a constructor rejection. Existing set member
lists and map removal lists remain constant; sparse keys/indices remain
constant. This adds no dynamic membership or key grammar and no new runtime
effects to those constant positions.

For an admitted constructor, process supplied arguments in written order.
Within a sparse entry list, process entries in written order. Evaluate each
payload expression exactly once and independently retain its scalar or child
delta before processing the next expression. A nested delta constructor
completes this same sequence before its result is retained by the enclosing
entry. Omitted arguments/fields contribute no entries and invoke no defaults.
This orders expression effects and retention, not a sequence of endpoint
mutations; construction publishes nothing.

If an expression or its retention fails, stop: later payload expressions do
not execute and no partial constructed delta escapes. Failure during final
assembly also produces no completed delta. Earlier successful effects stand.
Propagate the existing phase-appropriate value-operation error; node-hook
failure follows the node error contract. This does not add transactionality
over other values, logging, global state or outputs.

## Data formation is not publication admission

A well-shaped ordinary delta can be constructed, retained and replaced
without any endpoint to update. Empty sparse data is representable; storing
it neither creates a tick nor denotes silence. A removal instruction does
not prove that the key/member exists in a later target. Formation checks
data shape and exact types, not an unspecified endpoint's current state.

Applying an owned delta through a matching runtime return, own-output
assignment or generator yield retains the existing publication profile.
The payload must have exact type `delta<T>`. Structural originating-shape matching,
state-dependent membership/removal requirements,
[empty sparse application](empty-delta-validity.md) and fresh eval trace
validation apply at their existing boundaries. Atomic children use complete V, including admitted empty lists.
Copying or storing a delta neither satisfies nor bypasses those requirements.

Invalidation, invalid-child membership and reference designation remain
separate contracts. Empty stored data is not `_`, null or a held snapshot;
its application may be silent only as specified by EMPTY-1–2. A successfully
stored value alone does not establish state-dependent application preconditions.

## Replay and recording

Use `list<TimedValue<T>>`, whose value field is `delta<T>`.
Every entry is present ordinary data; silence is omission of a timed entry,
not a special delta value. Eval keeps its dense
horizon separately. The existing generator rules govern replay's traversal
and time handling; yield applies the stored delta to the exact T output,
subject to the publication contract above.

Record initializes the exact typed list in start, then retains each admitted
input publication and `clock.evaluation_time` through ordinary construction
and push. The run owner obtains an independently owned result after stop.
No special replay or recording capability, runtime type test or recursive
delta introspection is required. Atomic publications explicitly carry whole
values; structural publications remain sparse.

The generic compute remains unchanged:

```hgl
fn pass_through<T>(value: T) -> T {
    when { return delta_value(value) }
}
```

The [complete source example](../../examples/ordinary-delta-types.hgl) uses
these relationships, and [cases](../../../runtime/cases_ordinary_delta_types.md)
state expected type, ownership and failure observations.

The immutable [owned-delta audit](https://github.com/hhenson/hgraph_spec_audit/blob/cf431ec659442a0a30d30be4bde443b3d8a8d320/runtime/validation/owned_deltas/README.md)
supports independent sparse-data retention through ordinary containers.
Source grammar, TimedValue's shape-parameter meaning, exact originating-shape
identity and matching remain HGL decisions. The audit preserves aliasing
divergences and distinguishes valid semantic copies from copying that changes
a removal's meaning. It does not
establish constructor effect order or arbitrary retention/allocation failures.
