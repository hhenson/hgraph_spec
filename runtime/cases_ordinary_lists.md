# Ordinary list value cases

These are expected source and value observations for
[ordinary list values](../language/docs/design/ordinary-list-values.md), under
VAL-1, VAL-3, VAL-16 and VAL-17. They do not describe temporal list outputs or
nullable replay sequences.

## LIST-EMPTY — type context and fixedness

| Source | Expected result |
|---|---|
| `var xs: list<i64> = []` | Writable unbounded list, length zero. |
| `let xs: list<i64> = []` | Read-only unbounded list, length zero. |
| `let xs: list<i64, 0> = []` | Read-only fixed list, length zero. |
| `var xs = []` | Checking error: element type not supplied. |
| `var xs: list<i64, 2> = []` | Checking error: fixed size mismatch. |
| `push(xs, 1.0)` after the first declaration | Checking error: element type mismatch. |

No absent slot or default element is created by empty construction.

## LIST-READ — length and bounds

Start with a writable `list<i64>` constructed empty. After `push(xs, 10)`
and `push(xs, 20)`, `len(xs)` is 2, `xs[0]` is 10 and `xs[1]` is 20.
Repeated reads leave all three results unchanged. Index -1 or 2 fails bounds;
an `f64` index fails checking. Indexing an empty list at zero fails bounds.
No failure yields `null` or zero as a fallback.

The same length and index rules apply to an already admitted fixed ordinary
list. Its length never exposes the unbounded sentinel. An attempted growth
past the largest nonnegative i64 length fails without changing the list.

## LIST-GROW — order and authority

Continue with xs containing 10, 20. `push(xs, xs[0])` succeeds and leaves
10, 20, 10, with length 3. Each push has no result value.

Push through `let`, an ordinary parameter, const configuration, an input or a
read-only projection fails checking. Push on `list<i64, 0>` or any positive
fixed-size list fails checking even if the owning binding is `var`.

With a preinitialized global entry of exact type `list<i64>`, a typed
`var xs: list<i64> = get(global_state, key)` borrows it. Pushing 30 updates
that entry, and a later nonoverlapping borrow sees the increased length and
last element 30. The same get bound by `let` rejects push. A second get or
separate set while the writable borrow is live remains a borrow conflict.
The primitive length may be retained without borrowing an aggregate child.
For the same writable borrow, `let first: i64 = xs[0]` makes an ordinary
owned value; a subsequent `push(xs, first)` is admitted. First keeps its
original integer after the push, and the final element equals that integer.

## LIST-RETAIN — independent children

Construct a writable Sample whose amount is 4 and push it into an empty
`list<Sample>`. Changing the source amount to 9 leaves `xs[0].amount` at 4.
The source remains usable. For a `list<list<i64>>`, append a child containing
1, then push 2 into the original child. The retained child still contains
only 1. These cases require recursive independence, not just copying a parent
handle.

Construct an empty `list<list<i64>>` named outer and push a child containing
1. Then `push(outer, outer[0])` retains a second independent child before
extending outer. `push(outer[0], 2)` is admitted through the first child's
writable projection. Outer now contains two children: the first contains
1, 2, and the second still contains only 1. This is content mutation through
an indexed projection, not replacement of an indexed element. It asserts the
same recursive independence when the append source belongs to its receiver.

An owning `var independent = source` made from a read-only owning list may
grow independently; source keeps its original length and contents. An owning
local made from an owning list's Sample element may change its amount without
changing the element. A corresponding projection from a writable global-entry
borrow cannot create a second live aggregate local alias. Push may consume
that direct projection as a retaining argument without allowing the borrow
to escape.

## LIST-ERROR — operation and earlier effects

After one successful push, a subsequent bounds failure leaves that earlier
append observable under the enclosing error contract. A failed precondition
does not partly extend the list. No whole-hook rollback is implied, and
if retaining the appended item or obtaining capacity fails, the list keeps
its previous length and recursively equal element values. That failure
propagates through the applicable error contract.
