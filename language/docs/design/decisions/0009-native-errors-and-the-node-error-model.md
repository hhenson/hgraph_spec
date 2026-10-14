# ADR 0009: Historical native failure contract

Status: superseded, 2026-10-14, by
[tagged results, canonical errors and explicit propagation](../result-errors.md).

The former native source exception clause and backend-specific exception
policy are retired. They are not accepted portable HGL syntax or declaration
matching rules. Recoverable native failures now appear in the complete
`result<T,error>` signature. Strict execution failures and explicit handler
propagation use the canonical failure channel; earlier completed writes stand.
The runtime's NOD-19/20/29 capture and lifecycle rules remain applicable.

Historical native examples are retained in the
[audit archive](https://github.com/hhenson/hgraph_spec_audit/blob/main/examples/documentation/language/docs/design/decisions/0009-native-errors-and-the-node-error-model.md#example-101).
Their backend mechanisms do not define the current portable source contract.
