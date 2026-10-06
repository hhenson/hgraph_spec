# Optional fields in complete atomic publications

Admit finite concrete ordinary structs with optional fields into the atomic
payload grammar. The existing declaration `field: V = null` permits the field
to be unset; it does not change V into a new nullable source type. Present V
recursively follows the finite ordinary payload grammar. Required fields,
non-null defaults, exact nominal identity and constructor checking retain
their existing rules. Recursive definitions use the separate
[finite recursive-value contract](recursive-atomic-publications.md); abstract
families remain separate.

Complete construction preserves each field's presence. Omitting an optional
field whose effective default is null leaves it unset; explicitly supplying
null is permitted only for an optional field. A present zero, empty string or
empty container differs from an unset field. An unset field requires no
payload construction or retention. Ordinary equality and harness comparison
compare presence before comparing any present payload.

For `atomic<S>`, `delta<atomic<S>>` is the complete ordinary S. Applying it
replaces every field, including presence: an unset field in the new snapshot
does not retain an old payload. A complete struct with every optional field
unset is still a present value and publishes, including equal repetitions.
Only `_` denotes silence in a dense eval sequence.

Use the existing generic pass-through, replay, record, `TimedValue` and owning
retention contracts. Independently retain each present payload recursively at
every owning boundary; retain unset fields as unset. Later source mutation,
another publication, graph teardown or another eval cannot alter a capture.
This applies to optional structs nested in ordinary lists, tuples or map values
and to complete atomic children within structural publications.

Eval validates complete values before target startup. Missing required fields,
null supplied to a required field, wrong present types and unsupported shapes
are errors under the existing checking/construction rules. This admission adds
no optional-field read, mutation or clearing operation. In particular, it does
not define null as a structural delta, whole-output invalidation, or clearing
an omitted field in a sparse publication.

See [examples](../../examples/optional-atomic-publications.hgl) and
[compiler cases](../../../compiler/cases_optional_atomic_publications.md).
