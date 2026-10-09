# Ordinary tuple construction

`(first, second, ...)` constructs an ordinary `tuple<T0, T1, ...>` whose
positions are typed independently. Elements may be runtime expressions in
ordinary-value contexts, including node hooks and value functions:

```hgl
fn pair(value: i64) -> tuple<i64, bool> {
    when { return (value, value > 0) }
}
```

Check the complete tuple's types, phase and access before execution. Evaluate
each element once in written order, independently retaining its typed value
before evaluating the next. Retention is recursive under
[value mutability](value-mutability.md): it preserves the source and grants no
additional borrowing or mutation permissions. Later source changes cannot
alter retained children. Nested construction completes before retaining its
containing element. Assemble retained children in positional order without
reevaluating expressions.

Evaluation or retention failure stops later elements; assembly failure also
produces no result. No partial tuple escapes. Earlier completed effects remain.
Propagate failure through the existing value-operation contract: constant
evaluation fails checking, wiring-time evaluation fails construction, and
node execution follows its error contract.

`(value)` remains grouping, `(value,)` is a singleton, and `()` is rejected.
Constant contexts still require admissible constant elements; tuple syntax
cannot make a runtime payload or graph connection constant. It neither
constructs graph connections nor changes phase classification. Node payload
reads retain existing validity requirements.

Construction itself does not publish, schedule or invalidate. Existing output,
harness `_` and contextual delta rules remain. Nonempty ordinary list literals
retain their [constant restriction](ordinary-list-values.md#typed-empty-construction).
No order is specified for arbitrary calls.

See [examples](../../examples/ordinary-tuple-values.hgl) and
[construction cases](../../../runtime/cases_tuple_construction.md).
