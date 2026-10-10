# Boolean scalar capability cases

These cases follow [BOOL-1](../language/docs/design/bool-scalar-capabilities.md)
and VAL-7/10. The [source example](../language/examples/bool-scalar-capabilities.hgl)
uses direct bool values; boxed dispatch is covered in the separate Any corpus.

| Case | Required result |
|---|---|
| Same Boolean value under `==`/`!=` | Equal/not unequal. |
| `false` versus `true` | Unequal; false precedes true. |
| `<`, `<=`, `>`, `>=` for all four Boolean pairs | Total order with ordinary reflexive non-strict relations. |
| Ordinary executed expression and readable node operands | Same value comparison results. |
| Reconstructed Boolean set member or map key | Equality/hash identify the same member or key. |
| Boolean versus numeric value | BOOL-1 introduces no conversion or heterogeneous direct comparison. |
| Zone scalars under equality/hash/order | Preserve the linked existing zone contracts; no new zone semantics. |

No hash bit pattern, numeric representation or additional Boolean arithmetic
is specified. These are required source results, not HGL backend validation.
