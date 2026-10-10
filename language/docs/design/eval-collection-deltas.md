# Eval with contextual collection publication deltas

Status: proposed publication profile, 2026-09-30. This extends the fresh dense
eval profile using [contextual collection publication deltas](contextual-collection-deltas.md).
Empty sparse application follows [EMPTY-1–4](empty-delta-validity.md); this
does not introduce full-state replay.
The [ordinary delta type extension](ordinary-delta-types.md) supplies the
storable data and generic source relationships for this profile. Remaining
boundaries are listed in [ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md#unresolved-source-contracts).

## Publication shape admission

Admit the recursive scalar, set, fixed-list, positional/named-bundle and
map shapes with [admitted keys](scalar-collection-keys.md), including
[bytes leaves/keys](bytes-values.md), defined by the contextual collection contract, plus
[finite atomic shapes](atomic-delta-publications.md) at the top level or as
children. Atomic publications are complete values, not sparse patches.
Exact concrete input/output shapes must be known before graph construction.
Nominal identity, child types, tuple positions and fixed sizes are part of
the profile; matching a scalar leaf representation is insufficient.
Unsupported concrete eval shapes are checking errors. This is an eval
profile boundary, not a new source constraint predicate or a capability that
infers a generic implementation's storage type. Generic replay/record source
checking uses the explicit `delta<T>` relationship and its ordinary type
formation/matching rules.

The eval caller supplies only the target and typed delta sequences. Eval
constructs replay sources configured with ordinary const data, the target
compute and a record sink configured with a const string key. Record stores
an ordinary typed recording through run-wide `global_state`. No storage
injectable knows which temporal shape a node consumes or produces. The
generic target applies its input delta to its own output:

```hgl
fn pass_through<T>(value: T) -> T {
    when { return delta_value(value) }
}
```

## Publication data and owned recording

A present input slot supplies a contextual `delta<T>` for its exact
parameter shape. For example,
`delta<map<i64, i64>>(upsert: [1: 11])` supplies that sparse update, not a
complete map or other keys from earlier slots. `delta_value(v)` reads the
current publication from a live endpoint. The [nullable indexing rules](nullable-replay-indexing.md)
require presence before a nullable element's payload can be used. Ordinary
replay data instead contains present `TimedValue<T>` entries whose value
field is `delta<T>`,
with the dense horizon retained separately; no nullable element is needed.

A recorder retains an independent owned copy of each output delta and its
evaluation time. Later source changes, member removal, another eval and graph
teardown cannot change an earlier capture. No borrowed child view, member
range or endpoint reference escapes. Immutable physical sharing is allowed
only when these independence guarantees hold. Neither generic store access
nor ordinary input access schedules, publishes, applies a delta, deduplicates
or inserts silent cells. [Generic source bodies](../../examples/ordinary-delta-types.hgl)
use ordinary typed list construction, delta retention and push; no
role-specific capability stands in for those contracts.

## Dense literals, validation and comparison

Each supplied input is a finite dense sequence. `_` means no publication;
every other admitted slot contains a scalar or contextual delta for the
exact input shape. Use the collection constructors for structural slots.
The existing tuple harness shorthand `(1, _)` is equivalent in that context
to `delta<tuple<i64, i64>>(items: [0: 1])`; an underscore child is omitted,
not invalidated. This does not add `_` to ordinary runtime expressions.

Before starting the graph, validate the supplied trace from a fresh invalid
endpoint with no live set/map memberships. Each present structural delta
must satisfy the collection contract, including state-dependent
[empty sparse application](empty-delta-validity.md):
set additions are absent members, removals are present members, map removals
are present keys, and a newly added map child receives a delta making it
valid. Apply this validation recursively, preserving unchanged child state
between positions, even when applying an entry causes no tick. The original
input horizon is unchanged. At an admitted atomic boundary validate a complete V and
replace that child's whole state; an empty ordinary list is valid present data. An out-of-profile input raises the catalogued `eval.input_delta_profile`
admission error while executing eval, after ordinary argument evaluation and
before any graph starts, with a message beginning `eval: input delta outside publication profile`,
identifying its parameter and zero-based position. It is not a silent slot. This admission check
does not change the separate tolerant collection-mutation APIs.

The returned dense harness sequence retains its own contextual presence
contract; ordinary delta types do not create a general nullable sequence
type. Harness equality compares exact shape and each position's presence, then its
contextual delta. Set/key/index entry order is immaterial. Named/positional
children compare by field/index, recursively. Scalar leaves use their existing
comparison rules. An omitted child differs from a present equal scalar
publication. Structural equality must not fill omissions from held values.

```hgl
fn forward_map(value: map<i64, i64>) -> map<i64, i64> => pass_through(value)

test sparse_map_publications {
    assert eval(forward_map, value: [
        delta<map<i64, i64>>(upsert: [1: 10, 2: 20]),
        delta<map<i64, i64>>(upsert: [1: 11]),
        delta<map<i64, i64>>(remove: [2]),
        _
    ]) == [
        delta<map<i64, i64>>(upsert: [1: 10, 2: 20]),
        delta<map<i64, i64>>(upsert: [1: 11]),
        delta<map<i64, i64>>(remove: [2]),
        _
    ]
    assert eval(forward_map, value: [_, _, _]) == [_, _, _]
    assert eval(forward_map, value: []) == []
}
```

The exact wrapper fixes the shape even when no slot supplies a payload.
Repeated scalar child publications are retained. A map/set becoming empty
through a nonempty removal is admitted. Empty and all-silent input still
start/stop the graph and create an empty recording; dense materialization
uses the supplied horizon as in [eval composition](eval-operator-composition.md).

## Boundaries

Empty sparse inputs are admitted and applied under
[EMPTY-1–4](empty-delta-validity.md); record captures actual output ticks.
Invalidations, invalid-child membership creation and REF designation remain
outside this input profile. Do not encode those excluded state changes as
empty deltas, null, omitted entries or held snapshots.

Growing lists use the separate [net delta extension](growing-list-publications.md).
Rolling windows use the separate [arrival extension](rolling-publications.md).
Signals, atomic payloads outside the
finite atomic profile, timed-input syntax,
persistence and checkpoint/restart retain their separate contracts. No raw
reference literal or generic REF fixture is introduced.
