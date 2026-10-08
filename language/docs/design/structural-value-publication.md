# Ordinary structural-value publication

Returning an admitted ordinary structural value T, or assigning it to the
node's own T output, reconciles that output with the retained value's structure
and child validity. This is distinct from applying `delta<T>`.

A tuple, fixed-length list or named struct keeps its declared positions or fields.
Write each present child value; an unset child in the retained observation
invalidates the corresponding output child, even if that child was previously
valid. An unset observed child is state, not a sparse omission or a new
source-level nil literal.

For a map, every invalid child in the source snapshot must have a key already
present in the destination. Retain exactly the snapshot's keys: remove output
keys absent from it and reconcile present children. An invalid child preserves
its existing key membership while invalidating that child. Apply these rules
recursively within the admitted shape. Output ownership is independent of the
source under existing retention rules; source endpoint timestamps are not copied.

This bounded clarification covers nonempty structures retaining a valid child;
growing lists and their length transitions are outside its scope.
It does not admit wholly invalid values, zero-child or empty structures, new
invalid map membership, or decide empty-event forwarding. Existing publication,
notification and child-invalidation rules determine observations.

Sparse `delta<T>` application is unchanged: omitted children preserve their
previous values and validity, and map keys disappear only through explicit
removal entries. Ordinary value copying does not convert a held value into a
delta. Constructor evaluation and ownership rules are unchanged.

See [independent state-observation examples](../../examples/structural-value-publication.hgl)
and [bounded conformance cases](../../../runtime/cases_structural_value_publication.md).
