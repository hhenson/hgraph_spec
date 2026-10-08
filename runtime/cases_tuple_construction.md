# Ordinary tuple construction cases

These expected observations follow the
[ordinary tuple contract](../language/docs/design/ordinary-tuple-values.md).

| Case | Required observation |
| --- | --- |
| Runtime values | On admitted values 7, -1 and 0, `(value, value > 0)` produces `(7, true)`, `(-1, false)` and `(0, false)`. |
| Evaluation order | `(mark(2), mark(1))`, where mark logs and returns its argument, logs 2 then 1 once each and yields `(2, 1)`. |
| Recursive retention | After `let pair = (values, values)`, later permitted mutation of the original owning list changes neither child. |
| Early failure | If the first element logs then fails, the log remains, the second element never executes, and no completed tuple is produced. |
| Retention failure | A failure retaining one child prevents later expressions and yields no completed tuple. |
| Nesting | An inner tuple finishes before the next outer element begins. |
| Arity | `(7)` is i64, `(7,)` is `tuple<i64>`, and `()` is rejected. |

The [positive example](../language/examples/ordinary-tuple-values.hgl) includes
ordinary value-function assertions, independent aggregate retention and runtime
construction. Source checking must reject both isolated negative fixtures:

- [Dynamic const argument](../compiler/tuple_construction/reject-dynamic-const-argument.hgl):
  wrapping temporal connections does not supply a constant parameter.
- [Dynamic default](../compiler/tuple_construction/reject-dynamic-default.hgl):
  a constant parameter default cannot read a runtime payload.

These are ordinary source-check failures, outside passing-example discovery.
No diagnostic code is added for them. They do not use expected-error metadata
because their diagnostic conditions have no enumerated code in the catalogue.
