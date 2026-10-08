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
construction. Its `tuple_mark` helper injects the existing logger, emits its
argument with `info(logger, str(value))`, and returns the argument. It adds no
log-capture API. Run these named tests separately and capture their helper
messages externally:

| Executable test | Exact helper-message order | Asserted result |
| --- | --- | --- |
| `ordinary_tuple_written_order` | `"2"`, `"1"`, once each | `(2, 1)` |
| `ordinary_tuple_nested_written_order` | `"3"`, `"2"`, `"1"`, once each | `((3, 2), 1)` |

Source assertions check completed values; conformance also requires the stated
captured trace, including no duplicate helper messages.

Source checking must reject both isolated negative fixtures:

- [Dynamic const argument](../compiler/tuple_construction/reject-dynamic-const-argument.hgl):
  wrapping temporal connections does not supply a constant parameter.
- [Dynamic default](../compiler/tuple_construction/reject-dynamic-default.hgl):
  a constant parameter default cannot read a runtime payload.

The [fixture manifest](../compiler/tuple_construction/cases.json) lists both as
required source-check rejections. They remain outside passing-example discovery.
No diagnostic code is added for them. They do not use expected-error metadata
because their diagnostic conditions have no enumerated code in the catalogue.
