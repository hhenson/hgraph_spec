# Temporal scalar publication cases

These check the [three added scalar leaves](../language/docs/design/temporal-scalar-publications.md)
through the unchanged generic delta pass-through and ordinary replay/record.

| Case | Required result |
|---|---|
| Each new scalar input `[a, a, _, b]` | Same four output slots; equal publications remain separate ticks. |
| Civil value with microseconds | Preserve exact civil fields without adding a zone or offset. |
| Zone values `US/Eastern` and `America/New_York` | Preserve both spellings as distinct ordinary values. |
| Zoned values with equal instants and different zones | Preserve instant, zone and offset; ordinary equality distinguishes them. |
| Retain into TimedValue, a list, global state and a final recording | Preserve independent values across later publications, teardown and another eval. |
| New leaf nested in an admitted structural or atomic payload | Use its ordinary scalar delta and equality; retain the enclosing shape's rules. |
| Empty/all-silent input with an exact typed wrapper | Preserve the ordinary recording lifecycle and dense horizon. |
| Eval input literal requires zone/offset validation | Make the required run context available during input materialization; evaluate each expression once in written order, before any target starts. |
| Input construction or strict decoding fails | Abort before target start; do not defer the failure until replay. |
| Replay an already constructed valid zoned scalar | Preserve its exact value without provider re-resolution. |
| Read delta without particular-input validity/modification proof | Reject under the existing guard rules. |
| Use zoned_time as an eval/publication leaf | Reject this unsupported profile shape; general source type syntax is unchanged. |
