# Graph projections through references

These cases follow [wiring-time access](../language/docs/design/type-extensions.md#wiring-time-access),
WIR-5 and TS-14/16/17/25. The [source example](../language/examples/reference-wiring-projections.hgl)
supplies ordinary structural eval inputs and returns scalar observations;
it introduces no raw REF trace, REF literal or empty-reference constructor.

| Case | Required result |
|---|---|
| Graph body selects a declared field of `ref<Struct>` | Wire a reference-producing child projection; do not read its runtime payload while describing the graph. |
| Graph body selects a constant in-range index of `ref<list<T, N>>` | Wire the corresponding fixed-child projection with its declared type. |
| Ordinary T consumer binds to that projected `ref<T>` | Follow the selected child; preserve each child publication, including equal values. |
| Parent publishes only a sibling field/index | Do not publish the projected child's held value. |
| Parent is valid but projected child has not published | The ordinary consumer remains invalid and publishes nothing until that child is valid. |
| Parent reference changes between targets whose projected values are equal | Sample the newly designated valid child; endpoint identity is distinct even when payloads are equal. |
| Selected child publishes while the selector is silent | The ordinary consumer follows its publication without reevaluating the selector. |
| Selector republishes the same parent designation | No REF tick or additional sampling/publication. |
| Node `when` body accesses fields/indices below its REF input | Reject opaque-value traversal; graph projection admission does not change node access. |
| Unknown field or constant fixed index outside its declared range | Reject under existing field/index checking rules. |

Dynamic indices, unbounded-list projection, saved-reference storage and empty
REF construction retain their separate source contracts. This clarification
does not add them.
