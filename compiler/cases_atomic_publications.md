# Atomic publication cases

These check the [finite atomic profile](../language/docs/design/atomic-delta-publications.md)
using the existing generic delta pass-through and replay/record operators.

| Case | Required result |
|---|---|
| An admitted scalar S is spelled `atomic<S>` | Use the same canonical scalar shape, delta and TimedValue specialization as S. |
| Atomic tuple `(1, "a")`, then `(2, "b")` | Two complete tuple publications, no sparse child merge. |
| Atomic list `[1, 2]`, `[]`, `_`, `[]` | Present full list, present empty list, silence, present empty list. |
| Atomic nested list, then changed inner list | Complete nested replacement with independent retention. |
| Atomic nominal constructor omits a defaulted field | Use its ordinary default, not the previous snapshot's field. |
| Atomic nominal constructor omits a required field | Reject; earlier publications cannot complete this value. |
| Sparse map adds an atomic empty-list child | Nonempty map update creates a valid child containing `[]`. |
| Mutate a source after recording it twice | Both captures preserve their original complete values. |
| Mutate one independently owned returned capture | Other captures and original source remain unchanged. |
| Delta read without validity/modification proof | Reject under the existing guard rules. |
| Composite payload alone is offered to infer T in `delta<T>` | Reject unresolved T; do not guess an atomic boundary. |
| Explicit `TimedValue<atomic<list<i64>>>` contains `[]` | Accept; its exact type fixes replay's temporal shape. |
| Empty/all-silent eval with an exact atomic wrapper | Preserve its dense horizon and ordinary empty recording lifecycle. |
| Optional/recursive/native or other excluded atomic payload | Reject the unsupported concrete shape. |

These are source-contract expectations. Shape inference and source admission
follow the specified checking rules.
