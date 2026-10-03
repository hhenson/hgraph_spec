# Retained candidate specialization

These [compiler conformance](conformance.md) cases apply the
[materialization rules](../language/docs/developer-guide/syntax-and-semantics.md).
They specify compile-time language behavior, not a runtime reification API.

| Case | Expected result |
|---|---|
| A candidate requested as `instantiate op<_>` uses retained `T` in a local or storage type; a call binds `T` to an admitted concrete type. | Substitute `T` throughout the body and storage and check that specialization before execution. Retention alone is not a rejection reason. |
| Two calls bind the same retained candidate to different admitted concrete types. | Both specializations preserve their own type identities and storage layouts; a binding cannot leak into the other call. |
| Matching leaves a slot unresolved that an instantiated body or storage needs. | Reject before graph execution; do not infer it from payload contents or a runtime schema. |
| Matching succeeds, but a substituted body operation or storage shape is unsupported. | Reject that specialization before graph execution. Matching alone does not establish body validity. |
| A body reads a retained `const` parameter as a numeric value, without an explicit reification contract. | Reject; compile-time type substitution does not grant body-visible generic values. |

For the ordinary-delta extension, `instantiate replay<_>, record<_>` follows
the same rules. A call binding `T` substitutes it into `delta<T>` and
`TimedValue<T>` wherever the selected body or storage uses them. Formation
still obeys the [ordinary-delta admission rules](../language/docs/design/ordinary-delta-types.md);
no new delta shape or reification capability is introduced.
