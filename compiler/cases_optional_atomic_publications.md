# Optional atomic publication cases

| Case | Required result |
|---|---|
| Complete optional field omitted with null default | Construct an unset field; retain no payload. |
| Explicit null for optional field | Construct the same unset state. |
| Explicit null or omission for required field without default | Reject before evaluating constructor arguments. |
| Present zero, empty string or empty ordinary list | Preserve as present; do not substitute unset. |
| Complete snapshot changes present field to unset | Replace its old payload; publish the new complete value. |
| All optional fields unset | Preserve a present complete struct; repeated values still tick. |
| Optional field has the wrong present type | Reject under ordinary exact field checking. |
| Optional struct nested inside finite ordinary or atomic data | Retain nominal identity, presence and independent owned children. |
| Source list changes after it was retained in a present optional field | Previously captured value stays unchanged. |
| Empty/all-silent input with exact atomic wrapper | Preserve lifecycle and dense horizon; create no present capture. |
| Null used as sparse structural field clearing | Remains outside this extension. |
