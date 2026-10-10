# Finite composite collection keys

Extend [collection keys](scalar-collection-keys.md) to finite positional tuples
and fully applied concrete structs whose components recursively support equality
and hashing. Leaves use the admitted scalar-key rules, including exact enum and
temporal identity and non-NaN floating-point values. Optional fields compare and
hash their presence as well as any present payload. No ordering capability is
required. Recursive keys, abstract families, native opaque values, collections
inside keys and references remain outside this admission.

Use ordinary complete values as keys and set members, not structural deltas.
Tuple position, concrete nominal identity, field presence and every component
participate in equality. Equal independently constructed values identify the
same key; equal keys have equal hashes. Distinct concrete structs with equal
layouts do not become interchangeable. Hash collisions never establish equality.

Sparse delta key/member expressions are constants of exact K. The existing
ordinary tuple and struct constructors may form those constants, recursively;
provider-dependent children retain their cold recipes. Evaluate and retain each
expression once in written order, and complete construction and duplicate/overlap
validation before any target starts. This adds no per-tick key construction or
new literal/constructor spelling.

Retaining a key owns its full ordinary value. Later mutation of the source
binding cannot change stored membership, lookup identity or a captured delta.
The usual set net-membership and map child-publication rules still apply;
removal followed by a later insertion preserves the exact key. Harness
comparison ignores entry order while comparing full keys and child deltas.
The same K is admitted in ordinary atomic set/map snapshots. This extension
does not change [empty sparse application](empty-delta-validity.md),
invalidation or implicit-conversion rules.

See [examples](../../examples/composite-collection-keys.hgl) and
[compiler cases](../../../compiler/cases_composite_collection_keys.md).
