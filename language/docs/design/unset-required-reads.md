# Required reads of unset ordinary observations

This extension admits initializing an owning `let` from a child of a retained
ordinary structural observation, preserving its exact type and unset state.
For example, `let child = retained[i]` retains that observation without
requiring its payload; the same applies to a field projection. Whole partial
values remain independently retainable.

An already admitted operation requiring that payload fails with
`value.unset_read` when it is unset. Scalar arithmetic and Boolean conditions
require payloads, as does `len` on an unset list even with known fixed size.
Present zero, false and empty values keep their ordinary behavior. Failure
produces no result; [execution-error rules](execution-error-assertions.md)
govern order, propagation and cleanup.

This adds no collection operation, nullable type, nil literal or publication
rule. Direct temporal-input validity, bounds and missing-key failures remain
distinct. Retention itself does not consume an unset payload.

See [negative tests and present controls](../../examples/unset-required-reads.hgl).
