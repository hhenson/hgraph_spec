# Atomic publication deltas

## Finite profile

Admit `atomic<V>` at the top level and as a child of the existing
[collection publication profile](contextual-collection-deltas.md). V is an
ordinary value formed recursively from the [admitted scalar leaves](temporal-scalar-publications.md),
including [bytes](bytes-values.md), fixed or
unbounded ordinary lists, positional tuples, and fully applied concrete nominal
structs whose fields are required, defaulted or optional under the
[optional-field publication contract](optional-atomic-publications.md). Values and nominal
expansion are finite, except for the nominal edges admitted by the
[finite recursive-value contract](recursive-atomic-publications.md). The separate
[atomic family contract](abstract-atomic-publications.md) admits complete nonrecursive
family values. Other scalars, native types, endpoint shapes and
structural delta objects are outside this extension. An unbounded ordinary
list is a finite snapshot, not a growing temporal list. The separate
[ordinary set/map extension](atomic-set-map-publications.md) adds finite
set/map payloads and their exact typed constructors.

For non-composite S, `atomic<S>` normalizes to S under
[atomic scalar equivalence](atomic-scalar-equivalence.md). This profile does
not create a second scalar shape or extend the admitted scalar leaves.

## Complete values and publication

`delta<atomic<V>>` is exactly V. Construct it with ordinary V expressions and
constructors; no new `delta<atomic<V>>(...)` constructor form is added.
Required fields and ordinary defaults apply when constructing V. A publication
replaces the whole held V: it does not merge fields, retain an old omitted
field or apply sparse collection operations.

An empty list is a present complete snapshot. Equal snapshots at distinct
times remain separate publications. `_` alone denotes silence in an eval
sequence. Sparse outer collections use their canonical update rules, including
[empty sparse application](empty-delta-validity.md). An atomic child containing
an empty list is a valid present child, not an empty sparse patch.
Invalidation remains outside this profile.

`delta_value(endpoint)` requires that exact endpoint to be valid and modified,
under the existing evaluation-phase guards. It returns this cycle's complete
V, never a held value substituted for a silent cycle. Existing ordinary
read-only access and borrowing rules apply. Each owning constructor, list push,
global-entry write, output publication and final recording result independently
retains V, recursively. Later source mutations, another capture, graph teardown,
or changes to one owned result cannot change another retained snapshot.
No borrowed endpoint view escapes. Ordinary V operations remain available;
this adds no operation on sparse structural delta objects.

## Matching, eval and recording

A composite V alone does not determine T from `delta<T>`: V may be a complete
atomic payload rather than a structural publication delta. Bind T from an
endpoint, another known temporal shape, an expected exact TimedValue type or
an explicit `TimedValue<atomic<V>>` application. An unresolved shape is a
checking error. Existing scalar and originating-structural-delta matching
remain unchanged; inference never inspects runtime payload contents.

Use the same `TimedValue<T>`, replay and record operators. For atomic T, its
value field is V. Generic `pass_through` remains
`when { return delta_value(value) }`; it publishes the complete snapshot to
its own output. No additional storage or replay capability is introduced.

Eval fixes exact shapes before construction, validates every present slot as
complete V before start, and retains the existing dense horizon and lifecycle.
No earlier slot supplies missing fields. Harness comparison uses presence and
complete recursive ordinary values, preserving nominal types and fixed sizes;
it does not compare sparse patches. Empty and all-silent sequences retain their
existing meaning. Unsupported concrete shapes are checking errors; invalid
present input data uses the existing `eval: input delta outside publication profile`
diagnostic with parameter and zero-based position.

See [source examples](../../examples/atomic-delta-publications.hgl) and
[compiler cases](../../../compiler/cases_atomic_publications.md).
