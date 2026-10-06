# Atomic set and map publication cases

Check [ordinary construction and complete publication](../language/docs/design/atomic-set-map-publications.md).

| Case | Required result |
|---|---|
| Exact `set<K>(items: [...])` or `map<K,V>(items: [key: value,...])` | Construct the complete ordinary type; exact K/V do not depend on runtime contents. |
| Runtime ordinary member/key/value expressions | Accept in their existing phase; the sparse-delta constant restriction does not apply. |
| Map key, value, key, value with observable expression order | Evaluate and independently retain in written order, once each. |
| Duplicate member/key including +0/-0 | Reject by K equality; if discovered during execution, stop before later expressions and before a duplicate key's value expression. |
| Missing `items`, unknown argument or untyped constructor | Reject; this is one exact typed construction form. |
| Atomic set/map input `[full, empty, _, empty]` | Same four slots; empty snapshots publish and equal empties remain distinct ticks. |
| `map<str,list<i64>>(items: ["a": [1, 2], "b": [3]])` followed by complete `map<str,list<i64>>(items: ["a": [4]])` | Second held state has only "a", whose complete child is [4]; do not retain "b" or merge "a"'s list. |
| Expected entries in different order | Equal ordinary value and harness result. |
| Independently retain map with list/struct/set/map children | Later source/capture mutation or another eval cannot alter retained siblings. |
| Empty set/map as an atomic child in a nonempty structural publication | Present complete child data, not an empty sparse patch. |
| Bare composite payload used to infer a temporal shape without exact context | Reject under the existing atomic inference boundary. |
| Optional/recursive fields, composite keys, NaN keys, invalidation or references | Retain their separate unsupported profile boundaries. |
