# Enum publications

Admit every declared enum E as a scalar publication leaf, including within the
existing structural and finite atomic profiles. `delta<E>` is E and
`atomic<E>` normalizes to E. The twelve built-in scalar leaves remain admitted.

Use the existing generic `pass_through`, guarded `delta_value`, `TimedValue<E>`,
replay and record contracts. Each present member publishes, including equal
repeats; `_` is silence. Owning captures preserve the enum member across later
publications, graph teardown and another eval. Exact wrappers determine E for
empty and all-silent input. Harness comparison retains E's nominal identity and
assigned member number; it does not compare an integer or a display string in
place of an enum value.

The existing [enum identity and conversion rules](type-extensions.md#enum-identity-and-explicit-conversion)
apply. Another enum with the same names or numbers, and an integer carrying
the assigned number, cannot replace E. There is no implicit conversion or
new enum construction form. Declared members, including the existing signed
`i64` endpoints, are admitted; handling unnamed values imported from a native
enum remains outside this extension. Existing construction failure phases
apply before publication.

This adds scalar leaves, not new collection key or set element types. Sparse
publication, finite atomic shape, provider, presence and guard rules otherwise
retain their existing meaning. References remain outside the publication
profile.

See [source examples](../../examples/enum-publications.hgl) and
[compiler cases](../../../compiler/cases_enum_publications.md).
