# Growing-list publication cases

Check the [net growing-list delta contract](../language/docs/design/growing-list-publications.md).

| Case | Required result |
|---|---|
| Append 0/1, update 1 and append 2, silence, then remove 1/2 and update 0 | Own-output pass-through preserves exactly those sparse deltas and final length 1. |
| Repeat an equal scalar child update | Preserve another publication. |
| Remove every live tail index, then append at 0 | Publish nonempty removal, hold a valid empty list, and regrow from zero. |
| Removed indices listed in another order | Same delta identity; truncate once to the lowest removed index. |
| Append structural or atomic child publication | Apply its exact recursive child contract; atomic empty data is present. |
| Capture delta then grow/shrink again | Retained removed indices and child deltas remain independent. |
| Gap while appending, non-tail/out-of-range removal or removed/modified overlap | Reject the supplied trace before target start with the existing input-profile diagnostic and slot. |
| Duplicate/negative/nonconstant index or unknown constructor argument | Reject during checking. |
| Empty ordinary delta used as present input | Outside the nonempty publication profile. |
| Empty/all-silent sequence through exact wrapper | Normal lifecycle and dense horizon; no fabricated child publication. |
| `list<S>` versus `atomic<list<S>>` or fixed `list<S,N>` | Preserve distinct temporal shape identities. |
| Invalid-child growth, invalidation or references | Remain outside this bounded publication profile. |
