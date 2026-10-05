# Complete atomic family publications

Admit an abstract struct family as the declared type of a complete atomic
publication. The existing inheritance and import rules determine its concrete
members at wiring completion. This admission covers finite concrete values
under the existing atomic payload grammar; it adds no inheritance syntax,
implicit field projection, cast, type test or source-level dynamic dispatch.

`delta<atomic<A>>` carries a complete ordinary value of an admitted concrete
descendant of A. Retain its exact concrete nominal type and all its fields,
including fields absent from A. Equal-layout descendants remain distinct.
A concrete value may initialize an ordinary binding of its admitted ancestor
family type without losing the concrete type. Unrelated values and incomplete
or unapplied specializations fail ordinary type checking. Abstract parents
remain nonconstructible.

Publication replaces the complete member and its contents, including a change
to a different concrete member. Equal values at different times still tick;
`_` alone is silence. Optional fields retain their presence independently of
zero or empty values. Equality and harness comparison check the concrete tag
before comparing the member's fields. Complete owning copies preserve all
present descendants and isolate mutable children under the existing retention
rules.

Eval fixes the declared family and checks each concrete input's membership
before any target starts. Generic pass-through, replay, record and
`TimedValue<atomic<A>>` preserve that family shape and the concrete value.
Empty and all-silent inputs still use the explicit family-shaped wrapper.
No runtime discovery of family members is introduced.

This extension does not admit structural family roots or settle multiple-parent
field ordering. Recursive family edges remain subject to ADR 0012; this finite
nonrecursive family slice does not extend the recursive publication profile.
References remain separate.

See [examples](../../examples/abstract-atomic-publications.hgl) and
[compiler cases](../../../compiler/cases_abstract_atomic_publications.md).
