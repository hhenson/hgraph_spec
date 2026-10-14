# Nullable dictionary lookup cases

Required source contract under [LOOKUP-1–4](../language/docs/design/nullable-map-lookup.md).
The [ordinary/atomic examples](../language/examples/nullable-map-lookup.hgl)
and [rejection fixtures](../language/examples/reject/nullable-map-lookup.hgl)
are normative requirements, not claimed target execution.

| Case | Required result |
|---|---|
| Present zero, missing key, explicitly empty dictionary | Preserve present zero; absent safe reads produce null and require a check. |
| Guarded immutable local, early absent return | Continuing path admits the exact V payload. |
| Explicit fallback of exact V | Result is ordinary V; no presence check required. |
| Fallback of another type | Reject `lookup.default_type`. |
| Payload used before a presence check | Reject `nullable.required_presence`. |
| Guard one read, use a second read | Reject: presence belongs to the local binding. |
| Ordinary/atomic bracket or at read misses | Escaping execution failure retains `value.missing_key`; assert raises may match it. |
| Borrowed temporal child present but invalid | This extension supplies no safe-lookup policy. |
