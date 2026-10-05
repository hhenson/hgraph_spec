# Temporal scalar publication cases

These check the [four added scalar leaves](../language/docs/design/temporal-scalar-publications.md)
through the unchanged generic delta pass-through and ordinary replay/record.

For the provider-name cases, use a catalog containing `UTC`, `Etc/UTC`,
`America/New_York` and `US/Eastern`, and no entries named `utc`,
`america/new_york`, `Etc/Unknown` or `Missing/Zone`.

| Case | Required result |
|---|---|
| Each new scalar input `[a, a, _, b]` | Same four output slots; equal publications remain separate ticks. |
| Civil value with microseconds | Preserve exact civil fields without adding a zone or offset. |
| Zone values `US/Eastern` and `America/New_York` | Preserve both spellings as distinct ordinary values. |
| Zoned values with equal instants and different zones | Preserve instant, zone and offset; ordinary equality distinguishes them. |
| Retain into TimedValue, a list, global state and a final recording | Preserve independent values across later publications, teardown and another eval. |
| New leaf nested in an admitted structural or atomic payload | Use its ordinary scalar delta and equality; retain the enclosing shape's rules. |
| Empty/all-silent input with an exact typed wrapper | Preserve the ordinary recording lifecycle and dense horizon. |
| Exact catalog identifiers `@[UTC]`, `@[Etc/UTC]`, `@[America/New_York]`, `@[US/Eastern]` | Accept and preserve the exact name, including link spelling. |
| Wrong-case names `@[utc]`, `@[america/new_york]` | Reject during provider-backed literal validation; do not case-fold into catalog membership. |
| Absent names `@[Etc/Unknown]`, `@[Missing/Zone]` | Reject during provider-backed literal validation; no synthetic unknown-zone or UTC fallback. |
| Zoned literal with an absent or wrong-case name and otherwise valid date/time/offset | Reject the name even if the offset would agree with another catalog zone; syntactic validity is insufficient. |
| Eval input literal requires zone/offset validation | Make the required run context available during input materialization; evaluate each expression once in written order, before any target starts. |
| Input construction or strict decoding fails | Abort before target start; do not defer the failure until replay. |
| Replay an already constructed valid zoned scalar | Preserve its exact value without provider re-resolution. |
| Read delta without particular-input validity/modification proof | Reject under the existing guard rules. |
| Zoned-time input `[@09:30:00.123456[US/Eastern], @09:30:00.123456[US/Eastern], _, @09:30:00.123456[America/New_York]]` | Preserve all four slots, microseconds and both exact names; aliases compare unequal. |
| Zoned times differing only by one microsecond | Compare unequal and retain each exact wall-clock value. |
| Capture or replay a zoned time | Carry only wall-clock time and zone; do not invent a date, instant or offset, or resolve a date-dependent offset. |
| Zoned-time literal with an absent or wrong-case zone name | Reject during the existing provider-backed input materialization, before target start. |
| `zoned_time` in a structural delta or finite atomic snapshot | Admit under the existing recursive leaf rule; preserve sparse versus complete publication behavior. |
