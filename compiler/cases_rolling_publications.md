# Rolling publication cases

Check the [arrival delta and retained-window contract](../language/docs/design/rolling-publications.md).

| Case | Required result |
|---|---|
| `rolling<i64,2,2>` with `[10,_,20,30]` | Same arrivals; retained windows [10], [10,20], [20,30], readiness false/true/true. |
| `rolling<i64,2>` and `rolling<i64,2,2>` | Identical type; `valid` from first arrival and `all_valid` from second retained arrival. |
| `rolling<i64,5us>` and `rolling<i64,5us,5us>` | Identical type; `valid` from first arrival and `all_valid` when retained arrivals span 5us. |
| Duration Max=5us/Min=1us arrivals at t0, t0+2us, t0+8us | Same three arrivals; retained windows [10], [10,20], [30], readiness false/true/false. |
| Duration Min=0 | Ready and valid from first arrival. |
| Value exactly Max old on new arrival | Retain boundary value; evict only values older than Max. |
| Idle dense slots, even after duration Max elapses | No arrival, eviction or fabricated recording entry. |
| Equal repeated arrivals | Publish each arrival separately. |
| First valid/modified arrival below minimum | Permit guarded `delta_value`; do not require all-valid or substitute nil. |
| `delta<rolling<V,Max,Min>>`, retained in `TimedValue` or a list | Exactly complete ordinary V, independently owned; no window state or removed values. |
| Bare V used where a rolling shape is unresolved | Do not infer rolling sizes/kind; retain existing scalar inference and exact-shape requirements. |
| Same-shaped same-cycle pass-through | Reconstruct independent output window with the same arrival history. |
| Delayed publication of retained arrival | New publication time controls output eviction; do not preserve an earlier window timestamp implicitly. |
| Exact wrappers with empty/all-silent data | Preserve lifecycle and dense horizon with no arrivals. |
| Different kind, Max, Min or V | Unequal temporal shape; reject mismatching exact harness binding. |
| Ordinary finite composite V | Complete arrival value, with recursive ordinary copy/equality and no sparse child merge. |
| Rolling shape inside atomic payload, reference payload, invalidation or timer eviction | Remain outside this extension. |
