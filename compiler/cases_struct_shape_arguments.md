# Generic struct shape-argument cases

These check [generic struct shape arguments](../language/docs/design/generic-struct-shape-arguments.md).
Use Publication, Batch and Mixed from that contract.

| Case | Required result |
|---|---|
| `Publication<map<i64, i64>>` | Ordinary field has exact `delta<map<i64, i64>>` type. |
| `Batch<map<i64, i64>>` | Forwarded delta-formation requirement is satisfied. |
| T appears only under an admitted `delta<T>` | Accept T even if it is outside `value_type`. |
| That T is forwarded through Batch or a generic parent | Preserve the same requirement; no blanket value-type restriction. |
| The same T also initializes an ordinary T field, as in Mixed | Reject if T is outside `value_type`. |
| A forwarded parameter also has a `requires` constraint | Satisfy both the constraint and every occurrence requirement. |
| A concrete T has no admitted delta type | Reject before constructing a specialization. |
| Constructor leaves T unresolved or conflicting | Reject; no inference from runtime payload contents. |

The outside-value_type rows apply only when the delta publication profile
admits such a temporal shape; this rule adds no publication shapes itself.
Complete source arguments determine invariant nominal identity, even if two
admitted shapes derive the same ordinary payload type.
