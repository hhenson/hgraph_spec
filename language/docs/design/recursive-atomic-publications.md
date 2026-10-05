# Finite recursive atomic publications

Admit complete ordinary values of fully applied concrete recursive structs
into the atomic payload grammar. Declarations obey [ADR 0012](decisions/0012-recursive-struct-fields.md):
each recursive edge is an optional direct `atomic<T>` field with a null
default, and the graph of concrete type specializations is finite. Present
values are finite trees terminating at unset edges. Self recursion and
same-module mutual recursion use their existing declaration and import rules.

No new constructor or recursive value type is introduced. Ordinary struct
construction preserves exact nominal identity, field presence and each
present descendant. Equality and harness comparison traverse the finite value:
compare concrete types and field presence before comparing present payloads.
They do not compare object identity, discard deeper fields or expand a
recursive schema indefinitely. Type checking and shape matching retain nominal
recursive edges and terminate independently of the value's depth.

For `atomic<S>`, `delta<atomic<S>>` is the complete ordinary S. A publication
replaces the entire root and descendants. An unset recursive edge in the new
snapshot removes the previous descendant; it does not merge with the old tree.
A leaf with every recursive edge unset is present data. Equal trees at separate
times still publish, and `_` alone is silence.

Use the existing guarded generic pass-through, `TimedValue`, replay, record
and owning retention contracts. Each owning capture independently retains all
present descendants, including mutable ordinary children at any depth. Later
source mutation, another capture, graph teardown or another eval cannot alter
it. No borrowed recursive node or shared writable subtree escapes a capture.
Physical sharing is permitted only if these ordinary value observations remain
unchanged.

Eval fixes exact atomic shapes and validates complete finite values before
any target starts. The optional-field construction and error rules apply at
every depth. Recursive values may occur inside admitted finite ordinary
containers or as complete atomic children of structural publications; a
recursive edge through a container remains forbidden. This admission does not
add a recursive nominal root to the structural publication profile.

Required or non-atomic recursive edges, cyclic values, unbounded specialization
expansion and recursion through containers remain rejected under ADR 0012.
Abstract families, sparse optional-field clearing, invalidation and references
remain separate boundaries.

See [examples](../../examples/recursive-atomic-publications.hgl) and
[compiler cases](../../../compiler/cases_recursive_atomic_publications.md).
