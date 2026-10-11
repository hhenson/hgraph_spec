# Any value and publication cases

These check [ANY-1–5](../language/docs/design/any-values.md).

| Case | Required result |
|---|---|
| `any()`, `any(value)`, `any(any(value))` | Present empty or independently retained typed contents; no nested box layer. |
| Wrong count, named arguments, boxing an endpoint/view handle/opaque resource | Checking failure. No implicit boxing conversion. |
| Same content/type; i64 0 versus bool false | Equal boxes equate; different contained types remain unequal and unordered. |
| Empty/populated comparison; different-type relational operators; empty keys | Empty precedes populated; unordered relations are false; empty boxes equate/hash and are keys. |
| Contained type lacks equality, hash or order | Executed operation fails with `value.capability`; required constant evaluation rejects. |
| Mutable list/struct/recursive/family payload changes after capture | Retained contents, optional presence and concrete identity remain independent. |
| Empty/equal/mixed payloads and silent positions through pass-through | Complete boxes preserve each tick and the dense horizon. |
| `atomic<any>`, `delta<any>`, `TimedValue<any>` | One canonical leaf identity and complete box delta. |
| Box in sparse fixed/growing list, tuple, struct or map child | Outer structural delta stays sparse; each box replaces its whole contents. |
| Boxed structural delta | Preserve originating type and sparse entries as owning ordinary data; no implicit application. |
| Complete compound payload and rolling arrivals | Independently retain complete boxes, including empty and repeated arrivals. |
| Typed timed replay and record across fresh runs and source mutation | Exact box contents and times; independent storage and lifecycle. |
| Lexical global-state any borrows and independent boxing | Let/var follow aggregate borrow conflicts; replacement keeps entry type any and copied boxes do not alias. |
| Boxed keys with same/different contained types | Exact content/type identity; missing key capabilities fail before graph start; boxed NaN remains outside key admission. |
