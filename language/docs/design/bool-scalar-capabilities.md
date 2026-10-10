# Boolean scalar capabilities

**BOOL-1.** `bool` has equality, hash and total order. Equality compares its
Boolean value; `false` precedes `true`. The six comparison operators follow
that order, returning `bool`. Equality and order introduce no numeric
conversion. Equal Boolean values have equal hashes under VAL-7.

The same rule applies in ordinary evaluation and node value phases. It does
not change atomic/delta normalization, publication ticks or graph operator
activation. See the [source example](../../examples/bool-scalar-capabilities.hgl)
and [compiler cases](../../../compiler/cases_bool_scalar_capabilities.md).

This index collects the capability contracts needed by scalar consumers:

| Scalar | Equality | Hash | Order |
|---|---|---|---|
| `bool` | Boolean value, BOOL-1 | Yes, VAL-7 | Total, false before true, BOOL-1 |
| `timezone` | Exact zone name | Yes | None |
| `zoned_time` | Wall-clock time and exact zone name | Yes | None |
| `zoned_datetime` | Instant, exact zone name and resolved offset | Yes | None |

The three zone rows use the existing [temporal comparison rules](../developer-guide/syntax-and-semantics.md#arithmetic-and-comparison)
for equality and absent order, and [scalar key admission](scalar-collection-keys.md)
with VAL-7/10 for hash. Zone aliases remain distinct names. The index adds no
zone ordering or timeline equality and prescribes no hash representation.
