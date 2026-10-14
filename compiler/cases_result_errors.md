# Result and propagation cases

Required source contract under [RESULT-1–3, ERROR-1/2 and PROP-1–3](../language/docs/design/result-errors.md).
The [source examples](../language/examples/result-errors.hgl) and
[rejection fixtures](../language/examples/reject/result-errors.hgl) are
normative; their presence claims no compiler execution.

| Case | Required result |
|---|---|
| Ok/Err with expected result<i64,error> | Infer both sides; retain only the selected payload. |
| No expected E for ok(1) | Reject result.context_type. |
| Guarded immutable result, including early failure return | Permit only the proven tag's property. |
| Unguarded property, wrong tag, mutable alias, or different guarded expression | Reject result.required_variant. |
| Value helper ? success/failure | Unwrap success; propagate unchanged E; stop remaining statements on failure. |
| Value helper declared i64 or another E | Reject result.propagation_type. |
| Node when ? with E other than error | Reject result.propagation_type. |
| Nullable operand or graph-phase ? | Reject propagation type or phase respectively. |
| Result output publishes Ok, Err, equal Err, silence | Three complete outcome ticks and one silent slot; Err does not fail the node. |
| Error without code/context | Both properties are null; message is readable. |
| Graph failure propagates through nested ownership | Preserve code/message and first failing-node provenance. |
| Equal diagnostics at different graph times | Equal error values, separate failure events. |
| Node when ? propagates catalogued error | Escaping failure preserves code for assert raises after cleanup. |
| Recoverable native function | Declare result<T,error>; no source exception clause. |
| Ordinary-T strict native failure or result helper's operational contract violation | Execution failure remains distinct from declared recoverable Err data. |
| Retired source exception clause | Reject under ordinary function grammar; see [rejection fixture](../language/examples/reject/retired-throws.hgl). |
