# Owned REF output routes

[GRF-27](../runtime/graph.md) admits an explicit REF result position that exports
a branch-owned endpoint through its owning node's output boundary, including
positions nested in structural outputs. This narrows
the same-graph/enclosing-graph restrictions in GRF-24 and
[TS-20](../runtime/time_series.md); it does not provide general cross-graph
access. [Source example](../language/examples/reference-owner-export.hgl).

| Case | Required behavior |
|---|---|
| Branch scalar producer, then REF relay, then parent ordinary consumer | Follow the actual child designation after its owner completes evaluation. No stable parent scalar proxy replaces it. |
| Two nested conditional results explicitly declared `ref<i64>` | Each owning output boundary exports the route; rank is checked using the relevant owner at each parent level. |
| Exported bundle/fixed-list child-designation tree | Preserve only the exported tree and existing TS-16 identity; no access to unrelated child endpoints. |
| Multiple escaping results pack a REF slot and an ordinary slot into a generated bundle | Export the REF slot's designation under GRF-27; preserve the ordinary slot's schema without making the bundle an outer REF. |
| REF-typed positions lie inside nested structural fields or collection elements | Apply export at each declared REF position, preserving its own endpoint/tree, owner and lifetime; ordinary siblings grant no additional routes. |
| Later nested consumer captures a parent-visible exported route | Existing GRF-10 input bindings retain the live route; evaluation still follows the exporting owner. |
| Attempt to bypass an owning boundary or route directly into a parallel graph | Reject the route; no new source error code is assigned here. |
| Route would evaluate its consumer before the effective owning producer | Reject the rank violation under GRF-15/TS-20. |
| Retain an exported designation after its actual owner stops | Export does not extend endpoint lifetime; GRF-19/23/25 and TS-23 still govern teardown. |

The source example follows current branch results only: switching selects a
new child designation and returning to a branch creates a fresh instance.
It introduces no raw REF eval inputs, ordinary REF state, empty constructor,
node read-through, or reference comparison spelling. The composed-boundary
and rejection rows require the listed contracts; this matrix claims no HGL
target execution.
