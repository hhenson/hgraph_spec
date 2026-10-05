# Complete atomic family publication cases

| Case | Required result |
|---|---|
| Abstract parent constructor | Reject; only a concrete descendant constructs a value. |
| Concrete descendant initializes admitted ancestor binding | Retain the concrete nominal tag and all fields. |
| Same-layout distinct descendants | Compare as distinct concrete values. |
| Alternate members through generic pass-through | Replace the complete value, retaining each member's fields. |
| Equal repeated members and silence | Preserve repeated ticks and dense silent cells. |
| Optional field unset versus present zero/empty | Retain presence independently of its payload. |
| Concrete generic specialization mismatches family specialization | Reject before graph execution. |
| Unrelated concrete type | Reject membership before any target starts. |
| Owning capture followed by source mutation | Preserve every previously retained child. |
| Replay then another eval | Retain independent concrete tags and complete values. |
| Empty or all-silent input with exact family wrapper | Keep the declared family; record no present values. |
| Structural family root or unresolved multiple-parent field order | Remain outside this admission. |
