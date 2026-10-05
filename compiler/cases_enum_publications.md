# Enum publication cases

These check [enum leaf admission](../language/docs/design/enum-publications.md)
through the unchanged generic delta pass-through and ordinary replay/record.

| Case | Required result |
|---|---|
| E input `[E::a, E::a, _, E::b]` | Same four output slots; repeated members remain publications. |
| Members assigned negative values and either signed `i64` endpoint | Preserve exact E and assigned number, with no narrowing or ordinal conversion. |
| Exact E wrapper with empty/all-silent input | Preserve recording lifecycle and dense horizon; do not infer E from an absent member. |
| Store in `TimedValue<E>`, a list, global state or a recording | Retain E's member independently of subsequent publications and another eval. |
| E nested in an admitted structural delta or finite atomic snapshot | Preserve E's nominal identity and the enclosing sparse or complete publication rules. |
| `atomic<E>` compared with E | Equal canonical type under scalar atomic equivalence. |
| Supply an integer or another enum F to an E input, including identical names/numbers | Reject exact-type mismatch; do not convert or erase enum identity. |
| `delta_value` without exact endpoint validity/modification proof | Reject under the existing scalar guard rules. |
| Unknown number/name supplied through E's checked constructor | Retain existing phase-specific conversion failure; do not create an unnamed member or silence. |
| Use E as a set element or map key under the current collection profile | Reject this unsupported profile shape; this extension admits E as a publication leaf only. |
