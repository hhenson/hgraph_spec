# Scalar collection key and member cases

Check the [scalar-key extension](../language/docs/design/scalar-collection-keys.md)
with the same generic delta pass-through as existing collections.

| Case | Required result |
|---|---|
| Each admitted scalar K and declared enum as `set<K>` and `map<K, i64>` | Accept canonical nonempty add/upsert/remove traces with silence and retain exact K. |
| Keyed scalar child publishes the same value twice | Retain two publications at that key. |
| Reorder set members or map entries in the expected delta | Equal harness delta; iteration order is not identity. |
| Distinct enum types or integer substitute at an enum key | Reject exact-type mismatch. |
| Same wall clock in zoned-time aliases, or same instant in zoned-datetime aliases | Retain distinct keys according to ordinary complete scalar identity. |
| Zone-dependent literal key/member | Validate once with required context during cold materialization before target start; retain the result for replay. |
| Wrong-case or absent zone in a key/member | Fail materialization before target start under the existing strict name rules. |
| Duplicate or overlapping keys/members | Reject by exact K equality during checking if known, otherwise before target start. |
| `+0.0` and `-0.0` as duplicate map keys, set additions, or add/remove overlap | Reject as equal keys/members; hash must agree. |
| Positive and negative infinity supplied as already constructed f64 constants | Distinct supported keys/members; no finite-only restriction. |
| NaN key/member | Outside this bounded extension; no NaN equality/application behavior is asserted. |
| Composite or opaque native key/member | Outside this scalar extension, even if a runtime representation happens to be hashable. |
| Runtime-dependent key/member constructor expression | Reject under the retained constant-entry grammar. |
| Empty sparse patch | Apply [EMPTY-1–2](../language/docs/design/empty-delta-validity.md) without changing key identity. |
| Invalidation or invalid-child membership event | Retain the existing excluded boundary. |
| Immutable ordinary key alias initialized by a constant/cold recipe | Reuse its retained value once prepared; do not rerun its initializer. |
| Alias chain | Preserve exact K and the originally retained key value. |
| Temporal or mutable alias as sparse key | Reject; this admission remains constant-only. |
