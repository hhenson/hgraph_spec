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

## Dependency candidates

These cases use a precompiled provider with `instantiate op<_>` and a body
whose local or storage type depends on retained `T`.

| Case | Expected result |
|---|---|
| A downstream target first binds `T` to an admitted type unknown when the provider was compiled. | The consuming compiler checks and lowers the specialization from the descriptor-referenced artifact; `hgl check` requires no execution of provider code. |
| Two downstream calls bind different admitted types. | The target manifest records both canonical bindings; their types and storage remain distinct. |
| The body refers to a provider-private helper or type with the same short name as a consumer declaration. | Resolve through the artifact's provider binding closure; do not capture consumer declarations or expose private declarations to imports. |
| The provider omits a required artifact or supplies an incompatible version. | Reject during checking, including `hgl check`; do not discard the candidate or select another overload. |
| A substituted body operation is invalid although its candidate signature matches. | Checking and native builds reject the same specialization before execution. |
| The descriptor, artifact or linked provider fingerprints disagree. | Reject before wiring. |
| A consumer tries to specialize outside the requested candidate pattern. | The artifact provides no additional candidate; the call does not match this candidate. |

For the ordinary-delta extension, `instantiate replay<_>, record<_>` follows
the same rules. A call binding `T` substitutes it into `delta<T>` and
`TimedValue<T>` wherever the selected body or storage uses them. Formation
still obeys the [ordinary-delta admission rules](../language/docs/design/ordinary-delta-types.md);
no new delta shape or reification capability is introduced.
