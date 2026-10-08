# Ordinary structural-value publication cases

Expected observations for the bounded
[complete-value publication rule](../language/docs/design/structural-value-publication.md).
Each case retains a valid sibling; no empty or wholly invalid structure is used.
Every invalid source map child has a key already present in the destination.
Positional lists have fixed length; growing-list length transitions are not covered.

| Case | Complete ordinary value | Explicit delta control |
| --- | --- | --- |
| Switch from complete A to independently partial B | After writing A `(1,true)`, copying B `(2,unset)` leaves first child 2 and invalidates the second; `all_valid` becomes false. | B's first-child-only delta updates that child and retains the output's valid true sibling. |
| Remove one of two map keys at the source | Copying the held map leaves exactly the remaining key. | An explicit removal entry removes the same key. |
| Invalidate an existing map child | Copying retains its key but invalidates its child; the other key stays valid. | Omitting the child from a sparse update preserves the old output child. No empty delta need be applied. |

An independent step drives copying and state observations. Observe the output
again after an idle source cycle; validity and membership changes must persist.
Do not infer child validity from a recorded sparse delta. The map membership
query and `all_valid` observation are separate checks.

For fixed tuple/list and named-struct cases, B must be a separate endpoint that
never held A's missing child. Sending a partial update to A itself would retain
its old child and would not exercise complete-value reconciliation.

The [HGL example](../language/examples/structural-value-publication.hgl) checks
named-struct selection and both map transitions. The corresponding positional
case uses `tuple<i64, bool>`, or `list<i64, 2>` with an integer second child,
and observes the same validity difference. Owning retention, constructor order and scalar/atomic
replacement rules are not changed by these cases.
