# Eval with contextual collection publication deltas

Status: draft source extension, 2026-09-30. This extends the fresh dense eval
profile of [ADR 0016](decisions/0016-eval-scalar-buffer-capabilities.md) using
[contextual collection publication deltas](contextual-collection-deltas.md).
It does not decide empty-event application or introduce full-state replay.

## Admission and ordinary operator contracts

Admit the recursive scalar, set, fixed-list, positional/named-bundle and
integer-key-map shapes defined by the contextual collection contract.
Exact concrete input/output shapes must be known before graph construction.
Nominal identity, child types, tuple positions and fixed sizes are part of
the buffer type; matching a scalar leaf representation is insufficient.

The ordinary operator signatures remain:

```hgl
operator replay<T>() -> T
operator record<T>(ts: T)
```

For this extended profile, the finite scalar `requires` list in ADR 0016 is
replaced by the shape-admission rule above. This is a contextual checking
rule on the replay/capture capabilities, not a new source constraint predicate
or a promise that every T is supported. Unsupported instantiations are
checking errors. These are the same operator contracts with a wider domain;
do not register a competing universal overload beside the scalar contract.
Their generic HGL bodies retain ADR 0016's alarm, clock, cache, lifecycle and
`when` behavior, with the finite scalar `requires` clauses removed. Generic
checking retains the capability's shape obligation while T is unresolved.
After ordinary candidate selection, a concrete specialization must satisfy the
recursive shape grammar before graph construction; failure is a diagnostic,
not a false `requires` premise or fallback to another candidate. This rule
adds no type-reflection function and changes no candidate-ranking rule.

The eval caller supplies only the target and typed delta sequences. Eval
constructs and binds the replay sources, target compute and record sink.
It does not replace a compute with graph identity wiring. The generic target
continues to apply its input delta to its own output:

```hgl
fn pass_through<T>(value: T) -> T {
    when { return delta_value(value) }
}
```

## Typed buffers and exact method relationships

The following table uses `delta_of(T)` solely as specification notation,
not callable HGL syntax or an ordinary source type annotation.

| Capability method | Contextual result or argument |
|---|---|
| `replay_input.length() -> i64` | Unchanged: number of present and absent slots. |
| `replay_input.has_tick(index: i64) -> bool` | Unchanged: the slot's presence flag. |
| `replay_input.delta_at(index: i64)` | Owned contextual result `delta_of(T)`, where T is the source output shape. |
| `capture.begin()` | Unchanged: create the present empty recording. |
| `capture.append(time: datetime, delta: delta_of(T))` | Contextual delta argument for the sink's exact input shape T; independent owned capture. |

`delta_at` is admitted only in evaluation, with an in-bounds present slot.
Its contextual result can be returned directly by replay. `append` remains
evaluation-only, and `begin` remains start-only. No method is admitted in
stop. Existing bounds, absence, beginning, timestamp and allocation-failure
rules, validation order and diagnostic prefixes remain unchanged.

The source and sink capabilities still borrow only their configured
run-owned, node-scoped buffers. Missing/wrong-role/wrong-run/wrong-shape
bindings and multiple capture writers fail graph construction before start.
No capability value escapes, enters state/cache, or falls back to ambient
storage. Shape admission adds no resource API or storage intrinsic.

A successful `delta_at` result owns all nested scalar and sparse-entry data
needed by the delta. `append` completes a recursive owned capture before
returning. Later source changes, member removal, another eval and graph
teardown cannot change an earlier capture. No borrowed child view, member
range or endpoint reference escapes. Immutable physical sharing is allowed
only when these independence guarantees hold.

The methods do not schedule, publish, apply, deduplicate, advance a cursor,
or insert silent cells. The HGL record evaluation remains:

```hgl
when { capture.append(last_modified(ts), delta_value(ts)) }
```

## Dense literals, validation and comparison

Each supplied input is a finite dense sequence. `_` means no publication;
every other admitted slot contains a scalar or contextual delta for the
exact input shape. Use the collection constructors for structural slots.
The existing tuple harness shorthand `(1, _)` is equivalent in that context
to `delta<tuple<i64, i64>>(items: [0: 1])`; an underscore child is omitted,
not invalidated. This does not add `_` to ordinary runtime expressions.

Before starting the graph, validate the supplied trace from a fresh invalid
endpoint with no live set/map memberships. Each present structural delta
must be a canonical nonempty publication under the collection contract:
set additions are absent members, removals are present members, map removals
are present keys, and a newly added map child receives a delta making it
valid. Apply this validation recursively, preserving unchanged child state
between positions. An out-of-profile input is a graph-construction error
with a message beginning `eval: input delta outside publication profile`,
identifying its parameter and zero-based position. It is not a silent slot. This admission check
does not change the separate tolerant collection-mutation APIs.

No first-class source delta type is introduced for the returned sequence.
Harness equality compares exact shape and each position's presence, then its
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
uses the supplied horizon as in ADR 0016.

## Boundaries

This profile admits targets whose structural publications remain in the
ordinary nonempty domain. Actual empty set events, empty fixed/map patches,
invalidations, invalid-child membership creation and REF designation remain
outside it. No success or conformance claim may cover those events based on
this contract. Diagnosing an excluded runtime event as unsupported does not
choose its eventual application semantics. Dropping it, substituting null or
a held snapshot, or presenting a successful empty recording cannot establish
a conforming result for that event. This extension leaves the semantic choice
for applying an empty publication to a later contract rather than treating
payload emptiness as general proof that no tick occurred.

Growing structures, windows, signals, atomic boundaries, timed-input syntax,
persistence and checkpoint/restart retain their separate contracts. No raw
reference literal or generic REF fixture is introduced.
