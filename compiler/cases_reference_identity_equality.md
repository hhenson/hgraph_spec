# Node reference identity equality

These cases make the HGL spelling of existing node reference comparison
explicit under [REF-EQ-1](../language/docs/design/type-extensions.md#node-access-and-ticks),
[TS-16](../runtime/time_series.md), and reference opacity. The
[source example](../language/examples/reference-identity-equality.hgl) supplies
ordinary scalar inputs to eval; graph bindings obtain every designation.
This document records required results, not target validation.

| Case | Required result |
|---|---|
| Readable node operands of the same resolved `ref<T>` type use `==` | Return bool from endpoint/designation-tree identity; no read through either operand. |
| The same operands use `!=` | Negate their identity equality. |
| One endpoint reaches both operands through different forwarding routes | Equal, regardless of that endpoint's current scalar value or value ticks. |
| Separate endpoints carry equal scalar contents | Unequal; target-value equality is irrelevant. |
| Fixed-child or bundle designation trees differ | Compare their TS-16 trees; do not traverse payloads below REF. |
| Node comparison would require reading below REF | Reject such a payload read; REF-EQ-1 does not allow it. |
| Reference order, hash or heterogeneous equality | No new admission; retain existing contracts. |

Existing readability/validity guards remain required. This clarification adds
no reference literal, empty constructor, ordinary reference storage, target-value
comparison or graph-composition operator candidate. Invalid/expired reference
observations and child-graph lifetime retain their separate contracts.
