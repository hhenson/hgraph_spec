# Atomic scalar equivalence

For every non-composite value type S, `atomic<S>` and S are the same canonical
type. This includes `bool`, `i64`, `f64`, `str`, all temporal scalar types and
enums. Scalar identity is preserved: `atomic<Mode>` is Mode, not `i64`.

Where the grammar admits `atomic<S>`, normalize it to S before type equality,
generic occurrence checks, inference, overload matching and specialization
identity. Apply normalization recursively in type arguments and structural
children. Thus `list<atomic<i64>, 2>` is `list<i64, 2>`, and
`TimedValue<atomic<i64>>` is `TimedValue<i64>`. Equivalent spellings do not
create distinct overloads or require conversions. When `delta<S>` is admitted,
`delta<atomic<S>>` is the same delta type and scalar matching binds T to S.

This does not extend the grammar: `atomic` still marks a temporal boundary and
is not admitted in a `value_type`-only position such as a `const` parameter
annotation. Normalization does not admit otherwise unsupported types or
publications.

Lists, tuples, sets, maps and nominal structs are composite, including empty
and single-field structs. Their atomic boundary remains significant:
`atomic<list<i64>>` carries one complete list while `list<i64>` has temporal
children. Internal storage representation does not turn a scalar such as
`str` into a composite type.

[Compiler cases](../../../compiler/cases_atomic_scalar_equivalence.md) cover
canonical identity and the composite boundary.
