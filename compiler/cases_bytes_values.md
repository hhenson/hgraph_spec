# Bytes value and publication cases

These check [BYTE-1–6](../language/docs/design/bytes-values.md).

| Case | Required result |
|---|---|
| `bytes()` and `bytes([])` | Present empty bytes with length zero; ordinary equality equates them. |
| Construct from fixed/unbounded ordinary i64 lists | Preserve octets and order, including 0 and 255. |
| `bytes(value)` in a guarded node with `atomic<list<i64>>` input | Evaluate the readable complete ordinary list once, then construct bytes in input order. |
| Runtime argument effects | Evaluate once before checking the list's octets; preserve preceding effects on failure. |
| Wrong count/type, named argument, integer-length or string argument | Checking failure; no implicit conversion or overload. |
| Executed octet -1 or 256 | `value.byte_range`; no result, truncation or wrap. |
| Invalid constant in required constant evaluation | Reject the constant; execution-error expectations do not catch source rejection. |
| Invalid constant argument inside an executed test/value-function/runtime body | Preserve execution failure; do not reject through optional constant folding. |
| Same contents; different length/content; proper prefix; 127 versus 128 | Exact equality and unsigned lexicographic order; equal contents have equal hashes. |
| `[empty, empty, _, a, a, b]` through generic delta pass-through | Preserve all six slots; present empty/equal values tick and silence stays absent. |
| Empty/all-silent sequences with exact wrapper | Preserve recording lifecycle and dense horizon. |
| `atomic<bytes>` and `delta<bytes>` | Same canonical type as bytes; preserve generic matching and specialization identity. |
| Bytes child in sparse structure or complete atomic payload | Preserve byte identity and the enclosing sparse/complete publication rules. |
| Bytes set members/map keys | Content equality/hash determines identity, duplicates and overlap under existing key checks. |
| Byte rolling arrivals, including empty and equal values | Retain each present arrival; delta is bytes, not a held window. |
| Capture through TimedValue, list/global state, publication and recording | Independently retain contents across source-list changes, replacement, another eval and teardown. |
| Byte indexing, mutation or text encoding/decoding | Outside this extension. |
