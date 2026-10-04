# Contextual local binding cases

These specify [binding type and category stability](../language/docs/design/contextual-local-bindings.md).
Fixtures with required checking errors are intentionally invalid; they are not
positive language examples. Results below are requirements, not observations.

| Fixture | Required result |
|---|---|
| [`scalar_to_port`](contextual_bindings/scalar_to_port.hgl) | Reject during checking. |
| [`port_to_scalar`](contextual_bindings/port_to_scalar.hgl) | Reject during checking. |
| [`let_scalar_reassign`](contextual_bindings/let_scalar_reassign.hgl) | Reject during checking. |
| [`let_port_reassign`](contextual_bindings/let_port_reassign.hgl) | Reject during checking. |
| [`scalar_type_change`](contextual_bindings/scalar_type_change.hgl) | Reject during checking. |
| [`port_type_change`](contextual_bindings/port_type_change.hgl) | Reject during checking. |
| [`node_atomic_composite`](contextual_bindings/node_atomic_composite.hgl) | Reject during checking. |
| [`unused_category_change`](contextual_bindings/unused_category_change.hgl) | Reject during checking. |
| [`wire_rebind`](contextual_bindings/wire_rebind.hgl) | Accept; run its test. |
| [`node_scalar_local`](contextual_bindings/node_scalar_local.hgl) | Accept; run its test. |
| [`graph_scalar_local`](contextual_bindings/graph_scalar_local.hgl) | Accept; run its test. |

| [`conditional_initial_scalar_change`](contextual_bindings/conditional_initial_scalar_change.hgl) | Reject during checking. |
| [`conditional_initial_port_change`](contextual_bindings/conditional_initial_port_change.hgl) | Reject during checking. |
| [`conditional_uninitialized_result`](contextual_bindings/conditional_uninitialized_result.hgl) | Accept. |
| [`ordinary_widening`](contextual_bindings/ordinary_widening.hgl) | Accept. |

| [`graph_compound`](contextual_bindings/graph_compound.hgl) | Accept; compound result remains temporal. |

| [`uninitialized_scalar`](contextual_bindings/uninitialized_scalar.hgl) | Accept. |
| [`uninitialized_scalar_to_port`](contextual_bindings/uninitialized_scalar_to_port.hgl) | Reject during checking. |
| [`conditional_unused_scalar_write`](contextual_bindings/conditional_unused_scalar_write.hgl) | Reject during checking. |
| [`conditional_repeated_result`](contextual_bindings/conditional_repeated_result.hgl) | Accept. |

Also check annotated and inferred declarations, assignments in nested blocks,
ordinary widening into a fixed `f64`, and differing temporal composite shapes.
For uninitialized typed declarations, reject reads before definite assignment
and category disagreement across reaching branches. Temporal conditional
results remain connections; an existing scalar binding cannot become one.
Normal lifting at call/return boundaries remains permitted. Failures concern
binding/type validation, not malformed tokens or a new grammar production.
